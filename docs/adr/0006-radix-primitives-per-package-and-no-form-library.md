# Radix primitives per package, no form library

The client's accessible behaviour comes from Radix Primitives, installed one package per primitive (`@radix-ui/react-dialog`, `@radix-ui/react-slider`, ...), and its components are written in `client/src/ui/` on top of them with Tailwind CSS v4 and our tokens. Forms use a homegrown hook calling the validation module of `shared/`, with no form library.

Under the admissibility test of ADR 0001, no library that validates input may enter the dependency tree, and an evaluator reads `package.json`, not our intentions. Radix's own `Form` primitive validates; installing primitives one by one keeps it out, where the unified `radix-ui` package would pull it in. Form libraries all embed validation rules, so they fail the same test.

## Considered Options

- **Base UI** (`@base-ui/react`): actively maintained and shadcn's default, but one package that ships `Field` and `Form` with `validate`, present even when unused.
- **shadcn/ui CLI**: its Radix flavour imports the unified `radix-ui` package and adds `class-variance-authority` and `tailwind-merge`; its `form` component depends on react-hook-form and zod.
- **React Aria Components**: validation props are part of every field's API; no Avatar, Toast still unstable.
- **Headless UI**: no Toast, Slider, Tooltip or Avatar.
- **react-hook-form, Formik, TanStack Form**: embed validation rules.
