# Express serves the whole app behind a shared edge Caddy

In production, one Express process serves the JSON API, `/ws` and the built SPA (static files plus an `index.html` fallback): Express is the "built-in server" of R-GEN-07, and no web server sits inside Matcha's own stack. TLS is terminated by a separate `edge` Compose stack on the VPS, an official Caddy image that owns ports 80 and 443 for every site on the host and only proxies to app containers over an external Docker network. This keeps the application within the letter of R-GEN-07 and R-GEN-04, while the host gets automatic certificates without certbot wiring.

The edge is host infrastructure, not part of either app: it lives in `/opt/edge`, owned by root and changed only by a human (wizard), and its files are versioned in this repo under `deploy/edge/` so the hosting is reproducible from Git (R-GEN-12). Blackjack, whose web image currently owns 80/443 with its own Caddy, is moved behind the edge: its Caddy keeps its headers but listens on plain HTTP and publishes no port.

## Considered Options

- **nginx with certbot at the edge**: named in R-GEN-07, but needs manual certificate renewal and explicit WebSocket headers and timeouts. Since Express already satisfies R-GEN-07, the edge does not need to be named in the subject.
- **Caddy serving Matcha's static files**: makes Caddy part of the application, which reopens the R-GEN-07 question, and couples the edge to Matcha's build output.
- **Edge inside Matcha's Compose project, or Blackjack's Caddy as the edge**: either couples one app's releases to the other's availability.

## Consequences

- BlackjackOnline needs a release that drops its published ports, sets `SITE_ADDRESS=:80` and joins the `edge` network; the cutover happens before Matcha's first deployment.
- Matcha sets its own security headers in Express; the edge adds nothing but TLS, HTTP to HTTPS redirect and compression.
- A coturn service for the calls bonus would run with host networking on 3478 and 5349 (tcp/udp) plus a UDP relay range, outside the edge.
