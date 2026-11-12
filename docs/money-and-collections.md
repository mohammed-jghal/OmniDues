# Money and collections — deep dive

This document goes deep on the two halves of the product: the money rules that must never be wrong, and the collections engine that turns an overdue invoice into a collected payment. It is the companion to [architecture.md](architecture.md) (the shape) and [diagrams.md](diagrams.md) (diagrams 4, 6, 7, and 8). Nothing here is sealed that the evidence document does not seal; 5.0.1 is sealed (commit `e30aab9`, Entry 0263), and its only deviation is the documented sweep-timing exception.

## Part 1 — Money

### The three primitives

Every money fact in the system is built from three recorded primitives:

- **A — amount.** The invoice amount, in the invoice currency, as Decimal / NUMERIC. Never a float.
- **P — payments.** Recorded payments allocated to the invoice, each with an amount, a date, and an optional reference.
- **C — adjustments.** Recorded credit notes and adjustments against the invoice.

The balance is always **B = A − P − C**. It is derived at read time, never stored as a writable field. There is no fourth primitive and no manual override: a collector cannot type a balance, and an administrator cannot set one. If the primitives are right, the balance is right; if they are wrong, the correction is a new recorded primitive (an adjustment or a reversal), not an edit to the balance.

### The invoice state machine

Diagram 6 in [diagrams.md](diagrams.md) shows the states. In words:

- An invoice is created as **DRAFT**, then issued to become **OPEN**. A draft or cancelled invoice passes through as-is — its status is not derived from the balance, because it was never a live claim.
- **OPEN** means issued with a balance above zero and no payment or adjustment recorded yet.
- **PARTIALLY_PAID** means at least one payment or adjustment has been recorded, and the balance is still above zero.
- **PAID** means the balance is at or below zero. At or below, not exactly zero: an overpayment closes the invoice rather than leaving a one-dirham remainder that no one can collect.
- **OVERDUE** is not a separate financial state — it is OPEN or PARTIALLY_PAID past the due date with a balance above zero. The queue ranks overdue invoices by days overdue, so the oldest debt is worked first.
- A financial record is **never hard-deleted**. A wrong payment is reversed by a recorded reversal; a wrong invoice is corrected by a recorded adjustment or credit note. The history stays complete, which is what makes the audit chain meaningful: a deleted record would break the story the chain tells.

### Payment recording — why each step exists

Diagram 7 shows the sequence. Each step has a reason:

1. **Request fingerprint.** A retried request — a double-click, a network retry, a browser refresh after a slow response — carries the same fingerprint and returns the original result instead of creating a second payment. Without it, the most ordinary user behaviour (clicking twice) would double-charge a debtor.
2. **Row lock (`FOR UPDATE`).** Two collectors recording payments against the same invoice at the same moment cannot both read the old balance and both allocate against it. The lock serialises them: the second waits, then reads the balance the first one left.
3. **Ledger guard.** In 5.0.1, a 30-digit value overflowed the Decimal context and returned a 500. The fix is a guard (`LEDGER_MAX`, 9999999999999999.99) that rejects the value before it reaches the context — a loud rejection at the boundary, not a silent wrong number inside.
4. **Atomic allocation.** The payment insert and its allocation to the invoice commit together or not at all. A payment without an allocation would be money recorded nowhere; an allocation without a payment would be money allocated from nothing. Atomicity forbids both.
5. **Audit entry.** The payment is written to the hash-chained audit trail in the same transaction, so the money movement and its proof cannot diverge.

### Rounding — the two rounding rules, and why they differ

There are two rounding rules in the system, and they apply to different kinds of figures:

- **FX consolidation (ledger figures): ROUND_HALF_UP, per currency, before summation.** Amended by ADR-012 on 2026-11-01, committed in the 5.0.1 seal (commit `e30aab9`). The rule quantizes each currency's amount first, then sums — so the sum is a sum of rounded figures, not a rounding of a sum. The original ADR-012 text is preserved verbatim as history; the amendment is appended openly.
- **Penalty engine (advisory, non-ledger figures): HALF_EVEN.** Untouched by the amendment. Penalty figures are advisory computations, not ledger entries, and their rounding rule is documented separately.

The divergence is recorded, not hidden, because a reader who finds HALF_EVEN in the penalty code and HALF_UP in the FX code would otherwise conclude the codebase is inconsistent. It is consistent in a subtler way: ledger figures round one way, advisory figures round another, and the boundary between them is the ledger. A figure that enters the ledger must use the ledger rule; a figure that never enters it need not.

### What money the system will never move on its own

- No payment is allocated automatically from a bank match. A match is a suggestion; a human confirms it.
- No credit signal becomes a limit, a hold, or a block. Signals are advisory reads only.
- No outbound correspondence is sent without human approval. The engine drafts; a human sends.
- No invoice is written off by the system. A write-off, if it ever exists, is a recorded human decision, not a sweep action.

The four refusals share one principle: **the system proposes, a human disposes.** An automated error moves real money before anyone notices — a wrong match allocates a real payment, a wrong send contacts a real debtor, a wrong signal blocks a real customer — while a human error is one transaction, visible in the audit chain, reversible by a recorded correction. Automation in this product is therefore confined to drafting, scoring, and suggesting, and every step that changes money or contacts a debtor waits for a human decision recorded in the system. The cost is speed: nothing happens while no human is watching. The benefit is that no error, human or machine, can move money at machine speed.

## Part 2 — Collections

### Playbooks, enrollments, and the sweep

A **playbook** is a sequence of steps with conditions: contact the debtor, wait, escalate the tone, offer a payment plan, and so on. Two seeded playbooks ship with the demo data: Standard (5 steps) and Ferme (7 steps). They differ in step count and conditions, not in the rules below — the rules apply to every playbook.

An **enrollment** is one invoice in one playbook, advancing step by step. The **sweep worker** drives enrollments: it wakes on its schedule, takes the enrollments due for advancement, checks each one against the tenant's timezone and quiet hours, checks for an open dispute, advances the step, and records what it did. The sweep is the engine; the collector is the driver. The sweep never sends a message, never records a payment, and never closes an invoice — those are human actions.

### Quiet hours and timezones

Quiet hours are per-tenant, and the sweep respects them: no step advances inside quiet hours, even if the invoice is badly overdue. Timezones are per-tenant too, so "today" for a promise, "due today" for the queue, and "inside quiet hours" for the sweep are all computed in the tenant's local date, not in the server's timezone. A system that reminded a debtor at 3 a.m. because the server was in another timezone would be a system that burns the relationship it exists to save.

### Call outcomes — the five outcomes, and the two rejections

When a collector calls a debtor, the outcome is one of five: **PROMISE, PAID, NO_ANSWER, REFUSED, CALLBACK**.

- **PROMISE** requires a promised date inside the legal window: today through today + 180 days, in the tenant's local date. A past date is rejected. The window exists because a promise without a date is not a commitment, and a promise dated a year out is not a collection plan — 180 days is the bound the product enforces.
- **PAID** requires a recorded payment. A PAID outcome without a recorded payment is rejected, and the collector is pointed at the payments page: record the payment first, then mark the outcome. The order matters — the money record is the truth, and the outcome is the note about it, not the other way around.
- **NO_ANSWER, REFUSED, CALLBACK** carry no such precondition, but each is recorded with the invoice, the debtor, and the time, so the next collector sees the full history instead of starting cold. A CALLBACK without a date is rejected in the same spirit as a PROMISE without one: a callback that no one scheduled is a callback that never happens.

### Disputes — the pause that cannot be skipped

Opening a dispute pauses the active enrollment automatically, and the sweep skips enrollments with an open dispute. Resolving a dispute resumes the enrollment only when no open dispute remains. A paused-for-dispute enrollment is never silently resumed.

The rule exists for a commercial reason as much as a technical one: chasing a debtor who has opened a legitimate dispute — a wrong amount, a damaged delivery, a missing credit note — destroys trust and can destroy the commercial relationship. The system therefore treats a dispute as a stop sign, not as a note: the machinery halts for that invoice until a human resolves the dispute. The cost is that a debtor could open a dispute to buy time; the product accepts that cost, because the alternative (chasing through a genuine dispute) is worse.

### Bank matching — suggestion, never decision

Statements import with a per-entry fingerprint (so a file imported twice is recognised as a duplicate), parser logs, and an account-binding check. Candidates are scored on amount, reference, and counterparty-name overlap. Confirming, rejecting, or resolving a match is a human operator action gated by permission. **Money is never allocated automatically.**

The scoring is deliberately described without weights in this document: the record states the three factors (amount, reference, counterparty-name overlap) and does not state a formula, because no formula is verified in the record. A reader should not assume equal weights, or any particular threshold — the human confirmation is the control, not the score.

### The legal ladder

The legal ladder runs from penalty rule sets (activation requires owner-level review) through stored penalty computations to evidence-hashed escalation packs and debt acknowledgements. Each rung is gated: penalty rules do not activate without review; escalation packs are evidence-hashed, so the pack a debtor receives is the pack the system can prove it sent; debt acknowledgements are recorded, not assumed. The ladder is a legal process with a technical record, not a technical process with legal language pasted on it — and its activation is counsel-gated (L.1), so no rung is climbed before counsel clears it.

Why each rung is gated: a penalty rule that activates without review is a legal claim against a real debtor, computed by a formula nobody approved — the review exists because the claim's consequences are legal, not technical. An escalation pack that is not evidence-hashed is a letter the debtor can deny receiving and the system cannot prove it sent; the hash exists so the pack is provable in a dispute, not merely sent. And the counsel gate sits above all of it because escalation is legal action — a demand letter, a debt acknowledgement — and legal action taken by software without counsel is a liability the product would carry for the tenant. The gates are the product refusing to practice law by accident.

### The demo numbers, traced

Demo tenant Demo Distribution SARL, fictional data: **14 invoices, open 415,150.00 MAD across 11 invoices, overdue 326,500.00 MAD across 8 invoices.** Aging: **88,650 + 44,300 + 124,350 + 44,100 + 113,750 = 415,150 MAD.** The queue ranks the 8 overdue invoices by days overdue. The treasury forecast on the dashboard is a projection from those primitives, not a promise of collection — it shows what would be collected if the promises held, and the promises are human commitments recorded in the system, not system guarantees.

## Worked numerical examples — check the arithmetic yourself

All figures are illustrative (MAD), using the rules above — not demo data, so a reader can verify the rules without the demo tenant.

**Example 1 — partial payment.** Invoice amount A = 100,000. Payment P = 40,000. Adjustment C = 10,000. Balance B = 100,000 − 40,000 − 10,000 = **50,000**. Status: **PARTIALLY_PAID** (a payment exists, balance above zero). A second payment of 50,000 brings B to 0 → **PAID**.

**Example 2 — overpayment.** A = 100,000, P = 105,000, C = 0. B = −5,000, at or below zero → **PAID**. The invoice closes; the 5,000 overpayment is a recorded fact, not a remainder the queue keeps chasing.

**Example 3 — draft passes through.** A draft invoice for 80,000 with no payments has B = 80,000, but its status is **DRAFT**, not OPEN — drafts and cancelled invoices pass through as-is and are never derived from the balance.

**Example 4 — FX rounding (HALF_UP, per currency, before summation).** Two foreign-currency amounts consolidate: currency X quantizes to 1,000.005 → 1,000.01 (half rounds up); currency Y quantizes to 2,000.004 → 2,000.00. Sum = 3,000.01. The rounding happens per currency *before* the sum — rounding the unrounded sum instead would give a different figure, and the ADR-012 amendment exists precisely to fix which order applies. Penalty figures, by contrast, use HALF_EVEN and never enter the ledger — a reader who finds both rules in the code is looking at two different kinds of figures, not an inconsistency.

**Example 5 — promise window.** Tenant local date today is day T. A promised date of T + 180 is accepted; T + 181 is rejected; T − 1 (yesterday) is rejected. The window is computed in the tenant's local date, so a promise made at 23:00 in the tenant's timezone is not judged by the server's date.

**Example 6 — dispute pause.** An invoice 60 days overdue is on playbook step 3. A dispute opens. The sweep skips the enrollment on its next run — step 3 does not advance, no reminder is drafted, and the queue still shows the invoice (with its dispute visible) rather than silently dropping it. The dispute is resolved with no other dispute open: the enrollment resumes at step 3, not at step 1 — the pause froze the state, it did not reset it.
