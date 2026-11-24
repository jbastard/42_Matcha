# Back-end micro-frameworks and front-end tools under R-GEN-02 to R-GEN-04

Research for issue #2. Date: 2026-11-24. Rules cited from `docs/subject-rules.md`.

## Source trust levels

- **Primary, authoritative**: the Matcha subject, `fr.subject.matcha.pdf` version 6.0 (local, git-ignored), chapters III and VI.1. Its footnote says its micro-framework definition "fera autorité lors de la soutenance".
- **Primary, factual**: official framework docs and npm registry metadata (`npm view`, run 2026-11-24).
- **Secondary, weak**: public student repos. They show what students used, not what passed a defense: none of them records its grade, and at least one breaks a rule outright (see below).
- **Not found**: the official Matcha evaluation sheet (scale). It lives on the 42 intranet and no public copy turned up. Searches covered GitHub code search, `truthciao/42_evaluation_sheets` (no Matcha file), and the web. **Everything below about what the evaluation sheet checks is unverified.**

## What the subject actually says (primary)

From chapter III of the subject PDF (`pdftotext -layout fr.subject.matcha.pdf`, lines 85-115):

1. "Vous pouvez utiliser des micro-frameworks et toutes les bibliothèques nécessaires pour ce projet." (R-GEN-02, R-GEN-03)
2. A micro-framework "inclut un routeur et éventuellement du templating, mais qui n'inclut pas d'ORM, de validateurs ou de gestionnaire de comptes utilisateurs". (R-GEN-02)
3. Suggested list: "Express pour Node (oui, nous considérons cela comme un micro-framework)". Express is the only Node framework the subject names. Silex is excluded "en raison de son intégration avec Doctrine". (R-GEN-02)
4. UI libraries "telles que React, Angular, Vue, Bootstrap, Semantic, ou toute combinaison" are allowed. (R-GEN-03)
5. Queries are written by hand. A homegrown query-simplifying library is allowed. (R-GEN-05)

From chapter VI.1: "Tout ce qui n'est pas explicitement autorisé est strictement interdit" (R-GEN-04), plus "Valider toutes les entrées de formulaire et les téléchargements de fichiers" (R-SEC-05).

Gap in `docs/subject-rules.md`: R-GEN-02 does not mention that the subject names Express as a micro-framework, and R-GEN-03 does not repeat "toutes les bibliothèques nécessaires". This note does not change the rules file. The decision ticket can decide whether to update it.

### How to read the definition

The ban targets what the **framework** includes: an ORM, validators, or a user account manager. Taken alone, the text does not forbid a separate library for one of those jobs, since "toutes les bibliothèques nécessaires" are allowed. Three things narrow that reading in practice:

- R-GEN-05 separately requires hand-written queries and allows only a **homegrown** helper. That rules out third-party ORMs **and** query builders, whatever the framework.
- R-GEN-04 is "not explicitly allowed means forbidden". An evaluator can argue that a validation library (Zod, Joi, class-validator) or an auth library (Passport) brings in the very thing the definition excludes.
- The subject says its definition is authoritative "quelles que soient les définitions alternatives trouvées en ligne". The evaluator applies the subject's wording, not marketing labels like "micro" or "minimalist".

Treat these as red lines, not as legal certainty. The evaluation sheet that would settle them could not be checked.

## Back-end frameworks (Node/TypeScript)

| Framework | What core ships (primary source) | Verdict |
|---|---|---|
| **Express 5** (5.3.0) | Self-described "Fast, unopinionated, minimalist web framework for Node.js" and "a lightweight and flexible routing framework with minimal core features" ([expressjs.com](https://expressjs.com/)). npm dependencies: `router`, `body-parser`, `serve-static`, `cookie`, no validator, ORM or auth (`npm view express dependencies`). | **Safe.** Named in the subject itself. |
| **Koa 3** (3.3.0) | "Koa does not bundle any middleware within its core" ([koajs.com](https://koajs.com/)), so there is no router in core. Routing comes from `@koa/router` (15.7.1). | **Safe on the definition**, but not named in the subject. Koa core is not even a router, so the defense argument is "Koa + @koa/router = router". |
| **Hono 4** | Ships, as opt-in subpath exports, `hono/validator` ("Hono provides only a very thin Validator", [hono.dev/docs/guides/validation](https://hono.dev/docs/guides/validation)) and auth helpers `basic-auth`, `bearer-auth`, `jwt`, `jwk`, `csrf` (`npm view hono exports`). | **Risky.** The package literally includes a validator. That the validator is opt-in is a defense argument, not a guarantee. |
| **Fastify 5** (5.12.8) | "Route validation relies on Ajv v8, a high-performance JSON Schema validator" ([fastify.dev validation docs](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/)). `@fastify/ajv-compiler` is a hard dependency of core (`npm view fastify dependencies`). | **Risky.** Core includes a validator, which fails the subject's definition on a literal read. |
| **NestJS** | Built-in `ValidationPipe` (class-validator) and `StandardSchemaValidationPipe` (Zod, Valibot...). Its docs have a "Data" section covering TypeORM, Prisma, Sequelize, MikroORM ([docs.nestjs.com validation](https://docs.nestjs.com/techniques/validation)). | **Fails.** It ships validators and is a full framework, not "a router plus templating". |
| **AdonisJS** | "batteries such as routing, middleware, validation, and ORM out of the box" ([docs.adonisjs.com](https://docs.adonisjs.com/guides/preface/introduction)). | **Fails.** It ships an ORM and validation. |
| **Next.js / Remix (React Router framework mode) as back end** | Next.js Route Handlers "create custom request handlers for a given route" and its docs pitch it as a backend ([nextjs.org route handlers](https://nextjs.org/docs/app/getting-started/route-handlers)). | **Risky.** No ORM, validator or account manager in core, but it is a full-stack framework well beyond "router plus templating". The evaluator would have to accept it as a micro-framework, and the subject gives nothing to lean on. Not checked in Remix's docs. |

## Front-end tools

The subject allows UI libraries and "toutes les bibliothèques nécessaires" (R-GEN-03). It says nothing about build tools or client routers.

- **React**: explicitly named. Safe.
- **Vite**: a build tool/dev server, not a framework, and none of the three excluded features applies. Safe by reasoning. No subject or evaluation-sheet text covers it (unverified).
- **React Router (library/declarative mode), TanStack Router**: client-side routing libraries. Safe by reasoning (unverified).
- **Next.js, Remix, React Router framework mode**: they also run server code (see above), which pulls them under R-GEN-02. Avoid them.

## Add-ons that would disqualify or endanger a stack

| Category | Examples | Status | Basis |
|---|---|---|---|
| ORM / ODM | Prisma, TypeORM, Sequelize, MikroORM, Drizzle ORM, Mongoose | **Forbidden** | R-GEN-02, R-GEN-05 |
| Third-party query builders | Knex, Kysely | **Forbidden (high risk)**: not an ORM, but R-GEN-05 allows only hand-written queries and a homegrown helper | R-GEN-05 |
| Raw drivers | `pg`, `mysql2`, `neo4j-driver` with parameterized SQL | **Allowed** | R-GEN-05, R-SEC-04 |
| Validation libraries | Zod, Joi, Yup, class-validator, express-validator, Ajv | **Avoid**: the subject bans validators in the framework, and R-GEN-04 lets an evaluator extend that. Write validation by hand (R-SEC-05). | R-GEN-02, R-GEN-04 |
| User account managers / auth frameworks | Passport, Auth.js, Lucia, better-auth, Supabase Auth, Firebase Auth | **Forbidden**: "gestionnaire de comptes utilisateurs" | R-GEN-02 |
| Session / token primitives | `express-session`, `cookie-parser`, `jsonwebtoken`, `bcrypt`/`argon2` | **Likely allowed**: they store a session or sign a token, they do not manage accounts. Not verified against the evaluation sheet. | R-GEN-03 |
| Other libraries | `socket.io`/`ws`, `nodemailer`, `multer`, `helmet`, `cors` | **Likely allowed** | R-GEN-03 |

## Secondary evidence: student repos

Dependencies read with `gh api` on 2026-11-24. None of these repos shows its defense outcome.

- [erikpeik/Matcha](https://github.com/erikpeik/Matcha) `server/package.json`: express 4, express-session, pg, bcrypt, multer, nodemailer, socket.io.
- [yeomin4242/42OUTER](https://github.com/yeomin4242/42OUTER) `WEB/matcha/BE/package.json`: express 4, express-session, jsonwebtoken, pg, helmet, socket.io.
- [Elliop/42-Matcha](https://github.com/Elliop/42-Matcha) `Matcha/backend/package.json`: express 4, mysql, jsonwebtoken, **passport + passport-local** (an auth library, which would count as an account manager under R-GEN-02).
- [Chaleira/42_Matcha](https://github.com/Chaleira/42_Matcha) `matcha-express/package.json`: express, **mongoose** (ODM on a NoSQL DB, against R-GEN-02 and R-GEN-05). This shows that public repos are not evidence of acceptance.
- [nristorc/Matcha](https://github.com/nristorc/Matcha): describes itself as Express + MySQL with a "HomeMade orm/odm", which is the R-GEN-05 escape hatch.

Pattern: Express + raw driver + express-session/JWT + socket.io is the dominant Node stack. Fastify, Hono and Koa barely appear in Matcha repos found by code search.

## Recommendation for the decision ticket

- **Back end: Express 5** + a raw PostgreSQL driver (`pg`) behind a small homegrown query helper. Express is the only Node framework the subject names as a micro-framework, so the defense argument is one sentence long.
- **Validation and auth by hand.** Write validation functions and the account flow (registration, verification, reset, login) ourselves. Session or token primitives (`express-session` or signed cookies, `bcrypt`/`argon2`) are acceptable risk.
- **Front end: React + Vite + a client-side router in library mode**, as a SPA. No Next.js and no Remix.
- **Avoid Fastify and Hono.** Their core packages include a validator, which fails the subject's own definition on a literal read, and nothing gained is worth that argument at a defense.

## Open questions (unverified)

- The exact items on the official Matcha evaluation sheet. Ask a 42 student or staff member with intranet access to confirm whether it checks dependencies line by line, and whether it explicitly bans validation libraries or query builders.
