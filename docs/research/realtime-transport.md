# Real-time transport options compatible with R-GEN-02

Research for issue #6. Feeds the decision ticket #19.

## Question

Which real-time transports (WebSocket, Server-Sent Events, long polling) and which Node libraries (ws, socket.io, uWebSockets.js, native) meet R-CHAT-01 and R-NOTI-01 (at most 10 s delay) while staying within R-GEN-02 / R-GEN-04? How do they behave behind an HTTPS reverse proxy, how do they reconnect, and can they carry WebRTC signaling for an audio/video bonus (R-BONUS-01)?

## Short answer

- All three transports can meet the 10 s bound (R-CHAT-01, R-NOTI-01). The deciding factors are the proxy setup, reconnection and the bonus signaling need.
- WebSocket is the only one that is bidirectional on a single connection, which fits chat and WebRTC signaling directly. SSE is server-to-client only, so the client must send with ordinary HTTP requests.
- `ws` is a plain WebSocket library (no router, no ORM, no validators, no account manager), so it is the lowest-risk choice against R-GEN-02. Socket.IO is also a library, not a framework, but it ships its own protocol plus reconnection, buffering, acks and rooms, which makes "the implementation is ours" (R-CHAT-01) a weaker argument at defense (interpretation, not verified against any 42 ruling).
- Recommendation: WebSocket via `ws` attached to the existing `node:http` server with `noServer` + `handleUpgrade`, cookie session check and Origin check at upgrade, server ping/pong heartbeat, client reconnection with exponential backoff and a re-sync fetch on reconnect. One socket carries chat, notifications and (later) WebRTC signaling.

## Transports

### WebSocket (RFC 6455)

- Bidirectional over one connection; browser `WebSocket` API available across browsers since July 2015 (R-GEN-08). Source: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
- No backpressure in the `WebSocket` API; `WebSocketStream` is the alternative. Irrelevant at chat volumes. Source: same MDN page.
- Authentication is not prescribed by the protocol; servers can use cookies, HTTP auth or TLS auth during the handshake (RFC 6455 section 10.5). Servers SHOULD verify the `Origin` header and reply 403 when unexpected (section 10.2). This matters for R-SEC-*: a cookie-authenticated socket without an Origin check is open to cross-site WebSocket hijacking. Source: https://www.rfc-editor.org/rfc/rfc6455
- Reconnection is not automatic. RFC 6455 section 7.2.3 says clients SHOULD back off after abnormal closure: random initial delay (0 to 5 s suggested), then e.g. truncated binary exponential backoff. MDN's `WebSocket` page documents no automatic reconnection (not stated explicitly, inferred from absence). Source: https://www.rfc-editor.org/rfc/rfc6455
- Ping frames may serve as keepalive (RFC 6455 section 5.5.2, a NOTE, not normative). Source: https://www.rfc-editor.org/rfc/rfc6455

### Server-Sent Events (`EventSource`)

- One-way: "you can't send events from a client to a server". Chat sends would go through `fetch()` POSTs. Source: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Automatic reconnection by the browser; `retry` field sets the delay in ms; on reconnect the UA sends `Last-Event-ID` with the last `id` seen, which allows resuming missed events. No reconnect after a failed connection (non-200 status or wrong Content-Type). Source: https://html.spec.whatwg.org/multipage/server-sent-events.html (sections 9.2.1 to 9.2.4)
- Spec recommends a comment line (`:`) every 15 s or so to keep legacy proxies from dropping the connection. Source: same WHATWG page, section 9.2.7.
- Without HTTP/2, browsers cap SSE at 6 connections per browser + domain across all tabs; with HTTP/2 the limit is negotiated (default 100 streams). A user with several tabs open over HTTP/1.1 would hit this. Source: MDN "Using server-sent events" above.

### Long polling

- No browser API or spec of its own; built from plain `fetch()`. Meets 10 s if the server holds each request and answers on the first event. Each message costs a full request cycle, and the client must handle reconnection and ordering itself. Socket.IO uses it as a fallback transport. Source: https://socket.io/docs/v4/ (transports list). No other primary source consulted; trade-offs here are general knowledge, not verified.

## Node libraries

| Option | What it is | R-GEN-02 fit | Notes |
|---|---|---|---|
| `ws` | WebSocket client and server for Node; "does not work in the browser" (browser uses native `WebSocket`) | Library only, no framework features | Integrates with an existing `http` server via `noServer: true` + `handleUpgrade`; README shows auth in the upgrade handler (401 + destroy socket on failure) and a 30 s ping/pong heartbeat to terminate dead peers. Source: https://github.com/websockets/ws/blob/master/README.md |
| Socket.IO | "NOT a WebSocket implementation": own protocol over long-polling, WebSocket or WebTransport; plain WebSocket clients cannot talk to it | Library, not a router/ORM/validator/account manager, so allowed under R-GEN-03; but it does reconnection, buffering, acks and rooms for us (defense risk, interpretation) | Auto-reconnect with exponential back-off, heartbeat, packet buffering while disconnected, acknowledgements, rooms. Source: https://socket.io/docs/v4/ |
| uWebSockets.js | C++ web server exposed as a native V8 addon | Brings its own HTTP server and router, so it is closer to a server framework; also a native binary | Not on the npm registry, installed from a GitHub tag; licensing statement is conditional ("all rights reserved" plus Apache 2.0 where noted). Source: https://github.com/uNetworking/uWebSockets.js/blob/master/README.md |
| Node native | Global `WebSocket` is a browser-compatible client (stable since v22.4.0); global `EventSource` is experimental behind `--experimental-eventsource` | N/A | Node core documents no WebSocket server, so a server needs `ws` or a hand-written RFC 6455 framer. SSE needs no library: it is a `text/event-stream` response on `node:http`. Source: https://nodejs.org/api/globals.html |

Socket.IO and uWebSockets.js are not disallowed outright by R-GEN-02's wording; the risk is judgement at defense. `ws` and hand-written SSE carry the least risk.

## Behind an HTTPS reverse proxy

### nginx

- WebSocket: `Upgrade` and `Connection` are hop-by-hop and must be set explicitly with `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";` (or a `map` for mixed locations). Source: https://nginx.org/en/docs/http/websocket.html
- `proxy_http_version` defaults to 1.1 only since nginx 1.29.7; before that it defaulted to 1.0 and must be set to 1.1 for WebSocket. Check the deployed version. Source: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- `proxy_read_timeout` defaults to 60 s between two reads; idle sockets are closed unless the server pings or the timeout is raised. Source: both nginx pages above.
- SSE: `proxy_buffering` is on by default; disable per location or send `X-Accel-Buffering: no` from the app, otherwise events can be held in the buffer. Source: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- Socket.IO: nginx `proxy_read_timeout` must exceed Socket.IO `pingInterval + pingTimeout` (45 s by default). Source: https://socket.io/docs/v4/reverse-proxy/

### Caddy

- Supports WebSocket by performing the upgrade and switching to a bidirectional tunnel. Responses with `Content-Type: text/event-stream` are flushed immediately regardless of `flush_interval`. Source: https://caddyserver.com/docs/caddyfile/directives/reverse_proxy
- Whether WebSocket needs any explicit configuration is not stated on that page (not verified).

### TLS

- Behind HTTPS the browser must use `wss://` (same-origin path, e.g. `/ws`). TLS terminates at the proxy; the Node process speaks plain HTTP to it. (General knowledge; no primary source fetched for mixed-content rules.)

## WebRTC signaling (bonus)

- "WebRTC doesn't specify a transport mechanism for the signaling information"; WebSocket is "a common solution" and MDN's example uses it. Source: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling
- Signaling needs server push to a specific peer at any time (offers, answers, trickled ICE candidates), and the server treats payloads as opaque. Source: same page. A WebSocket already used for chat covers this with a new message type. SSE + POST also works but adds a request per ICE candidate (inference).

## Recommendation for #19

1. Transport: WebSocket, single connection per tab, path `/ws`, same origin.
2. Library: `ws` in `noServer` mode on the app's `node:http` server. Authenticate from the session cookie in the `upgrade` handler and reject unexpected `Origin` with 403 (RFC 6455 10.2, R-SEC-*).
3. Liveness: server ping every <= 30 s (below nginx's 60 s `proxy_read_timeout`), terminate peers that miss a pong.
4. Reconnection: client-side exponential backoff with jitter (RFC 6455 7.2.3); after reconnect, fetch unread messages/notifications over HTTP so nothing is lost while offline (R-CHAT-02, R-NOTI-02).
5. Proxy: nginx with explicit `Upgrade`/`Connection` headers and `proxy_http_version 1.1` if version < 1.29.7.
6. Fallback if `ws` is ruled out: SSE on plain `node:http` (no library) + POST for sends, with HTTP/2 at the proxy to avoid the 6-connection limit and `X-Accel-Buffering: no`.

## Not verified

- No official 42 ruling on whether Socket.IO counts as a framework under R-GEN-02; the risk assessment above is interpretation.
- Caddy's need for explicit WebSocket config.
- Long-polling trade-offs and `wss://` mixed-content rules (general knowledge, no primary source fetched).
- uWebSockets.js supported Node versions and platforms (README does not list them).
