# Seeding 500 realistic profiles with photos

Research for issue #5. Question: how to generate at least 500 distinct, realistic profiles (R-GEN-06) with photos whose licence allows redistribution in a public repo or download at seed time, and how to cluster locations so proximity matching (R-BROW-03, R-BROW-04) is demonstrable.

Sources were read on 2026-11-24. "Unverified" marks claims that could not be traced to a primary source.

## Constraints from the subject

- R-GEN-06: at least 500 distinct profiles in the database at evaluation.
- R-PROF-01: up to 5 photos per profile, one of them the profile picture.
- R-VIEW-03: a user can only like if they have a profile picture, so seeded profiles need one to take part in likes.
- R-BROW-03 / R-BROW-04: proximity weighs in matching and same-area users get priority, so seeded locations must cluster.
- R-GEN-12: only the Git repo is evaluated, so the seed script (and either the photos or the code that downloads them) must be in the repo.
- R-GEN-11: any API key for a photo service lives in `.env`, never in Git.
- R-GEN-03: any library is allowed, so a fake-data library is fine.

## Photo sources compared

| Source | Licence | Commit to public repo | Use on a public site | Volume | Real people? |
|---|---|---|---|---|---|
| faker `personPortrait` assets | CC0-1.0 | Yes | Yes | 100 male + 100 female, 32 to 512 px | No, AI (Stable Diffusion 3) |
| randomuser.me portraits | Not stated, points to UI Faces FAQ (unverified) | Unclear | Unclear | 100 men + 100 women | Unverified |
| Unsplash (site downloads) | Unsplash License | Allowed, but not compiling to replicate a similar service | Yes | Large | Yes |
| Unsplash API | Unsplash License + API guidelines | No: must hotlink | Yes, with attribution | Large, rate limited | Yes |
| Pexels | Pexels License | Not on other stock platforms | Yes, but not showing people "in a bad light" | Large | Yes |
| Generated.Photos (free) | Proprietary | No | No (personal, non-commercial only) | n/a | No, AI |
| FFHQ (NVIDIA) | Dataset CC BY-NC-SA 4.0, per-image Flickr licences | Per image, some need attribution | Non-commercial | 70,000 | Yes |

### faker personPortrait assets (recommended)

- `faker.image.personPortrait({ sex, size })` returns "static images of fictional people created by an AI, Stable Diffusion 3"; `sex` is `female`, `male` or `generic`, `size` is 512, 256, 128, 64 or 32 (https://fakerjs.dev/api/image).
- The images live in `faker-js/assets-person-portrait`, licensed CC0-1.0 per the GitHub API (`gh api repos/faker-js/assets-person-portrait` returns `license.spdx_id: CC0-1.0`). The README states the images are AI-generated and gives the layout `/{male,female}/{32,64,128,256,512}/{0...99}.jpg` (https://github.com/faker-js/assets-person-portrait).
- So there are 200 distinct faces. Checked: `female/512/99.jpg` returns 200, `female/512/100.jpg` returns 404 on jsDelivr.
- Measured size of single samples: about 200 KB at 512 px, 27 KB at 256 px, 8 KB at 128 px. All 200 faces at 256 px should fit in roughly 5 to 6 MB (extrapolated from one sample), small enough to commit.
- CC0 removes the licence question for both committing the files and serving them publicly. AI faces also avoid putting identifiable real people on a dating site.
- The README offers to replace a picture that resembles a real person too closely, and ships per-image metadata (prompt, model) to regenerate more with Stable Diffusion 3 at about $0.035 per image on Replicate. The licence of newly generated SD3 outputs was not checked (unverified).
- Limitation: 200 faces for 500+ profiles means each face is reused about 2.5 times. R-GEN-06 asks for distinct profiles, not distinct photos; whether duplicated faces would be challenged at defense is a judgement call for the decision ticket.

### randomuser.me

- The API is MIT (`gh api repos/RandomAPI/Randomuser.me-Node` returns MIT) and can return up to 5,000 users per request, with `seed`, `gender`, `nat` (FR included) and `inc`/`exc` parameters (https://randomuser.me/documentation).
- On photos it only says they "were hand picked from the authorized section of UI Faces" and refers to the UI Faces FAQ (https://randomuser.me/). No licence text for UI Faces was found: the photo licence is unverified. Do not commit these images; hotlinking them on a public site carries the same uncertainty.
- Checked: `portraits/men/99.jpg` returns 200 and `portraits/men/100.jpg` returns 404, so it also has 200 faces and adds no volume over faker.

### Unsplash

- Licence: irrevocable, worldwide right to "download, copy, modify, distribute, perform, and use images from Unsplash for free", no attribution needed; excludes selling without significant modification and "the right to compile images from Unsplash to replicate a similar or competing service" (https://unsplash.com/license).
- A dating site is not an Unsplash-like service, so committing a few hundred portraits is arguably allowed, but collecting them by hand is tedious.
- Via the API, images must be hotlinked from `photo.urls`, the `download_location` endpoint must be triggered, and the photographer plus Unsplash must be credited with a link (https://help.unsplash.com/en/articles/2511245-unsplash-api-guidelines). Hotlinking makes seeded profiles depend on Unsplash at runtime, and per-photo attribution is awkward in a dating UI. Rate limit numbers were not on that page (unverified).
- Real, identifiable people would appear as fictional dating profiles. The licence page does not address this; it remains a reputational risk on a public site.

### Pexels

- Free use, credit optional; "Identifiable people may not appear in a bad light or in a way that is offensive"; no redistribution on other stock or wallpaper platforms; no implied endorsement by people on the imagery (https://www.pexels.com/license/).
- Presenting real people as fake dating profiles is arguably close to the "bad light" and "implied endorsement" limits. Not recommended.

### Generated.Photos

- Free tier: one copy "for personal, non-commercial usage only", mandatory attribution link, no derivatives, no posting Materials online "in a downloadable format", written consent needed for use in online applications (https://intercom.help/generated-photos/en/articles/7872726-generated-photos-terms-and-conditions). Ruled out for a public repo and site.

### FFHQ

- Dataset metadata is CC BY-NC-SA 4.0; each image keeps its own Flickr licence (CC BY 2.0, CC BY-NC 2.0, PDM, CC0, US Gov), some requiring attribution; 70,000 images, 89 GB at 1024 px (https://github.com/NVlabs/ffhq-dataset). Real people and per-image attribution: too heavy for this need.

## Fake-data generator for TypeScript

`@faker-js/faker` (latest 10.6.0 per `npm view`, MIT per its LICENSE file in faker-js/faker):

- Reproducible output with `faker.seed(123)`; reseeding restarts the sequence, and values can change between Faker versions (https://fakerjs.dev/guide/usage). Pin the version so the seeded set stays stable.
- French data through the `fakerFR` export (also `fakerFR_BE`, `fakerFR_CH`, ...) (https://fakerjs.dev/guide/localization).
- Sex-consistent names: `const sex = faker.person.sexType(); faker.person.firstName(sex)` (https://fakerjs.dev/guide/usage). The same `sex` can feed `personPortrait({ sex })` so name and face match.
- Tags, orientation and popularity are app concepts picked from our own lists; the `faker.helpers` API for weighted picks was not re-read here (unverified in detail).

## Clustering locations

- `faker.location.nearbyGPSCoordinate({ origin: [lat, lon], radius, isMetric })` returns a point within `radius` of `origin`; `radius` defaults to 10 and is in miles unless `isMetric: true` (https://fakerjs.dev/api/location). Its distribution inside the radius was not checked (unverified).
- Approach: a short hand-written list of cluster centres (for example the 42 Paris campus and a few large French cities, each with its city name), assign each profile a centre with weights (heavier on the defense city), then draw coordinates with `nearbyGPSCoordinate({ origin, radius: 5 to 15, isMetric: true })`. Store the centre's city name as the approximate location (R-PROF-08), so no reverse geocoding is needed at seed time.
- A heavy cluster around the defense location makes nearby suggestions show first (R-BROW-04), while the other clusters show distance sorting and filtering working (R-BROW-05, R-SRCH-01).

## Recommendation for the decision ticket

1. Use `@faker-js/faker` with `fakerFR`, a fixed seed and a pinned version.
2. Photos: faker `personPortrait` assets (CC0, AI, no real people). Commit the 200 faces at 256 px (about 5 to 6 MB) so the seed works offline and is part of what is evaluated (R-GEN-12), and copy them into the uploads store like user uploads. Give each seeded profile 1 to 5 photos, always with a profile picture (R-PROF-01, R-VIEW-03).
3. Accept face reuse across 500+ profiles, or, if reuse is judged risky at defense, regenerate extra faces from the repo's metadata (about $0.035 per image; check the output licence first).
4. Avoid randomuser.me photos (licence unverified), Unsplash and Pexels (real people as fake dating profiles, attribution or hotlinking) and Generated.Photos (terms forbid it).
5. Cluster locations with weighted city centres and `nearbyGPSCoordinate` in km.
