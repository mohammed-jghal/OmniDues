# Commercial reading — honest assessment

This document is the commercial half of the repository: what a buyer, a partner, or an investor should conclude from the engineering record. It is deliberately separate from the architecture and security documents, because engineering evidence and commercial readiness are different questions with different answers. Diagram 20 in [diagrams.md](diagrams.md) shows the scores.

## The desk assessment — 513 / 1000, and what it is

On 2026-11-05, OmniDues was scored as a B2B project-quality testing company would score it: 13 aspects, each out of 1000, unweighted mean. The result was **513 / 1000 overall**. The verdict: **FAIL for a production release; conditional PASS for a controlled pilot with stated limits only.**

What this assessment is: a desk assessment from the recorded gates, logs, and documents — not a live independent audit. No auditor ran the system, interviewed a user, or tested a live tenant, because no live tenant exists. The scores are therefore a reading of the evidence record, and they inherit its limits: they are as honest as the record, and no more.

What it is not: a market verdict, a price, or a prediction. It does not say the product cannot sell; it says the product as it stands on 2026-11-05 is not ready to be sold as a production system, and is ready to be piloted under limits.

## The 13 aspects

| Aspect | Score / 1000 | One-line reading |
|---|---|---|
| Functional completeness | 560 | The money cycle is built end to end; the live money loop is not |
| Test quality | 720 | A real gate with real coverage; the sweep timing is the open question |
| Security | 680 | Layered and tested; no external penetration test, no live KMS |
| Tenant isolation and data protection | 700 | RLS + composite keys + proven through PgBouncer; CNDP not cleared |
| Performance and scalability | 470 | Create p50 ~1,700 ms on demo hardware; no production-like measurement |
| Reliability and disaster recovery | 520 | Restore drill ~73 s; RPO unbounded, no backup encryption, no schedule |
| User experience | 600 | Two UX rounds built; the owner's verdict on round 2 is still owed |
| Architecture and maintainability | 640 | Clean modular monolith with strict layers; the create-path bottleneck is known |
| Compliance and governance | 380 | Binding decisions exist; counsel gates unopened, DSAR-vs-backup unresolved |
| Operational readiness | 350 | Five metrics, no alerting, no status page, no SLA, no live CI |
| Documentation | 700 | Extensive and honest — this repository is the evidence for the score |
| Commercial and market readiness | 150 | No pilot tenant, no first dirham, no paying tenant, no distribution route open |
| Integrations and payments | 200 | Built behind Tier 3 gates; nothing live — no bank feed, no PSP, no WhatsApp Cloud |

Arithmetic: 560 + 720 + 680 + 700 + 470 + 520 + 600 + 640 + 380 + 350 + 700 + 150 + 200 = 6,670. 6,670 / 13 = 513.07, stated as **513 / 1000**. The mean is unweighted — every aspect counts equally, so a strong engineering score cannot hide a weak commercial one. That is deliberate: weighting engineering more heavily would have produced a higher number and a less honest verdict.

## The two clusters — the real story

The 13 aspects fall into two clusters, and the gap between them is the story of this project:

- **Engineering cluster (functional, test, security, isolation, architecture, documentation): 666.7** — mean of 560 + 720 + 680 + 700 + 640 + 700 = 4,000 / 6. The system is well built, well tested, and honestly documented. A technical evaluator who reads the gate record will find a serious engineering effort.
- **Product / commercial cluster (performance, reliability, user experience, compliance, operations, commercial, integrations): 381.4** — mean of 470 + 520 + 600 + 380 + 350 + 150 + 200 = 2,670 / 7, **including user experience (600)**. The six non-UX aspects alone mean 345 (2,070 / 6) and range 150–520. The system is not yet a product that can be sold, operated, or trusted with live money at scale. No tenant, no payment path, no operations surface, no compliance clearance. An earlier version of this document stated the clusters as "~650" and "~200–350" and placed UX outside the commercial cluster in the text while diagram 20 placed it inside it; the figures above are the corrected, recomputed ones, and the text and the diagram now agree.

The same conclusion was reached independently by an external audit (Audit 2): "engineering-complete, not yet a product." Two independent readings agreeing is not proof, but it is a strong signal that the gap is real and not an artifact of one scorer's bias.

## What moves the score — in order

1. **A live money loop with a real tenant (commercial 150 → ~600 alone).** Onboarding → real invoices → real reminder → real reply → promise recorded → **real payment collected via a real path → renewal**. The first dirham collected, then the first renewal. Nothing else in this list moves the score as much, because nothing else proves the product does what it claims to do.
2. **A sealed 5.0.1 foundation — done 2026-11-12 (commit `e30aab9`, Entry 0263, owner decision D03).** The engineering scores assumed a sealed base; that item is now closed. What remains is everything below it.
3. **Buyer trust:** a live Trust Center, a contracting pack (MSA + SLA with remedies + signed DPA), a documented customer exit plan, and an external penetration test.
4. **Live SaaS operations:** a status page with incident communications, per-tenant SLA measurement and reporting, product analytics and customer-success signals, and load/soak testing on production-like infrastructure (not the 2-vCPU demo VM).
5. **Legal clearance as a pilot precondition:** the CNDP gate, counsel gate L.1, and a DPA — real debtor data without CNDP clearance is a legal liability, not a growth shortcut.
6. **Performance proof:** fix the create bottleneck (p50 ~1,700 ms) and the sweep timing under full load (369–566 s vs the 300 s bound, four measurements).

The acceptance checklist for the leap — sealed 5.0.1, one real pilot tenant, first dirham, first renewal, live payment path, CNDP cleared, live CI, live status page, penetration-test report, live Trust Center — would move the overall score from 513 to roughly **700–750**. That range is an estimate, labelled as such: the scores are a judgment, and the post-leap scores would be a new judgment on new evidence, not a calculation from this one.

## What does NOT move the score

- More lines of code. The tree grew by 6,006 lines between the 2026-10-27 and 2026-11-05 measurements and the score moved zero points. **The baseline behind that growth figure (406,967 lines) is not in any local record and is unverifiable locally** — the growth claim is retained as recorded, with its baseline flagged, not presented as a verified measurement.
- More documentation. This repository is already the highest-scoring aspect (700); more pages cannot fix a missing payment path.
- More features before the loop is proven. Breadth before a live money loop is the error this roadmap is designed not to repeat.
- Publishing the code. Full code publication would change verifiability for technical readers, not the commercial score — Moroccan distributors do not read GitHub; they buy trust, references, a working product, and support.

## The blocker, named plainly

The leap has no route to a real tenant today: direct outreach was cancelled by the owner, and the expert-partner path was dropped by the owner. The leap therefore starts with reopening a pilot route — not with more building, and no longer with the D03 decision, which was made 2026-11-12 (5.0.1 sealed, commit `e30aab9`). Until a pilot route reopens, the commercial score stays where it is, however much the engineering score improves. That is the hardest sentence in this repository, and it is stated here rather than buried in the roadmap.
