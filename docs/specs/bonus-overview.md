# Bonus overview

Which bonuses are built, in what order, and the hooks the mandatory part leaves for them (R-BONUS-01). Each bonus gets its own detailed spec later; this document fixes only selection, order and hooks. Feasibility research: issue 9.

## Selection and order

All five bonuses of the subject are designed. They are built in this order, and none starts before the mandatory part is complete and tested end to end (R-BONUS-01):

1. Interactive map
2. Meetups
3. Photo gallery
4. OAuth login (42 intra and GitHub)
5. Audio and video calls: stretch goal, designed but not committed

A bonus is merged only when finished. There are no feature flags.

## Interactive map

- The viewer's own position uses the browser Geolocation API with `enableHighAccuracy`, only to centre the map, with consent.
- Other users are placed at their stored point rounded server-side to a 0.01 degree grid (about 1 km). The raw point never leaves the server (ADR 0003, R-PROF-07).
- Shown users follow the browse eligibility (complete profile, compatible, no block either way), restricted to the visible bounds, at most 200, with marker clustering.
- Leaflet 1.9.4 with OpenStreetMap tiles; attribution displayed and the default `Referrer-Policy` kept so tile requests carry the origin.

Mandatory hook: none beyond ADR 0003.

## Meetups

- A Meetup is proposed by one connected user to the other with a time, a place and an optional note; the other accepts or declines; either can cancel. Two people only.
- A Meetup is cancelled automatically when the Connection breaks (uncrush, block, fake report).
- New notification types: meetup proposed, accepted, declined, cancelled, following the creation rule of the chat and notifications spec.

Mandatory hook: notification types are an open list in `shared/`, not a closed database enum that forbids additions without a migration review.

## Photo gallery

- The gallery is the profile's photo manager upgraded: drag-and-drop upload, crop, rotate and filters in the browser (Cropper.js and canvas `filter`). The cap stays at 5 photos (R-PROF-01).
- Edits are baked into the pixels before upload; the server validates and re-encodes the result through the same pipeline as any upload (R-SEC-03, R-SEC-05, issue 8).

Mandatory hook: the photo upload endpoint accepts any valid image, with no assumption about its origin.

## OAuth login

- Two providers, 42 intra and GitHub, through a hand-written authorization code flow: `state` always, PKCE where the provider supports it. No library that manages users or sessions (R-GEN-02, ADR 0001).
- Signing up through a provider creates a user without a password; the user confirms a username prefilled from the provider. The provider's email counts as verified only when the provider asserts it; otherwise the usual verification email is sent (R-AUTH-03).
- An existing account is never linked automatically on a matching email. Linking a provider is done from the settings while logged in.
- A user without a password can set one through the password reset flow.

Mandatory hooks: the password hash column is nullable; a separate table holds external identities (provider, subject at the provider, user).

## Audio and video calls

- One-to-one audio and video between connected users, over WebRTC.
- Signaling goes through HTTP `POST` requests relayed to the other user as WebSocket events, so the WebSocket stays push-only (ADR 0005).
- A coturn TURN server on the VPS, with short-lived HMAC credentials (`use-auth-secret`). It needs ports 3478 and 5349 and a UDP relay range open.

Mandatory hook: WebSocket events are a typed union in `shared/` that new event types can extend.
