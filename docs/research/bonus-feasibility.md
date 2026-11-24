# Bonus feasibility

Research for issue #9. Question: for each bonus in chapter V of the subject, is it allowed, what does it cost, and what does it depend on in the mandatory design?

Sources were read on 2026-11-24. Anything not confirmed in a primary source is marked **(unverified)**.

## Framing: what "allowed" means here

- Chapter V lists five bonus features verbatim: OmniAuth strategies, a personal photo gallery with drag-and-drop upload and basic editing (crop, rotate, filters), an interactive user map "nécessitant une localisation GPS plus précise via JavaScript", video or audio chat for connected users, and planning real-world dates or events for matched users (`fr.subject.matcha.pdf`, chapter V).
- Because each feature is named in the subject, building it satisfies R-GEN-04 ("anything not explicitly allowed is forbidden") at the feature level. R-GEN-04 still applies to *how* it is built: the tools must pass R-GEN-02 (micro-framework: router plus optional templating, no ORM, no validators, no user account manager) and R-GEN-03 (any library, as long as R-GEN-02 holds).
- R-BONUS-01: no bonus is evaluated unless the mandatory part is perfect. Every hour spent on a bonus is only worth it once the mandatory part is done.
- R-GEN-01 (zero console errors or warnings) and R-SEC-01 to R-SEC-05 (any flaw scores 0) apply to bonus code as much as to mandatory code.

## Summary

| Bonus | Allowed | Rough cost | Main dependencies |
|---|---|---|---|
| OAuth login (OmniAuth-style) | Yes, with a protocol-only client; avoid "auth frameworks" that own users and sessions | Medium (2-4 days) | #18 auth/session design, R-AUTH-01/03/04 account model |
| Photo gallery, DnD, crop/rotate/filters | Yes | Low-medium (2-3 days) | #8 upload validation, R-PROF-01 photo model |
| Interactive user map | Yes | Low-medium (2-3 days) | #3 geolocation, #14 geospatial storage, R-PROF-07 privacy |
| Audio/video chat (WebRTC) | Yes, but needs a TURN server on the VPS | High (5-8 days incl. TURN ops) | #6/#19 real-time transport (signaling), #10/#21 VPS ports and TLS |
| Dates/events between matched users | Yes, no third-party library needed | Low-medium (2-3 days) | #15 domain model, #19 notifications, R-VIEW-04/05/09 |

Cost estimates are judgment, not sourced.

## 1. OAuth login ("OmniAuth strategies")

### What OmniAuth is

OmniAuth is a Ruby library; "Each OmniAuth strategy is a Rack Middleware". It was "intentionally built not to automatically associate with a User model" and "does not perform any actions beyond setting some environment information on the callback request" ([omniauth README](https://github.com/omniauth/omniauth)). So the subject's reference point is a protocol-only layer: the app keeps ownership of user records and sessions. In TypeScript the equivalent is an OAuth 2.0 / OIDC client library, or the authorization code flow written by hand.

### Is it allowed?

- Feature: yes, named in chapter V (R-GEN-04 satisfied).
- Library: R-GEN-02 forbids a "user account manager". A protocol client that only builds the authorize URL, exchanges the code and returns tokens does not manage accounts, same as OmniAuth itself. Full auth frameworks that create users, link accounts and own sessions are the risky category at a defense. My reading of R-GEN-02: they would count as a user account manager. The subject does not name any JS library, so this is interpretation, not a quoted rule.
- Candidates:
  - `openid-client` 6.9.1, MIT (`npm view`). OAuth 2 and OpenID Connect client, certified for OpenID Connect Basic, FAPI 1.0 and FAPI 2.0 Relying Party profiles, runs on Node.js 20+, supports PKCE; the app must "store the code_verifier and state in the end-user session" itself ([openid-client README](https://github.com/panva/openid-client)). Protocol-only: safest choice.
  - Hand-written code flow with `fetch`: three HTTP calls, no dependency. Viable because only 1-2 providers are needed.
  - Arctic: deprecated in July 2026 per its README ([arctic repo](https://github.com/pilcrowonpaper/arctic)). Do not use.
  - Passport 0.7.0 (MIT) plus `passport-42` 1.2.6 (MIT) (`npm view`). Passport describes itself as "authentication middleware for Node.js" that delegates "data access to the application" ([Passport overview](https://www.passportjs.org/concepts/authentication/)). Its session integration was not checked **(unverified)**; it is defensible but brings more surface than needed.

### Security requirements (any flaw scores 0)

From RFC 9700 (OAuth 2.0 Security BCP) ([RFC 9700](https://datatracker.ietf.org/doc/html/rfc9700)):
- Use the authorization code flow; clients "SHOULD NOT use the implicit grant" (2.1.2).
- PKCE is RECOMMENDED for confidential clients (2.1.1); if PKCE support is not confirmed, use a one-time `state` for CSRF (4.7.1).
- Authorization servers use exact redirect URI matching (2.1, 4.1.3).
- Client secret goes in `.env` (R-GEN-11).

### Providers

- 42 intra: the web application flow guide sits behind a 42 login, so it could not be read. A search snippet confirms the standard code flow with `client_id`, `state` and client secret exchange ([42 web application flow](https://api.intra.42.fr/apidoc/guides/web_application_flow)). Rate limits, PKCE support and email verification status are **(unverified)**. Most relevant provider for evaluators, who all have a 42 account.
- GitHub: authorize at `https://github.com/login/oauth/authorize`, token at `https://github.com/login/oauth/access_token`, PKCE supported with `S256` only, `state` "Strongly recommended", code expires after 10 minutes, redirect must match the callback URL exactly by default ([GitHub docs](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)). How to get the email (`user:email` scope) was not checked **(unverified)**.
- Google: redirect URIs "must use the HTTPS scheme" and "Hosts cannot be raw IP addresses" (localhost exempt) ([Google OAuth web server](https://developers.google.com/identity/protocols/oauth2/web-server)). Needs a domain name on the VPS, which is planned anyway. Whether basic `openid email profile` scopes need app verification was not confirmed **(unverified)**.

### Dependencies on mandatory design

- R-AUTH-01 requires email, username, last name, first name, password at registration; R-AUTH-03 requires email verification; R-AUTH-04 requires username/password login. OAuth must not create a second kind of account that skips these. Decision for #18: either OAuth only links to an existing verified account, or an OAuth sign-up completes the missing fields (username, password) before the account is usable. Trusting the provider's email as verified depends on the provider guaranteeing it **(unverified for 42)**.
- Session creation must reuse the mandatory session mechanism (#18), not a library's.

## 2. Photo gallery with drag-and-drop and basic editing

### Is it allowed?

Yes: named in chapter V, and client-side UI libraries are explicitly allowed (R-GEN-03).

### Building blocks

- Drag-and-drop: native HTML Drag and Drop API; the drop zone must cancel `dragover` for `drop` to fire, then read files via `dataTransfer.items` / `getAsFile()`; a hidden `<input type="file">` behind a `<label>` gives the click fallback ([MDN File drag and drop](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API/File_drag_and_drop)). No library needed.
- Crop and rotate: Cropper.js 2.3.1, MIT ([site](https://fengyuanchen.github.io/cropperjs/), `npm view`). v2 is a set of custom elements (`@cropper/elements`: "A series of custom elements for the Cropper", `npm view`); `cropper-image` exposes `$rotate`, `$zoom`, `$scale`, `$skew`; `cropper-selection` exposes `$toCanvas()` ([cropper-image API](https://github.com/fengyuanchen/cropperjs/blob/main/docs/api/cropper-image.md), [cropper-selection API](https://github.com/fengyuanchen/cropperjs/blob/main/docs/api/cropper-selection.md)).
- Filters: `CanvasRenderingContext2D.filter` accepts CSS filter functions (`grayscale()`, `sepia()`, `brightness()`, `contrast()`, `saturate()`, `blur()`, ...) ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/filter)). Not Baseline (Safari only behind a flag), but supported since Chrome 52 and Firefox 49 ([MDN browser-compat-data](https://github.com/mdn/browser-compat-data/blob/main/api/CanvasRenderingContext2D.json)), which covers R-GEN-08.

### Dependencies on mandatory design

- R-PROF-01 caps the profile at 5 photos, one being the profile picture. Decide in #20 whether the gallery is the same 5 photos with editing, or a separate gallery (then the cap does not apply to it, but storage and moderation grow).
- Edited images are still uploads: R-SEC-03 and R-SEC-05 apply. The server must validate and re-encode whatever the client sends; reuse the pipeline from #8. Alternative: send crop/rotate/filter parameters and apply them server-side, so the client never produces the stored bytes.

## 3. Interactive user map

### Is it allowed?

Yes: named in chapter V. Map libraries fall under R-GEN-03.

### Libraries

- Leaflet 1.9.4 stable, BSD-2-Clause; 2.0 is still beta (`2.0.0-beta.5`) (`npm view leaflet dist-tags`). About 40 KB of JS, tile layers, markers, popups ([leafletjs.com](https://leafletjs.com/)). Clustering via `leaflet.markercluster` 1.5.3, MIT (`npm view`). Use 1.9.4, not the beta, to limit R-GEN-01 risk.
- MapLibre GL JS 6.19.0, BSD-3-Clause (`npm view`): "uses WebGL to render interactive maps from vector tiles", no API key required by the library; needs CSP `worker-src 'self'` and `img-src data: blob: 'self'` ([MapLibre docs](https://maplibre.org/maplibre-gl-js/docs/)).

### Tiles

- OSM `tile.openstreetmap.org` ([tile usage policy](https://operations.osmfoundation.org/policies/tiles/)): attribution must be shown clearly and not hidden; browsers must send a valid `Referer` (no restrictive `Referrer-Policy`); honour caching headers; no bulk download or offline use; "Availability is best-effort" with no SLA; access may be blocked "without notice" if usage degrades the service. Fine for a student project's traffic if those rules are met; the app's `Referrer-Policy` header must not be `no-referrer`.
- OpenFreeMap: free public instance, "no limits on the number of map views or requests", "no registration, no user database, no API keys, and no cookies", attribution required (automatic with MapLibre), self-hostable under MIT, no SLA ([openfreemap.org](https://openfreemap.org/)). Pairs with MapLibre.
- No API key in either option, so nothing to protect under R-GEN-11.

### Precise GPS

- `navigator.geolocation.getCurrentPosition()` needs a secure context (HTTPS) and explicit user permission; `enableHighAccuracy: true` gives the best result at the cost of time and power ([MDN getCurrentPosition](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition)). `coords.accuracy` is the radius in meters at 68% confidence ([MDN accuracy](https://developer.mozilla.org/en-US/docs/Web/API/GeolocationCoordinates/accuracy)).

### Dependencies on mandatory design

- Reuses #3 (geolocation and consent flow) and #14 (geospatial storage). R-PROF-07 requires explicit consent, and R-PROF-08 users have only a city/neighbourhood: the map must handle approximate locations.
- Privacy and security: plotting other users' exact coordinates exposes home addresses. Store precise coordinates if needed for distance, but render others at neighbourhood precision (snap or jitter server-side) and exclude blocked users (R-VIEW-09). This is a design choice, not a subject rule.

## 4. Audio/video chat (WebRTC)

### Is it allowed?

Yes: named in chapter V ("pour les utilisateurs connectés"). WebRTC is a browser API; a TURN server is infrastructure, not a framework, so R-GEN-02 is not in play. Signaling rides on the mandatory real-time channel (R-CHAT-01).

### What it takes

- `getUserMedia()` works only in secure contexts (HTTPS or localhost) and always needs user permission ([MDN getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)). HTTPS is already planned (#10).
- ICE uses STUN to discover the public address; behind symmetric NAT "we need to use TURN", which relays all media through the server ([MDN WebRTC protocols](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)). webrtc.org: "For most WebRTC applications to function a server is required for relaying the traffic between peers", and `username`/`credential` "are required when connecting to a TURN server"; it names coturn as the open-source self-hosted option ([webrtc.org TURN](https://webrtc.org/getting-started/turn-server)).
- coturn: "free open source implementation of TURN and STUN Server", Docker images on Docker Hub, GHCR and Quay, TLS 1.3 support ([coturn README](https://github.com/coturn/coturn)); BSD-style license (LICENSE header, full terms not read). Defaults: 3478 UDP/TCP, 5349 TLS, UDP relay range 49152-65535 ([turnserver.conf](https://raw.githubusercontent.com/coturn/coturn/master/examples/etc/turnserver.conf)). `use-auth-secret` + `static-auth-secret` enables the TURN REST API: username `timestamp:userid`, password `base64(HMAC(secret, username))`, so our back end mints short-lived credentials only for connected users. The secret lives in `.env` (R-GEN-11).

### Cost and risks

- VPS work: open 3478 and 5349 plus a narrowed relay port range in the firewall, give coturn the TLS certificate, set `external-ip` if the VPS is behind NAT **(unverified for this VPS, see #11)**, and bandwidth for relayed media.
- Testing needs two networks (e.g. a phone on mobile data); NAT edge cases are the main time sink.
- R-GEN-01: WebRTC and media permission errors surface in the console; every failure path must be handled.
- Authorization: credentials and calls only between connected users (R-VIEW-04), cut when a like is removed (R-VIEW-05) or a user is blocked (R-VIEW-09).

### Dependencies on mandatory design

- #6/#19: the transport must carry signaling messages (offer, answer, ICE candidates) between two users, which a WebSocket/SSE+POST design already does.
- #10/#21: the deployment topology must leave room for a coturn container with host networking or the UDP range published.

## 5. Dates and events between matched users

### Is it allowed?

Yes: named in chapter V. Pure domain feature; no library required, so R-GEN-02 is not in play.

### What it takes

- New domain entity (proposal with time, place, status: proposed, accepted, declined, cancelled), hand-written SQL (R-GEN-05), validated inputs (R-SEC-05), escaped output (R-SEC-02).
- Optional: `.ics` export so users can add the date to their calendar; the format is RFC 5545 **(not read for this research)**.
- Optional: place picker reusing the map from bonus 3.

### Dependencies on mandatory design

- Only between connected users (R-VIEW-04); pending dates cancelled on unlike (R-VIEW-05) or block (R-VIEW-09).
- Notifications reuse the R-NOTI-01 channel (#19); new notification types go beyond the mandatory list, which is fine for a bonus.
- #15 domain model and the word replacing "like".

## Recommendation for #20

1. Nothing before the mandatory part is complete (R-BONUS-01).
2. Best value per effort: **interactive map** (reuses #3/#14, no secrets, no new infra), **dates/events** (reuses domain, notifications and real-time), **photo gallery editing** (reuses #8, client-only libraries).
3. **OAuth with 42** is worth it for the defense if #18 keeps a clean account model: use `openid-client` or a hand-written code flow with PKCE and `state`, never a framework that owns users or sessions.
4. **WebRTC** last: highest cost, new infrastructure (coturn, UDP ports), hardest to test, and the most likely to produce console noise. Only if time remains.

Mandatory decisions that should leave room for bonuses, even if none is built: HTTPS with a domain name (#10, needed by Google OAuth, getUserMedia, precise geolocation), a `Referrer-Policy` that still sends the origin (OSM tiles), and a real-time channel able to carry arbitrary typed messages between two users (#6/#19, WebRTC signaling).
