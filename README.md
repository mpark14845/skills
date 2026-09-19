# skills

A personal collection of [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) that encode this user's default architectural standards, tech stack choices, and conventions. Each skill lives in its own directory with a `SKILL.md` (frontmatter + instructions) that Claude Code loads automatically when a request matches its `description`.

## Skills

| Skill | Directory | Applies to |
|---|---|---|
| [node-express-backend](node-express-backend/SKILL.md) | `node-express-backend/` | Node.js + Express backend architecture — routing, controllers/services/models layering, API conventions, error handling, validation, security essentials |
| [vanilla-js-frontend](vanilla-js-frontend/SKILL.md) | `vanilla-js-frontend/` | Frontend UI work — plain HTML/CSS/JS only; frameworks (React, Vue, Angular, etc.) and CSS libraries (Tailwind, Bootstrap) are prohibited by default |
| [sqlite-standards](sqlite-standards/SKILL.md) | `sqlite-standards/` | **Default** persistence choice for web/desktop apps — schema, migrations, queries, transactions via `better-sqlite3` |
| [postgresql-standards](postgresql-standards/SKILL.md) | `postgresql-standards/` | Opt-in upgrade from SQLite when a project needs multi-writer concurrency, scale, or advanced querying (JSONB, full-text search) |
| [sql-server-standards](sql-server-standards/SKILL.md) | `sql-server-standards/` | Opt-in for projects targeting an existing SQL Server / Azure SQL instance |
| [json-persistence](json-persistence/SKILL.md) | `json-persistence/` | Opt-in lightweight alternative to a database — flat JSON files for config/settings/small datasets |
| [auth-identity](auth-identity/SKILL.md) | `auth-identity/` | Authentication (local email/password + OIDC federated login) and authorization (RBAC + ABAC) standards |
| [company-design](company-design/SKILL.md) | `company-design/` | Brand design system (colors, type, spacing, components) for UI work matching the company's visual identity |


## How these fit together

- **Persistence is layered by default → opt-in.** `sqlite-standards` is the default for any new persistence need. `postgresql-standards` and `sql-server-standards` are explicit opt-ins for scale or an existing cloud instance; `json-persistence` is an opt-in for small, file-based storage. Only one persistence skill should govern a given project.
- **`node-express-backend`** defines the layered architecture (routes → controllers → services → models); the active persistence skill fills in the "Database & Data Access" section with library- and query-specific details.
- **`auth-identity`** builds on top of `node-express-backend` and whichever persistence skill is active — it defines the auth/authz domain model (users, identities, roles, permissions) but delegates table/migration/query style to that persistence skill.
- **`vanilla-js-frontend`** governs all client-side UI code and explicitly forbids frontend frameworks; **`company-design`** supplies the design tokens (colors, type, spacing, components) that frontend work should pull from when a `tokens.css`/`design.md` exists in the project.

## Shared principles

Several skills repeat the same philosophy, worth calling out once:

- **"Always is cheap, sometimes is expensive"** — build for the common case; don't add abstractions, config options, or pluggable providers for scenarios the project doesn't have yet.
- **Parameterized queries, always** — every persistence skill treats string-built SQL as a non-negotiable defect, not a style preference.
- **Ask before assuming scope** — `auth-identity` requires confirming identity providers and role models before implementing; the persistence skills require justifying a choice other than SQLite before scaffolding it.
