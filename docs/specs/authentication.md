# Authentication

Registration, email verification, login, password reset and logout (R-AUTH-01..06, R-SEC-01, R-SEC-05). Sessions and CSRF defence follow ADR 0004; libraries follow the admissibility test of ADR 0001.

## Identifiers

- **Username**: `[a-z0-9_]{3,20}`, stored lowercase, unique, never changes (R-PROF-03 does not list it as editable). Login uses the username only (R-AUTH-04).
- **Email**: unique, compared case-insensitively. Changing it is part of the account lifecycle, specified elsewhere.

## Registration

Fields: email, username, first name, last name, password; nothing else (R-AUTH-01). Birth date and the rest of the profile come when the user completes their profile.

Each field is validated by hand on the server (R-SEC-05). A taken username or email is reported plainly. Before rejecting a conflict, the server deletes any unverified user created more than 7 days ago that holds the same username or email, so abandoned registrations do not squat identifiers. There is no scheduled purge.

On success the user is created unverified and a verification email is sent (R-AUTH-03). No session is opened.

## Password policy

- 10 to 128 characters, no composition rules (NIST SP 800-63B).
- Rejected when, after normalisation, it exactly matches an entry of the committed common-password list (R-AUTH-02, research in issue 4) or the user's username, first name, last name or email.
- The client shows the rule and a strength meter while typing; the server enforces it.
- Hashed with argon2id through the `argon2` package, OWASP parameters m=19 MiB, t=2, p=1 (R-SEC-01).

## One-time tokens

Email verification and password reset share one mechanism: a random 256-bit token sent in the email link, stored only as its SHA-256 with its purpose, user id and expiry, deleted on use.

| Purpose | Lifetime | On use |
|---|---|---|
| Email verification | 24 h | user becomes verified |
| Password reset | 1 h | password replaced, all the user's sessions deleted, user becomes verified |

Issuing a new token of a purpose deletes the user's previous tokens of that purpose.

Email links open SPA routes (`/verify?token=...`, `/reset-password?token=...`). The page then sends the token to the API in a POST; the API never consumes a token on GET, because mail scanners prefetch links.

## Email verification

- `POST /api/auth/verify` with the token: verifies the user.
- `POST /api/auth/verify/resend` with the username and password: sends a new link if the credentials are correct and the user is unverified.

## Login

`POST /api/auth/login` with username and password.

- Unknown username or wrong password: one generic message, "invalid username or password".
- Correct password, unverified user: "email not verified", with the option to resend the link. This is shown only after a correct password, so it reveals nothing to a third party.
- Success: a new session token is issued (protection against session fixation) and the cookie is set.

There is no account lockout; rate limiting covers brute force without letting an attacker lock a victim out.

## Password reset

- `POST /api/auth/password-reset/request` with an email: always answers the same message, whether or not a user has that email (no enumeration). If one exists, a reset link is sent (R-AUTH-05).
- `POST /api/auth/password-reset` with the token and the new password: applies the password policy, then the effects in the token table above.

## Sessions

Mechanism per ADR 0004: opaque token in the `__Host-sid` cookie, SHA-256 stored in a `session` table.

- Idle expiry: 7 days, sliding on each authenticated request.
- Absolute expiry: 30 days after login.
- No "remember me" option.
- Changing the password deletes every other session of the user.
- The WebSocket upgrade authenticates with the same cookie and checks `Origin`.

## Logout

`POST /api/auth/logout` deletes the session row, clears the cookie and closes the WebSocket connections opened by that session. A logout button sits in the header of every page (R-AUTH-06, R-GEN-09). There is no "log out everywhere" control.

## CSRF

Per ADR 0004, without tokens:

- SameSite=Lax session cookie;
- API bodies accepted only as `application/json`, except multipart on upload routes;
- every non-GET request rejected unless its `Origin` header equals the app's origin.

## Rate limiting

`express-rate-limit` with its in-memory store, enough for a single instance. Every threshold is configurable through environment variables. Over the limit the API answers 429 with `Retry-After`.

| Route | Limits |
|---|---|
| Login | 10 per 15 min per IP + username; 100 per 15 min per IP |
| Registration | 5 per hour per IP |
| Password reset request | 3 per hour per email; 10 per hour per IP |
| Verification resend | 3 per hour per email; 10 per hour per IP |

## Dependencies

To record in `docs/dependencies.md` when they are added (ADR 0001):

- `argon2`: password hashing; no validation, account modelling or SQL.
- `express-rate-limit`: request counting; no validation, account modelling or SQL.

## Later

- OAuth login (bonus): the user model must allow a user without a password. Decided in the bonus selection ticket.
- Email change and account deletion: account lifecycle.
