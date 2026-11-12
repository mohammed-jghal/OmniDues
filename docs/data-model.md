# Data model — deep dive

This document describes the data model as it stands at the v5 seal: **119 tables in 14 groups**, in a single models file of **4,140 lines**. 5.0.1 holds **120 tables** (migration 0073 adds email verification), sealed in commit `e30aab9`. Diagrams 15 in [diagrams.md](diagrams.md) shows the groups and their relationships. No table is listed by name here beyond what the record verifies — the full enumeration is private, and this document states counts and relationships, not the complete schema.

## Why one file

The whole schema lives in one file deliberately: a reader can see every table, every relationship, and every tenant boundary in one place, without chasing imports across modules. The cost is a 4,140-line file that is hard to edit safely — a change in one group can affect a foreign key in another, and the file's size makes that easy to miss. The mitigation is the boot discipline (the application refuses to boot on a migration-head mismatch) and the gate (every schema change runs the full suite). The trade was made for readability at review time, not for convenience at edit time.

## The 14 groups, and what each one holds

| Group | Tables | Holds |
|---|---|---|
| Tenancy and identity | 23 | Tenants, users, memberships, roles, sessions, impersonation records, token versions |
| Customers and tags | 3 | Debtor master data (including encrypted email / phone / ICE) and its labels |
| Invoices, credit notes, numbering, taxes, currencies | 9 | The money primitives: invoices, credit notes, numbering sequences, tax records, currencies |
| Payments, promises, mandates | 8 | Recorded payments, payment promises, debit mandates, allocations and reversals |
| Banks and PSP | 8 | Bank accounts, statements, payment-provider records, matching records |
| Collections | 7 | Playbooks, enrollments, steps, call outcomes, sweep state |
| Disputes and legal | 7 | Disputes (OPEN / RESOLVED / CLOSED only) and the legal-escalation ladder |
| Integrations | 6 | External-system bindings (ERP and others), with their gate states |
| Compliance, privacy, audit, security | 12 | Audit chain, consent records, retention settings, deletion certificates |
| Regional e-invoicing | 4 | ZATCA and DGI surfaces (ZATCA: specs and tooling; DGI: design only, frozen off) |
| AI and forecasting | 9 | Forecasting inputs and outputs — advisory only, never a decision input |
| Platform operations and documents | 6 | Operational records and generated documents |
| Credit | 8 | Advisory credit-signal records — counts and k-anonymized sums only, never a limit |
| Statements, digests, communications | 9 | Statement records, digest records, outbound-communication records (drafts, sends) |

The groups are not equal in weight: tenancy and identity (23 tables) and compliance / audit (12) together are 35 of 119 tables — nearly a third of the schema exists to answer "who are you, which tenant are you in, and what did you do?" That proportion is the design in numbers: identity and evidence are not features of the product; they are the product's foundation.

The groups also mirror the service boundaries: the collections tables are the tables the collections services touch, the banks tables are the tables the bank services touch, and so on. The mirror exists so a reader — or a change — can find the tables a subsystem owns without searching a 4,140-line file, and so a change to one subsystem's tables is visible as a change inside one group rather than scattered across the file. The group boundary is a blast-radius boundary: a mistake in the collections group cannot silently rewrite a bank row, because the tables it may touch are the ones in its own group.

## Relationships — the rules that hold the model together

- **Composite foreign keys.** Every child row references its parent with a composite `(tenant_id, X_id)` key. A child cannot point at another tenant's parent even if row-level security were bypassed — the schema itself refuses the write. This is the backstop behind the RLS backstop.
- **Derived, never stored.** Invoice balances are derived from primitives at read time. No table stores a balance as a writable field, so no table can hold a stale balance.
- **Never hard-deleted.** Financial records are corrected through adjustments and reversals, never deleted. The audit chain and the money history therefore tell the same story: nothing disappears.
- **Disputes gate collections.** A dispute row in OPEN state pauses the enrollment for its invoice; the sweep reads that state before advancing any step. The relationship is enforced in code and visible in the model: collections reads disputes, disputes do not read collections.
- **Consent gates credit.** A credit-signal read requires an explicit opt-in consent row naming its purpose; any other purpose is refused. The network gate (NOT_CLEARED until counsel clears it) sits above the consent rows — consent without clearance still refuses.

## The plans exception — the one table without FORCE RLS

118 of 119 tables at seal are under FORCE row-level security, with 729 logical policies (953 physical rows including per-partition copies). The single exception is the **plans** reference table: it carries no `tenant_id`, and the application role reads it only. It is reference data (the plan catalogue), not tenant data — there is nothing tenant-specific in it to leak. In 5.0.1 (sealed) the census is 119 of 120 tables (733 logical policies, 957 physical rows), and plans remains the single exception. A reader who finds a table without FORCE RLS should check whether it carries a `tenant_id`: if it does, that is a defect; if it does not, like plans, it is reference data by design. The distinction is the rule, not the count.

## Migration 0073 — the one-table change in 5.0.1

Migration 0073 adds email verification: one new table, taking the census from 119 to 120 tables. It is the only schema change in 5.0.1, and it is sealed in commit `e30aab9` (Entry 0263). The decisive gate 501d ran red on the sweep timing only (369.65 s vs a 300 s bound, counts exact) — that timing failure is the seal's documented exception, not a gap in the migration: a one-table migration sealed with a one-test timing exception is the smallest possible distance between "built" and "sealed", and this document states the distance rather than closing it by wording.

## What the model does not hold

- No stored balances, no stored credit limits, no stored risk scores. Anything that looks like a decision is either derived at read time or advisory-only.
- No plaintext PII for email, phone, ICE, or tax IDs — ciphertext at rest (AES-256-GCM), with blind indexes where search is required. Two fields remain plaintext by an undecided question: `tenant_entities.if_number` and `registration_number`.
- No production secrets, no API keys, no credentials of any kind. Configuration lives outside the database; the model holds business data, not access to other systems.
