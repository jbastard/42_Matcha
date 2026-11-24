# Research: safe image upload validation in Node

Ticket: #8. Rules in scope: R-SEC-03 (no upload of unauthorized content), R-SEC-05 (every file upload is validated), R-PROF-01 (up to 5 photos, one of them the profile picture). Also relevant: R-GEN-01 (zero errors), R-GEN-04 (anything not explicitly allowed is forbidden), R-GEN-11 (credentials in `.env`), R-GEN-12 (only the Git repo is evaluated).

Sources checked on 2026-11-24. npm versions from `npm view <pkg> version engines.node` the same day.

## Answer in short

Treat every upload as hostile bytes. Cap the request size at the parser, sniff magic bytes against a 3-format allowlist (JPEG, PNG, WebP), then decode and re-encode with sharp under pixel limits so the stored file is a fresh image with no metadata. Store it on local disk outside any static directory under a server-generated name, and serve it through an app route with a fixed `Content-Type` and `X-Content-Type-Options: nosniff`. Keep sharp current: its bundled decoders have had high-severity CVEs.

## Findings

### 1. What OWASP asks for

From the OWASP File Upload Cheat Sheet (https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html):

- Allowlist extensions; blocklists are weak on their own.
- "don't trust the Content-Type header as it can be spoofed".
- File signature checks "should not be used on its own".
- For images: "decode and re-encode to an allowed image format". The sheet also warns that rewriting does not guarantee removal of all malicious content, so the image library must be kept up to date and run with resource limits.
- Generate the stored filename server-side (UUID); do not reuse the user's filename.
- "Set a file size limit."
- "Store the files on a different server. If that's not possible, store them outside of the webroot."
- Serve through "a handler that gets mapped to filenames inside the application".
- "Only allow authorized users to upload files." and "Protect the file upload from CSRF attacks."
- Least-privilege file permissions.

Not covered by the sheet (verified absent): SVG, `Content-Disposition`, `X-Content-Type-Options`, image pixel-dimension bombs. Those points below come from other sources.

### 2. Magic-byte sniffing: `file-type`

https://github.com/sindresorhus/file-type

- Detects binary formats by magic number via `fileTypeFromBuffer`, `fileTypeFromFile`, `fileTypeFromStream`; returns `{ext, mime}` or `undefined`.
- Explicitly a hint, not a guarantee: "File type detection is based on binary signatures (magic numbers) and is a best-effort hint." and "It does not guarantee the file is actually of that type or that the file is valid/not malformed."
- Does not detect text formats such as SVG.
- ESM-only. Current version 22.3.0 requires Node >= 22 (`npm view`).

Assessment: useful as a cheap early reject, but redundant if sharp is restricted to an allowlist of decoders (sharp's `metadata().format` already reports the decoder that parsed the header, see section 3). Either is acceptable; neither replaces re-encoding.

### 3. Decode, check and re-encode: `sharp`

Current version 0.35.5, Node >= 20.9.0 (`npm view`).

Constructor limits (https://sharp.pixelplumbing.com/api-constructor):

- `limitInputPixels`, default `268402689` (about 16384 x 16384). Lower it for profile photos (for example 40 megapixels) to cap memory per decode.
- `failOn`, default `'warning'`: "Use the default 'warning' level with untrusted input."
- `unlimited`, default `false`; leaving it false keeps "safety features that help prevent memory exhaustion".
- `pages` default `1` and `animated` default `false`: only the first frame of GIF/WebP/TIFF is read.
- `limitInputChannels`, default `5`.

Header-only inspection (https://sharp.pixelplumbing.com/api-input): `metadata()` gives "Fast access to (uncached) image metadata without decoding any compressed pixel data", including `format`, `width`, `height`, `pages`. Note that `width`/`height` ignore EXIF orientation.

Metadata stripping (https://sharp.pixelplumbing.com/api-output): "By default all metadata will be removed, which includes EXIF-based orientation." So GPS and camera EXIF disappear on re-encode unless `keepMetadata`/`withMetadata`/`keepExif` is called. Because orientation is also dropped, the pipeline must auto-orient before output (`autoOrient: true` constructor option, https://sharp.pixelplumbing.com/api-constructor) or photos will appear rotated.

Decoder allowlist (https://sharp.pixelplumbing.com/api-utility): `sharp.block({ operation: [...] })` / `sharp.unblock(...)` (since 0.32.4) restrict libvips operations at runtime, which lets us block every loader except JPEG, PNG and WebP. The page also mentions a `VIPS_BLOCK_UNTRUSTED` environment variable; its exact operation list is not documented on that page (not verified).

### 4. Known CVEs in the image stack

From https://github.com/lovell/sharp/security/advisories:

- GHSA-54xq-cgqr-rpm3 / CVE-2023-4863 (libwebp heap overflow, the one exploited in browsers): sharp < 0.32.6 affected, fixed in 0.32.6. "Almost anyone processing untrusted input with versions of sharp prior to 0.32.6" is affected. https://github.com/lovell/sharp/security/advisories/GHSA-54xq-cgqr-rpm3
- GHSA-f88m-g3jw-g9cj, five libvips CVEs (CVE-2026-33327, -33328, -35590, -35591, -69242): sharp < 0.35.0 affected, page recommends 0.35.3+ (libvips 8.18.3). Workaround blocks the GIF, TIFF and VIPS loaders. https://github.com/lovell/sharp/security/advisories/GHSA-f88m-g3jw-g9cj
- Also listed: librsvg (GHSA-wq5f-xc86-pv6w, 2026-09-30) and libheif (GHSA-rgj7-g3m4-5g8c, 2026-08-27) advisories. Their affected versions were not opened (not verified).

Takeaways: decoder bugs are the realistic attack surface, the SVG (librsvg) and HEIF loaders have their own advisories, and blocking unused loaders is the vendor's own mitigation. Pin sharp to a current release and track advisories (Dependabot or `npm audit`).

### 5. Parsing the multipart request

multer README (https://github.com/expressjs/multer), current version 2.5.0:

- `limits.fileSize`, `files`, `parts` and `fields` all default to `Infinity`; they must be set explicitly.
- `memoryStorage` can "run out of memory" with large or many uploads; acceptable only with a small `fileSize` limit.
- "Never add multer as a global middleware since a malicious user could upload files to a route that you didn't anticipate."
- `fileFilter` sees the client-declared `mimetype` only, so it is a fast pre-filter, not validation (consistent with the OWASP note on Content-Type).

If the chosen micro-framework is Fastify, `@fastify/multipart` (10.1.9) plays the same role; its limits were not reviewed here (not verified). Either library is a parser, not a validator, so R-GEN-02 is not at stake; final say belongs to the framework decision ticket.

### 6. Storage: local disk vs object storage

- OWASP: different server, or else outside the webroot, server-generated names, least-privilege permissions (section 1).
- Object storage (S3-compatible) gives the "different server" property but adds credentials to manage in `.env` (R-GEN-11) and an external service the subject does not explicitly allow (R-GEN-04). Whether evaluators accept it is not verified; it is a decision, not a fact.
- Local disk (a Docker volume) outside any statically served directory, with UUID filenames and the photo row in the database holding the mapping, satisfies OWASP's fallback with no extra moving parts.

### 7. Serving uploads

- Serve through an application route that looks the ID up in the database (OWASP handler mapping) and enforces the same visibility rules as the profile (blocked users, R-VIEW-09).
- Set `Content-Type` from the format we re-encoded to, never from the upload, and `X-Content-Type-Options: nosniff`, which makes the browser trust the declared type and blocks script/style loads of the wrong MIME type (https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Content-Type-Options).
- Never accept SVG: it is a text format that carries script (OWASP lists stored XSS among the upload risks), `file-type` cannot detect it, and librsvg has its own advisory.
- `Content-Disposition` and a separate cookie-less domain are common hardening steps but not sourced here (not verified as required; re-encoding plus nosniff already removes the polyglot risk).

## Recommendation for the decision ticket

1. Upload route only (no global parser), authenticated, CSRF-protected, max 5 photos per user enforced server-side (R-PROF-01).
2. Parser limits: `fileSize` around 5 to 10 MB, `files: 1`, small `fields`/`parts`.
3. Allowlist JPEG, PNG, WebP. Block every other libvips loader with `sharp.block`/`unblock` at startup; optionally pre-check with `file-type`.
4. `sharp(buf, { limitInputPixels: ~40e6, failOn: 'warning', autoOrient: true })`, check `metadata()` (format in allowlist, minimum dimensions), resize to a max edge (for example 1600 px), re-encode to one output format (WebP or JPEG). Metadata is stripped by default. Any sharp error = reject with a clean 4xx, never a 500 (R-GEN-01).
5. Store the re-encoded output only, under a UUID name, on a volume outside the static root; never store the original.
6. Serve via an app route with a fixed `Content-Type`, `nosniff`, and the profile's access rules.
7. Keep sharp >= 0.35.3 and watch its advisories page.

Open points: object storage acceptability at the defense (R-GEN-04) and the exact `@fastify/multipart` limits if Fastify is chosen.
