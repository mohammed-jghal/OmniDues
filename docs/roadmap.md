# Roadmap

Everything below is **planned, not done**. Items move by decision, not by date: the sequence holds, and scope is cut by explicit decision, never by silently dropping a gate. Exit criteria are stated where they exist, so progress is checkable instead of asserted.

**Current state: no pilot tenant exists yet, no first dirham has been collected, and no paying tenant exists.** Nothing below is presented as in progress beyond 5.0.1, which is sealed (commit `e30aab9`, Entry 0263, 2026-11-12 — see [evidence-and-gates.md](evidence-and-gates.md)).

## How to read a wave

Each wave below states four things: its **purpose** (what it proves), its **preconditions** (what must be true before it starts), its **contents** (what is built), and its **exit criterion** (the observable evidence that it is done). A wave with no exit evidence is not done, however much was built. A wave whose precondition is a legal or owner decision does not start by writing code — it starts by obtaining that decision.

## Wave 0b — pilot (planned)

**Purpose.** Prove the product with real users before any live money loop is built. The pilot answers one question: does a real collector, with real invoices and real debtors, get money back faster with this system than without it? Everything in the wave serves that question; nothing in it serves scale, distribution, or automation.

**Preconditions.** CNDP gate first — no real personal data enters the system before the data-protection declaration and counsel posture are in place. The tenant model already carries a declaration-number field; the gate is procedural and legal, not a code task. A pilot tenant must also exist: a real distributor willing to bring real invoices and to have a real collector work them. No pilot tenant is identified yet, and no outreach route to one is open — that is the live blocker for the whole roadmap, stated in the README.

**Contents.**

- **CSV import** for bringing a pilot tenant's invoices in.
- **Manual payment recording**, including cheque and commercial paper as recordable payment methods, with their full lifecycle.
- **Due-today list and invoice PDF** for the collector's daily work.
- **Draft-with-human-approval mode.** The engine drafts; the collector reviews; the collector sends. Every step is audit-logged. **There is no automated sending in this wave.**
- **Manual messaging as the default channel.** WhatsApp drafts via manual send links (the operator sends; the system never sends on its own). Email via a provider account is the second channel, with delivery rates measured per channel.

**Exit criterion for wave 0b:** a real message out → a real reply → a promise recorded → the **first dirham collected**. No first dirham means no Wave 1. The exit evidence is a recorded payment against a real invoice from a real debtor, not a demo payment and not a promise alone.

**Non-goals for 0b.** No live PSP payment, no live bank feed, no automated sending, no debtor portal, no self-serve trial. Any of those built during 0b would be scope drift, not progress.

## Wave 1 — live money loop (planned)

**Purpose.** Turn the proven pilot into a loop that collects money repeatedly and renews: a tenant pays for the system, uses it to collect, and pays again. The wave answers a second question: does the money loop sustain itself without the founder in the room?

**Preconditions.** Wave 0b exit (first dirham collected). Counsel gate for legal escalation (L.1). Live PSP / bank / WhatsApp contracts. CNDP cleared. D03 was decided 2026-11-12, so Wave 1 builds on a sealed foundation (commit `e30aab9`) rather than an uncommitted tree.

**Contents.**

- Live PSP payments and payment links.
- Live bank feeds.
- Legal escalation (after the counsel gate).
- Debtor portal v2, disputes handling, and self-billing.
- Live trust signals, a status page, and an SLA.
- Plus the deferred items: full UI deepening, the WhatsApp API upgrade as a second option (manual stays available), the cheque lifecycle, the audit-anchor design, remaining field-encryption decisions, PII-masking inventory, and a drilled incident-response exercise.

**Exit criterion for Wave 1:** paying tenants end to end, plus the **first renewal**. A renewal is the evidence that the loop works: a tenant who collected money with the system chose to pay for another period. One payment is a sale; a renewal is a loop.

## Wave 2 — distribution (planned)

**Purpose.** Acquire tenants through others — partners, accountants, and listings — instead of only through the founder's direct effort. The wave answers a third question: can someone who is not the founder sell this?

**Preconditions.** Wave 1 exit (paying tenants plus first renewal). Partner contracts in place. A public read-first API only when a real integrator asks for it — the API is built on demand, not on speculation.

**Contents.**

- Odoo Apps listing.
- Accountant channel.
- Partner portal.
- Self-serve trial.
- Product analytics.

**Exit criterion for Wave 2:** the first tenant acquired through a partner. That tenant's origin — a partner, not the founder — is the evidence, and it must be recorded as such.

## Later decisions — not planned builds

After the distribution wave, three decision points open, in this order of governance: network effects, financing (legal first — this is regulated activity, so a license or a licensed partner comes before any build), and scale or region. None of these is a build plan today.

## What the roadmap deliberately does not contain

The leap from a built system to a proven live money loop is a category change, not a version number. The roadmap therefore does not contain: a bigger line count as a goal; distribution before the loop is proven; financing before the legal gate; new features before the pilot exit. Breadth before the loop is proven is the error this roadmap is designed not to repeat. The acceptance checklist for that leap — sealed 5.0.1, one real pilot tenant, first dirham, first renewal, live payment path, CNDP cleared, live CI, live status page, external penetration test, live Trust Center — is the standard the waves above are measured against, and none of its items is claimed as done here.

## Decision log — D01–D06, where each one stands

| Decision | Question | State |
|---|---|---|
| D01 | PC readiness for trial and gates | Resolved in effect — the trial runs and the gates ran on the owner's PC |
| D02 | Docs-push presence (owner live for the approval card) | Open — this 2026-11-12 iteration is the push candidate for his line-by-line review; the push waits on his presence, plus the SECURITY contact (undecided). LICENSE is decided: proprietary, all rights reserved (2026-11-12). |
| D03 | Seal 5.0.1 with the documented exception? | **Decided 2026-11-12 by the owner** («seal it»): seal with the documented exception; commit `e30aab9`, Entry 0263 |
| D04 | Approve FINAL_PLAN rev2 + set X (10 paying tenants or first licensed finance partner) | Open — rev2 governs, X unconfirmed |
| D05 | Verdict on UX round 2 | Open — the merge proceeded per programme; his verdict is still owed |
| D06 | Pick from the six proposed additions (wrap-up card, promises-due strip, outcome history, keyboard-first queue, WhatsApp templates, send/share invoice) | Open — proposals only, he picks |

## What blocks each wave today

| Wave | Blocker, named plainly |
|---|---|
| 0b pilot | No pilot tenant identified, and no outreach route to one is open (direct outreach cancelled, expert-partner path dropped, both by the owner's decisions). Plus the CNDP gate, which is procedural and legal, not a code task. |
| Wave 1 | 0b exit (first dirham) has not happened; no PSP / bank / WhatsApp contracts; counsel gate L.1 unopened. (D03 is decided — 2026-11-12 — and no longer blocks Wave 1.) |
| Wave 2 | Wave 1 exit (paying tenants + first renewal) has not happened; no partner contracts. |

## Explicitly not in scope

- No automated sending before human approval in the pilot wave.
- No real personal data before the CNDP gate.
- No AI credit decisions, ever (binding architecture decision).
- No renaming of pinned cryptographic constants or protocols (the brand rename already proved this boundary: surface only).
- No production activation (Tier 3) without the owner accounts, counsel decisions, and owner decisions that gate it.
- No feature is built because it was requested in this document. Every item above waits for its gate.
