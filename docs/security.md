# Security

Security posture, stated as built — with the open items stated alongside, not omitted. This document describes sealed v5 plus wave 0a. The 5.0.1 security fixes are sealed (commit `e30aab9`, Entry 0263, 2026-11-12): all six security P1 fixes from the 5.0.1 audit are sealed, including verification-gated registration and per-account MFA throttling.

## Posture in one paragraph

The database is the first line of defence. Enforcement is layered, never single-point: per-endpoint permission checks, then row-level security as the backstop for direct data access, then field masking for viewer and masked custom roles. Authentication invalidates aggressively (any credential, role, or MFA change kills prior tokens), money moves only through idempotent, locked, atomic operations, and every claim in this document traces to code, a gate log, or a record.

## Threat model — who we defend against, and what we do not claim

| Attacker | What they can try | What stops them here | Residual risk, stated plainly |
|---|---|---|---|
| A user of tenant A trying to read tenant B's data | Guessing IDs, omitting the tenant header, calling an endpoint with another tenant's ID | Permission check, then row-level security, then composite foreign keys — three layers, any one of which fails closed | None claimed as zero: the layers are tested (RLS proof 6 / 6 through PgBouncer in wave 0a), but no external penetration test has been run, so an unknown bypass is not ruled out |
| A stolen session or token | Replaying a Bearer token or a session cookie after the user changed a credential | Token-version invalidation: any credential, role, or MFA change bumps the version and kills prior tokens; logout is a global version bump | A token stolen and used before any change is made still works until that change — there is no per-request revocation list |
| A hostile input in a name, note, or document | Stored XSS through a debtor name or a generated document | XSS battery (hostile-input cases against rendered output, including URI-scheme annotations and hostile names in generated documents); CSP derived from the configured API origin with a validated-origin guard | `script-src 'unsafe-inline'` remains in the CSP (documented); a production CSP proof has not been run |
| An attacker on the network path | Intercepting or tampering with traffic to the API or to an external service | HSTS and pinned-transport egress (HTTPS only, public addresses only after DNS resolution) | No production deployment exists, so production TLS configuration is unproven |
| A breached password reused from another site | Credential stuffing against login | Per-IP and per-token rate limits; per-account MFA throttling (5 failures / 15 minutes, sealed in 5.0.1); breach-corpus checking procedure (see below) | The large breach corpus itself has not been imported (F-6.9-4, OPEN); CAPTCHA was not adopted (F-6.9-5, recorded decision) |
| An operator or platform admin abusing break-glass access | Impersonating a tenant and changing security settings | Impersonation is pinned to exactly one tenant; security modifications under impersonation tokens are forbidden; every impersonation action is stamped with a platform-admin actor in the audit chain | Break-glass access is still powerful by design — attribution exists, prevention of a determined insider does not |

What this threat model deliberately does not cover: physical access to the owner's PC (the trial runs there); the security of the owner's email account (password-reset delivery depends on it); and the legal-compliance layer (CNDP, counsel gates), which is a procedural gate, not a code defence.

## Authentication and sessions — the full lifecycle

- **Registration (5.0.1, sealed).** Verification-gated: no session is created on register, no 409 oracle leaks whether an email exists, and login refuses an unverified account with `403 EMAIL_NOT_VERIFIED`. Existing accounts are grandfathered. The consequence is stated in the README: until live email delivery exists or an operator-stamp path is defined, a brand-new self-registration cannot complete verification.
- **Login.** Bearer JWT with token-version invalidation. Any credential, role, or MFA change invalidates all prior tokens for that user.
- **Multi-factor authentication** with backup codes. Per-account throttling in 5.0.1 (sealed): 5 failures in 15 minutes, enforced by a definer function.
- **Platform-admin impersonation is pinned and limited.** An impersonation token is bound to exactly one tenant, and security modifications under impersonation tokens are forbidden.
- **View-as sessions are read-only by construction**, through a dedicated middleware, with a hash recorded per access and session invalidation.
- **Wave 0a sessions: httpOnly cookies with CSRF protection.** Session cookies are HttpOnly and SameSite-lax; state-changing requests carry a double-submit CSRF token. Bearer tokens remain for programmatic clients only. Logout is global (a token-version bump).
- **Rate limits apply per IP and token per minute, and per registration email per hour.** The limits sit early in the request path so abusive traffic is refused before domain work executes.

The lifecycle, end to end: a user registers (no session until verified), verifies (once email delivery exists), logs in (JWT issued with the current token version), optionally completes MFA, works under a tenant context, and logs out (version bump kills every token, everywhere). If an administrator changes the user's role mid-session, the next request fails on the version check — the session does not linger with stale permissions. That is the point of version invalidation: permissions are checked against the current record, not against the moment the token was issued.

### 5.0.1 security fixes — sealed, listed completely

All six security P1 fixes from the 5.0.1 audit are sealed in 5.0.1 (commit `e30aab9`, Entry 0263):

1. Impersonation default-deny.
2. Platform token moved to an httpOnly cookie (`wf_platform`); the JSON `access_token` is retained for Bearer programmatic clients only.
3. Verification-gated registration (migration 0073), as above.
4. Reset-token hygiene: fingerprint-only mail log, and dev-token exposure behind an explicit opt-in.
5. MFA per-account throttling (5 failures / 15 minutes).
6. Login CSRF: a foreign Origin returns 403; a missing Origin is a documented programmatic exemption (see Residuals below).

### Why each fix exists — the mechanism behind the list

1. **Impersonation default-deny.** The old behaviour allowed an impersonation path that forgot to set its limits to act with the caller's full power. Default-deny inverts the failure: a path that forgets its limits now gets nothing, and the omission is visible as a refusal instead of silent over-privilege. The rule is fail-closed at the exact point where a future developer is most likely to forget.
2. **Platform token in an httpOnly cookie.** A JSON `access_token` lives in JavaScript-accessible storage, so any cross-site scripting hole can read it and exfiltrate it. An httpOnly cookie cannot be read by JavaScript at all — the theft surface moves from "any script that runs" to "a stolen cookie replayed before invalidation", which token-version invalidation then bounds. Bearer is retained for programmatic clients because they have no browser to hold a cookie; the two credential forms exist because the two client kinds have different theft surfaces.
3. **Verification-gated registration.** A session created at registration would let an account act before proving it owns the email it registered with — an attacker could register as a victim's address and act in their name. No session until verification closes that window. No 409 oracle for the same reason in reverse: confirming "this email exists" hands an attacker an enumeration oracle for the user base, so registration answers the same way whether the email exists or not.
4. **Reset-token hygiene.** A full reset token in a mail log is a credential sitting in a log file — anyone who can read the log can reset the account. A fingerprint (a hash prefix) lets an operator correlate a reset request with its log entry without the log ever holding anything that can be used as the token. The dev-token opt-in exists for the same reason: a token printed to a console in development must never be printable in production by default.
5. **MFA throttling per account, not just per IP.** Per-IP limits are defeated by rotating source addresses — a botnet has as many IPs as it wants. A per-account limit binds the throttle to the target being attacked, so no number of source addresses buys more guesses against one account. Five failures in fifteen minutes is tight enough to make guessing impractical and loose enough that a real user who mistypes is not locked out of their own account.
6. **Login CSRF.** A login form posted from another origin is the CSRF shape: the victim's browser carries the session, the attacker's page supplies the request. A foreign Origin is proof the request did not come from the product's own pages, so it is refused outright. The missing-Origin exemption is the documented cost: programmatic clients send no Origin header, so refusing all no-Origin requests would break them — and a browser request with a stripped Origin would pass the same exemption. That cost is stated in Residuals rather than hidden, because the exemption is a hole with a reason, not a hole nobody noticed.

## Row-level security

- At the v5 seal: 118 tables under FORCE row-level security, 729 logical policies (953 physical rows including per-partition copies). In 5.0.1 (sealed): 119 of 120 tables under FORCE row-level security (733 logical policies, 957 physical rows); the plans reference table remains the single exception (it carries no `tenant_id`, and the application role reads it only).
- The tenant identity is applied as a transaction-local setting at every transaction start; a missing tenant matches nothing (fail-closed).
- Composite foreign keys (`tenant_id`, `X_id`) prevent cross-tenant parent references at the schema level, independent of policy enforcement.
- Wave 0a proved isolation **through PgBouncer**: session settings are transaction-local, and isolation holds with the pooler's server-reset query emptied. Migrations and backups connect directly to the database, never through the pooler.

Why transaction-local matters: PgBouncer in transaction-pooling mode hands the same server connection to different clients between transactions. If the tenant context were set once per session, a later transaction on the same connection could inherit the previous tenant's identity. Setting it transaction-local — at the start of every transaction — means the context cannot leak across transactions even when the underlying connection is shared. The wave 0a proof (6 / 6) tested exactly this: isolation held with the pooler's server-reset query emptied, which is the configuration where a leak would have shown.

## PII encryption

- Customer email and phone are stored as **AES-256-GCM ciphertext** behind a key-provider abstraction, with blind indexes for search. Production boot fails without a configured key.
- Wave 0a extended the same treatment to customer ICE and to entity tax IDs: ciphertext at rest, no blind index where search is not required.
- Key providers are environment, cloud KMS, or vault — selected by configuration, never defaulted to a usable secret. No usable secret ships in any example configuration.

What is encrypted, and what is not: email, phone, ICE, and tax IDs are ciphertext at rest. Amounts, dates, invoice references, and playbook state are not PII and are stored in plaintext — encrypting them would break the queries the product runs on every page (aging, queue ranking, reconciliation) without adding protection, because the values themselves are not identifying. Two fields remain plaintext by an undecided question, stated here so it is not hidden: `tenant_entities.if_number` and `registration_number` are still plaintext, and the decision to encrypt them has not been made.

## HTTP hardening

Shipped security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, a content security policy of `default-src 'none'; frame-ancestors 'none'`, HSTS, and `Cache-Control: no-store`.

Wave 0a built an **XSS battery** (hostile-input cases against rendered output, including URI-scheme annotations and hostile names in generated documents) and derived the browser connection policy from the configured API origin with a validated-origin guard. A production CSP proof remains an open item — it is not claimed here.

The battery, in plain terms: the test suite feeds hostile strings — script tags, javascript: URIs, hostile names — into every place user input is rendered, and asserts that the output is escaped or rejected rather than executed. 38 / 38 cases passed in wave 0a. The battery proves the rendering layer handles the cases it was written for; it does not prove that no hostile input exists that it was not written for. That is why the CSP proof is still open, and why `script-src 'unsafe-inline'` remaining in the policy is stated rather than hidden.

## Egress control

All outbound connections pass a pinned-transport guard: HTTPS only, and public addresses only after DNS resolution. Anything else is refused. There is no ambient outbound access from application code. The guard exists because the application fetches URLs that configuration supplies — webhook targets, integration endpoints, e-invoicing channels — and a configured URL that points at an internal address would be server-side request forgery: the server itself fetching the metadata service, the database, or another tenant's internal service. Resolving the DNS name and requiring a public address closes that shape before the connection is made, and HTTPS-only means no credential or token ever travels in plaintext to a fetched endpoint. The guard is a single choke point, so a new integration cannot accidentally open its own outbound path around it.

## Audit chain

- Audit records live in append-only tables with a hash-chained head: each entry hashes the previous hash together with its framed fields, so any tampering breaks the chain from that point forward.
- Chain verification walks from the genesis entry, recomputing every hash, and is available as an operator tool plus an export endpoint.
- Impersonation actions are stamped with a platform-admin actor, so break-glass access is attributable.

How verification works, step by step: start at the genesis entry (the first entry, whose previous hash is the defined genesis value); recompute the hash of each entry from its framed fields plus the previous hash; compare with the stored hash; move to the next entry. The first mismatch names the exact entry where the chain broke, and every entry after it is untrusted — because each later hash was computed over the tampered value. Tampering is therefore detectable, not preventable: the chain proves that a record was changed, it does not stop someone with write access from changing it. Prevention of that write is the database-role separation (the application role owns nothing; only the migrator owns the schema).

## Backup and restore

- The backup role is select-only (it bypasses row-level security for reads, which is exactly what a backup needs, and can write nothing).
- A restore drill on a demo database measured **restore time of about 73 seconds against a under-15-minute target, with row counts matching after restore**. Backup encryption is not shipped — the mechanism is an owner decision, not a silently missing feature. See [evidence-and-gates.md](evidence-and-gates.md) for the drill record.

The drill, in plain steps: take a logical backup of the demo database (168 tables, 10,000 invoices); restore it into a fresh database; measure the restore time (71.4 seconds, plus 1.8 seconds for the backup itself — about 73 seconds total); compare row counts after restore (matching). The target was under 15 minutes, so the drill passed with wide margin. What the drill does not prove: that a backup taken under production load would restore as fast, that an encrypted backup would restore at all (none exists), or that a backup taken on a schedule would exist when needed (no schedule ships — RPO is unbounded by default).

## Breach-corpus checking (HIBP)

- A documented, fail-closed breach-corpus import procedure exists: binary search over a sorted digest corpus, union with (never replacement of) existing data, and closed failure on any error.
- The breadth of the corpus itself remains open (finding F-6.9-4): the procedure is delivered, the large-corpus import is an operational step that has not been run. It is not presented as complete.

The procedure's three rules, and why each exists: binary search over a sorted corpus (so a lookup is fast even over hundreds of millions of digests); union, never replacement (so importing a new corpus cannot delete the breach data already held); fail-closed on any error (so a failed import never leaves the system believing a breached password is safe). The corpus itself — about 17 GB at 847 million digests in the documented procedure — has not been imported, because the import is a large operational step, not a code step. Until it runs, breach checking covers only the small seed corpus (9,227 lines).

## What is NOT yet live — Tier 3 gates

These are built but deliberately not activated. Activation requires owner accounts, counsel decisions, or owner decisions — never a code change alone:

- Live bank feeds, live PSP payments, WhatsApp Cloud, live email delivery, and live e-invoicing credentials (including ZATCA production credentials and DGI activation, which is frozen off at schema level).
- External penetration testing (a zero-exit gate must open first).
- Live KMS, live object-lock storage, and hosted AI/OCR services.
- Email verification for new self-registration (see README: until live email delivery or an operator-stamp path exists, a new self-registration cannot complete verification).

Why these are gates and not code switches: a live integration needs three things no code change can supply — an owner account with the provider, a contract or terms the owner has accepted, and (where the law requires it) counsel clearance. A code switch would let a deploy silently activate a connection whose account, contract, or clearance does not exist; a gate makes the missing piece visible and names who must supply it. That is why the gate list is written as "built, not activated" rather than "not built": the engineering is done, and the remaining work is legal and commercial, not technical — and the document refuses to let a technical reader mistake one for the other.

## Residuals — the exact limits of the 5.0.1 fixes

These are the limits of the sealed 5.0.1 security work, stated so no reader over-reads the fixes:

- `APP_ENV` unset does not fail closed globally.
- A missing Origin is a documented programmatic exemption from login CSRF (programmatic clients send no Origin; the exemption is deliberate, and its cost is that a browser request with a stripped Origin would also pass it).
- Impersonation has a direct-call skip without a Request object.
- The platform JSON token is retained for Bearer programmatic clients.
- There is no `/verify-email` frontend page (no new UX was built in 5.0.1); verification completes through the API path only.

## Open items and recorded decisions

- **F-6.9-4 (breach-corpus breadth): OPEN**, as stated above.
- **F-6.9-5 (CAPTCHA friction): OPEN as a recorded non-adoption.** CAPTCHA was not adopted because it requires an external service and account; no provider has been chosen. This is a decision, not an oversight.
- **Incident-response plan: drafted, undrilled.** No simulated exercise and no operator alerting configuration exist yet.
- **Live CI: not run. Production SLSA: none claimed.** The deployment manifests are written and reviewed, not proven in a live pipeline.
- **DSAR vs backup retention: needs counsel.** Deletion certificates and retention policy settings exist, but the interaction between subject-rights deletion and backup retention requires a counsel opinion (L-Q5) and a written policy before it is presented as settled.
- **Frontend monitoring inventory, application-log retention policy, and operator alerting: not present in code.** Stated here so no reader assumes them.

## A session attack, walked through

An attacker steals a session cookie from a collector's browser. What happens, step by step:

1. The attacker presents the cookie. The session is valid — the cookie itself is genuine — so the request is authenticated. Cookie theft is not prevented by httpOnly (httpOnly stops JavaScript from reading the cookie; it does not stop a stolen cookie from being replayed). This is stated plainly because httpOnly is often over-claimed.
2. The attacker tries to change a security setting. Under a normal session this would succeed if the collector's role allowed it — the session carries the collector's permissions. Under an impersonation token it is forbidden outright. The defence here is role scope, not the cookie.
3. The collector changes their password. The token version bumps. The attacker's replayed session dies on the next request — the version check fails. This is the real defence against a stolen session: aggressive invalidation, not the cookie flags.
4. The collector logs out. Global version bump — every token dies everywhere, including the attacker's copy.

The lesson, stated without softening: the system's session security rests on version invalidation and on short-lived trust in the token, not on the cookie being unstealable. A stolen cookie works until the user acts. That window is the residual risk, and it is why MFA throttling and rate limits sit in front of login rather than behind it.

## What an external penetration test would test first

No external penetration test has been run — the zero-exit gate must open first. When one does run, these are the first targets, stated here so a reader can see what is unproven rather than assume it is proven:

1. Tenant isolation under a hostile caller: omitting the tenant header, guessing another tenant's IDs, and calling endpoints with a second tenant's ID while authenticated as the first.
2. The CSRF exemption: a browser request with a stripped Origin against the login endpoint (the documented programmatic exemption).
3. The `script-src 'unsafe-inline'` CSP entry, and whether a stored hostile input can execute despite the XSS battery.
4. The impersonation direct-call skip (a call without a Request object).
5. Rate-limit behaviour under a distributed login attempt (per-IP limits do not stop a botnet).
6. The backup role: a select-only role that bypasses RLS is powerful by design — what happens if its credential leaks?

Each item is a test to run, not a vulnerability claimed. Listing them is the honest form: the system has not been attacked by a professional yet, and this document does not pretend otherwise.

## Key rotation — what is documented, and what is not

Key providers are environment, cloud KMS, or vault, selected by configuration, never defaulted to a usable secret. What is **not** documented in the record: a key-rotation procedure, a rotation schedule, or a re-encryption procedure for existing ciphertext under a new key. That absence is stated here as an open item, not filled in by invention: until a rotation procedure exists and is tested, a compromised key has no documented recovery path in this repository. Boot failing without a configured key is a start-up guard, not a rotation plan.
