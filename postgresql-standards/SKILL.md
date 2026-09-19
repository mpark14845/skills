---
name: postgres-persistence
description: Enforces PostgreSQL as the database and data-access standards for schema design, migrations, connection pooling, queries, transactions, and backups. Use this skill whenever the user explicitly asks for Postgres/PostgreSQL, or when a project's requirements genuinely need what Postgres offers over SQLite — multiple server instances writing concurrently, high write concurrency, horizontal scaling, advanced querying (full-text search, JSONB, window functions), or an existing project already uses Postgres. Do not use this skill by default: this user's default persistence choice is SQLite (`sqlite-persistence` skill); only reach for Postgres on an explicit request or a clear scale/feature need, and say so when recommending it over the default.
---

# PostgreSQL Persistence Standards

You are an expert in relational database architecture. When assisting with Postgres persistence or data-access code, strictly adhere to the following.

## 1. Architectural Philosophy

- Postgres is the right choice when a project genuinely needs what SQLite can't provide: multiple app servers/processes writing to the same database concurrently, high write throughput, advanced query features (JSONB querying, full-text search, window functions, extensions like PostGIS), or strong tooling for read replicas and horizontal scaling.
- This user's default is SQLite. When a request would reach for a database, use this skill only if the user explicitly named Postgres, an existing project already runs on it, or the stated requirements clearly need multi-writer/scale characteristics — and say plainly why Postgres over the default when you make that call.
- Apply "always is cheap and sometimes is expensive": don't add read replicas, sharding, or exotic extensions until the project's actual load requires them. A single well-indexed Postgres instance with a connection pool covers the overwhelming majority of real apps.

## 2. Tech Stack & Library Choice

- **Default driver:** [`pg`](https://node-postgres.com/) (`node-postgres`) with its built-in `Pool` — mature, minimal, and gives direct SQL control.
- **When the project wants type-safe queries and a non-trivial schema:** [Drizzle ORM](https://orm.drizzle.team/) with its Postgres driver, for the same reasons it's preferred in `sqlite-persistence` — it stays close to SQL and generates migrations without a heavy runtime. Prefer it over Prisma unless Prisma is already in use.
- Do not mix a raw-SQL layer and an ORM in the same project — pick one data-access style and keep it consistent, same rule as the SQLite skill.

## 3. Project Structure

```
migrations/
  0001_init.sql
  0002_add_users.sql
src/
  config/
    db.js              # creates and exports the shared Pool
  models/               # one file per table/entity, all queries for it live here
```

- Connection details come from a single `DATABASE_URL` environment variable (never hardcoded host/user/password), with `.env.example` documenting the required shape but no real credentials.

## 4. Connection & Pooling

Create one `Pool` per process and share it — never open a new client per request:

```js
// config/db.js
import pg from "pg";

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === "production" ? { rejectUnauthorized: true } : false,
  max: 10,                        // tune to expected concurrent load, not "as high as possible"
  idleTimeoutMillis: 30_000,
  statement_timeout: 10_000,      // fail a runaway query instead of hanging the pool
});

export default pool;
```

- Require SSL in production; most managed providers (RDS, Supabase, Render, Fly) need `ssl: true` or a specific `rejectUnauthorized` setting — check the provider's docs rather than guessing.
- Set `statement_timeout` so one slow query can't exhaust the pool and take the app down with it.
- Size the pool to the deployment, not by default habit — a serverless/edge deployment with many concurrent instances needs a much smaller `max` per instance (or a pooler like PgBouncer/Supabase's pooler) than a single long-running server.

## 5. Schema & Migrations

- Every schema change is a numbered, ordered `.sql` file in `migrations/` (or Drizzle Kit's generated migrations if using Drizzle) — never hand-alter a live schema.
- Track applied migrations in a `_migrations` table, same pattern as `sqlite-persistence`, so a runner script applies only what's new and migrations stay reproducible across environments.
- Define foreign keys with explicit `ON DELETE` behavior, and add indexes in the same migration as the column/table they support — don't leave a query pattern's index for "later."
- Use `snake_case` for table and column names (Postgres convention) and plural table names (`users`, `order_items`).

## 6. Query Patterns

- **Always use parameterized queries** (`$1`, `$2`, …) — never string-concatenate or template-literal user input into SQL.
  ```js
  // Good
  const { rows } = await pool.query("SELECT * FROM users WHERE email = $1", [email]);

  // Never
  await pool.query(`SELECT * FROM users WHERE email = '${email}'`); // injection risk
  ```
- For multi-statement writes, check out a client and use a real transaction — don't call `pool.query` multiple times and hope nothing fails in between:
  ```js
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    await client.query("INSERT INTO orders (...) VALUES (...)", [...]);
    await client.query("UPDATE inventory SET qty = qty - 1 WHERE id = $1", [id]);
    await client.query("COMMIT");
  } catch (err) {
    await client.query("ROLLBACK");
    throw err;
  } finally {
    client.release(); // always release back to the pool, success or failure
  }
  ```
- Keep all SQL for a table in that table's `models/` file — controllers/services call model functions, never raw SQL, matching the `node-express-backend` layering.

## 7. Data Types & Conventions

- Use `UUID` (`gen_random_uuid()`, via the `pgcrypto`/`pgcrypto`-free `gen_random_uuid()` in PG 13+) or `BIGSERIAL`/`IDENTITY` for primary keys — pick one convention per project, don't mix.
- Use `TIMESTAMPTZ` for all timestamps, never bare `TIMESTAMP` — store in UTC, convert at the display layer.
- Use `NUMERIC`/`DECIMAL` for money and anything requiring exact precision — never `FLOAT`/`REAL`, which introduce rounding error.
- Use `JSONB` (not `JSON`) for semi-structured data that needs to be queried or indexed; add a `GIN` index on a `JSONB` column if it's filtered on regularly.
- Prefer a real column with a `CHECK` constraint or a Postgres `ENUM` type over a free-text status field, so invalid states are rejected by the database, not just the app.

## 8. Concurrency & Scaling

- Postgres uses MVCC and handles concurrent writers natively — unlike SQLite, this is not a limiting factor for a normal web app's traffic.
- Index every column used in a `WHERE`, `JOIN`, or `ORDER BY` on a table of meaningful size; an unindexed query that's fine at 100 rows will not be fine at 100,000.
- Keep transactions short — no `await`ing an external API call or long computation between `BEGIN` and `COMMIT`; long-held transactions cause lock contention and pool exhaustion under load.
- For read-heavy scaling beyond a single instance, a read replica (most managed providers offer one) is the first lever — reach for it before anything more exotic like sharding.

## 9. Backups

- Use `pg_dump`/`pg_restore` for logical backups; for anything production-grade, enable continuous WAL archiving / point-in-time recovery (most managed providers do this automatically — verify it's actually turned on rather than assuming).
- Test the restore path at least once — a backup that's never been restored is unverified.

## 10. Testing

- Run tests against a real Postgres instance (local Docker container or a dedicated test database), not a mock — Postgres-specific behavior (constraints, JSONB queries, case sensitivity) doesn't reliably surface in a mocked layer.
- Wrap each test in a transaction that's rolled back at the end (`BEGIN` in setup, `ROLLBACK` in teardown) for fast, isolated tests without recreating the schema each time; use a fresh migrated schema per test run in CI.

## 11. Security

- Parameterized queries are non-negotiable (§6) — the primary injection vector.
- The application's database role should follow least privilege: not a superuser, granted only the schema/table permissions it needs. Use a separate, more restricted role for any read-only reporting access.
- Require SSL/TLS for any non-local connection (§4). Never commit `DATABASE_URL` or credentials — environment variables or a secrets manager only.
- Consider row-level security (`ROW LEVEL SECURITY` policies) for multi-tenant data where a bug in application-layer filtering could otherwise leak another tenant's rows.

## 12. Integration with Other Skills

- This skill fills in the "Database & Data Access" section of `node-express-backend` the same way `sqlite-persistence` does: `models/` holds the queries, `config/db.js` is the shared client, migrations live alongside the rest of the project.
- Relationship to the other persistence skills: `sqlite-persistence` is the default; `json-file-persistence` is the opt-in lightweight alternative for small/local data. This skill is the opt-in upgrade path when a project outgrows SQLite (per that skill's §8) or the user asks for Postgres directly. Don't run two persistence skills' patterns in the same project — one database, one data-access style.

## 13. Example: End-to-End

```sql
-- migrations/0002_add_users.sql
CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```js
// models/userModel.js
import pool from "../config/db.js";

export async function createUser({ email, passwordHash }) {
  const { rows } = await pool.query(
    "INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING *",
    [email, passwordHash]
  );
  return rows[0];
}

export async function findByEmail(email) {
  const { rows } = await pool.query("SELECT * FROM users WHERE email = $1", [email]);
  return rows[0] ?? null;
}
```

## 14. Instructions for the Agent

1. Acknowledge that you are applying the `postgres-persistence` skill.
2. Confirm this is the right call before scaffolding: if nothing about the request needs Postgres over the default SQLite, say so and suggest `sqlite-persistence` instead, unless the user has already stated they want Postgres.
3. Default to the `pg` driver with raw parameterized SQL; only introduce Drizzle if the schema is complex or the user asks for type safety — don't add both.
4. Always write parameterized queries and never string-interpolate values into SQL, with no exceptions.
5. When scaffolding a new project, create the `migrations/` folder and an initial migration, and wire up `config/db.js` with pooling and `statement_timeout` from the start rather than adding them later under load.
