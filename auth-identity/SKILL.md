---
name: auth-identity
description: Enforces standards for authentication (local email/password plus OpenID Connect federated login via external identity providers) and authorization (RBAC and ABAC). Use this skill whenever the user asks to add login, sign-up, user accounts, "sign in with Google/LinkedIn/etc.", SSO, roles, permissions, access control, or protecting routes/resources by who the user is — even if they don't say "authentication" or "authorization" explicitly (e.g. "only admins should see this page", "let users log in with their Google account", "the API should reject requests from users who don't own the resource"). Before implementing any OIDC/social login, this skill requires asking the user which identity providers to support — never assume a default set.
---

# Authentication & Authorization Standards

You are an expert in identity, authentication, and access control. When assisting with login, user accounts, or permissions, strictly adhere to the following.

## 1. Architectural Philosophy

- Support two authentication paths into the same local user identity: **local credentials** (email + password, hashed) and **federated login via OpenID Connect** (Google, LinkedIn, Microsoft, GitHub, Apple, etc.). A user is one row in `users` regardless of how they authenticate — federated logins link to that row, they don't replace it.
- Apply "always is cheap and sometimes is expensive": don't build a generic pluggable-provider abstraction for providers the user hasn't asked for. Implement exactly the providers requested, in a way that adding one more later is a small, contained change (see §5), not a rewrite.
- Authentication (who is this?) and authorization (what can they do?) are separate concerns handled by separate middleware/layers — never fold a role check into a login handler, and never let a controller re-implement an authorization check inline instead of calling the shared authorization layer (§7, §8).
- Self-host this rather than reaching for a managed identity platform (Auth0, Clerk, Firebase Auth) by default — it's consistent with this user's preference for owning the data layer (see the persistence skills). If the user explicitly asks for a managed provider instead, that overrides this default.

## 2. Before Writing Any Code: Ask What's Needed

Two things must be confirmed with the user before implementation, not assumed:

1. **Which identity providers to support.** Never default to a fixed set (e.g. always adding Google + LinkedIn because they were mentioned once). Ask explicitly — common options to offer as a starting menu: Google, Microsoft, LinkedIn, GitHub, Apple — but the user picks, and the list may be shorter or include something else entirely.
2. **What the roles/permissions model actually needs to express** for this project (see §8) — don't invent a role list (`admin`/`editor`/`viewer`) without confirming it matches the real use case, and don't build ABAC policies (§9) for attribute checks the project doesn't have.

## 3. Tech Stack

- **Local auth:** `bcrypt` or `argon2` for password hashing (never a custom hash, never MD5/SHA1/plain SHA256 for passwords).
- **OIDC:** [`openid-client`](https://github.com/panva/node-openid-client) — a spec-compliant, actively maintained library that talks directly to each provider's discovery/token/userinfo endpoints without a heavyweight strategy-per-provider framework like Passport. One integration pattern covers every OIDC-compliant provider.
- **Sessions:** server-side sessions (session ID in an httpOnly cookie, session data in the database via the project's active persistence skill), not JWTs in `localStorage` (see §6 for why).
- Do not add Passport.js, Auth0 SDK, Firebase Auth, or NextAuth unless the user explicitly asks for one of them or a project already uses it.

## 4. Project Structure

```
src/
  config/
    oauthProviders.js   # per-provider client config, built from confirmed provider list
  middleware/
    authenticate.js      # resolves req.user from session/token
    authorize.js          # requireRole(), requirePermission(), ABAC policy checks
  models/
    userModel.js           # local user records
    identityModel.js        # linked federated identities (provider, subject, user_id)
    sessionModel.js          # server-side sessions
    roleModel.js              # roles, permissions, user_roles
  routes/
    auth.js                    # /login, /register, /logout, /auth/:provider, /auth/:provider/callback
  services/
    authService.js               # password verify/hash, session create/destroy
    oidcService.js                 # provider discovery, code exchange, claim validation
    policyService.js                 # ABAC policy functions (§9)
```

## 5. Local Authentication

- Hash passwords with `bcrypt` (cost factor ≥ 12) or `argon2id` — never store or log plaintext passwords.
- Enforce a minimum password policy (length over complexity rules — NIST guidance favors length; e.g. 12+ characters) at the validation boundary, same as the `node-express-backend` skill's validation rules.
- Rate-limit login and password-reset endpoints specifically (tighter than general API rate limits) to slow credential-stuffing and enumeration attacks.
- Return the same generic error ("invalid email or password") whether the email doesn't exist or the password is wrong — never reveal which one, to avoid user enumeration.
- Password reset: a single-use, short-expiry (e.g. 15–30 min) random token, hashed before storage (same principle as a password), emailed as a link — never email the password itself or a predictable token.
- Email verification: send a verification link on sign-up; treat `email_verified` as false until confirmed, and don't rely on an unverified email for account linking (§5.1).

## 6. OpenID Connect Federated Login

- Use the Authorization Code flow with PKCE for every provider, even confidential clients — PKCE is cheap and closes a real attack class.
- Use each provider's discovery document (`/.well-known/openid-configuration`) via `openid-client` rather than hardcoding endpoint URLs — providers rotate signing keys and occasionally endpoints.
- Validate the `state` parameter on callback (CSRF protection for the OAuth flow) and the `nonce` claim in the returned ID token (replay protection) — both are required, not optional hardening.
- Verify the ID token's signature, `iss`, `aud`, and `exp` — `openid-client` does this as part of token validation; don't hand-roll JWT verification.
- **Account linking**: look up an existing `identities` row by `(provider, subject)` first. If none exists, only auto-link to an existing local user by email when the provider asserts `email_verified: true` for that email — otherwise require the user to explicitly confirm the link (e.g. while logged in) rather than silently merging, since an unverified email claim is an account-takeover vector.
- Store per-provider identity as its own row (`identities`: `id`, `user_id`, `provider`, `subject`, `email`, `email_verified`) rather than columns bolted onto `users` — a user can have multiple linked providers.

## 7. Session & Token Management

- Default to server-side sessions: an opaque, random session ID in an `httpOnly`, `secure`, `sameSite=lax` (or `strict` if the flow allows) cookie, with the session record (user id, created/expires, metadata) in the database. This avoids the XSS-exposure risk of a JWT readable by client-side JS.
- If the project is a separate API consumed by a non-cookie client (mobile app, third-party integration), use short-lived JWT access tokens (10–15 min) plus a long-lived refresh token — store refresh tokens hashed in the database, rotate on use, and support revocation (deleting the stored token invalidates it; a stateless JWT alone cannot be revoked before it expires).
- Logout must invalidate the session/token server-side (delete the session row / revoke the refresh token), not just clear the client-side cookie — a copied cookie or token must stop working after logout.

## 8. RBAC — Role-Based Access Control

- Schema: `roles` (`id`, `name`), `permissions` (`id`, `name` — e.g. `posts:write`, `users:manage`), `role_permissions` (join table), `user_roles` (join table, supports a user having more than one role).
- Keep roles coarse-grained and few (confirmed with the user per §2) — RBAC answers "can this kind of user reach this kind of action at all," not fine-grained per-resource questions (that's ABAC, §9).
- Implement as middleware, applied per-route, not inline in controllers:
  ```js
  router.delete("/posts/:id", authenticate, requireRole("admin"), postController.remove);
  ```
- Look up permissions once per request (attached to `req.user` during `authenticate`), not with a fresh database query inside every route handler.

## 9. ABAC — Attribute-Based Access Control

- Use ABAC for the fine-grained checks RBAC can't express: ownership ("only the author can edit their own post"), relationship ("a manager can approve their direct reports' requests but not others'"), or context (time window, resource state).
- Implement as small, named policy functions that take `(user, resource, action, context)` and return allow/deny — not a generic rules-engine/DSL unless the project's policy complexity genuinely warrants one (confirm with the user before introducing that complexity):
  ```js
  // services/policyService.js
  export function canEditPost(user, post) {
    if (user.roles.includes("admin")) return true;
    return post.authorId === user.id;
  }
  ```
- Call ABAC policy functions from the controller/service after the RBAC middleware has confirmed the user's role can act on this *kind* of resource at all — RBAC is the coarse gate, ABAC is the fine-grained check on the specific instance.
- Keep policy functions pure and testable (no direct database calls inside them where avoidable) — pass in the already-loaded user and resource.

## 10. Database Schema (fills in the active persistence skill)

```sql
CREATE TABLE users (id, email UNIQUE, password_hash NULL, email_verified, created_at);
CREATE TABLE identities (id, user_id FK, provider, subject, email, email_verified, UNIQUE(provider, subject));
CREATE TABLE roles (id, name UNIQUE);
CREATE TABLE permissions (id, name UNIQUE);
CREATE TABLE role_permissions (role_id FK, permission_id FK);
CREATE TABLE user_roles (user_id FK, role_id FK);
CREATE TABLE sessions (id, user_id FK, expires_at, created_at);
```

- `password_hash` is nullable — a user who only ever signs in via an OIDC provider has no local password.
- Use whichever persistence skill is active in the project (`sqlite-persistence` by default) for the actual table definitions, migrations, and query style — this skill defines the auth-specific schema and logic, not a new data-access pattern.

## 11. Security Checklist

- Parameterized queries for every auth-related query — same non-negotiable rule as every persistence skill.
- PKCE + `state` + `nonce` on every OIDC flow (§6) — no exceptions, even for a "trusted" provider.
- Generic error messages on login/reset to prevent user enumeration (§5).
- Rate-limit auth endpoints separately and more tightly than general API endpoints.
- `httpOnly`, `secure`, `sameSite` cookies for sessions; never store a session ID or token in `localStorage`/`sessionStorage` where JS (and therefore any XSS) can read it.
- Redirect URIs for OIDC must be allowlisted exactly (no wildcard subdomains) in both the provider's app config and server-side validation of the `redirect_uri` used.
- Logout revokes server-side state, not just the client cookie (§7).

## 12. Testing

- Local auth: test registration, login (correct/incorrect credentials), password reset flow, and rate-limiting behavior.
- OIDC: mock the provider's discovery/token/userinfo responses (don't hit a real IdP in automated tests) and test both the new-user and account-linking paths, including the "email not verified — don't auto-link" case.
- RBAC: test that each role can/cannot reach each protected route.
- ABAC: unit-test policy functions directly with varied `(user, resource)` combinations — these are pure functions and the cheapest part of this system to get full coverage on.

## 13. Instructions for the Agent

1. Acknowledge that you are applying the `auth-identity` skill.
2. **Before writing any OIDC integration code, ask the user which identity providers to support.** Offer Google, Microsoft, LinkedIn, GitHub, and Apple as a starting menu, but do not assume any of them without confirmation, and do not silently add a provider that wasn't requested.
3. Before building RBAC/ABAC, confirm the actual role list and the specific fine-grained (ownership/attribute) rules the project needs — don't invent a generic `admin`/`user` split if the project needs something more specific, and don't over-build ABAC policies for checks the project doesn't have.
4. Keep authentication (§5–7) and authorization (§8–9) as separate middleware layers, always applied in that order (`authenticate` before `authorize`).
5. Use the project's active persistence skill for the actual table/migration/query style; this skill defines the auth domain model and logic on top of it.
