# Hand-rolled opaque sessions with tokenless CSRF defence

A logged-in browser holds one random 256-bit token in a `__Host-sid` cookie (HttpOnly, Secure, SameSite=Lax, Path=/). The server stores only its SHA-256 in a `session` table with the user id, creation time and last-use time, so a database leak yields no usable session. Logout, password change and password reset delete rows, which makes revocation immediate (R-AUTH-06). The same cookie lookup authenticates the WebSocket upgrade.

CSRF is handled without tokens: SameSite=Lax, a JSON-only API (multipart only on upload routes) and a strict `Origin` check on every non-GET request, which together suffice for a same-origin SPA (ADR 0002).

## Considered Options

- **`express-session`**: passes the admissibility test of ADR 0001, but needs a second dependency or a homegrown store for PostgreSQL, and its generic session object invites a debate about R-GEN-02 at the defense for code that is about a hundred lines by hand.
- **JWT access and refresh tokens**: stateless tokens cannot be revoked without a denylist, which recreates a session table with more moving parts; one-click logout and revocation on reset would be approximate.
- **Synchronizer CSRF tokens**: redundant with SameSite plus `Origin` checking on a same-origin API, and extra state to keep in the SPA.
