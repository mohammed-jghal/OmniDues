# OmniDues — New Diagrams (English)

Six new diagrams for the 5.0.1 seal, built 2026-11-12. English is the primary public tree; the Arabic parallel is `diagrams-ar.md` (same diagrams, Arabic labels, Latin node IDs).

**Status: 5.0.1 is SEALED** — commit `e30aab9`, EVIDENCE_LOG Entry 0263, 2026-11-12, by owner decision D03. The seal certifies the engineering build only; the strict B2B desk assessment scores the product 513/1000 — FAIL for production, conditional pilot only.

**Sources for every number below:** `docs/gates/EVIDENCE_LOG.md` Entries 0262 and 0263, and `FINAL_REPORT_501.md`. No number here comes from a summary or a memory. Where the record states a cause as unknown, the diagram states it as unknown.

**Demo data:** any product screen referenced by these diagrams shows demo data for the fictional tenant Demo Distribution SARL — never real customers.

---

## 1. Email verification lifecycle (migration 0073)

Registration is verification-gated. No session is issued at register time, and the register endpoint discloses nothing about whether an address exists (no 409 oracle).

```mermaid
flowchart TD
    A["Register: email + password"] --> B["No session issued"]
    B --> C["Verification code issued"]
    C --> D{"Verify code"}
    D -->|"Valid"| E["Account verified"]
    D -->|"Invalid or expired"| F["Rejected: no activation"]
    C --> G["Resend requested"]
    G --> H["Old code invalidated"]
    H --> C
    E --> I["Login gate opens"]
    I --> J{"Login attempt"}
    J -->|"Unverified"| K["403 EMAIL_NOT_VERIFIED"]
    J -->|"Verified"| L["Session issued"]
    M["Pre-0073 accounts: grandfathered"] --> I
    N["Consumed code is never revived"] -.-> H
    O["No activation without verification"] -.-> F
```

Sources: Entry 0263 — migration `0073_email_verification`; login refuses unverified accounts with 403 `EMAIL_NOT_VERIFIED`; grandfathering for pre-0073 accounts. Resend rules (old code invalidated on resend, consumed code never revived, no activation without verification) from the 501b targeted fix, device-proven green 3/3 in 14.65s.

---

## 2. The 5.0.1 seal chain

From the v5 seal to the 5.0.1 seal. Every gate in the chain is red on the sweep timing only; every sweep count is exact.

```mermaid
flowchart TD
    A["v5 SEALED<br/>5,192 passed / 0 failed in 3,398.02s<br/>Entry 0262, 2026-10-25"] --> B["Wave 0a<br/>commit 5d029b5<br/>Documented exception: sweep 384s vs 300s"]
    B --> C["Gate 501b<br/>3 failed / 5,298 passed in 1,229.37s<br/>2x register drift AS-API-0056, fixed 3/3 in 14.65s<br/>Sweep 566.89s"]
    C --> D["Gate 501c<br/>2 failed / 5,300 passed in 886.20s<br/>Sweep 378.04s + prop_fx tie<br/>Tie resolved by ADR-012 HALF_UP"]
    D --> E["Gate 501d<br/>1 failed / 5,301 passed / 11 skipped in 781.72s<br/>Sweep 369.65s vs 300s<br/>Counts 10,000/10,000, 0 failed"]
    E --> F["5.0.1 SEALED<br/>commit e30aab9, Entry 0263, 2026-11-12<br/>Authority: owner decision D03"]
    E -.->|"Timing only"| G["Documented exception<br/>Solo 101.95s PASS<br/>Reduced 42 tests in 125.46s PASS"]
    F -.-> H["Coverage at seal<br/>Overall 84.3 percent (20,525/24,358), floor 81.8<br/>Critical 96.1 percent (7,749/8,061), floor 85.4"]
```

Sources: Entries 0262 and 0263; FINAL_REPORT_501.md. The exception covers timing only — any functional failure reopens the seal. The sweep-timing cause is unknown (resource contention vs DB contention vs a scaling limit — undistinguished); holding the seal does not fix the timing.

---

## 3. CSRF and session flow

Bearer-first authentication with a cookie fallback, a double-submit CSRF token, and a global logout. Includes the 5.0.1 cookie-path fix.

```mermaid
flowchart TD
    A["Login with email + password"] --> B["wf_session cookie issued<br/>HttpOnly, SameSite=lax, Path=/api/v1"]
    B --> C["wf_csrf cookie issued<br/>Double-submit token"]
    C --> D{"Request authentication"}
    D -->|"Header: Bearer token"| E["Bearer accepted first"]
    D -->|"No header"| F["Cookie fallback"]
    E --> G["TOKEN_EPOCH = 2 enforced"]
    F --> G
    G --> H{"CSRF check on state-changing calls"}
    H -->|"Foreign Origin"| I["403: login CSRF"]
    H -->|"No Origin"| J["Documented programmatic exemption"]
    H -->|"Origin matches"| K["Request served"]
    L["Logout"] --> M["Global logout: token version bumped"]
    N["5.0.1 fix: wf_csrf path /api/v1 to /"] -.-> O["Browser login was 403 while API login worked<br/>Platform pages sit outside /api/v1"]
    P["Platform token: wf_platform HttpOnly cookie"] -.-> B
    Q["JSON access_token retained for Bearer programmatic clients only"] -.-> P
    R["MFA: 5 failures / 15 min per account"] -.-> A
```

Sources: Entry 0263 — security P1 fixes; the `wf_csrf` cookie-path fix (`/` instead of `/api/v1`); login CSRF (foreign Origin to 403; no-Origin as documented programmatic exemption); platform token moved to the `wf_platform` HttpOnly cookie; MFA per-account throttling (5 failures / 15 min, definer-function limiter).

---

## 4. Sweep load-test anatomy

One test, four in-gate timings, zero functional failures. The test enrolls 10,000 tenants' worth of collections work and must finish inside 300 seconds.

```mermaid
flowchart TD
    A["Sweep load test<br/>test_sweep_load_sanity_10k_enrollments<br/>10,000 enrollments, bound 300s"] --> B{"Where it runs"}
    B -->|"In the full gate"| C["501d: 369.65s<br/>FAIL on timing only<br/>Counts 10,000/10,000, 0 failed"]
    B -->|"Standalone"| D["Solo: 101.95s<br/>PASS"]
    B -->|"Reduced, -n 4"| E["42 tests in 125.46s<br/>PASS"]
    C --> F["In-gate series, same signature<br/>Wave 0a 384s<br/>501b 566.89s<br/>501c 378.04s<br/>501d 369.65s"]
    F --> G["Counts exact every time<br/>Zero functional failures in any run"]
    G --> H["Cause: UNKNOWN<br/>Resource contention vs DB contention<br/>vs a scaling limit, undistinguished"]
    H --> I["Documented exception<br/>Holding the seal does not fix the timing<br/>Any functional failure reopens the seal"]
```

Sources: Entry 0263; FINAL_REPORT_501.md; gate logs `device_gate_501b.log`, `device_gate_501c.log`, `device_gate_501d.log` (solo and reduced runs in `sweep_solo_501*.log`, `sweep_reduced_501*.log`).

---

## 5. Money invariants map

Money is Decimal and NUMERIC end to end. Two guards and one recorded divergence.

```mermaid
flowchart TD
    A["Money rule: Decimal in Python<br/>NUMERIC in PostgreSQL<br/>Binary floats for money are a review-blocking defect"] --> B["LEDGER_MAX guard<br/>LEDGER_MAX = Decimal('9999999999999999.99')"]
    B --> C["30-digit values overflowed the Decimal context<br/>and returned HTTP 500<br/>Now guarded before the ledger"]
    A --> D["FX consolidation<br/>Quantize per currency ROUND_HALF_UP<br/>before summation"]
    D --> E["ADR-012 amendment 2026-11-01<br/>Tie: 12.90 x 4.25 = 54.825<br/>HALF_UP 54.83 vs HALF_EVEN 54.82"]
    E --> F["Rationale: same figure must not round<br/>differently across engines<br/>Tax engine uses ROUND_HALF_UP"]
    A --> G["Penalty engine: HALF_EVEN<br/>Advisory, non-ledger figures only"]
    G --> H["Recorded divergence, not a contradiction<br/>Scope: FX consolidation only"]
```

Sources: Entry 0263 (gate fixes: `LEDGER_MAX` guard; ADR-012 amended to HALF_UP, original Decision sentence preserved verbatim as history); ADR-012 itself (exact tie 12.90 x 4.25 = 54.825; scope FX consolidation only; penalty HALF_EVEN untouched as a recorded divergence).

---

## 6. Pilot onboarding path

The designed route from registration to renewal. The email-delivery parts are designed, not live — the diagram marks them as such and does not hide the gap.

```mermaid
flowchart TD
    A["Registration<br/>Verification-gated, no session"] --> B["Email verification"]
    B --> C{"Email delivery"}
    C -->|"Brevo live"| D["Code delivered by email"]
    C -->|"Brevo NOT live"| E["DESIGNED, NOT LIVE<br/>Email delivery absent<br/>Operator-stamp path still to be defined"]
    D --> F["Account verified"]
    E -.->|"Decision needed before pilot onboarding"| F
    F --> G["First real reminder<br/>wa.me manual, human approval"]
    G --> H["Real reply from the debtor"]
    H --> I["Promise recorded"]
    I --> J["First dirham collected"]
    J --> K["Renewal<br/>Exit of the live money loop"]
    L["Pilot route today: no execution path<br/>Direct outreach cancelled 2026-10-27<br/>Expert-partner path dropped 2026-10-27"] -.-> A
```

Sources: Entry 0263 — registration/Brevo consequence (a brand-new self-registration cannot complete verification until email delivery exists or an operator-stamp path is defined); the 0b pilot design (wa.me manual as the default channel, draft-with-human-approval mode); Entry 0263 — the pilot route is currently without an execution path.

---

## Rendering note

Diagrams are Mermaid `flowchart` blocks, rendered natively by GitHub. Node IDs are Latin-only in both language files; only the labels change between `diagrams-en.md` and `diagrams-ar.md`. If a diagram fails to render, the first suspect is the `timeline`-style construct — none is used here; all six are `flowchart`.
