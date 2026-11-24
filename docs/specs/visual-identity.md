# Visual identity and tone

The name, voice and visual direction of the product. This document fixes intent and constraints; the design agent turns it into tokens, components and screens (see the design system brief). Font and icon names are references the design agent may adjust if it keeps the stated character. Decided in issue 22.

## Name and language

- The product is called **Matcha**: the name in the logo, the page titles and the emails.
- The interface is in English only, with no internationalisation layer. Copy uses the vocabulary of `GLOSSARY.md` (Crush, Connection, Suggestion, Profile Picture, ...), never the terms it lists under _Avoid_.

## Voice

Warm and light: short sentences, addressing the user directly, a touch of humour in empty states and confirmations.

The humour stops where trust starts: errors, security (login, password, email verification), blocking and fake-account reports use a plain, calm and precise tone.

| Situation | Do | Don't |
| --- | --- | --- |
| Empty crushes list | "No crushes yet. Go say hi." | "You have 0 crushes." |
| New connection | "It's mutual! You can now chat with Lea." | "Match found." |
| Wrong password | "Wrong username or password." | "Oops, nice try!" |
| Block confirmation | "You and Lea won't see each other anymore." | "Bye bye Lea!" |
| Unverified email | "Check your inbox to verify your email, then log in." | "Almost there, lovebird!" |

## Colour intent

- **Primary**: a deep matcha green, for the brand, primary actions and links.
- **Accent**: a warm coral, reserved for what carries romantic meaning: Crush, Connection, and the notification and unread-message badges.
- **Neutrals**: slightly warm greys and off-whites, never pure black on pure white.
- **Feedback colours** (error, warning, success, info) are distinct from the accent: an error is never coral.
- Every text and background pair meets WCAG 2.2 AA contrast (4.5:1 for body text, 3:1 for large text and UI components), in both themes.

## Typography

- **Headings**: Fraunces, an expressive variable serif.
- **Body and UI**: Inter.
- Both fonts are self-hosted with the application; no request goes to a third-party font service.

## Imagery and icons

- The only photographs are profile photos.
- Icons come from one consistent line icon set (Lucide as reference). No illustrations: empty states and onboarding use an icon plus copy.

## Themes

- Light and dark themes, following the operating system preference (`prefers-color-scheme`). No manual toggle, no stored preference.
- Every colour token is defined for both themes from the start.

## Shape and density

- Generous rounding: about 16px on cards, pill-shaped buttons.
- Airy spacing and very light shadows.
- Profile cards are photo-first and must read well on small screens (R-GEN-10).

## Motion

- Discreet transitions, 150 to 250 ms.
- One signature micro-animation when a Crush is sent and when a new Connection appears.
- All animation is disabled under `prefers-reduced-motion`.

## Logo

- A wordmark: "matcha" in lowercase Fraunces, primary green, with no pictogram.
- The favicon is the letter "m" on a primary green background.

## Inclusivity

Gender is man or woman and Sexual Preference is a set of genders (`GLOSSARY.md`, R-BROW-02). The interface presents them factually and neutrally:

- No gendered colour coding (no blue for men, pink for women) and no gender symbols.
- The preference field is labelled "Interested in", never "Orientation"; a preference holding both genders reads "Everyone".
- The binary model is a constraint of the subject, not a topic of the interface: no explanatory or apologetic copy about it.
- Seed photos vary in age and origin (R-GEN-06).
