# Code-Hosting Channel Adapters — Design

## Overview

The GitHub polling adapter lets AI agents monitor GitHub for tasks by polling the notifications API and posting agent responses as issue/PR comments. Unlike IM adapters (real-time webhooks/long-poll), this adapter polls on an interval.

## Architecture: Notification as Wake-up Signal

The core insight: platform notifications are **thread-level** and **mutable** — any activity (comment, push, label change) bumps `updated_at`. Notifications cannot be used as a reliable per-comment event stream.

Instead, notifications serve only as a **wake-up signal** ("something happened on this thread"). The adapter then enumerates actual comments via the platform's comments API, using a per-thread watermark to determine which comments are new.

## GitHub: Cursor-Based Comment Window

GitHub provides a server-side per-thread read marker (`last_read_at`), but it cannot be relied upon for dedup:

- `PUT /notifications` (global mark) returns **202 Accepted** and runs asynchronously
- The mark uses `last_read_at` as a cutoff: notifications with `updated_at > last_read_at` are **not** marked
- The bot's reply bumps the notification's `updated_at` past the cutoff before the async process runs → the mark never takes effect
- The next poll re-fetches the still-unread notification → duplicate processing

Therefore, correctness comes from a **cursor-based comment window**, not from the notification's read status:

Poll cycle:

1. `GET /notifications?since={cursor-1s}` — discover unread threads
2. Save `windowSince = cursor.lastProcessedAt` (the cursor **before** this poll advances it)
3. `markNotificationsAsRead(maxUpdatedAt)` — best-effort global mark (cleans up non-issue notifications)
4. Advance global cursor to `max(updated_at)`
5. Per thread: `listComments(since=windowSince)` — enumerate comments
6. Filter: bot's own comments, `updated_at > maxUpdatedAt` (upper bound), `updated_at <= windowSince` (lower bound, exclusive)
7. Process: mention detection → envelope → `handleInbound`

The effective comment window is `(windowSince, maxUpdatedAt]`. Comments processed in a previous poll have `updated_at <= windowSince` (the previous poll's `maxUpdatedAt`) and are excluded. This prevents duplicates regardless of whether `PUT /notifications` succeeded.

The global mark is still called (step 3) to clean up non-issue/PR notifications and reduce the unread list, but it is not load-bearing for dedup.

### Scenario Behavior

| Scenario                          | Behavior                                                                                                                      |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| New thread (@bot in comment)      | Appears (unread) → enumerate since cursor → process                                                                           |
| Existing thread, new comment      | Reappears (unread) → enumerate since cursor → old comments excluded by `<= windowSince` → only new                            |
| Non-comment activity (push/label) | Appears → zero new comments in window → skip                                                                                  |
| User marks read on github.com     | Disappears from API → not processed                                                                                           |
| markNotificationsAsRead fails     | Cursor window still prevents duplicates → no impact on correctness                                                            |
| Crash after markRead, before done | Comments lost (best-effort) → user re-mentions to retry                                                                       |
| Bot replies to a thread           | `updated_at` bumped → notification may stay unread → next poll fetches it → comments excluded by cursor window → no duplicate |
| New issue with @bot in body       | No comments → body contains mention → feed body as trigger (deduped via `dispatchedBodies`)                                   |

## PollingChannelBase

`PollingChannelBase<Cursor>` (in `packages/channels/base/`) extends `ChannelBase` and provides the poll loop infrastructure:

- **Poll loop**: start/stop via `startPollLoop()`/`stopPollLoop()`, called from `connect()`/`disconnect()`
- **Poll interval**: read from channel config `pollInterval` (ms), defaults to 60000
- **Cursor persistence**: JSON cursor saved atomically after each successful `pollOnce()`; loaded on construction (corrupt → fallback to `createInitialCursor()`)
- **Backoff**: exponential 2s → 30s on poll errors, reset on success

Subclasses implement only:

- `pollOnce()` — do the work, mutate `this.cursor`
- `createInitialCursor()` — first-run default value

The `Cursor` generic is any JSON-serializable object. GitHub uses `{ lastProcessedAt: string }`.

## Mention Detection

Body-based, case-insensitive regex. Separate functions for detection (`testBotMention`) and stripping (`stripBotMention`):

- Detection: explicit regex match returning boolean — never inferred from strip-before/after comparison (whitespace differences cause false positives)
- Stripping: removes only `@bot`, preserves all other formatting (no whitespace collapsing)

## Session Scope

Polling adapters use `chat_thread` scope: routing key = `channel:chatId:threadId`. This prevents cross-repo session collision (`repo-a/issue:42` vs `repo-b/issue:42`).

## Error Handling

Delivery is **best-effort**. On `handleInbound` failure, an error comment is posted on the thread; the user re-mentions to retry. If the process crashes mid-processing, the cursor is not saved (it is persisted only after `pollOnce()` completes), so the next startup re-fetches the same notifications — but the cursor-based comment window excludes already-processed comments, preventing duplicates.

Duplicate prevention does **not** depend on `PUT /notifications` succeeding. The global mark is best-effort cleanup; the cursor window is the load-bearing dedup mechanism.
