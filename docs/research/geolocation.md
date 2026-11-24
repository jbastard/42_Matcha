# Research: geolocation, IP fallback, geocoding and distance

Ticket: #3. Rules in scope: R-PROF-07, R-PROF-08, R-PROF-09, R-BROW-03, R-BROW-04, R-BROW-05, R-BROW-06, R-SRCH-01, R-GEN-01, R-GEN-03, R-GEN-04, R-GEN-05, R-GEN-06, R-GEN-08, R-GEN-11.

Sources were read on 2026-11-24. Anything marked "unverified" was not confirmed from a primary source.

## 1. Browser Geolocation API (R-PROF-07)

- Secure context only: "This feature is available only in secure contexts (HTTPS)". https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API
- `http://localhost` and `127.0.0.0/8` are potentially trustworthy, so local dev works without TLS. https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts
- The browser itself prompts the user when `getCurrentPosition()` or `watchPosition()` is called, unless permission is already `granted` or `denied`. The Permissions API exposes `granted` / `denied` / `prompt` for `geolocation`. Grant lifetime depends on the user agent. https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API
- `getCurrentPosition()` is one-shot; `watchPosition()` fires on each position change. A dating profile only needs one-shot, refreshed on demand (R-PROF-09). Same source.
- Options: `enableHighAccuracy` (default `false`), `timeout` (default `Infinity`), `maximumAge` (default `0`). https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition
- Errors: `1 PERMISSION_DENIED`, `2 POSITION_UNAVAILABLE`, `3 TIMEOUT`. https://developer.mozilla.org/en-US/docs/Web/API/GeolocationPositionError/code
- `coords.accuracy` is in meters at 68% confidence. https://developer.mozilla.org/en-US/docs/Web/API/GeolocationCoordinates/accuracy
- Permissions-Policy default allowlist for `geolocation` is `self`. Only relevant if the app is framed or sets a restrictive policy header. https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API

Consequences for us:
- Set a finite `timeout` (otherwise the default `Infinity` can hang the UI) and `enableHighAccuracy: false` (neighbourhood precision does not need GPS-grade fixes).
- Every error code must lead to the manual-entry path of R-PROF-08, and must be handled without a console error (R-GEN-01).
- "Explicit consent" (R-PROF-07): the browser prompt is consent at the browser level. Whether the defense expects an additional in-app opt-in is an interpretation, not verified. A cheap safe choice is an in-app button "Use my position" that triggers the browser prompt, never a call on page load.

## 2. IP-based fallback

| Option | Terms | Source |
|---|---|---|
| MaxMind GeoLite City (local `.mmdb`) | Free account and license key required; EULA requires keeping data current and deleting a release within 30 days of a new one; 30 downloads/day; web service 1000 lookups/day; "IP geolocation is inherently imprecise", "should not be used to identify a particular address or household"; free City "considerably less accurate" than paid. Attribution requirement not stated on the page (unverified). | https://dev.maxmind.com/geoip/geolite2-free-geolocation-data/ |
| DB-IP IP to City Lite (local `.mmdb` / CSV) | CC BY 4.0; attribution and a link back to DB-IP.com on pages that display or use the results; updated monthly; no account visible for the download. | https://db-ip.com/db/download/ip-to-city-lite |
| ip-api.com free endpoint | "strictly limited for a non-commercial purpose"; 45 req/min; SSL listed as a pro feature, so the free endpoint is likely HTTP only (inferred, not stated). | https://ip-api.com/docs/legal |

Consequences for us:
- IP geolocation is city-level at best (MaxMind's own disclaimer), so it cannot satisfy "down to the neighbourhood" (R-PROF-07).
- Locating someone by IP without asking is arguably locating them without consent. R-PROF-08 says a user who refuses GPS "must enter" an approximate location, so the subject's fallback is manual entry, not IP. At most, an IP lookup can pre-fill the city field that the user then confirms or edits. This is an interpretation; the subject does not mention IP.
- If used, a local `.mmdb` (DB-IP Lite needs no key; MaxMind needs a key in `.env` per R-GEN-11) avoids a third-party call per request. Behind a reverse proxy the client IP comes from `X-Forwarded-For`, which must be trusted only from our own proxy.

## 3. Geocoding and reverse geocoding (R-PROF-07, R-PROF-08)

### Nominatim (OSM), public instance

Usage policy, https://operations.osmfoundation.org/policies/nominatim/ :
- Absolute maximum 1 request/second.
- A valid HTTP Referer or User-Agent identifying the application; stock library User-Agents are not acceptable.
- Results must be cached on our side; repeated identical queries can get the client blocked.
- Clear attribution (OSM data is ODbL).
- No autocomplete, including client-side on top of the API.
- Forbidden: systematic queries (e.g. reverse queries on a grid), scraping, reselling.
- Do not submit personal data or confidential material.
- Must be switchable to another service on request without a software update (i.e. the endpoint is configuration).

API:
- Search: free-form `q` or structured `city`, `county`, `state`, `country`, `postalcode`; `countrycodes` filter; `featureType=city|settlement` (`settlement` covers state down to neighbourhood); `limit` up to 40; `format=jsonv2`; `accept-language`; `email` recommended for higher volumes. https://nominatim.org/release-docs/latest/api/Search/
- Reverse: `zoom` sets address detail. 10 = city, 12 = town/borough, 13 = village/suburb, 14 = neighbourhood. Formats `xml`, `json`, `jsonv2`, `geojson`, `geocodejson`. Address keys such as `suburb`, `neighbourhood`, `city` appear in examples; the exact key list is not fixed on that page, and `geocodejson` is recommended for stable admin levels. https://nominatim.org/release-docs/latest/api/Reverse/

### Photon (komoot), public instance

"You can use the API for your project, but please be fair - extensive usage will be throttled." No numeric limit, no availability guarantee. It advertises search-as-you-type, but its terms do not state that public-instance autocomplete is allowed (unverified). https://photon.komoot.io/

### Consequences for us

- Call Nominatim from the backend, not the browser: one place to set the User-Agent, cache, and rate-limit to 1 req/s.
- City input (R-PROF-08): a "search" button, not keystroke autocomplete (forbidden by the policy). Use `featureType=settlement`, `limit` around 5, and let the user pick one result.
- GPS result (R-PROF-07): reverse-geocode with `zoom=14` to get a neighbourhood label for display.
- Seeding 500 profiles (R-GEN-06) must not call Nominatim in a loop over a grid (systematic queries are banned). Generate coordinates offline from a small, hand-picked list of places, or geocode a few dozen distinct place names once with throttling and cache them.
- Endpoint URL and User-Agent go in configuration (`.env`, R-GEN-11), which also satisfies the "switchable" clause.
- Rounding coordinates before sending them for reverse geocoding limits the personal-data exposure the policy warns about. Whether raw GPS coordinates count as "personal data" under the policy is an interpretation.

## 4. Distance computation (R-BROW-03, R-BROW-04, R-BROW-05, R-BROW-06, R-SRCH-01)

### PostGIS `geography`

- Spherical model with great-circle paths; measurements in meters; SRID 4326 by default; GiST index on a geography column uses a sphere-based index. https://postgis.net/docs/using_postgis_dbmanagement.html
- `ST_DWithin(geography, geography, meters, use_spheroid = true)`, with `use_spheroid = false` faster on the sphere; it includes a bounding-box comparison that uses available indexes. https://postgis.net/docs/ST_DWithin.html
- `<->` supports geography (sphere distance) and gives index-assisted KNN in `ORDER BY`, but only when one side is a constant. https://postgis.net/docs/geometry_distance_knn.html

### PostgreSQL `earthdistance` (contrib, requires `cube`)

- Earth assumed perfectly spherical. Cube-based: `ll_to_earth(lat, lon)`, `earth_distance()` in meters, `earth_box()` for indexed search with `@>` plus a second exact check. Point-based `<@>` returns statute miles and has edge issues near the poles and the antimeridian. https://www.postgresql.org/docs/current/earthdistance.html

### Haversine in plain SQL

- Works with two `double precision` columns and no extension, but no index can bound the search, so every filter by radius is a full scan of candidates. The formula itself was not re-checked against a primary source here; it is standard spherical trigonometry.

### Comparison

| | PostGIS | earthdistance | Haversine SQL |
|---|---|---|---|
| Extra install | PostGIS extension (separate package) | contrib, ships with PostgreSQL | none |
| Units | meters | meters (cube) | whatever radius we pick |
| Index for radius filter | GiST + `ST_DWithin` | GiST on `ll_to_earth(...)` + `earth_box` (index type unverified on that page) | none |
| Accuracy | spheroid by default | sphere | sphere |

Note on R-GEN-02 / R-GEN-05: these are database extensions, not ORMs; queries stay hand-written. Whether an evaluator could object to PostGIS is not settled by the subject; R-GEN-03 allows "any library needed".

At 500 to a few thousand profiles (R-GEN-06), all three are fast enough; the index matters little at that scale (not benchmarked).

## Recommendation for the decision ticket

1. Primary: an explicit "Use my position" button calling `getCurrentPosition({ enableHighAccuracy: false, timeout: ~10s, maximumAge: some minutes })`. On success, the backend reverse-geocodes with Nominatim at `zoom=14` and stores the point plus a neighbourhood/city label.
2. Fallback (R-PROF-08): mandatory manual city/neighbourhood entry, geocoded server-side via Nominatim search (button, no autocomplete, `featureType=settlement`, user picks a result). Matching stays disabled until a location exists.
3. IP: do not locate silently. Optionally pre-fill the manual field from a local DB-IP Lite `.mmdb` (CC BY 4.0, attribution link) and require confirmation. Skipping IP entirely is also compliant.
4. Distance: PostGIS `geography(Point, 4326)` with a GiST index, `ST_DWithin` for radius filters and `ST_Distance` for sorting, if the deployment image can carry PostGIS. Otherwise `cube` + `earthdistance`. Plain haversine is the zero-dependency fallback.
5. Nominatim hygiene: backend-only calls, custom User-Agent, 1 req/s throttle, result cache table, visible OSM attribution, endpoint in `.env`, offline coordinates for the 500-profile seed.

Open points not verified here: whether evaluators expect an in-app consent step on top of the browser prompt; the exact Nominatim address keys per country; PostGIS availability in our chosen Docker image.
