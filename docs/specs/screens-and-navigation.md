# Screens and navigation

Every screen of the SPA, its route, who may open it, the states it must render, and the navigation shell around it (R-GEN-09, R-GEN-10, R-CHAT-02, R-NOTI-02, R-AUTH-06). Routing follows ADR 0002; vocabulary follows `GLOSSARY.md` and voice follows `docs/specs/visual-identity.md`. Decided in issue 24; the prototype that settled it lives on the `prototype/navigation-shell` branch.

## Page skeleton

Every screen renders inside the same skeleton: `<header>`, `<main>`, `<footer>` (R-GEN-09). No screen, including errors, redirects and email-link pages, renders without them.

- **Header**: depends on the visitor's state (see below). Sticky at the top.
- **Main**: the screen itself, centred, at most about 1040px wide.
- **Footer**: the wordmark and static links (About, Privacy, Terms). It sits at the end of the content, not fixed; on phones it scrolls above the tab bar.

## Shell

Two layouts, switched at the 768px breakpoint (Tailwind `md`).

### Wide (768px and up)

Header, left to right: wordmark, text links Discover / Search / Activity, then icon buttons Chat (with badge), Notifications (with badge), Profile Picture (opens My profile), Settings, and a labelled Log out button.

### Narrow (below 768px)

- Header: wordmark, Notifications (with badge), Log out icon button.
- A bottom tab bar, fixed, with five destinations: Discover, Search, Activity, Chat (with badge), Profile.
- Settings is reached from My profile.

### Header by visitor state

| Visitor | Header contents |
|---|---|
| Logged out | Wordmark, Log in, Sign up |
| Logged in, profile not complete | Wordmark, Settings, Log out. No badges, no browse links, no tab bar: a user without a Complete Profile neither browses nor appears to others |
| Logged in, Complete Profile | The full shell above |

The wordmark links to Landing when logged out, Onboarding when the profile is not complete, Discover otherwise.

## Global indicators

- **Chat badge**: number of visible conversations holding an unread message (R-CHAT-02). On the Chat link in the header (wide) or the Chat tab (narrow).
- **Notification badge**: number of unread notifications (R-NOTI-02). On the Notifications button, always in the header.
- Both are driven by the initial fetch and the `counts` event (`docs/specs/chat-and-notifications.md`). They use the accent colour and are hidden at zero.
- **Notifications button**: on wide screens it opens a popover listing the latest notifications with a "See all" link to `/notifications`; on narrow screens it navigates to `/notifications`. Opening either marks them read.
- A live `notification` event also shows a toast, except a `message` whose conversation is open.

## Logout

Log out is a button in the header on every logged-in screen, on every breakpoint, one click with no confirmation and never inside a menu (R-AUTH-06). It calls `POST /api/auth/logout` and lands on Log in.

## Access and redirects

Each route has one access level, enforced by the router before rendering (and by the API independently):

| Access | Who may open it | Otherwise |
|---|---|---|
| public | logged-out visitors only | a logged-in user goes to Discover, or Onboarding if the profile is not complete |
| auth | any logged-in user | a logged-out visitor goes to Log in, then back to the requested route after login |
| complete | logged-in users with a Complete Profile | logged out: Log in; not complete: Onboarding |
| any | everyone | |

## Screen inventory

| Screen | Route | Access | States | Rules |
|---|---|---|---|---|
| Landing | `/` | public | default | |
| Sign up | `/register` | public | form, field errors, link sent ("Check your inbox") | R-AUTH-01, R-AUTH-02 |
| Log in | `/login` | public | form, wrong credentials, unverified (with resend), rate limited | R-AUTH-04 |
| Email verification | `/verify?token=` | any | verifying, verified, invalid or expired | R-AUTH-03 |
| Forgot password | `/forgot-password` | public | form, sent (same message whether or not the email exists) | R-AUTH-05 |
| Reset password | `/reset-password?token=` | any | form, done, invalid or expired | R-AUTH-05 |
| Onboarding | `/onboarding` | auth | step 1 about you (birth date, gender, Interested in, bio), step 2 Interest Tags, step 3 photos and Profile Picture, step 4 location (consent asked, or refused then entered by hand) | R-PROF-01, R-PROF-07, R-PROF-08 |
| Discover | `/discover` | complete | suggestions, loading, empty (filters too narrow), error | R-BROW-01..06 |
| Search | `/search` | complete | criteria, results, loading, no results, error | R-SRCH-01, R-SRCH-02 |
| Profile view | `/profiles/:id` | complete | no relation, crushed you, you crushed, connected, viewer has no Profile Picture (Crush disabled with a link to add one), unavailable, loading | R-VIEW-01..10 |
| Activity | `/activity` | complete | Visitors tab, Crushes received tab, empty, loading | R-PROF-04, R-PROF-05 |
| Conversations | `/chat` | complete | list, empty, loading, error | R-CHAT-01 |
| Conversation | `/chat/:id` | complete | open, closed (connection broken: input disabled), loading | R-CHAT-01, R-VIEW-05 |
| Notifications | `/notifications` | complete | list, empty, loading | R-NOTI-01, R-NOTI-02 |
| My profile | `/me` | auth | form, saving, field errors; shows own Popularity Score, photos, location with "Use my current location" | R-PROF-01, R-PROF-03, R-PROF-06, R-PROF-09 |
| Settings | `/settings` | auth | account (first name, last name, email; username read-only), password change | R-PROF-03 |
| Not found | `*` | any | default | |

Profile ids in routes are the user's `uuidv7` (ADR 0003). "Unavailable" covers a missing user, a block or fake report in either direction, and an incompatible or incomplete profile: the screen never says which.

Bonus screens and additions (`docs/specs/bonus-overview.md`) are specified with each bonus: a Map destination, Meetups inside Conversation, the gallery inside My profile, OAuth buttons on Log in and Sign up, linked accounts in Settings.

## State conventions

- **Loading**: skeletons shaped like the content, no full-page spinner.
- **Empty**: an icon and one warm sentence with a next action (`docs/specs/visual-identity.md`).
- **Error**: a plain sentence and a "Try again" button; field errors sit under their field.
- **Redirect**: silent, except the Onboarding gate, which states why ("Complete your profile to browse, search and chat").
