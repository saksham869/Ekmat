# DRUNIX Hackathon Proposal

**Challenge:** CHL-7007 — Drunix Hackathon in collaboration with Citi
**Organiser:** India Blockchain Forum
**Problem Statement:** 4 — Financial Inclusion
**Submitted by:** Satyam Mishra · B.Tech CSE, KIET Group of Institutions, Ghaziabad (AKTU), 2027
**Repository:** https://github.com/saksham869/Ekmat

> Each section below maps to a field on the submission form.

---

## Proposal Title

Ekmat: A Shared Event Ledger for Co-Lending Dispute Adjudication on DRUNIX

---

## Problem Understanding

The Reserve Bank of India (Co-Lending Arrangements) Directions, 2025 (RBI/DOR/2025-26/139, issued 6 August 2025, effective 1 January 2026) created a reconciliation surface and did not create anything to reconcile it on.

Read four provisions together.

**Para 25** requires each regulated entity to maintain the borrower's account individually for its respective share. **Para 26** routes every disbursement and repayment through an escrow account. **Para 31** requires each entity to report to credit information companies separately, for its own share of the same loan. **Para 33** requires that if either entity classifies its exposure as SMA or NPA, the same classification applies to the other, with information shared on a near-real time basis and in any case latest by end of the next working day.

One borrower. One loan. Two lenders. Two sets of books. Two bureau submissions. And a regulatory obligation that both parties reach the same answer within a day.

The Directions name no lead lender and no system of record. Para 13 requires a single point of interface with the customer, but that is a servicing obligation, not a shared ledger. Nowhere does the text say whose version of the loan's history is authoritative when the two diverge.

They do diverge. The Indian Institute of Banking and Finance surveyed 16 banks and 20 NBFCs and documented partners running different loan management systems, taking one to two quarters to integrate with each new partner, with IRAC and Ind-AS divergence producing conflicting classification positions on the same exposure. Prashant Kumar, Managing Director and CEO of Yes Bank, said in May 2025: *"I think everybody is struggling. Fundamentally, it means your underwriting standards should be aligned with those of the originator."* Executives at UGRO, Tyger Capital, Sa-Dhan, ICRA and Vivriti have raised inconsistent classification across partners on record.

The borrower absorbs it. When two lenders file contradictory positions on one loan, the credit bureau records both, and a dispute appears on a file the borrower did not cause and cannot trace to a source.

This runs on a base of over Rs 1.1 lakh crore of NBFC co-lending AUM as at 31 March 2025, growing 35 to 40 percent annually, per CRISIL Ratings.

Co-lending exists to channel bank capital to borrower segments that NBFCs reach and banks do not. That is a financial inclusion instrument. When its reconciliation layer fails, the cost lands on exactly the borrowers the arrangement was built to serve.

Primary source: https://www.rbi.org.in/Scripts/NotificationUser.aspx?Id=12888&Mode=0

---

## Solution Description

Ekmat is a shared, append-only event ledger for a co-lending arrangement. Both lenders keep their own books exactly as Para 25 requires. What they share is the sequence of events those books are derived from.

It is not a replacement for either loan management system and it is not a payments rail. It is the adjudication layer the Directions assume exists and do not provide.

**One agreed event sequence.** Every servicing event on the loan, from disbursement tranche to repayment to escrow appropriation to classification change, is written once, co-signed by both lenders, and ordered. Neither party can later assert a different sequence, because neither party holds the log alone.

**Convergence before bureau submission.** Para 33 requires both lenders to reach the same classification within a working day. Ekmat makes that a state transition on the shared ledger rather than a file exchange. One lender proposes a classification change; the other endorses it or raises a challenge with its own evidence on the same log; the bureau submission is gated on convergence. A contradictory pair of submissions cannot leave the system, because the submission reads from the agreed state rather than from either party's private view.

**Adjudicable disputes.** When the parties disagree on escrow appropriation order, interest apportionment, or the date a payment landed, the dispute is resolved against a co-signed ordered record that neither wrote alone. A supervisor inspecting the arrangement reads the same log.

What this is not: a claim that a ledger creates trust between the parties. They already have a contract. The asset here is non-repudiable event ordering. When a bank and an NBFC submit conflicting DPD on one loan, the dispute is whose timestamp was right, and a co-signed append-only log makes that question answerable.

Full network topology, data model, endorsement policy and the Para 33 state machine:
https://github.com/saksham869/Ekmat/blob/main/docs/ARCHITECTURE.md

Event walkthroughs, phase plan, testing strategy and demonstration script:
https://github.com/saksham869/Ekmat/blob/main/docs/WORKFLOW.md

---

## Implementation Approach

### What already exists

**Regulatory reporting pipeline.** During a paid global internship with the Mifos Initiative I built CB-ILD inside openMF, an open source core banking platform used by institutions serving underbanked segments. It ingests financial records, validates each against a regulatory taxonomy, and handles the rejections, retries and disputes that come back. Eleven REST APIs in Java 21 and Spring Boot 3, three-role access control, audit logging on every write, eleven database migrations. Six pull requests merged into the production codebase, each reviewed by senior maintainers. Publicly verifiable at https://github.com/openMF

**Append-only ledger primitives.** Arbiter implements a hash-chained log with immutable audit storage, idempotent writes keyed by request so a retry or duplicate processes exactly once under concurrent load, and a policy engine that authorises state transitions against versioned rules before they commit. Seventy passing tests, including an adversarial suite of nine attacks covering concurrent duplicates, replay and gate bypass. All nine blocked, running on every commit. https://github.com/saksham869/Arbiter

### Platform

DRUNIX, NPCI's open source distributed ledger, described in its own README as an enhanced fork of Hyperledger Fabric. https://github.com/npci/drunix

Permissioned MSP membership is the correct model: a co-lending arrangement has a known, contractual counterparty set, not an open network. Channels per arrangement and private data collections let the two lenders share the event sequence without exposing either party's wider book. And DRUNIX supports Java chaincode, which I verified against the platform's own source, where the supported-platforms registry in `core/chaincode/platforms` contains a fully implemented Java platform in v1.0.0. That means the contract logic is written in the language I have shipped production code in.

I will be straightforward that Java chaincode on DRUNIX is inherited from Fabric rather than documented by NPCI, and every DRUNIX sample I found uses Go. Phase 0 exists to retire that risk before anything depends on it.

### Phased build

**Phase 0 — De-risk the platform (Days 1 to 2)**
Deliverable: a DRUNIX network running locally with trivial Java chaincode deployed.
Done when: a Java contract commits a transaction and reads it back on a multi-org network. If Java proves unworkable, this fails fast and the build moves to Go, with two days lost instead of two weeks.

**Phase 1 — Network and data model (Days 3 to 5)**
Deliverable: four organisations, Bank, NBFC, Escrow Bank and Regulator, with a channel per arrangement and the loan event schema.
Done when: each org reads the shared sequence and no org can write on another's behalf.

**Phase 2 — Co-signed event append (Days 6 to 9)**
Deliverable: servicing events appended only on endorsement by both lending parties, keyed idempotently.
Done when: a duplicated or replayed submission appends exactly once under concurrent load, demonstrated by a test firing simultaneous identical submissions. This is the critical correctness milestone and is scheduled early for that reason.

**Phase 3 — Classification convergence (Days 10 to 13)**
Deliverable: the Para 33 state machine. One party proposes an SMA or NPA transition, the other endorses or challenges, and the arrangement holds a single agreed classification with the next-working-day deadline enforced in contract logic.
Done when: a divergent classification attempt is rejected at the chaincode, and an unendorsed proposal past the deadline raises a breach event visible to the Regulator org.

**Phase 4 — Gated bureau submission (Days 14 to 16)**
Deliverable: submission reads from the agreed state rather than either party's private view, using the validation and exception handling already built in CB-ILD.
Done when: two lenders cannot produce contradictory submissions on one loan, and an injected format rejection is tracked to resolution rather than dropped.

**Phase 5 — Dispute adjudication and demonstration (Days 17 to 20)**
Deliverable: an escrow appropriation dispute raised, adjudicated against the ordered log, and resolved.
Done when: a full arrangement lifecycle runs end to end and the adversarial suite passes against the integrated system.

Phases 0 to 4 are the minimum viable path. Phase 5 is separable and absorbs slippage.

### Testing

Tests ship with each phase, not after. Concurrency tests fire genuinely simultaneous submissions rather than sequential ones, because concurrency defects do not appear in tests that run one request at a time. Fault injection on the event path: dropped endorsements, duplicate submissions, out-of-order arrival, events for an arrangement already closed. Chain verification runs as a test, so tamper-evidence is asserted rather than assumed.

Nine adversarial scenarios are specified, each with a required outcome, in
https://github.com/saksham869/Ekmat/blob/main/docs/WORKFLOW.md

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Java chaincode may not work on DRUNIX | Phase 0, two days, fail fast. Go fallback identified. |
| DRUNIX's SQL state database has no demonstrated Java path | The design does not depend on it. Phase 5 optimisation, not a dependency. |
| No live co-lending counterparty available to a student team | Two simulated lender organisations against synthetic loan data modelled on the Directions' requirements. No production deployment claimed. |
| Scope exceeds the window | Phases 0 to 4 ordered first. Phase 5 separable. |

### The objection I expect

A signed append-only log in a conventional database would deliver much of this. That is fair, and I would rather raise it than be caught by it.

**The answer is custody, not cryptography.** A signed log in one party's database is still that party's log. In a co-lending arrangement the two parties have opposed incentives on NPA timing and collection sequencing, which is precisely why neither should host the record the other is bound by. Dual endorsement with a Regulator org as observer gives an ordering neither party can unilaterally revise and a supervisor can inspect directly.

I would also note what killed the previous generation of consortium ledgers. we.trade, TradeLens, Marco Polo and Contour failed on getting many counterparties onto one platform, not on technology. A co-lending arrangement is two parties with a pre-existing contract and a regulatory obligation to agree. That is a materially easier adoption problem, and it is why I chose this use case rather than a broader one.

---

## Technology Stack

**Distributed ledger:** DRUNIX (NPCI), an enhanced fork of Hyperledger Fabric. Permissioned MSP membership, four organisations, channels per arrangement, private data collections, Raft ordering. https://github.com/npci/drunix

**Smart contract layer:** Java chaincode, validated against DRUNIX's supported-platforms registry in v1.0.0.

**Application layer:** Java 21, Spring Boot 3, REST APIs, role-based access control, audit logging on every write.

**State and reporting:** PostgreSQL for read models and projections, Liquibase migrations, and the regulatory validation and exception-handling pipeline carried over from CB-ILD. The ledger is authoritative; if the projection disagrees, it is rebuilt from events.

**Integrity primitives:** hash-chained append-only event log, idempotent writes keyed by event reference, immutable audit storage.

**Frontend:** Next.js 14 and TypeScript for the arrangement dashboard, dispute view and event timeline. Angular 20 for the operator console pattern already built in openMF.

**Testing:** JUnit 5, Mockito, integration tests, concurrency tests with genuinely simultaneous submission, fault injection, chain verification as a test.

**Infrastructure:** Docker, GitHub Actions CI/CD, Linux.

Architecture detail: https://github.com/saksham869/Ekmat/blob/main/docs/ARCHITECTURE.md

---

## Expected Impact

**For the borrower.** Two lenders cannot file contradictory positions on one loan, because bureau submission reads from an agreed state rather than either party's private view. A borrower does not acquire a dispute on their credit file that originated in a disagreement between two institutions they never chose to have. On a Rs 1.1 lakh crore and fast-growing base, serving segments co-lending exists to reach, that is the inclusion outcome.

**For the lending partners.** The Para 33 obligation to converge within a working day becomes an enforced state transition rather than a file exchange and a phone call. Disagreement over escrow appropriation or the date a payment landed is adjudicated against an ordered record neither party authored alone.

**For the supervisor.** A Regulator organisation on the network reads the arrangement's event history directly, in sequence, without requesting reconciled statements from either party. Breach of the next-working-day requirement is a recorded event rather than something reconstructed afterwards.

**Scale characteristics.** The design is an append with dual endorsement and a deterministic state transition, bounded by the arrangement's servicing event rate rather than by volume on any payment rail. I am not claiming measured throughput. Load characterisation is Phase 5 work, not a claim made here.

**Why this problem.** The Directions came into force on 1 January 2026, nine months ago. The obligation to converge exists. The infrastructure to converge on does not. That gap is narrow enough to build against and consequential enough to matter.

RBI Deputy Governor M. Rajeshwar Rao, TransUnion CIBIL Credit Conference, 1 July 2025: *"Without this, duplication and misreporting remain risks. We must move towards a unique borrower identifier, which is secure, verifiable, and consistent across the system."*
https://www.bis.org/review/r250702e.htm

---

## GitHub Repository URL

https://github.com/saksham869/Ekmat

---

## Pitch Deck URL

*To be filled with the share link for the Ekmat deck.*

---

## Sources

Every claim above traces to a primary regulator document, an industry body, a rating agency, or a named executive on the record. No vendor estimates.

- [RBI (Co-Lending Arrangements) Directions, 2025](https://www.rbi.org.in/Scripts/NotificationUser.aspx?Id=12888&Mode=0) — RBI/DOR/2025-26/139
- IIBF, *Asymmetric Information and Market Failure in Bank-NBFC Co-Lending Model* — survey of 16 banks and 20 NBFCs
- CRISIL Ratings — co-lending AUM and growth
- Business Standard, May 2025 — Prashant Kumar, Yes Bank
- [M. Rajeshwar Rao, RBI Deputy Governor, 1 July 2025](https://www.bis.org/review/r250702e.htm)
- [github.com/npci/drunix](https://github.com/npci/drunix)

---

*[README](../README.md) · [ARCHITECTURE](ARCHITECTURE.md) · [WORKFLOW](WORKFLOW.md)*
