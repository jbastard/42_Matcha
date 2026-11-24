# Express 5 with hand-rolled validation, accounts and queries

The back end runs on Node (Active LTS) with TypeScript and Express 5, the only Node framework the subject names as a micro-framework (R-GEN-02). Input validation, the account flow and SQL query handling are written by hand, because R-GEN-02 forbids validators, ORMs and user account managers, R-GEN-04 forbids anything not explicitly allowed, and R-GEN-05 requires hand-written queries.

## Library admissibility test

A runtime library is allowed only if it does none of the following:

1. validate input against a schema;
2. model users, accounts or authentication flows;
3. build or abstract SQL queries.

Under this test `pg`, `ws`, `argon2`, `nodemailer`, `sharp`, `helmet` and a multipart parser pass; Zod, Joi, express-validator, Ajv, Passport, Auth.js, Prisma, Drizzle, Knex and Kysely do not. Database access goes through a raw driver plus a small homegrown helper for parameterised queries. Every runtime dependency is listed with its justification in `docs/dependencies.md`, created with the first one.

## Considered Options

- **Fastify**: core depends on Ajv for route validation.
- **Hono**: ships `hono/validator` and auth helpers.
- **Koa + `@koa/router`**: passes the definition but is not named in the subject, which invites a debate at the defense.
- **NestJS, AdonisJS**: bundled validation and ORM integration.
- **Next.js, Remix**: run server code, so they fall under R-GEN-02 as full frameworks.
