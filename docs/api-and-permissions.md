# API and permissions — deep dive

This document describes the API surface and the permission system as they stand at the v5 seal. Diagram 14 in [diagrams.md](diagrams.md) shows the surface; diagram 13 shows the roles. At the 5.0.1 seal (commit `e30aab9`): **300 paths / 367 operations**, re-counted from the committed OpenAPI snapshot — verification endpoints added; the live count was not re-measured (see [evidence-and-gates.md](evidence-and-gates.md)).

## The surface — 296 paths, 363 operations at the v5 seal; 300 paths, 367 operations at the 5.0.1 seal

At the v5 seal, the committed OpenAPI snapshot records **296 paths / 363 operations**. The live surface at seal counted 296 paths / 365 operations; the difference is exactly the two HEAD health operations — no path added or removed, just the two HEAD variants the live server exposes and the snapshot does not list. Stating both numbers, with the reason for the difference, is the point: a reader who counts the live surface and gets 365 is not finding a discrepancy; they are finding the two HEAD operations this paragraph names. At the 5.0.1 seal the snapshot records **300 paths / 367 operations** — four paths and four operations added, each one operation: the CSRF token endpoint (`GET /api/v1/auth/csrf`), email verification (`POST /api/v1/auth/verify-email`), verification resend (`POST /api/v1/auth/verify-email/resend`), and platform logout (`POST /api/v1/platform/auth/logout`); nothing was removed. The live count at 5.0.1 was not re-measured, so no live figure is asserted here.

What 296 paths means in practice: the API is wide. It covers customers, invoices, payments, collections, disputes, legal escalation, bank import and matching, reconciliation, credit-signal reads (advisory only), integrations, platform operations, and documents. Breadth is not depth — 296 paths is a count of endpoints, not a measure of how well each one works. The gate record (5,192 passed / 0 failed at seal) is the evidence for how well they work; the path count is only the size of the surface the gate covered.

## Permissions — the 46-entry RBAC catalog

At seal, permissions are a **46-entry RBAC catalog**. Five system roles exist: **OWNER, ADMIN, COLLECTOR, FINANCE, VIEWER**. Custom roles are also supported, with one rule that defines the whole system: **a custom role replaces its system template entirely — it never merges with it.** Unknown permissions are rejected at validation.

Why "replaces, never merges" matters: in a merging system, a custom role built from the VIEWER template would silently inherit every VIEWER permission, including ones added later — a template change would quietly widen every custom role built from it. In a replacing system, a custom role holds exactly the permissions it declares, and nothing else. The cost is verbosity (every permission must be listed); the benefit is that a role's power is exactly what its definition says, and a template change cannot widen it by accident. For a money system, that trade is correct: a permission granted by accident is worse than a permission that must be typed out.

Per-endpoint authorization: each endpoint declares its required permission or role, and the check happens before any domain work. A request that fails authorization never touches a money rule, never reads a balance, and never writes an audit entry for the attempted action itself — it is refused at the gate, not inside the vault.

## The five system roles, in one line each

| Role | What it can do |
|---|---|
| OWNER | Everything within the tenant, including role management and the gates that other roles cannot open |
| ADMIN | Tenant administration, without the owner's exclusive gates |
| COLLECTOR | The daily collections work: queue, call outcomes, promises, drafts for review |
| FINANCE | Money work: payments, adjustments, bank matching confirmation, reconciliation |
| VIEWER | Read-only — sees the data, changes nothing, and sees masked fields where masking applies |

Field masking applies to VIEWER and to masked custom roles: some fields are hidden or masked for those roles even though the row itself is visible. Row-level security decides *which rows* a user can see; masking decides *which fields* inside a visible row they can see. The two layers are independent, and both must pass for a field to be shown.

## Authentication on the API — Bearer first, cookie second

Two credential forms reach the API, in a defined order: **Bearer first (exact "Bearer"), cookie fallback.** Programmatic clients use Bearer tokens; browser sessions use httpOnly cookies (`wf_session`, SameSite=lax, Path=/api/v1 for the API cookie — and, since the 5.0.1 CSRF fix, the CSRF cookie path is `/` so it covers platform pages outside `/api/v1`). A state-changing browser request carries a double-submit CSRF token (`wf_csrf`). Logout is global: a token-version bump kills every token, everywhere.

The Bearer-first order matters for one failure mode: a request carrying both a Bearer token and a cookie is judged by the Bearer token. A stale cookie cannot override a valid Bearer token, and a valid cookie cannot rescue a revoked Bearer token. The order is the rule; without it, two credentials would mean two possible answers to "who is calling".

## Rate limits

Rate limits apply per IP and per token per minute, and per registration email per hour. They sit early in the request path so abusive traffic is refused before domain work executes. In 5.0.1 (sealed), MFA gains per-account throttling: 5 failures in 15 minutes, enforced by a definer function. What rate limits do not do: they do not stop a distributed attempt (per-IP limits do not bind a botnet), and they do not distinguish a legitimate burst (a collector working fast) from an abusive one (a script hammering login). Those limits are stated in the security document as residual risk, not hidden.

## Deprecation and versioning

Deprecation headers follow RFC 8594; the deprecation log is empty — no endpoint is currently deprecated. The API version string `3.0.1-rc1` is internal lineage, not a product claim. When an endpoint is deprecated, the header will say so and the log will record it; until then, "no endpoint is deprecated" is a fact about the current surface, stated with its date (the v5 seal), not a promise about the future.

## What the API will never do

- Allocate money from a bank match on its own (human confirmation is required).
- Send a message on its own (draft-with-human-approval only).
- Return a credit limit, hold, or block (signals are advisory reads only).
- Serve a row from another tenant (fail-closed: a missing tenant matches nothing).
- Accept an unknown permission in a custom role (rejected at validation).

The five refusals share one principle: **the API is a gate, not an actor.** Every refusal names a decision the API will not make on anyone's behalf — allocating money, contacting a debtor, judging credit, crossing a tenant boundary, or granting a permission nobody defined. Each of those decisions belongs to a human (confirming a match, sending a draft, setting a limit) or to a gate that has not opened yet (counsel, CNDP, Tier 3). An API that made those decisions itself would be fast and wrong at machine speed; an API that refuses them is slow in exactly the places where slowness is the product's safety. The refusals are the boundary between what the system does and what it exists to let a human do.
