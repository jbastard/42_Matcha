# Push-only WebSocket from a single process, writes over HTTP

Every write that drives chat or notifications (sending a message, marking messages or notifications read) goes through the JSON API. The WebSocket, served by `ws` on the app's HTTP server, only carries events from server to client: `message`, `notification`, `counts` and `conversation-closed`. Messages and notifications are committed to PostgreSQL before they are pushed, so the push is a signal and the database the source of truth; a client that reconnects resynchronises over HTTP.

The registry of open sockets and the online state live in memory in one Node process. Running a second app instance would require a pub/sub layer between processes and moving presence out of memory.

## Considered Options

- **Bidirectional WebSocket protocol**: messages sent over the socket would need their own validation, rate limiting, error replies and acknowledgements, duplicating what the API already does under ADR 0001 and ADR 0004, and adding an inbound attack surface.
- **Socket.IO**: handles rooms, acknowledgements and reconnection, but speaks its own protocol on top of WebSocket, which makes "the implementation is ours" (R-CHAT-01) harder to defend under R-GEN-02.
- **Redis pub/sub from the start**: allows several instances, but adds a service and failure mode for a load one process handles easily.
