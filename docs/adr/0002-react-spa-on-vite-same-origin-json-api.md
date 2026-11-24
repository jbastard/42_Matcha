# React SPA on Vite, same-origin JSON API

The client is a React single-page application built with Vite, talking to the Express back end (ADR 0001) through a JSON API under `/api` and a WebSocket under `/ws`, all on one origin so session cookies work without CORS. A SPA keeps one WebSocket open across navigation, which the real-time chat and notifications need on every page (R-CHAT-01, R-CHAT-02); React is a UI library the subject names (R-GEN-03).

The repository is an npm-workspaces monorepo: `server/`, `client/` and `shared/`. `shared/` holds the API contract types and the homegrown validation module, so client and server apply the same rules; the server always revalidates (R-SEC-05). Routing uses React Router in library (declarative) mode. Server data on the client goes through a homegrown `fetch` client with TanStack Query for caching, and WebSocket events invalidate the affected queries. In development, the Vite proxy forwards `/api` and `/ws` to Express.

## Considered Options

- **Server-rendered pages (Express + templating)**: every navigation reloads the page and drops the WebSocket.
- **Vue, Angular**: allowed by the subject, not chosen. **Svelte, Solid**: not named in the subject (R-GEN-04 risk).
- **Next.js, Remix, React Router framework mode**: run server code, so they fall under R-GEN-02.
- **TanStack Router**: its `validateSearch` reads as a validator to a strict evaluator.
