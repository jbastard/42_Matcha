# UI stack

Libraries, styling, design tokens and the zero-warning guarantee for the React SPA of ADR 0002 (R-GEN-01, R-GEN-03, R-GEN-08). Component primitives and forms follow ADR 0006; every runtime library passes the admissibility test of ADR 0001 and is listed in `docs/dependencies.md`. Decided in issue 23.

## Styling

- Tailwind CSS v4, configured in CSS (`@theme`), with no JavaScript config file.
- No CSS-in-JS. Class names are composed with `clsx` only (no `class-variance-authority`, no `tailwind-merge`).

## Design tokens

- All tokens are CSS custom properties in one file, `client/src/styles/tokens.css`: colour, typography, spacing, radius, shadow, motion duration and easing.
- Two layers:
  - **Primitives**: raw values named by what they are (`--green-700`, `--coral-500`, `--radius-lg`).
  - **Semantic**: named by role (`--color-primary`, `--color-accent`, `--color-danger`, `--color-surface`, `--color-text`), each pointing at a primitive.
- Components use semantic tokens only, never primitives or literal values.
- The dark theme redefines semantic tokens only, inside `@media (prefers-color-scheme: dark)`. No `.dark` class, no JavaScript, no stored preference (`docs/specs/visual-identity.md`).
- Tailwind's `@theme` exposes the semantic tokens as utilities, so a utility class always resolves to a token.

## Components

- Accessible behaviour (focus, keyboard, ARIA) comes from Radix Primitives, installed one package per primitive (`@radix-ui/react-dialog`, `@radix-ui/react-slider`, ...). The unified `radix-ui` package and `@radix-ui/react-form` are never installed.
- Expected primitives: Dialog, AlertDialog, Popover, DropdownMenu, Tabs, Toast, Slider (two thumbs for ranges), Select, Checkbox, Switch, Tooltip, Avatar, VisuallyHidden. Each one added is justified in `docs/dependencies.md`.
- The project's own components live in `client/src/ui/`, written for this project on top of those primitives and the tokens. The shadcn/ui CLI is not used; its code may serve as reference only.
- Every Dialog and AlertDialog has a Title, wrapped in VisuallyHidden when it must not show.

## Forms

- No form library (react-hook-form, Formik, TanStack Form all embed validation rules).
- A homegrown `useForm` hook in `client/` holds field values, touched state and errors, and calls the validation module of `shared/` (ADR 0002). The server revalidates every input (R-SEC-05) and its field errors are shown in the same places as client errors.
- Every `<form>` has `noValidate`, so native browser bubbles never duplicate the application's messages.

## Fonts and icons

- Fraunces and Inter come from `@fontsource-variable/fraunces` and `@fontsource-variable/inter`, bundled by Vite and served with the application; no request reaches a third-party font service.
- Icons come from `lucide-react`, imported one by one so unused icons are dropped from the bundle.

## Zero console warnings

R-GEN-01 forbids any error, warning or notice in the browser console, on the latest Firefox and Chrome (R-GEN-08). Four mechanisms enforce it:

1. `tsc` in strict mode and ESLint (with `eslint-plugin-react-hooks` and `eslint-plugin-jsx-a11y`) run with `--max-warnings 0` and block CI.
2. The application renders inside `<StrictMode>` in development, so React warnings surface early.
3. A Playwright smoke test runs against the production build on Firefox and Chromium, visits every screen, and fails on any console message of type `warning` or `error` and on any `pageerror`.
4. The same test catches failed asset requests (favicon, fonts, source maps), since a 404 shows in the console.

The wider testing strategy is specified elsewhere.
