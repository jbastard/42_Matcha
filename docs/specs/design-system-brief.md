# Design system brief

The design agent's work order. It builds the design system, the navigation shell and every screen of Matcha, mandatory and bonus, as production code in the repository, working alone from this brief and the documents it points to. It decides nothing those documents already fix; where they leave room, it chooses and records the choice in its pull request. Decided in issue 25.

## Read first

| Document | What it fixes |
|---|---|
| `docs/specs/visual-identity.md` | Name, voice, colour intent, typography, icons, themes, shape, motion, logo, inclusivity |
| `docs/specs/ui-stack.md` | Tailwind v4, two-layer tokens in `tokens.css`, Radix one package per primitive, homegrown `useForm`, fonts and icons packages, zero console warnings |
| `docs/specs/screens-and-navigation.md` | Page skeleton, shell at 768px, header by visitor state, badges, every screen with its route and states, state conventions |
| `docs/specs/chat-and-notifications.md` | Notification types and wording, unread and collapsed notifications, closed conversations, presence |
| `docs/specs/bonus-overview.md` | What each bonus does: map, Meetups, gallery, OAuth, calls |
| `GLOSSARY.md` | The only vocabulary allowed in copy and in component and prop names |
| `docs/adr/0001-*`, `0002-*`, `0006-*` | Library admissibility test, monorepo layout and client stack, Radix per package and no form library |
| `docs/subject-rules.md` | R-GEN-01, R-GEN-08, R-GEN-09, R-GEN-10, R-BONUS-01 |

## Deliverable

Production code, not mock-ups: the files the application will ship.

- `client/src/styles/tokens.css`: every token, both themes.
- `client/src/ui/`: the components below.
- `client/src/screens/`: every screen, as a component that receives all its data and callbacks through props. It fetches nothing and holds no server state, so wiring the API later never changes its rendering.
- `client/src/fixtures/`: the fake data the showcase feeds to screens. Names, ages and places vary; text uses the glossary; photos are a handful of the seed portraits (issue 5), committed under `client/src/fixtures/photos/`.
- The `/_design` showcase (below).

### Scaffold

The repository holds no code yet. The agent creates the npm-workspaces root and the `client/` workspace only, as ADR 0002 lays it out: Vite, React, TypeScript strict, React Router (library mode), Tailwind v4, ESLint with `eslint-plugin-react-hooks` and `eslint-plugin-jsx-a11y`, Playwright. No `server/`, no `shared/`, no TanStack Query, no API client, no business routes. `docs/dependencies.md` is created with the first dependency, and every dependency added gets a line of justification against the admissibility test of ADR 0001.

## Showcase

A route `/_design`, mounted only in development and absent from the production bundle (a test builds for production and asserts the route and fixtures are not in it).

- An index of tokens (colour swatches per theme with their contrast ratios, type scale, spacing, radius, shadow, motion), components, and screens.
- Each component on its own page with all its variants and states.
- Each screen on its own page with a state picker covering every state listed for it in `screens-and-navigation.md`, rendered inside the real shell for the matching visitor state.
- A separate **Bonus** section for the bonus screens and additions.

No Storybook.

## Scope

### Shell

Page skeleton (`header`, `main`, `footer`), wide header, narrow header and bottom tab bar, the three header variants by visitor state, Chat and Notification badges, the notifications popover (wide), toasts, the Onboarding gate message. As in `screens-and-navigation.md`.

### Components

The minimum list; the agent adds what the screens need and removes nothing. Each is built on the Radix primitive named in `ui-stack.md` where one applies.

- **Actions**: Button (primary, secondary, ghost, danger, neutral for OAuth providers; sizes; icon-only with accessible label; loading), Link.
- **Forms**: TextField, TextArea with counter, PasswordField with show/hide, DateField (Birth Date), Select, Checkbox, Switch, RadioGroup (Gender), gender set picker ("Interested in", "Everyone"), Interest Tag input with suggestions and chips, RangeSlider (two thumbs: age, distance, Popularity Score), Field wrapper with label, hint and error, form-level error banner.
- **Profile**: Avatar (circle, Profile Picture or initials on a neutral background, Online dot), Profile Card (photo-first, 4:5 crop, name, age, distance band, Common Tags), photo carousel for Profile view, photo manager (up to 5, choose Profile Picture, delete, upload), Popularity Score display, Online / Last Seen indicator, Interest Tag chip (common tags highlighted).
- **Relations**: Crush button with its states (send, sent, crushed you, connected, disabled without Profile Picture) and the signature animation, Connection banner and its animation, Block and Fake Report actions with their confirmation dialogs.
- **Activity and chat**: list rows for Visitors and Crushes received, Conversation list row (Avatar, last Message, unread mark), Message bubble (mine, theirs, time), Message composer (disabled when the conversation is closed), Notification row (each type of `chat-and-notifications.md`, unread, collapsed count).
- **Overlays and feedback**: Dialog, AlertDialog, Popover, DropdownMenu, Tooltip, Toast, Tabs.
- **States**: Skeletons shaped per content, Empty state (icon and one sentence and an action), Error state (sentence and "Try again"), inline alert.
- **Brand**: wordmark, favicon (the "m" on primary green), in every size the shell and the browser need.

### Screens

All 17 screens of `screens-and-navigation.md`, each in every listed state.

### Bonus

Visual proposals; each bonus spec may correct them before it is built (R-BONUS-01).

- **Map**: the Map destination in both shells, Leaflet markers and clusters styled with the tokens (accent for people), a profile preview on marker click, the OSM attribution, empty and loading states. A static background or live OSM tiles in development are both acceptable.
- **Meetups**: proposal dialog (time, place, note) from Conversation, the Meetup card in the conversation in each state (proposed, accepted, declined, cancelled, cancelled because the Connection broke), Meetup notification rows.
- **Gallery**: the photo manager upgraded with drag-and-drop, crop, rotate and filters, the 5-photo cap visible.
- **OAuth**: 42 and GitHub buttons on Log in and Sign up, the username confirmation step, linked accounts in Settings (link, unlink, the password-less case).
- **Calls**: incoming call, outgoing call, call in progress (audio and video, mute, camera off, hang up), call ended and failed.

## Constraints

### Tokens

Exactly the two layers of `ui-stack.md`. Components and screens use semantic tokens only: no primitive token, hex value, pixel value or arbitrary Tailwind value (`[...]`) outside `tokens.css`. The accent is reserved for Crush, Connection, badges and map markers of people (`visual-identity.md`).

### Accessibility

- WCAG 2.2 level AA in both themes.
- Visible focus ring on every interactive element, distinct in both themes.
- Every action reachable and operable by keyboard; Radix behaviour is kept, never overridden.
- Touch targets at least 44 by 44px below 768px, at least 24 by 24px above.
- Every icon-only control has an accessible name; photos have alternative text naming the person.
- Motion off under `prefers-reduced-motion`, signature animations included.

### Responsive and browsers

- Layout works from 320px wide, with no horizontal scroll.
- Verified at 320, 375, 768 and 1280px.
- Latest Firefox and Chromium (R-GEN-08). Safari is neither tested nor specifically handled.

### Photos and avatars

Profile photos are cropped to 4:5 with `object-fit: cover` in cards, carousel and gallery; avatars are circles. Without a Profile Picture, initials on a neutral background, never a silhouette.

### Copy

All visible text follows the voice of `visual-identity.md` and the vocabulary of `GLOSSARY.md`, never a term listed under _Avoid_. The agent writes the copy for every state; notification wording follows `chat-and-notifications.md`.

## Acceptance criteria

A batch is accepted when all of these hold:

1. A script checks every text/background and UI/background semantic token pair against WCAG AA (4.5:1 body text, 3:1 large text and UI components) in both themes, and fails CI on any miss.
2. Playwright visits every `/_design` page and every screen state, on Firefox and Chromium, at 320, 375, 768 and 1280px, in both colour schemes, and fails on any console message of type `warning` or `error`, any `pageerror` and any failed request.
3. The same run checks each page with `@axe-core/playwright` and fails on any violation.
4. `tsc` strict and ESLint pass with `--max-warnings 0`.
5. A lint rule or script fails on primitive tokens, literal colours, literal sizes or arbitrary Tailwind values outside `tokens.css`.
6. The same Playwright run saves screenshots of every page, viewport and scheme for review; they are not committed as reference snapshots. The dev reviews them and approves.

## Delivery

Work happens on the `design-system` branch, delivered as four successive pull requests, each merged only after the dev's review before the next one opens:

1. Scaffold and tokens, with the contrast script and the showcase index.
2. Components and shell.
3. Screens.
4. Bonus.
