# Workflow

What actually happens, step by step, from a borrower's repayment to a credit bureau submission — and what happens when the two lenders disagree.

---

## Contents

1. [The cast](#1-the-cast)
2. [Walkthrough: a normal repayment](#2-walkthrough-a-normal-repayment)
3. [Walkthrough: the classification disagreement](#3-walkthrough-the-classification-disagreement)
4. [Walkthrough: an escrow appropriation dispute](#4-walkthrough-an-escrow-appropriation-dispute)
5. [Walkthrough: the deadline breach](#5-walkthrough-the-deadline-breach)
6. [Bureau submission](#6-bureau-submission)
7. [Build phases](#7-build-phases)
8. [Testing strategy](#8-testing-strategy)
9. [Demonstration script](#9-demonstration-script)

---

## 1. The cast

| Who | What they do here |
|---|---|
| **Borrower** | Takes one loan, funded jointly. Never interacts with Ekmat directly. |
| **Bank** | Funds the larger share. Runs its own LMS on IRAC. Endorses events. |
| **NBFC** | Originates and services. Runs its own LMS on Ind-AS. Endorses events. |
| **Escrow Bank** | Para 26 requires all money to route through it. Writes settlement events. |
| **Regulator** | Reads. Cannot write. Cannot endorse. |
| **Credit Bureau** | Receives one submission per lender share, derived from one agreed state. |

Throughout: **Bank holds 80%, NBFC holds 20%.** Each RE must retain at least 10% under the Directions.

---

## 2. Walkthrough: a normal repayment

The uneventful case. Worth walking because everything else is a deviation from it.

```
 ┌──────────┐
 │ Borrower │  pays ₹12,000 EMI
 └────┬─────┘
      │
      ▼
 ┌─────────────────┐
 │  ESCROW BANK    │  receives funds
 └────┬────────────┘
      │
      │  ① writes SETTLEMENT event
      │     { amount: 12000, receivedAt: T }
      ▼
 ╔═══════════════════════════════════════╗
 ║        SHARED EVENT LEDGER            ║
 ║                                       ║
 ║  seq 47 · SETTLEMENT                  ║
 ║  endorsed by: EscrowBank + NBFC       ║
 ╚═══════════════════════════════════════╝
      │
      │  ② NBFC proposes the REPAYMENT event,
      │     splitting per shareRatio
      │     { bank: 9600, nbfc: 2400 }
      ▼
 ┌─────────────────┐         ┌─────────────────┐
 │      NBFC       │────────►│      BANK       │
 │   proposes      │ endorse?│  checks split   │
 └─────────────────┘         └────────┬────────┘
                                      │ ✓ endorses
                                      ▼
 ╔═══════════════════════════════════════╗
 ║  seq 48 · REPAYMENT                   ║
 ║  endorsed by: NBFC + Bank             ║
 ║  prevEventHash: h(seq 47)             ║
 ╚═══════════════════════════════════════╝
      │
      ▼
 Both lenders update their own books from
 the SAME committed event. No reconciliation
 required, because there was never a divergence.
```

**What made it uneventful:** the split was computed once, agreed once, and recorded once. Neither party derived it independently from its own view of the escrow statement.

That is the entire mechanism. Everything below is what happens when it does not go this smoothly.

---

## 3. Walkthrough: the classification disagreement

This is the demonstration. Para 33 in motion.

### Day 0 — divergence

```
  NBFC's LMS (Ind-AS)              Bank's LMS (IRAC)
  ─────────────────────            ────────────────────
  DPD: 31 days                     DPD: 29 days
  → moves to SMA-2                 → still Standard

         ⚠  Same borrower. Same loan. Two positions.
```

The gap is not incompetence. Ind-AS and IRAC count differently, and the two systems received the last repayment file at different times. **This is the normal state of a co-lending arrangement, and it is exactly what Para 33 exists to close.**

### Step 1 — proposal

NBFC proposes the transition. It is not an email or a shared spreadsheet; it is a signed, timestamped transaction.

```
 ╔═══════════════════════════════════════════════╗
 ║  seq 49 · CLASSIFICATION_PROPOSED             ║
 ║                                               ║
 ║  from:        STANDARD                        ║
 ║  to:          SMA_2                           ║
 ║  basis:       DPD 31, last payment 2026-08-14 ║
 ║  proposedBy:  NBFC                            ║
 ║  proposedAt:  2026-09-15T09:12:00Z  (claim)   ║
 ║  committedAt: 2026-09-15T09:12:04Z  (observed)║
 ║                                               ║
 ║  ⏱ deadline: end of 2026-09-16                ║
 ╚═══════════════════════════════════════════════╝

              State: PROPOSED
```

The deadline is computed by chaincode from `committedAt`, using the next-working-day rule. Nobody sets a reminder.

### Step 2 — the fork

```
                    ┌─────────────┐
                    │  PROPOSED   │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      ENDORSE         CHALLENGE        NO RESPONSE
          │                │                │
          │                │                │
          ▼                ▼                ▼
    ┌───────────┐   ┌───────────┐   ┌───────────────┐
    │ CONVERGED │   │CHALLENGED │   │DEADLINE_BREACH│
    └───────────┘   └─────┬─────┘   │  (§5)         │
                          │         └───────────────┘
                    adjudicated
                          │
                          ▼
                    ┌───────────┐
                    │ CONVERGED │
                    └───────────┘
```

### Step 3a — Bank endorses

Bank checks its own records, sees the payment date matches, accepts that Ind-AS timing is ahead of its own.

```
 ╔═══════════════════════════════════════════════╗
 ║  seq 50 · CLASSIFICATION_ENDORSED             ║
 ║  endorsedBy:  Bank                            ║
 ║  committedAt: 2026-09-15T14:33:00Z            ║
 ╚═══════════════════════════════════════════════╝

              State: CONVERGED
              agreedClassification: SMA_2
              convergedAt: 2026-09-15T14:33:00Z
```

Both lenders now hold `SMA_2`. Not because they negotiated it, because the ledger holds one value and both signed it.

### Step 3b — Bank challenges

Bank's records show a payment on 2026-09-01 that the NBFC's system has not applied.

```
 ╔═══════════════════════════════════════════════╗
 ║  seq 50 · CLASSIFICATION_CHALLENGED           ║
 ║                                               ║
 ║  challengedBy: Bank                           ║
 ║  grounds:      payment applied 2026-09-01     ║
 ║  evidence:     ref seq 44 (SETTLEMENT)        ║
 ╚═══════════════════════════════════════════════╝

              State: CHALLENGED
```

**The evidence is a reference to an event already on the shared log.** Bank is not asserting something the NBFC has to take on faith; it is pointing at `seq 44`, which the NBFC endorsed itself.

Adjudication reads the ordered sequence. If `seq 44` exists and the NBFC's DPD calculation did not account for it, the NBFC's proposal was wrong, and the record shows exactly where.

```
 ╔═══════════════════════════════════════════════╗
 ║  seq 51 · CLASSIFICATION_ENDORSED             ║
 ║  resolution: proposal withdrawn               ║
 ║  agreed:     STANDARD                         ║
 ╚═══════════════════════════════════════════════╝

              State: CONVERGED
```

**What changed versus today:** the dispute took one exchange instead of a week of emails, and it was settled by pointing at a record neither party could have altered.

---

## 4. Walkthrough: an escrow appropriation dispute

The subtler case, and the one that costs real money.

**The scenario.** A borrower pays ₹8,000 against an EMI of ₹12,000. Partial payment. In what order does it get appropriated — interest first, then principal? Bank's share before NBFC's, or pro rata? The arrangement agreement says pro rata. The escrow bank's system applied it interest-first across the whole exposure.

```
        BANK's view                  NBFC's view
        ───────────                  ───────────
        received ₹6,400              received ₹1,600
        (pro rata 80/20)             (pro rata 80/20)

        ESCROW BANK's actual appropriation:
        ₹8,000 → interest across full exposure
        → Bank ₹6,900, NBFC ₹1,100

              ⚠  Everyone's books now disagree.
```

**How Ekmat handles it:**

```
  ① Escrow Bank writes the SETTLEMENT event with its
     actual appropriation, not the expected one.

 ╔═══════════════════════════════════════════════╗
 ║  seq 62 · ESCROW_APPROPRIATION                ║
 ║  received:    8000                            ║
 ║  applied:     { bank: 6900, nbfc: 1100 }      ║
 ║  method:      INTEREST_FIRST                  ║
 ║  writtenBy:   EscrowBank                      ║
 ╚═══════════════════════════════════════════════╝

  ② NBFC raises a dispute — the arrangement says pro rata.

 ╔═══════════════════════════════════════════════╗
 ║  seq 63 · DISPUTE_RAISED                      ║
 ║  against:   seq 62                            ║
 ║  grounds:   appropriation method              ║
 ║  expected:  { bank: 6400, nbfc: 1600 }        ║
 ╚═══════════════════════════════════════════════╝

  ③ Resolution is adjudicated against the ordered record.
     The arrangement terms are themselves on the ledger
     (written at origination, seq 1).

 ╔═══════════════════════════════════════════════╗
 ║  seq 64 · DISPUTE_RESOLVED                    ║
 ║  finding:    seq 62 applied wrong method      ║
 ║  reference:  seq 1 (arrangement terms)        ║
 ║  correction: { bank: 6400, nbfc: 1600 }       ║
 ╚═══════════════════════════════════════════════╝
```

**Note what was never deleted.** `seq 62` stays on the log, wrong appropriation and all. The correction is `seq 64`. A supervisor reading the sequence sees that the error happened, when it was caught, and how it was resolved.

An append-only log does not hide mistakes. It makes them attributable.

---

## 5. Walkthrough: the deadline breach

Para 33's teeth.

```
  2026-09-15  09:12   NBFC proposes SMA_2
                      deadline computed: end of 2026-09-16

  2026-09-15  ...     Bank does not respond
  2026-09-16  ...     Bank still does not respond

  2026-09-16  23:59   ⏱ DEADLINE ELAPSED
```

```
 ╔═══════════════════════════════════════════════╗
 ║  seq 51 · DEADLINE_BREACH                     ║
 ║                                               ║
 ║  against:     seq 49 (CLASSIFICATION_PROPOSED)║
 ║  obligation:  Para 33, next working day       ║
 ║  elapsed:     2026-09-16T23:59:59Z            ║
 ║  unresponsive: Bank                           ║
 ╚═══════════════════════════════════════════════╝

         Visible to: Bank, NBFC, Regulator
         State: still PROPOSED — NOT submittable
```

Two things happen, and both matter.

**The breach is recorded, not alerted.** It is an event on the ledger with the same permanence as a repayment. Nobody has to have been watching a dashboard.

**Bureau submission stays blocked.** State never reached `CONVERGED`, and submission only reads from `CONVERGED`. The system will not let a contradictory pair of filings go out *because* someone missed a deadline. The obligation and the enforcement are the same mechanism.

---

## 6. Bureau submission

```
     ┌─────────────────────────┐
     │ State == CONVERGED?     │
     └───────────┬─────────────┘
            no ──┴── yes
             │        │
             ▼        ▼
        ┌────────┐  ┌──────────────────────────┐
        │ BLOCK  │  │ Read agreedClassification│
        │ submit │  └───────────┬──────────────┘
        └────────┘              │
                                ▼
                 ┌──────────────────────────────┐
                 │ Generate Bank's submission   │
                 │ Generate NBFC's submission   │
                 │   — both from the SAME       │
                 │     agreed state             │
                 └───────────┬──────────────────┘
                             ▼
                 ┌──────────────────────────────┐
                 │ Validate against bureau      │
                 │ taxonomy   (CB-ILD pipeline) │
                 └───────────┬──────────────────┘
                             ▼
                      ┌──────┴──────┐
                  accepted      rejected
                      │             │
                      ▼             ▼
          ╔═══════════════╗  ╔═══════════════════════╗
          ║ SUBMISSION_   ║  ║ SUBMISSION_REJECTED   ║
          ║ ACCEPTED      ║  ║ reason: <bureau code> ║
          ║ written to    ║  ║ status: OPEN          ║
          ║ ledger        ║  ║ tracked to resolution ║
          ╚═══════════════╝  ╚═══════════════════════╝
                                        │
                                        ▼
                             Visible to both lenders
                             AND the Regulator until
                             it is resolved.
```

Para 31 still requires **two separate submissions**, one per lender share. Ekmat does not change that. What it changes is that both are generated from one agreed state, so they cannot contradict each other.

**The rejection path is the part that matters most.** Today a bureau rejection is a line in one institution's log that nobody reads, and the borrower's file silently stays wrong. Here it is a ledger event visible to both parties and the supervisor until somebody closes it.

---

## 7. Build phases

Twenty days. The hardest problem is scheduled on day 6, not day 18.

### Phase 0 — De-risk the platform · Days 1–2

**Deliverable:** A DRUNIX network running locally with trivial Java chaincode deployed.

**Done when:** A Java contract commits a transaction and reads it back on a multi-org network.

**Why first:** Java chaincode support on DRUNIX is verified in the platform's source (`core/chaincode/platforms` contains a fully implemented `java.Platform{}` in v1.0.0) but is not documented by NPCI, and every DRUNIX sample found uses Go. If Java does not work, this fails on day 2 and the build pivots to Go — two days lost instead of two weeks.

### Phase 1 — Network and data model · Days 3–5

**Deliverable:** Four organisations (Bank, NBFC, Escrow Bank, Regulator), channel per arrangement, event schema, private data collections.

**Done when:** Each org reads the shared sequence, and no org can write on another's behalf.

**Checks:** Regulator org has no write path. Private data collections isolate correctly.

### Phase 2 — Co-signed event append · Days 6–9 ⚠ **critical**

**Deliverable:** Servicing events appended only on dual endorsement, keyed idempotently.

**Done when:** A duplicated or replayed submission appends **exactly once** under concurrent load, demonstrated by a test that fires simultaneous identical submissions.

**Why here:** This is the correctness milestone the entire system rests on. If idempotency under concurrency does not hold, nothing above it is trustworthy. Scheduling it early means discovering that on day 9, not day 19.

### Phase 3 — Classification convergence · Days 10–13

**Deliverable:** The Para 33 state machine. Propose, endorse, challenge, converge. Deadline enforced in contract logic.

**Done when:** A divergent classification attempt is rejected at the chaincode, and an unendorsed proposal past the deadline raises `DEADLINE_BREACH` visible to the Regulator org.

### Phase 4 — Gated bureau submission · Days 14–16

**Deliverable:** Submission reads from agreed state, using the CB-ILD validation and exception-handling pipeline.

**Done when:** Two lenders cannot produce contradictory submissions on one loan, and an injected format rejection is tracked to resolution rather than dropped.

### Phase 5 — Dispute adjudication and demonstration · Days 17–20

**Deliverable:** Escrow appropriation dispute raised, adjudicated, resolved. Full lifecycle demonstration.

**Done when:** A complete arrangement lifecycle runs end to end and the adversarial suite passes against the integrated system.

**Absorbs:** Slippage from earlier phases. Optional exploration of DRUNIX's SQL state database if time allows.

---

### Dependency order

```
  Phase 0 ──► Phase 1 ──► Phase 2 ──┬──► Phase 3 ──► Phase 4 ──┐
   (Java)      (network)  (idempot.) │   (Para 33)   (bureau)   │
                                     │                          ▼
                                     └──────────────────────► Phase 5
                                                            (demo)

  Minimum viable path: Phases 0–4.
  Phase 5 is separable and absorbs slippage.
```

---

## 8. Testing strategy

**Tests ship with each phase, not after it.**

### Concurrency tests

Fire **genuinely simultaneous** submissions, not sequential ones.

```java
// The shape that matters
CountDownLatch gate = new CountDownLatch(1);
List<Future<Result>> results = IntStream.range(0, N)
    .mapToObj(i -> executor.submit(() -> {
        gate.await();                 // all threads block here
        return submitEvent(sameEvent); // then fire together
    }))
    .toList();
gate.countDown();                     // release

// Assert: exactly one commit, N-1 idempotent returns
```

A test that submits the same event twice in sequence passes trivially and proves nothing. Concurrency defects only appear when requests genuinely overlap.

### Fault injection on the event path

| Injected fault | Expected behaviour |
|---|---|
| Dropped endorsement | Event uncommitted; deadline clock still runs |
| Duplicate submission | Exactly one commit |
| Out-of-order arrival | `seq` assigned at commit; proposer clock kept as a claim |
| Event for a closed arrangement | Rejected at chaincode |
| Counterparty peer unreachable | Proposal fails cleanly; retriable |
| Clock skew between orgs | `committedAt` governs; `proposedAt` recorded but not authoritative |

### Chain verification as a test

Tamper-evidence is **asserted, not assumed**:

```
1. Commit a sequence of events
2. Alter a stored event directly
3. Run chain verification
4. Assert: verification fails, and names the exact event
```

### Adversarial scenarios

Carried over in spirit from [Arbiter](https://github.com/saksham869/Arbiter), where nine attacks all blocked:

| # | Attack | Must be |
|---|---|---|
| 1 | Unilateral classification change by one lender | Rejected by endorsement policy |
| 2 | Replayed classification proposal | Idempotent, single commit |
| 3 | Bureau submission from an unconverged state | Blocked, no code path exists |
| 4 | Regulator org attempts a write | Rejected — no write path |
| 5 | Event backdated via `proposedAt` | `committedAt` governs; skew recorded |
| 6 | Historical event altered in storage | Chain break detected and located |
| 7 | Concurrent contradictory proposals from both parties | One commits, one is rejected as stale |
| 8 | Deadline evaded by withdraw-and-repropose | Original breach event persists |
| 9 | Private data leaked to the shared channel | Schema validation rejects |

---

## 9. Demonstration script

Eight minutes. What a judge sees, in order.

```
  0:00  ONE SLIDE OF CONTEXT
        RBI Co-Lending Directions, 2025. Para 25, 31, 33.
        Two lenders. One loan. No system of record.

  1:00  THE BROKEN STATE
        Two LMS dashboards side by side.
        Same borrower. Bank says Standard. NBFC says SMA-2.
        Both are about to file. Both filings are valid to them.

  2:00  PROPOSE
        NBFC proposes SMA-2 to the ledger.
        Show the transaction: signed, timestamped,
        deadline computed by chaincode.

  3:00  TRY TO CHEAT
        Attempt to commit the classification with only
        the NBFC's signature.
        → REJECTED by endorsement policy, before ordering.
        This is the moment that proves the claim.

  4:00  CHALLENGE AND ADJUDICATE
        Bank challenges, citing seq 44 — an event the
        NBFC endorsed itself.
        Resolution reads the ordered record.
        Converged.

  5:00  THE DUPLICATE
        Fire 50 simultaneous identical repayment events.
        → Exactly one commit. 49 idempotent returns.
        Show the ledger: one entry.

  6:00  TAMPER
        Alter a historical event directly in storage.
        Run chain verification.
        → Fails, and names the exact event.

  7:00  SUBMIT
        Bureau submission reads from CONVERGED state.
        Two filings, one agreed position, no contradiction.
        Then: inject a rejection. It becomes a tracked
        ledger event, visible to the Regulator, still open.

  8:00  THE HONEST SLIDE
        What is built. What is not. What Java chaincode
        risk remains. Why not Postgres.
```

**The two moments that carry the demonstration** are 3:00 and 6:00. Everything else is narrative; those two are the system refusing to do something it should refuse to do. Judges remember refusals more than features.

---

*[README](../README.md) · [ARCHITECTURE](ARCHITECTURE.md)*
