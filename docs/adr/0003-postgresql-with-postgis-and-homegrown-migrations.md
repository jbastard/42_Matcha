# PostgreSQL with PostGIS, homegrown migrations and repository modules

The database is PostgreSQL 18 with the PostGIS 3.7 extension, run from the pinned `postgis/postgis:18-3.7` image and reached through the raw `pg` driver (R-GEN-05, ADR 0001). PostGIS is a database extension, not a Node library, so the library admissibility test of ADR 0001 does not apply to it.

## Locations

A user's location is a `geography(Point,4326)` column with a GiST index. Proximity filters use `ST_DWithin` and distance sorts use `ST_Distance`, both in meters on the spheroid (R-BROW-03/04/05/06, R-SRCH-01/02). Alongside the point the row keeps the neighbourhood label, the source (`gps` or `manual`) and the update time. The point is stored as received and never leaves the server: the API exposes only another user's neighbourhood label and a distance rounded to the kilometre (R-PROF-07). Nominatim responses are cached in a `geocode_cache` table with a TTL, as its usage policy requires; the seed reuses it.

## Queries and migrations

SQL is written by hand in repository modules, one per domain, through a homegrown helper limited to `query`, `one`, `many` and `transaction` (R-GEN-05). Values only ever travel as `$n` parameters; the only dynamic SQL fragments, sort column and direction, come from an allowlist (R-SEC-04). Row types are written by hand in each repository, which maps snake_case columns to camelCase fields.

Migrations are numbered raw SQL files `NNNN_name.sql`, forward only, each applied in its own transaction and recorded in a `schema_migrations` table by a homegrown runner. The runner is an npm script and also runs when the server container starts.

Entities that appear in URLs (users, photos, conversations) use `uuid` keys defaulting to `uuidv7()`, so URLs do not reveal account counts; internal tables (likes, visits, notifications, messages) use `bigint generated always as identity`.

## Considered Options

- **MariaDB/MySQL**: spatial functions exist but spheroidal distance and indexing are weaker than PostGIS.
- **Neo4j**: fits the like/block graph, but age, distance and popularity filters and sorts are clumsier, and Cypher is harder to defend.
- **`earthdistance` or plain haversine**: workable at 500 profiles (R-GEN-06), but sphere-only or unindexed, for no real saving over PostGIS.
- **node-pg-migrate and similar tools**: their JavaScript migration DSL builds SQL, which fails point 3 of the admissibility test.
- **Type generation (pgtyped, kanel)**: one more build tool that parses SQL, to defend at evaluation for little gain.
- **Sequential integer ids everywhere**: expose account counts and invite enumeration in profile URLs.
