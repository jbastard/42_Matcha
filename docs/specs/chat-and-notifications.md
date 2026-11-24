# Chat and notifications

Real-time chat between connected users, notifications, unread indicators, online status and last seen (R-CHAT-01, R-CHAT-02, R-NOTI-01, R-NOTI-02, R-VIEW-05, R-VIEW-07, R-VIEW-09). Transport and process model follow ADR 0005; WebSocket authentication follows ADR 0004 and the authentication spec.

## Delivery model

- Every write goes through the JSON API. The WebSocket at `/ws` only pushes events from server to client.
- A message or notification is committed to the database, then pushed to every open socket of its recipient. The push is a signal; the database is the source of truth.
- Delay is bounded by one round trip after commit, well under the 10 s of R-CHAT-01 and R-NOTI-01. A recipient with no open socket sees everything on their next page load.
- One Node process holds the registry of open sockets in memory (user id to sockets). Each browser tab opens its own socket.

## WebSocket

- Upgrade authenticated with the session cookie and a strict `Origin` check; refused otherwise. Logout closes the sockets opened by that session.
- The server pings every 30 s and terminates a socket that missed the previous pong.
- The client reconnects with randomized exponential backoff (1 s doubling to 30 s). On every (re)connection it invalidates its chat and notification queries, which resynchronises over HTTP.
- Incoming client frames other than pongs are ignored.

Events, typed in the `shared/` package:

| Event | Sent to | Payload |
|---|---|---|
| `message` | recipient and the sender's other tabs | the full message |
| `notification` | recipient | the full notification, as listed by the API |
| `counts` | the affected user | unread conversations count, unread notifications count |
| `conversation-closed` | both users | conversation id |

`counts` is pushed after any change to either count: a new message or notification, a read in any tab, a block or an uncrush that hides unread items. The header badges are driven by it and by the initial fetch.

## Conversations

- One conversation per pair of users, created when they first connect. It is visible to both only while they are connected; otherwise it is absent from the conversation list and the profile, kept in the database, and reappears with its history if they connect again.
- Breaking the connection (uncrush, block, fake report) pushes `conversation-closed` to both: an open chat view disables its input and the conversation leaves the list.
- The conversation list is ordered by last message and shows the other user's profile picture, username, online state, a preview of the last message and whether it holds unread messages.

## Messages

- Plain text, 1 to 1000 characters after trim, validated on the server (R-SEC-05) and always rendered as text, never as HTML (R-SEC-02).
- `POST /api/conversations/:id/messages` checks, in the same transaction as the insert, that the two users are still connected; otherwise 403.
- `GET /api/conversations/:id/messages?before=<id>` returns 50 messages per page, cursor on the `uuidv7` id.
- Sending is rate limited to 30 messages per minute per user.

### Read state

- Each message has a `read_at`, set for the recipient only.
- Messages are marked read when the recipient opens the conversation, or receives them while the conversation is open and the tab visible: `POST /api/conversations/:id/read`.
- Reading a conversation also marks read the unread `message` notification from that sender.
- No read receipt is shown to the sender.

## Notifications

### Types

| Type | Recipient | Trigger |
|---|---|---|
| `crush` | the crushed user | a crush that does not form a connection |
| `connection` | the user who crushed first | a crush back that forms a connection; the user who crushed back gets nothing |
| `visit` | the visited user | a visit by another user |
| `message` | the recipient | a new message |
| `uncrush` | the uncrushed user | an uncrush that breaks a connection; an uncrush of a non-connected user notifies no one |

### Creation rule

A notification from an actor to a recipient is created only when all hold:

1. the actor is not the recipient;
2. there is no block or fake report between them, in either direction;
3. the recipient has not uncrushed the actor since last crushing them (R-VIEW-05): an uncrush is stored and lifted when the withdrawer sends a new crush;
4. the recipient has no unread notification of the same type from the same actor.

Rules 2 and 3 suppress notifications only. The underlying fact (visit, crush, message) is still recorded and still counts toward the popularity score.

### Listing and reading

- `GET /api/notifications?before=<id>` returns 20 per page, newest first, cursor on the id.
- Notifications whose actor is blocked or fake-reported, in either direction, are filtered out when listing and counting; they are not deleted.
- Opening the notification panel marks read everything up to the newest notification shown: `POST /api/notifications/read` with that id.
- Each notification shows the actor's profile picture and username, a link to their profile and a relative date. A `message` notification carries no preview of the text.
- On a `notification` event the client shows a toast, except for a `message` whose conversation is open.
- Notifications are kept indefinitely; there is no purge.

## Unread indicators

Visible in the header of every page (R-CHAT-02, R-NOTI-02):

- chat badge: number of visible conversations holding at least one unread message;
- notification badge: number of unread notifications after filtering.

## Online and last seen

- A user is online while at least one of their sockets is open (in-memory registry).
- `last_seen_at` is written when a socket opens and when the user's last socket closes. After a crash it may lag; this is accepted.
- A profile or conversation shows "online" or the date and time of last seen (R-VIEW-07). The value is read with the profile or conversation and refreshed by refetch on window focus and every 30 s; presence is not pushed.

## Dependencies

- `ws` for the WebSocket server (research in issue 6, admissible under ADR 0001).
- `express-rate-limit`, already used by authentication.
