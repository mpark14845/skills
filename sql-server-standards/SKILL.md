---
name: sqlserver-persistence
description: Enforces Microsoft SQL Server (including cloud-hosted/Azure SQL) database and data-access standards for schema design, migrations, connection pooling, T-SQL queries, transactions, and backups. Use this skill whenever the user explicitly asks for SQL Server, MSSQL, T-SQL, or Azure SQL, or an existing project already connects to one. Do not use this skill by default: this user's default persistence choice is SQLite (`sqlite-persistence` skill), with Postgres (`postgres-persistence`) as the opt-in choice for self-managed scale needs. SQL Server is available to this user as an occasional, explicitly-chosen option — typically because a cloud-hosted instance already exists — not something to reach for unprompted.
---

# SQL Server Persistence Standards

You are an expert in relational database architecture on Microsoft SQL Server. When assisting with SQL Server persistence or data-access code, strictly adhere to the following.

## 1. Architectural Philosophy

- SQL Server is used here when the user has an existing cloud-hosted instance (Azure SQL Database is the common case) they want to target — not chosen from scratch the way SQLite or Postgres might be. Confirm which flavor is in play (Azure SQL Database, Azure SQL Managed Instance, or a self-hosted SQL Server) before assuming Azure-specific behavior, since firewall, auth, and scaling details differ.
- Apply "always is cheap and sometimes is expensive": use the built-in Azure SQL features (automated backups, point-in-time restore, built-in firewall) rather than re-implementing what the platform already provides.
- Since this is an "occasional" database for this user rather than the default, keep the data-access layer isolated behind models (per `node-express-backend`) so the rest of the app doesn't need to know or care that this particular project talks to SQL Server instead of SQLite or Postgres.

## 2. Tech Stack & Library Choice

- **Default driver:** [`mssql`](https://github.com/tediousjs/node-mssql) (wraps `tedious`) — the standard Node driver for SQL Server, with built-in connection pooling and a Promise API.
- **When the project wants type-safe queries and a non-trivial schema:** Drizzle ORM's SQL Server support, or Prisma if the project already uses it — same preference order as the other persistence skills (stay close to SQL, avoid a heavy runtime, unless something heavier is already in place).
- Do not mix a raw-SQL layer and an ORM in the same project.

## 3. Project Structure

```
migrations/
  0001_init.sql
  0002_add_users.sql
src/
  config/
    db.js              # creates and exports the shared connection pool
  models/               # one file per table/entity, all queries for it live here
```

- Connection details come from environment variables (`DB_SERVER`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, or a single connection string) — never hardcoded. `.env.example` documents the required keys with no real values.

## 4. Connection & Pooling

Create one connection pool per process and share it — never open a new connection per request:

```js
// config/db.js
import sql from "mssql";

const pool = new sql.ConnectionPool({
  server: process.env.DB_SERVER,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  pool: {
    max: 10,                 // tune to expected concurrent load
    min: 0,
    idleTimeoutMillis: 30_000,
  },
  options: {
    encrypt: true,             // required for Azure SQL; keep on for any cloud instance
    trustServerCertificate: false, // true only for local dev against a self-signed cert
    requestTimeout: 15_000,     // fail a runaway query instead of hanging the pool
  },
});

export const dbReady = pool.connect(); // connect once; reuse the pool everywhere
export default pool;
```

- `encrypt: true` is required for Azure SQL and strongly recommended for any cloud connection — don't disable it to "fix" a local connection error; fix the certificate/firewall issue instead.
- For Azure SQL specifically: the server's firewall must allow the connecting IP (or "Allow Azure services" for platform-hosted apps) — a connection timeout is often a firewall rule, not a code bug. Check that first before changing connection code.
- Azure AD authentication (`authentication: { type: "azure-active-directory-*" }`) is available and preferable to SQL auth (username/password) when the hosting environment supports managed identity — use it if the project's deployment target supports it; otherwise SQL auth with a strong, rotated password is fine.

## 5. Schema & Migrations

- Every schema change is a numbered, ordered `.sql` file in `migrations/` — never hand-alter a live schema, especially on a shared cloud instance other people or apps may also depend on.
- Track applied migrations in a `_migrations` table, same pattern as the other persistence skills.
- SQL Server naming convention: `PascalCase` or `snake_case` are both common — match whatever convention the existing database already uses rather than imposing a new one on cloud instances that predate this project.
- Objects live in the `dbo` schema by default unless the instance already uses named schemas per module/tenant — check first rather than assuming `dbo`.

## 6. Query Patterns

- **Always use parameterized queries** via the `mssql` request API (`request.input(...)`) — never string-concatenate or template-literal user input into T-SQL.
  ```js
  // Good
  const result = await pool.request()
    .input("email", sql.NVarChar, email)
    .query("SELECT * FROM Users WHERE Email = @email");

  // Never
  await pool.request().query(`SELECT * FROM Users WHERE Email = '${email}'`); // injection risk
  ```
- Declare the SQL type on every `.input()` call (`sql.NVarChar`, `sql.Int`, `sql.UniqueIdentifier`, etc.) rather than letting the driver infer it — inferred types can silently pick the wrong one and cause implicit conversions that block index usage.
- Use `TOP` for row limiting (`SELECT TOP 10 ...`), not `LIMIT` — T-SQL syntax, not Postgres/SQLite syntax.
- For multi-statement writes, use an explicit transaction, not multiple independent `pool.request()` calls:
  ```js
  const transaction = new sql.Transaction(pool);
  await transaction.begin();
  try {
    const request = new sql.Request(transaction);
    await request.input("id", sql.Int, orderId).query("INSERT INTO Orders (...) VALUES (...)");
    await request.input("id", sql.Int, itemId).query("UPDATE Inventory SET Qty = Qty - 1 WHERE Id = @id");
    await transaction.commit();
  } catch (err) {
    await transaction.rollback();
    throw err;
  }
  ```
- Keep all SQL for a table in that table's `models/` file — controllers/services call model functions, never raw SQL, matching the `node-express-backend` layering.

## 7. Data Types & Conventions

- Use `UNIQUEIDENTIFIER` (with `NEWID()` or app-generated UUIDs) or `INT IDENTITY(1,1)` for primary keys — pick one convention per project.
- Use `DATETIME2` for timestamps (more precision and a wider range than the legacy `DATETIME`); store in UTC and convert at the display layer.
- Use `DECIMAL`/`NUMERIC` for money and anything requiring exact precision — never `FLOAT`/`REAL`.
- Use `NVARCHAR` (not `VARCHAR`) for any text that might contain non-ASCII characters — `VARCHAR` is single-byte and will silently mangle non-Latin text.
- Use `CHECK` constraints or a lookup table over a free-text status column so invalid states are rejected by the database.

## 8. Concurrency & Scaling

- SQL Server handles concurrent writers natively (locking + optional row versioning via `READ_COMMITTED_SNAPSHOT`) — this is not a limiting factor for typical app traffic the way SQLite is.
- Index every column used in a `WHERE`, `JOIN`, or `ORDER BY` on a table of meaningful size, and periodically check for missing-index recommendations if you have visibility into the instance (Azure SQL's Query Performance Insight surfaces these).
- Keep transactions short — no `await`ing an external call between `BEGIN TRAN` and `COMMIT`; long transactions cause blocking on a shared cloud instance that may have other consumers.
- If the instance is DTU/vCore-limited (common on lower Azure SQL tiers), watch for throttling under load — that's a capacity/tier problem, not something to code around with retries alone.

## 9. Backups

- Azure SQL Database handles automated backups and point-in-time restore natively — verify the retention window fits the project's needs rather than building a separate backup mechanism.
- For a self-hosted SQL Server instance, use native backup (`BACKUP DATABASE`) on a schedule, and periodically verify the restore path works.

## 10. Testing

- Run tests against a real SQL Server instance (a local SQL Server container, or a dedicated cloud test database/schema) — T-SQL-specific behavior doesn't reliably surface against a mock.
- Prefer a transaction-per-test that rolls back at the end for speed and isolation; if the shared cloud instance can't be used for test writes, seed and tear down a dedicated test schema instead.

## 11. Security

- Parameterized queries are non-negotiable (§6) — the primary injection vector.
- Use a least-privilege application login/user — not the server admin account — scoped to only the schema/tables the app needs.
- `encrypt: true` for any cloud connection (§4); never disable encryption to work around a connection error.
- Never commit connection strings or credentials — environment variables or a secrets manager (Azure Key Vault, if already in use) only.
- Since this is a shared/occasional cloud instance, double-check which database/schema a migration or destructive query targets before running it — a mistake here can affect more than just this project.

## 12. Integration with Other Skills

- This skill fills in the "Database & Data Access" section of `node-express-backend` the same way the other persistence skills do: `models/` holds the queries, `config/db.js` is the shared pool, migrations live alongside the rest of the project.
- Relationship to the other persistence skills: `sqlite-persistence` is the default, `json-file-persistence` is the lightweight opt-in, `postgres-persistence` is the self-managed scale opt-in, and this skill is the opt-in for when the user specifically wants to use their existing cloud SQL Server instance. Don't run two persistence skills' patterns in the same project — one database, one data-access style.

## 13. Example: End-to-End

```sql
-- migrations/0002_add_users.sql
CREATE TABLE Users (
  Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  Email NVARCHAR(255) NOT NULL UNIQUE,
  PasswordHash NVARCHAR(255) NOT NULL,
  CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

```js
// models/userModel.js
import sql from "mssql";
import pool from "../config/db.js";

export async function createUser({ email, passwordHash }) {
  const result = await pool.request()
    .input("email", sql.NVarChar, email)
    .input("passwordHash", sql.NVarChar, passwordHash)
    .query(
      "INSERT INTO Users (Email, PasswordHash) OUTPUT INSERTED.* VALUES (@email, @passwordHash)"
    );
  return result.recordset[0];
}

export async function findByEmail(email) {
  const result = await pool.request()
    .input("email", sql.NVarChar, email)
    .query("SELECT * FROM Users WHERE Email = @email");
  return result.recordset[0] ?? null;
}
```

## 14. Instructions for the Agent

1. Acknowledge that you are applying the `sqlserver-persistence` skill.
2. Confirm which SQL Server flavor is in play (Azure SQL Database, Managed Instance, or self-hosted) before assuming Azure-specific behavior like firewall rules or automated backups.
3. Default to the `mssql` driver with raw parameterized T-SQL; only introduce an ORM if the schema is complex or the user asks for type safety — don't add both.
4. Always use `.input()` with an explicit SQL type for every parameter, and never string-interpolate values into a query, with no exceptions.
5. Treat the instance as shared/occasional infrastructure: confirm the target database/schema before running migrations or destructive queries, and don't assume `dbo` or a specific naming convention without checking the existing schema first.
