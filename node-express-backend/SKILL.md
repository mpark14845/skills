---
name: node-express-backend
description: Enforces Node.js and Express.js architectural standards for backend API generation, server logic, routing, database access, and authentication. Use this skill whenever the user asks to build, extend, or debug a backend, API, server, route, controller, endpoint, or database layer — even if they don't say "Node" or "Express" explicitly (e.g. "add a login endpoint", "connect this to a database", "the server is throwing an error"). Also applies when scaffolding a new backend project or reviewing existing backend code for structure and security issues.
---

# Backend Architecture Standards

You are an expert backend software architect. When assisting with server-side logic, APIs, or database interactions, strictly adhere to the following boundaries.

## 1. Architectural Philosophy

- Apply the principle: "Always is cheap and sometimes is expensive." Handle the common path simply; don't add abstraction, config options, or defensive code for edge cases that rarely occur.
- Prioritize simplicity, maintainability, and standard web platform features over clever abstractions or extra dependencies.
- Every layer should have one job. Routes parse the request and call a controller. Controllers orchestrate and shape the response. Services hold business logic. Models/queries hold data access. Don't let a route handler touch the database directly, and don't let a service know about `req`/`res`.

## 2. Tech Stack

- **Environment:** Node.js (LTS).
- **Framework:** Express.js for routing and middleware.
- **Modules:** Default to whichever module system (CommonJS or ES Modules) the current project already uses. If starting fresh, use ES Modules (`"type": "module"`).
- Do NOT suggest Python, Django, Flask, Fastify, NestJS, or other frameworks/environments unless the user explicitly asks for them or the project already uses them.

## 3. Project Structure

Default to this layout for any new backend project; match an existing project's layout if one already exists rather than forcing a rewrite:

```
src/
  routes/        # thin — define endpoints, wire middleware, call controllers
  controllers/    # parse req, call services, shape res — no business logic
  services/       # business logic, orchestration — framework-agnostic
  models/         # schema + data access (ORM models or query functions)
  middleware/     # auth, validation, error handling, logging
  config/         # env loading, constants, third-party client setup
  utils/          # small stateless helpers
app.js            # express app assembly, middleware registration
server.js         # process entrypoint — starts the HTTP server
```

## 4. API Design Conventions

- Use standard RESTful conventions: plural nouns for collections (`/users`, `/orders/:id`), HTTP verbs carry the action (`GET`/`POST`/`PATCH`/`DELETE`), not the URL (`/getUsers`).
- Version the API from the start: prefix routes with `/api/v1`.
- Respond with a consistent JSON envelope:
  ```json
  { "data": { }, "error": null }
  ```
  On failure: `{ "data": null, "error": { "message": "...", "code": "..." } }`
- Use correct status codes: `200` success, `201` created, `204` no content, `400` validation error, `401` unauthenticated, `403` unauthorized, `404` not found, `409` conflict, `500` unhandled server error. Never return `200` with an error payload.

## 5. Error Handling

- Wrap async route handlers so rejected promises reach Express's error handling instead of crashing the process or hanging the request — either a small `asyncHandler(fn)` wrapper or a try/catch that calls `next(err)`.
- Register a single centralized error-handling middleware (four-argument signature) at the end of `app.js` that maps known error types to status codes and logs unexpected ones.
- Throw or return typed/known errors from services (e.g. a `NotFoundError`, `ValidationError`) rather than raw strings, so the central handler can map them without string-matching.

## 6. Validation

- Validate all external input (body, query params, route params) at the route boundary before it reaches a controller — use a schema library (`zod` or `joi`) rather than hand-rolled `if` checks.
- Reject invalid input with `400` and a specific message; never let invalid data reach a service or the database layer.

## 7. Security Essentials

Apply these by default on any new Express app, not only when asked:
- `helmet()` for secure HTTP headers.
- `cors()` configured to an explicit allowed-origin list — never a wildcard in production.
- Rate limiting on public or auth-related endpoints (e.g. `express-rate-limit`).
- All secrets (DB credentials, API keys, JWT secret) come from environment variables via `.env` + `dotenv`, never hardcoded or committed. Include a `.env.example` with keys but no values.
- Hash passwords with `bcrypt`/`argon2` — never store or log plaintext passwords or tokens.
- Sanitize/parameterize all database queries; never build a query with string concatenation from user input.

## 8. Database & Data Access

- Stay ORM-agnostic unless the project has already picked one (Prisma, Sequelize, Mongoose, or a raw query builder like Knex are all acceptable) — use what's already there.
- Keep all query logic inside `models/`; controllers and services call model functions, never write raw queries inline.
- Use migrations for schema changes, not manual/ad-hoc edits to a live schema.
- Use connection pooling and a single shared client instance (set up in `config/`), not a new connection per request.

## 9. Authentication & Authorization

- Default to stateless JWT auth (access token + refresh token) unless the project indicates session-based auth is already in use.
- Implement auth as middleware (`requireAuth`, `requireRole('admin')`) that populates `req.user`, applied per-route — never re-implement the check inline in each controller.
- Keep authorization (what a user can do) separate from authentication (who they are) — authz checks belong in middleware or the service layer, not scattered in controllers.

## 10. Testing

- Prefer integration tests over heavy mocking: use `jest` (or the project's existing test runner) with `supertest` to hit real routes against a test database or in-memory instance.
- Every new endpoint gets at least one happy-path test and one failure-path test (validation error or auth failure).

## 11. Logging

- Use a structured logger (`pino` or `winston`), not `console.log`, for anything beyond local debugging.
- Log request method, path, status, and duration for every request (via middleware); log errors with stack traces server-side only — never leak stack traces in API responses.

## 12. Example: Route → Controller → Service

```js
// routes/users.js
router.post("/", validate(createUserSchema), asyncHandler(userController.create));

// controllers/userController.js
async function create(req, res) {
  const user = await userService.createUser(req.body);
  res.status(201).json({ data: user, error: null });
}

// services/userService.js
async function createUser({ email, password }) {
  const hashed = await hashPassword(password);
  return userModel.insert({ email, password: hashed });
}
```

## 13. Instructions for the Agent

1. Acknowledge that you are applying the `node-express-backend` skill.
2. Before scaffolding a new project, briefly confirm the database and ORM choice if not already evident from the codebase — don't default silently to one.
3. When editing an existing project, match its existing structure and conventions rather than forcing this layout wholesale; apply these standards to new code and flag major deviations rather than silently rewriting unrelated files.
4. Default to standard CommonJS or ES Modules as used by the current project setup.
