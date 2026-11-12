# Evidence and gates

How to read this record: every number carries its denominator, and every state carries its date. Three states are never mixed — **sealed** (a complete gate passed), **committed** (built and committed, with any exception stated), and **in progress** (built but not through a green gate). A test count and a test-function count are different numbers and are never conflated: parametrization multiplies functions into collected items.

## What "green gate" means

A green gate means **0 failures and 0 setup errors**. A recurring flake is fixed by strengthening the test and diagnosing the mechanism — never by silently rerunning until the number passes. Coverage floors are fixed before measurement: **81.8% overall, 85.4% critical**. The critical set is defined before measurement and covers roughly a third of the code, measured by line coverage (no mutation testing is claimed for it).

What a green gate does and does not prove: it proves that the collected tests passed on the machine they ran on, against the code and schema at that commit, with coverage above the floors fixed before measurement. It does not prove that the product is correct for inputs the tests never exercised, that it is fast under production load, or that it is secure against an attacker the tests never modelled. The gate is a floor, not a ceiling — the minimum evidence for a seal, not a guarantee of a product.

Why the floors are fixed before measurement: a floor chosen after seeing the result is not a floor — it is a description of the result dressed as a standard. Fixing 81.8% overall and 85.4% critical before the gate runs means the gate can fail, and a gate that cannot fail proves nothing. The same discipline is why the bar is 0 failures and 0 errors rather than "few enough": an error count above zero means some tests never ran, so the pass count is a count of what happened to run, not of what the suite covers — a gate with errors is a gate with untested surface hiding behind a passing number.

## v5 seal — sealed

- **Seal gate: 5,192 passed / 0 failed, in 3,398.02 seconds.**
- **Coverage at seal: 84.3% overall (20,525 / 24,358 lines), 96.8% critical (7,573 / 7,827 lines)** — both above their floors.
- **Test corpus: 2,423 test functions in 236 Python files (83,844 lines), collected into 5,192 items** by parametrization. The two numbers are reported together, never as one. Method: `git archive` at the seal commit plus `wc -l` over the `.py` files only — fixtures and JSON data files in the test tree are not counted. At the 5.0.1 seal (commit `e30aab9`): 247 Python files, 87,859 lines, collected into 5,313 items.
- **API surface at seal: 296 paths / 363 operations** (committed OpenAPI snapshot). The live surface at seal counted 296 paths / 365 operations; the difference is exactly the two HEAD health operations, no path added or removed. At the 5.0.1 seal (commit `e30aab9`): **300 paths / 367 operations**, re-counted from the committed OpenAPI snapshot (`backend/tests/openapi_snapshot.json`) — verification endpoints added. The live count at 5.0.1 was not re-measured (the local environment lacks the database driver); the snapshot figure is the stated one, and the live figure is not asserted.
- **Schema at seal: 119 tables, 118 under FORCE row-level security, 729 logical policies (953 physical rows including per-partition copies), 71 migrations** (last: `0071_dpa_suite.py`). At the 5.0.1 seal: 120 tables, 119 under FORCE RLS, 733 logical policies (957 physical rows), 73 migrations (last: `0073_email_verification.py`). The physical counts are the `pg_policies` rows; they were not re-counted against a live database in this pass and are stated from the seal record and the test pins.
- **Permissions at seal: a 46-entry RBAC catalog**, with system roles OWNER, ADMIN, COLLECTOR, FINANCE, and VIEWER, plus custom roles that replace (never merge with) their system template.
- **Codebase size at the 5.0.1 seal (commit `e30aab9`): 412,989 lines in 1,483 tracked files**, measured by `git archive e30aab9` plus `wc -l` over every tracked file — a raw newline count, so it includes generated files such as `frontend/package-lock.json` and the newline bytes inside binary assets. An earlier working-tree figure (2026-11-05: 411,274 tracked lines / 412,973 total) **does not reproduce**: re-measured at `5d029b5` with the same method it is 407,587 tracked lines in 1,473 files. Cross-check: 407,587 + 5,403 (the seal commit's net line change) = 412,990, against 412,989 measured — off by one line, most likely a file without a trailing newline.

### Critical coverage by group (seal)

| Group | Coverage |
|---|---|
| Money | 95.9% |
| Collections | 98.7% |
| Tenancy / RLS | 96.4% |
| Banks | 94.2% |
| Auth | 96.6% |
| Legal | 96.6% |
| Credit | 97.5% |
| Safety | 100.0% |

The groups are read as follows: the lowest sealed group is Banks at 94.2%, still 8.8 points above the critical floor of 85.4%. Safety at 100.0% means every line in the safety set was executed by the gate — it does not mean the safety logic is proven correct for every input, because line coverage measures execution, not correctness. No mutation testing is claimed for any group. **Per-group denominators are not in the local record** — the percentages above are the seal gate's, as recorded in Entry 0262. Group denominators that appear elsewhere (for example Money 1,212 / 1,264) trace to the device gate-8 record below, which is not reproducible locally; they are not repeated here as seal figures.

### Device gate 8 (owner machine, after the seal) — device-logs-only

The eighth device run, after a chaos-test scoping fix that had been killing unrelated workers' connections, was reported fully green: **5,182 passed, 0 failed, 10 skipped, in 204.43 seconds**, with coverage 84.2% overall (20,503 / 24,358) and critical 96.8% (7,573 / 7,827 — identical to the seal figure). **These figures are device-logs-only: they are not found in any local primary record.** The local `gate_8.log` shows a different run (5,023 passed in 2,579.33 seconds). The figures are retained here because they were recorded from the device log at the time, with their arithmetic intact (20,503 is 22 lines below the seal's 20,525, a 0.1-point difference), but they cannot be independently recomputed from anything in this repository — a reader who wants to verify them needs the device gate log. Nothing else in this document depends on them.

## Wave 0a — committed, with a documented exception

Wave 0a committed as `5d029b5` (98 files): workers in Compose, httpOnly sessions with CSRF, XSS battery, row-level-security proof through PgBouncer, breach-corpus import procedure, restore drill, ICE / tax-ID encryption, and the backup-role fix.

The gate record for wave 0a, stated completely:

- The third gate run had **one failure, purely timing**: a 10,000-enrollment sweep load test ran **384 seconds against a 300-second bound**, with counts exact (**10,000 / 10,000**, zero failures). A solo rerun of the same test passed in about **102 seconds**.
- Coverage in that run was **84.2% overall and 96.6% critical**, above the 81.8% / 85.4% floors.
- The documented exception therefore reads: one timing-only failure with full numeric match and a fast solo pass, coverage above both floors, and **the exception is never reopened for any functional failure**.
- The earlier gate runs in the same wave are part of the record too: an initial red run (environment and setup failures, fixed one by one) and a second run with 4 failures, before the third run's single timing failure. The wave was committed by an explicit owner-delegated decision, recorded in the commit message together with the exception text. Wave 0a is **committed**, not sealed; v5 and 5.0.1 carry seal language.

### Restore drill (wave 0a)

A logical backup-and-restore drill on a demo database measured **about 73 seconds of restore time against a under-15-minute target, with row counts matching after restore**. Backup encryption is not shipped; the mechanism is an owner decision.

## 5.0.1 — SEALED 2026-11-12

5.0.1 is v5 plus wave 0a plus audit-finding fixes and a UX merge. Sealed: all three P0 fixes, all six security P1 fixes, correctness P1 fixes, and the UX merge (including a CSRF cookie-path fix found by the merge itself: the cookie path did not cover platform pages, so browser login failed while API login worked). Migration 0073 adds email verification; the sealed tree holds 120 tables, 119 of them under FORCE row-level security. **Seal: commit `e30aab9` ("Seal 5.0.1 (Entry 0263)"), Entry 0263 in the evidence log, by owner decision D03 given 2026-11-12.** The seal certifies the engineering build with one documented exception (the sweep timing, below); it does not certify commercial readiness — the desk assessment still reads 513 / 1000, FAIL for production.

### The 5.0.1 gate series — all four full gates, in order

| Gate | Result | Time | Sole / main failures |
|---|---|---|---|
| First device gate (5.0.1, before fixes) | **89 failed, 4,800 passed, 11 skipped, 412 errors** (5,312 collected) | 1,031.41 s | Shared test-helper failures (collapsed the 412 errors) plus two real product bugs: a 30-digit value overflowing the Decimal context (500), and a `Header(None)` object bypassing the key guard (500) |
| 501b | **3 failed / 5,298 passed / 11 skipped / 0 errors** (5,312 collected) | 1,229.37 s | 2 register-drift failures (fixed, then proven green in a targeted run: 3 passed in 14.65 s) + sweep timing 566.89 s vs 300 s, counts exact |
| 501c | **2 failed / 5,300 passed / 11 skipped** (5,313 collected) | 886.20 s | Register failures gone; sweep timing 378.04 s vs 300 s, counts exact; one new property-test failure on FX rounding (resolved by the ADR-012 amendment) |
| 501d (last) | **1 failed / 5,301 passed / 11 skipped / 0 errors** (5,313 collected) | 781.72 s | Sweep timing only: 369.65 s vs 300 s, counts exact (10,000 / 10,000, 0 failed) |

Coverage stayed green through the series: first gate 83.93% overall (20,916 / 24,921) / 95.2% critical (critical denominator not in the local record); 501b 84.3% overall (21,020 / 24,930) / 96.1% critical (7,749 / 8,061); 501c 84.3% / 96.1% (denominators not in the local record — Entry 0263 states the percentages only); 501d 84.3% overall (20,525 / 24,358) / 96.1% critical (7,749 / 8,061). The first gate's coverage was also green even while the gate itself was red — coverage and pass/fail are independent measures, and a green coverage figure never cancels a red gate.

**Anomaly, stated honestly (A7).** The 501d overall coverage — 20,525 / 24,358 — is numerically identical to the v5 seal figure despite added code between the two measurements. The 501d aggregate was re-run from `container /tmp/cov501d.json` on the gate machine; that file is not available locally, so the figure cannot be independently recomputed here. It is reproduced faithfully from Entry 0263 and flagged as an anomaly, not corrected by inventing a different figure: if the aggregate was computed against a stale coverage file, the true 501d overall coverage is unknown, and the seal's coverage claim for 501d rests on the recorded aggregate alone. The critical figure (7,749 / 8,061) is shared with 501b and is not anomalous in the same way.

What the series shows, read honestly: the functional failures went down monotonically (89 + 412 errors → 3 → 2 → 1), and the last remaining failure in every gate after the first was the same sweep timing test. The fixes worked. The timing did not move with them, which is the evidence that the timing failure is a load behaviour, not a functional defect hiding behind the fixes.

### The sweep series — the same test, four measurements

| Measurement | In-gate sweep time | Bound | Counts | Solo rerun | Reduced rerun |
|---|---|---|---|---|---|
| Wave 0a gate 3 | 384 s | 300 s | 10,000 / 10,000, 0 failed | about 102 s, pass | — (not recorded) |
| 501b | 566.89 s | 300 s | exact, 0 failed | 140.14 s, pass | 145.59 s, pass |
| 501c | 378.04 s | 300 s | exact, 0 failed | 95.40 s, pass | 133.93 s, pass |
| 501d | 369.65 s | 300 s | 10,000 / 10,000, 0 failed | 101.95 s, pass | 42 / 42 in 125.46 s, pass |

Four consecutive in-gate failures against the same 300-second bound, with exact counts every time and a solo pass every time. Holding the seal does not fix the timing; the pattern is a load behaviour, not a functional failure, and it is recorded as such. What is not known — and is stated as not known — is why the sweep is slower under full-gate load than alone: the candidates are resource contention on the gate machine, database contention from parallel workers, or a genuine scaling limit in the sweep itself. No measurement in the record distinguishes between them, so no cause is claimed here.

**D03 was decided by the owner on 2026-11-12** («seal it»): 5.0.1 is sealed with the documented exception, recorded as commit `e30aab9` and Entry 0263. The decision had been live since 2026-11-01 and was the owner's alone to make. The exception covers the sweep timing only; any functional failure reopens the seal.

### ADR-012 amendment (2026-11-01, committed in the 5.0.1 seal)

ADR-012 was amended so foreign-exchange consolidation quantizes per currency with ROUND_HALF_UP before summation. The original decision text is preserved verbatim as history and the amendment is appended openly. The penalty engine's documented HALF_EVEN rounding for advisory, non-ledger figures is untouched; the divergence is recorded. The amendment is committed in the seal (commit `e30aab9`).

### The two real product bugs the first 5.0.1 gate found

The first gate's 89 failures were mostly test-helper failures, but two were real product bugs, and they are recorded because a gate that finds real bugs is doing its job:

1. A 30-digit value overflowed the Decimal context and returned a 500. Fixed with a ledger guard (`LEDGER_MAX`, 9999999999999999.99) that rejects the value before it reaches the context.
2. A `Header(None)` object bypassed the key guard and returned a 500. Fixed so the guard requires a real non-empty string.

Both fixes were verified in the gate-fix run (2,194 green across 67 files, 0 failures) before the 501b gate ran.

## Load test — demo hardware only, not production capacity

Measured 2026-10-25 on a 2-vCPU demo virtual machine, single tenant, concurrency 20:

- **About 9 invoice creates per second** (9,985 successful creates).
- Create latency: **p50 about 1,700 ms, p95 about 3,300 ms, p99 about 4,300 ms** — the create path is the bottleneck.
- Read latency per page: **p50 about 213 ms** (515 rows) to about 219 ms (10,000 rows).
- **11 × HTTP 500 in about 10,000 creates (about 0.11%)**, from database statement timeouts; the source was not found in application code and remains an open follow-up.
- The plan wall works as designed (200 × 402 responses at the GROWTH 5,000/month cap) — that is a plan enforcement result, never quoted as performance.

These numbers describe demo hardware. They are not production capacity, and no load measurement on production-like infrastructure exists yet.

How to read the load figures: the read path scales flat (213 ms at 515 rows, 219 ms at 10,000 rows — the page cost barely moves with table size), while the create path is slow at every percentile (p50 1,700 ms means half of all creates took longer than 1.7 seconds). That split points at the create path itself — derived-balance computation, row locking, allocation, audit writes — not at the database size. The 0.11% of 500s (11 in about 10,000) is small in percentage and large in meaning for a money system: every one of those creates failed outright rather than recording a wrong balance, which is the fail-closed behaviour, but the source was not found in application code, so the failure mode is recorded as open rather than explained away.

## Open findings and recorded decisions

| Item | State |
|---|---|
| F-6.9-4 — breach-corpus breadth | OPEN. The import procedure is delivered; the large-corpus import has not been run. |
| F-6.9-5 — CAPTCHA friction | OPEN as a recorded non-adoption: CAPTCHA requires an external service and account; no provider chosen. |
| SSDF practices (42) | 20 evidenced, 14 by unit, 8 gap-with-disposition. The eight are gaps *with a disposition*, not open gaps without a position. |
| npm audit / veraPDF | Blocked by network egress (npm audit) and not run (veraPDF). Not claimed. |
| Live CI | Not run. Deployment manifests are written and reviewed, not proven live. |
| Production SLSA | None claimed. |
| Incident-response plan | Drafted, undrilled. No simulated exercise, no operator alerting. |
| DSAR vs backup retention | Needs counsel opinion (L-Q5) and a written policy. |
| Backup encryption | Not shipped. The mechanism is an owner decision. |
| RPO | Unbounded by default; no backup schedule ships. |
| Load testing at distributor scale | Measured on demo hardware only (see above). Not presented as production capacity. |
| 25,000-test plan | A stated ambition, not a measured commitment. The measured commitment is the current green gate (5,192) and the continuous gate loop. |

## What is deliberately not claimed

- No live tenant, no live money movement, no collected revenue, no renewal, no pilot tenant.
- No production deployment. The only running copy is a trial on the owner's PC (loopback only).
- No production security certification (ISO 27001 / SOC 2 superset is a design bar, not a held certificate).
- No external penetration test, no live KMS, no live object-lock storage.
- No CNDP clearance, no Brevo account, no CMI / PSP / bank contracts.
- No hand-written-production-code claim (see README: execution by agents under direction, judgment, and verification).

## How to read a gate log yourself

A gate log is not a single number — it is a record with four counts and a duration. Read it in this order:

1. **Errors first.** Any error count above zero means the gate did not run cleanly — setup failed, a fixture broke, or the environment was wrong. A gate with 412 errors (the first 5.0.1 gate) is not "89 failures" — it is 89 failures *plus* 412 things that never ran. Fix errors before reading failures.
2. **Failures second.** Each failure names a test. Read the test name: a timing failure (`test_sweep_load_sanity_10k_enrollments`) is a different kind of news from a functional failure (a wrong balance, a leaked row). The 501d gate has exactly one failure, and it is the timing test — that is why the gate is described as "red on the sweep timing only" and not as "red".
3. **Passed and skipped third.** 5,301 passed is the weight of the gate; 11 skipped is the weight of what was deliberately not run. Skipped tests are not passed tests — if the skip count jumped, ask why.
4. **Coverage last.** Coverage answers "how much code did the tests execute", not "did the tests pass". The first 5.0.1 gate was red with green coverage (83.93% / 95.2%) — coverage never cancels a failure.

## What would change the verdict

The verdicts in this document are not permanent. Each one has a condition that would change it:

- **v5 sealed** changes if a defect is found in the sealed tree that the gate did not catch — the seal would stand as a historical record, and the fix would be a new version with a new gate, not a silent edit to the seal.
- **Wave 0a committed-with-exception** changes only if a functional failure is found in the wave — the exception is never reopened for a functional failure, so a functional failure would reopen the wave itself.
- **5.0.1 sealed** changes only if a functional failure is found in the sealed tree — the exception covers timing only, so a functional failure reopens the seal, and the fix would be a new version with a new gate, not a silent edit to the seal record. The sweep timing changes when a measurement distinguishes the cause, as above.
- **The sweep timing** changes when a measurement distinguishes the cause — resource contention, database contention, or a scaling limit — and a fix is measured against the same 300-second bound under full-gate load, not alone.
- **The load-test verdict** changes only with a measurement on production-like infrastructure. A faster demo VM would not change it; the hardware is the variable, not the code alone.

## Arithmetic check — the seal coverage, recomputed

Overall: 20,525 / 24,358 = 84.26% (stated 84.3%). Critical: 7,573 / 7,827 = 96.75% (stated 96.8%). Floors: 81.8% overall (margin +2.5 points), 85.4% critical (margin +11.4 points). The margins are the point: the seal passed with room, not on a rounding edge. A reader who recomputes these numbers from the denominators will get the same figures — that is what "every number carries its denominator" means in practice.
