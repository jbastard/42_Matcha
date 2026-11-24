# Subject rules

Immutable rules imposed by the Matcha subject (`fr.subject.matcha.pdf`, version 6.0). Every design decision, spec, ticket and review respects them and cites them by ID. A rule changes only when the subject changes.

The subject reference (e.g. `III`, `IV.2`) points to the source chapter for the exact French wording.

## General (III, VI.1)

- **R-GEN-01**: Zero errors, warnings or notices, server-side and client-side (browser console).
- **R-GEN-02**: Any programming language. Frameworks are limited to micro-frameworks: a router plus optional templating, with no ORM, no validators and no user account manager. This definition is the one applied at the defense. Silex is explicitly forbidden (Doctrine integration).
- **R-GEN-03**: Any library needed for the project is allowed, including UI libraries (React, Angular, Vue, Bootstrap, Semantic, or any combination), as long as R-GEN-02 holds.
- **R-GEN-04**: Anything not explicitly allowed is forbidden.
- **R-GEN-05**: Free relational or graph database (MySQL, MariaDB, PostgreSQL, Cassandra, InfluxDB, Neo4j, etc.). Queries are written by hand; a homegrown library to simplify query handling is allowed.
- **R-GEN-06**: The database holds at least 500 distinct profiles for the evaluation.
- **R-GEN-07**: Web server of our choice (Apache, Nginx, built-in server).
- **R-GEN-08**: The whole application works on at least the latest Firefox and Chrome.
- **R-GEN-09**: Every page has at least a header, a main section and a footer.
- **R-GEN-10**: Mobile-friendly, with an acceptable layout on small screens.
- **R-GEN-11**: Credentials, API keys and environment variables live in a local `.env` file excluded from Git. Public credentials can fail the project.
- **R-GEN-12**: Only what is in the Git repository is evaluated.

## Security (III, VI.1)

Any security flaw scores 0.

- **R-SEC-01**: Passwords are never stored in plain text.
- **R-SEC-02**: No HTML or JavaScript injection through unprotected variables.
- **R-SEC-03**: No upload of unauthorized content.
- **R-SEC-04**: No SQL injection.
- **R-SEC-05**: Every form input and every file upload is validated.

## Registration and login (IV.1)

- **R-AUTH-01**: Registration asks at least for email, username, last name, first name and a secure password.
- **R-AUTH-02**: Commonly used English words are rejected as passwords.
- **R-AUTH-03**: After registration, the user receives an email with a unique link to verify the account.
- **R-AUTH-04**: Login uses username and password.
- **R-AUTH-05**: A user who forgot their password can request a reset email.
- **R-AUTH-06**: Logout takes one click from any page.

## User profile (IV.2)

- **R-PROF-01**: Once logged in, the user completes their profile: gender, sexual preferences, biography, interest tags (e.g. `#vegan`, `#geek`), and up to 5 photos, one of them the profile picture.
- **R-PROF-02**: Tags are reusable.
- **R-PROF-03**: The user can edit the profile information at any time, as well as last name, first name and email.
- **R-PROF-04**: The user can see who viewed their profile.
- **R-PROF-05**: The user can see who liked them.
- **R-PROF-06**: Every user has a public popularity score. Its definition is ours, with consistent criteria.
- **R-PROF-07**: The user is located by GPS down to the neighbourhood, only with explicit consent.
- **R-PROF-08**: A user who refuses GPS tracking must enter an approximate location (city or neighbourhood) to use matching.
- **R-PROF-09**: The user can change their location in their profile at any time.

## Browsing (IV.3)

- **R-BROW-01**: The user easily reaches a list of suggested profiles matching their preferences.
- **R-BROW-02**: Suggestions respect orientation: a heterosexual woman sees only male profiles. Bisexuality is handled. A user with no stated orientation is treated as bisexual.
- **R-BROW-03**: Matching weighs several criteria, at least: proximity to the user's location, number of shared tags, popularity score.
- **R-BROW-04**: Users in the same geographic area get priority.
- **R-BROW-05**: The suggested list is sortable by age, location, popularity score and common tags.
- **R-BROW-06**: The suggested list is filterable by age, location, popularity score and common tags.

## Search (IV.4)

- **R-SRCH-01**: Advanced search on one or more criteria: age range, popularity score range, location, one or more interest tags.
- **R-SRCH-02**: Search results are sortable and filterable by age, location, popularity score and interest tags.

## Profile view (IV.5)

- **R-VIEW-01**: A user can view other users' profiles, showing all available information except email and password.
- **R-VIEW-02**: Viewing a profile is recorded in the visit history.
- **R-VIEW-03**: A user can like another user's profile picture, but only if they have a profile picture themselves.
- **R-VIEW-04**: Two users who liked each other are connected and can chat.
- **R-VIEW-05**: A user can remove a like. That user's further notifications stop and the chat between them is disabled.
- **R-VIEW-06**: A user can see another user's popularity score.
- **R-VIEW-07**: A user can see whether another user is online, and otherwise the date and time of their last connection.
- **R-VIEW-08**: A user can report another user as a fake account.
- **R-VIEW-09**: A user can block another user. The blocked user no longer appears in search results, generates no notifications, and cannot chat.
- **R-VIEW-10**: The profile view clearly shows whether that profile liked the current user and whether they are connected, with the option to unlike or disconnect.

## Chat (IV.6)

- **R-CHAT-01**: Connected users chat in real time, with at most 10 seconds of delay. The implementation is ours.
- **R-CHAT-02**: From any page, the user sees when they receive a new message.

## Notifications (IV.7)

- **R-NOTI-01**: Real-time notifications, with at most 10 seconds of delay, when the user: receives a like, has their profile viewed, receives a message, gets a like back from a user they liked, is unliked by a connected user.
- **R-NOTI-02**: From any page, the user sees when they have unread notifications.

## Bonus (V)

- **R-BONUS-01**: Bonus features are evaluated only if the mandatory part is perfect: fully implemented and working without any malfunction.

## Left open by the subject

Not rules: design decisions to make and record elsewhere (ADRs, specs).

- Language, micro-framework, database and web server (within R-GEN-02 to R-GEN-07).
- Definition of the popularity score (R-PROF-06).
- Matching weights and what makes a profile "interesting" (R-BROW-03).
- Real-time transport for chat and notifications (R-CHAT-01, R-NOTI-01).
- The word replacing "like": the subject encourages a more explicit alternative.
- Which bonus features to build, if any.
