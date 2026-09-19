---
name: sqlite-persistence
description: Enforces SQLite as the default database and data-access standards for schema design, migrations, queries, transactions, and backups. Use this skill whenever the user needs persistence, storage, or a database in a web or desktop app — e.g. "save this to a database", "add a users table", "persist this data", "add login" (implies a users store), "add a schema/migration", "the app needs to remember state between runs" — even if they don't say "SQLite" or "database" explicitly. This is the default persistence choice for this user's projects; do not propose Postgres, MySQL, MongoDB, or a cloud database unless the user explicitly asks for one or an existing project already uses one.
---

# SQLite Persistence Standards

You are an expert in embedded relational data storage. When assisting with persistence, schema, or data-access code, strictly adhere to the following.

## 1. Architectural Philosophy

- SQLite is the default database for both web backends (paired with the `node-express-backend` skill) and Electron/desktop apps built with this user's stack — it needs no separate server process, ships as a single file, and is plenty for single-server or single-user workloads.
- Apply "always is cheap and sometimes is expensive": don't add a query builder, ORM, or connection pool abstraction the project doesn't need yet. Start with direct SQL and a thin data-access layer; only add an ORM (see §2) when the schema is genuinely complex or the user asks for type safety.
- Know SQLite's real limit and say so early: it handles one writer at a time. It is excellent for a single Node server, a desktop app, an internal tool, or read-heavy sites. It is the wrong choice for multi-server horizontal scaling or high-concurrency multi-writer workloads — flag this if a project's requirements point that direction, rather than silently building on SQLite anyway.

## 2. Tech Stack & Library Choice

- **Default:** [`better-sqlite3`](https://github.com/WiseLibs/better-sqlite3) — synchronous, fast, and simple; no callback/promise ceremony for what is a local file read/write. Use this unless the project already has a different SQLite library in place.
- **When the project wants type-safe queries and the schema is non-trivial:** [Drizzle ORM](https://orm.drizzle.team/) with its SQLite driver — it stays close to SQL, generates migrations, and doesn't pull in a heavy runtime. Prefer it over Prisma for SQLite projects unless Prisma is already in use.
- Do not add both a raw SQL layer and an ORM in the same project — pick one data-access style and keep it consistent.

## 3. Project Structure & File Location

```
data/
  app.db              # the SQLite database file — gitignored
  migrations/
    0001_init.sql
    0002_add_users.sql
src/
  config/
    db.js              # opens the connection, applies pragmas, exports the client
  models/               # one file per table/entity, all queries for it live here
```

- The `.db` file (and `-wal`/`-shm` sidecar files) are runtime data, not source — always add `data/*.db*` to `.gitignore`. Never commit a populated database file.
- Use a separate database file for tests (`data/test.db` or `:memory:`) — never point tests at the dev or prod file.

## 4. Connection & Configuration

Open one shared connection per process (SQLite doesn't benefit from a pool the way a network database does) and set these pragmas on startup:

```js
// config/db.js
import Database from "better-sqlite3";

const db = new Database(process.env.DB_PATH || "data/app.db");
db.pragma("journal_mode = WAL");     // concurrent readers + one writer, don't block on read
db.pragma("foreign_keys = ON");      // off by default in SQLite — always turn on
db.pragma("busy_timeout = 5000");    // wait instead of failing immediately on a lock

export default db;
```

- **WAL mode** is close to mandatory for anything with concurrent access (a web server) — it lets reads proceed while a write is in progress.
- **`foreign_keys = ON`** every time — SQLite silently ignores foreign key constraints unless this is set per connection.
- Never open a new `Database` instance per request; import the shared client from `config/db.js`.

## 5. Schema & Migrations

- Every schema change is a numbered, ordered `.sql` file in `migrations/` (`0001_init.sql`, `0002_add_users.sql`) — never edit an already-applied migration or hand-alter the schema on a live database.
- Track applied migrations in a `_migrations` table (`id`, `name`, `applied_at`) so a small runner script can apply only what's new:
  ```js
  // scripts/migrate.js — run new .sql files in order, record each in _migrations
  ```
- Write `CREATE TABLE IF NOT EXISTS` and explicit column types (`INTEGER`, `TEXT`, `REAL`, `BLOB`) even though SQLite doesn't strictly enforce them — it documents intent and enables type affinity.
- Define foreign keys with an explicit `ON DELETE` behavior (`CASCADE`, `SET NULL`, or `RESTRICT`) — don't leave it implicit.

## 6. Query Patterns

- **Always use prepared statements with bound parameters** — never string-concatenate or template-literal user input into SQL. This is the single most important rule in this skill.
  ```js
  // Good
  const stmt = db.prepare("SELECT * FROM users WHERE email = ?");
  const user = stmt.get(email);

  // Never
  db.prepare(`SELECT * FROM users WHERE email = '${email}'`); // injection risk
  ```
- Prepare statements once (module scope or a cached map), reuse them across calls — don't call `db.prepare()` inside a hot loop.
- Wrap multi-statement writes in a transaction so partial failures can't leave inconsistent data:
  ```js
  const insertMany = db.transaction((rows) => {
    for (const row of rows) insertStmt.run(row);
  });
  insertMany(rows);
  ```
- Keep all SQL for a table in that table's `models/` file — controllers and services call model functions (`userModel.findByEmail(email)`), never raw SQL.

## 7. Data Types & Quirks

- SQLite has dynamic typing (type affinity, not strict types) — don't rely on the database to reject a wrong-typed value; validate at the application boundary (per the `node-express-backend` skill's validation rules).
- **Booleans:** store as `INTEGER` (`0`/`1`) — there is no native boolean type. Convert at the model layer, not scattered through the app.
- **Dates/times:** store as ISO 8601 `TEXT` (`2026-09-17T14:30:00.000Z`) for readability and correct lexicographic sort order, not as a Unix integer unless the project specifically needs it.
- **JSON columns:** store as `TEXT` and `JSON.parse`/`JSON.stringify` at the model boundary; SQLite's `json_extract` is available if you need to query inside the JSON, but prefer a real column when a field is queried often.
- Auto-increment primary keys: use `INTEGER PRIMARY KEY` (SQLite's built-in rowid alias) — don't add `AUTOINCREMENT` unless the project specifically needs monotonically non-reused IDs, since it adds overhead.

## 8. Concurrency & Scaling Limits

- One process, one writer at a time is the model — WAL mode lets reads run concurrently with a write, but writes still serialize.
- Keep transactions short — don't hold a write transaction open across an `await` for a network call or another I/O operation; gather data first, then transact.
- If the app will run as multiple server instances/processes against the same file (not just multiple requests within one process), say so explicitly to the user — that's the point where SQLite stops being a safe default and a networked database (or a replication layer like Litestream) needs to be discussed.

## 9. Backups

- For a live app, back up with `VACUUM INTO 'backup.db'` or the SQLite backup API — never `cp` the file while it's open and being written to (WAL mode makes a naive file copy inconsistent).
- Desktop/Electron apps: back up to the user's app-data directory on a schedule or before destructive operations (schema migration, bulk delete).

## 10. Testing

- Use `:memory:` as the database path for unit/integration tests — fast, isolated, and automatically discarded.
- Apply the same migration scripts against the in-memory database at test setup so tests run against the real schema, not a hand-maintained copy of it.

## 11. Security

- Parameterized queries are non-negotiable (see §6) — this is the primary injection vector for a SQLite-backed app.
- The `.db` file must not be served by the web server or placed in a public/static directory — keep it outside anything reachable by a direct URL.
- File permissions: the database file should be readable/writable only by the application's process user, not world-readable, especially on shared hosting.

## 12. Integration with the Backend Skill

- This skill fills in the "Database & Data Access" section of `node-express-backend`: `models/` holds the SQLite queries described here, `config/db.js` is the shared client, and migrations live alongside the rest of the project per that skill's structure.
- When both skills are active, this one wins on database-specific decisions (library choice, pragmas, query style); `node-express-backend` still governs the route/controller/service layers around it.

## 13. Example: End-to-End

```sql
-- migrations/0002_add_users.sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
```

```js
// models/userModel.js
import db from "../config/db.js";

const insertStmt = db.prepare(
  "INSERT INTO users (email, password_hash) VALUES (?, ?)"
);
const findByEmailStmt = db.prepare("SELECT * FROM users WHERE email = ?");

export function createUser({ email, passwordHash }) {
  const info = insertStmt.run(email, passwordHash);
  return findById(info.lastInsertRowid);
}

export function findByEmail(email) {
  return findByEmailStmt.get(email);
}
```

## 14. Instructions for the Agent

1. Acknowledge that you are applying the `sqlite-persistence` skill.
2. Default to SQLite for any new persistence need without asking, unless the project already uses a different database or the user's stated scale requirements (multi-server writes, very high concurrency) call for flagging the limits in §8.
3. Default to `better-sqlite3` with raw SQL; only introduce Drizzle if the schema is complex or the user asks for type safety — don't add both.
4. Always write parameterized queries and never string-interpolate values into SQL, with no exceptions.
5. When scaffolding a new project, create the `migrations/` folder and an initial migration rather than a single ad-hoc `CREATE TABLE` buried in application code.
