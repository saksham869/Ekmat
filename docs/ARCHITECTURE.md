# Architecture

How Ekmat is put together, and why each piece is shaped the way it is.

---

## 1. The design constraint

Every decision below follows from one sentence in the RBI Co-Lending Directions, 2025:

> **There is no lead lender and no system of record.**

Para 25 requires each lender to keep its own borrower account. Para 31 requires each to report separately to credit bureaus. Para 33 requires both to hold the *same* classification within a working day. And nothing designates whose version wins.

So the architecture cannot put the record inside either lender. It also cannot put it inside a third-party vendor, because that just moves the custody problem to someone with a commercial interest in both parties' data.

**The record has to be jointly held, jointly written, and independently verifiable.** That is the whole reason a ledger is here rather than a database.

---

## 2. Network topology

One DRUNIX channel per co-lending arrangement. Four organisations.

```
╔═══════════════════════════════════════════════════════════════════╗
║                    CHANNEL: arrangement-<id>                      ║
║                                                                   ║
║   ┌──────────────────┐              ┌──────────────────┐          ║
║   │   Org: BANK      │              │   Org: NBFC      │          ║
║   │                  │              │                  │          ║
║   │  Peer (endorse)  │              │  Peer (endorse)  │          ║
║   │  CA + MSP        │              │  CA + MSP        │          ║
║   │                  │              │                  │          ║
║   │  ┌────────────┐  │              │  ┌────────────┐  │          ║
║   │  │ Private    │  │              │  │ Private    │  │          ║
║   │  │ Data Coll. │  │              │  │ Data Coll. │  │          ║
║   │  └────────────┘  │              │  └────────────┘  │          ║
║   └────────┬─────────┘              └────────┬─────────┘          ║
║            │                                 │                    ║
║            │    BOTH signatures required     │                    ║
║            └────────────────┬────────────────┘                    ║
║                             ▼                                     ║
║            ┌────────────────────────────────┐                     ║
║            │   ORDERING SERVICE (Raft)      │                     ║
║            │   3 orderer nodes              │                     ║
║            └────────────────┬───────────────┘                     ║
║                             ▼                                     ║
║            ┌────────────────────────────────┐                     ║
║            │   SHARED EVENT LEDGER          │                     ║
║            │   append-only · ordered        │                     ║
║            │   replicated to every peer     │                     ║
║            └────────────────────────────────┘                     ║
║                     ▲                  │                          ║
║                     │ writes           │ reads                    ║
║          ┌──────────┴──────┐    ┌──────▼─────────────┐            ║
║          │ Org: ESCROW     │    │  Org: REGULATOR    │            ║
║          │      BANK       │    │                    │            ║
║          │                 │    │  Peer (read-only)  │            ║
║          │ Peer            │    │  No endorsement    │            ║
║          │ Settlement      │    │  rights            │            ║
║          │ events only     │    │                    │            ║
║          └─────────────────┘    └────────────────────┘            ║
╚═══════════════════════════════════════════════════════════════════╝
```

### Organisation roles

| Org | Writes | Endorses | Reads | Why it is a separate org |
|---|---|---|---|---|
| **Bank** | Servicing events for its share | Yes | Full channel | Keeps its own books per Para 25; must not be able to write alone |
| **NBFC** | Servicing events for its share | Yes | Full channel | Same, and has opposed incentives on NPA timing |
| **Escrow Bank** | Settlement and appropriation events | No | Full channel | Para 26 routes funds through it, so its record of appropriation order becomes evidence rather than testimony |
| **Regulator** | Nothing | No | Full channel | Reads the arrangement's history directly instead of requesting reconciled statements from either party |

The Regulator org holds **no endorsement rights and no write path**. It is an observer by construction, not by policy — it cannot alter the record even if it wanted to.

---

## 3. Endorsement policy

This is the load-bearing security decision.

```
Servicing event, classification change, dispute resolution:
    AND('BankMSP.member', 'NBFCMSP.member')

Settlement / escrow appropriation event:
    AND('EscrowBankMSP.member', OR('BankMSP.member', 'NBFCMSP.member'))

Read:
    any channel member
```

**What this means in practice.** Neither lender can append an event to the shared sequence by itself. A classification change proposed by the NBFC does not exist on the ledger until the Bank's peer also signs it. This is not a workflow convention enforced in application code — it is a chaincode-level policy that the ordering service will reject a transaction for violating.

That is the difference between "we agreed to share a log" and "neither of us can rewrite the log."

---

## 4. Data model

### Core entity: Arrangement

```
Arrangement
├── arrangementId          (channel key)
├── borrowerRef            (hashed; PII stays in private data collections)
├── bankOrgId
├── nbfcOrgId
├── escrowAccountRef
├── shareRatio             { bank: 0.80, nbfc: 0.20 }   -- each RE ≥ 10%
├── originationDate
├── agreedClassification   { status, asOf, convergedAt, proposedBy }
├── eventSequence[]        -- the append-only log
└── openDisputes[]
```

### Event record

Every entry on the shared log has this shape:

```
Event
├── eventId                (idempotency key; deterministic from content)
├── arrangementId
├── seq                    (monotonic; assigned at commit, not at proposal)
├── type                   DISBURSEMENT | REPAYMENT | ESCROW_APPROPRIATION
│                          | CLASSIFICATION_PROPOSED | CLASSIFICATION_ENDORSED
│                          | CLASSIFICATION_CHALLENGED | DISPUTE_RAISED
│                          | DISPUTE_RESOLVED | DEADLINE_BREACH
├── payload                (type-specific; amounts, dates, classification)
├── proposedBy             orgId
├── proposedAt             (proposer's clock, recorded as a claim)
├── committedAt            (ordering service clock, authoritative)
├── endorsements[]         [{ orgId, signature, at }]
└── prevEventHash          (hash chain over the arrangement's sequence)
```

**Two timestamps, deliberately.** `proposedAt` is what the proposing party says. `committedAt` is what the ordering service observed. When the dispute is *whose timestamp was right*, you need both — the claim and the independent observation. Storing only one makes the dispute unanswerable.

### What stays private

```
ON THE SHARED CHANNEL          IN PRIVATE DATA COLLECTIONS
─────────────────────          ───────────────────────────
event type                     borrower name, address, PAN
amounts on this loan           internal risk grades
classification status          the lender's wider exposure to this borrower
timestamps                     pricing and margin detail
endorsement signatures         internal notes
hashes                         
```

The shared channel carries **what the two parties are jointly bound by**. It does not carry either party's wider book. A hash of the private record goes on-chain so the private detail is verifiable without being visible.

---

## 5. The classification state machine

Para 33 is the provision Ekmat exists to enforce. It is implemented as a state machine in chaincode, not as a workflow in application code.

```
                    ┌───────────────┐
                    │   AGREED      │◄──────────────────┐
                    │ (both hold    │                   │
                    │  same status) │                   │
                    └───────┬───────┘                   │
                            │                           │
                  one party proposes                    │
                  a classification change               │
                            │                           │
                            ▼                           │
                    ┌───────────────┐                   │
              ┌─────│   PROPOSED    │                   │
              │     │ (awaiting     │                   │
              │     │  counterparty)│                   │
              │     └───────┬───────┘                   │
              │             │                           │
              │      ┌──────┴──────┐                    │
              │      │             │                    │
     deadline │  endorsed      challenged               │
     elapses  │      │             │                    │
              │      │             ▼                    │
              │      │     ┌───────────────┐            │
              │      │     │  CHALLENGED   │            │
              │      │     │ (evidence on  │            │
              │      │     │  the same log)│            │
              │      │     └───────┬───────┘            │
              │      │             │                    │
              │      │        adjudicated                │
              │      │             │                    │
              │      └──────┬──────┘                    │
              │             ▼                           │
              │     ┌───────────────┐                   │
              │     │  CONVERGED    │───────────────────┘
              │     │ (one status,  │
              │     │  both signed) │
              │     └───────┬───────┘
              │             │
              │             ▼
              │     ┌───────────────┐
              │     │  SUBMITTABLE  │──► bureau submission
              │     │ to bureau     │    reads from HERE,
              │     └───────────────┘    not from either
              │                          party's own view
              ▼
      ┌───────────────┐
      │ DEADLINE      │  breach event written to the log,
      │ BREACH        │  visible to the Regulator org
      └───────────────┘
```

**The critical invariant:** bureau submission is only reachable from `CONVERGED`. There is no code path from either party's private classification to a bureau submission. That is how two contradictory filings become structurally impossible rather than procedurally discouraged.

**The deadline is in the contract.** Chaincode computes the next-working-day boundary from `committedAt` on the proposal. A proposal still in `PROPOSED` past that boundary triggers `DEADLINE_BREACH` — a recorded event, not an alert someone has to notice.

---

## 6. Idempotency and concurrency

Two lenders, asynchronous systems, retrying clients. Duplicate submissions are not an edge case here, they are the normal weather.

**The failure mode being designed against:**

```
   t0   Bank's system submits repayment event E
   t1   Network hiccup; Bank's client does not see the ack
   t2   Bank's client retries E
   t3   Both submissions reach the ordering service

        Naive design: the repayment is applied TWICE.
        The borrower's outstanding is wrong. Both parties'
        books now disagree with each other AND with reality.
```

**The fix, carried over from [Arbiter](https://github.com/saksham869/Arbiter):**

`eventId` is derived deterministically from the event's content and the arrangement — not generated by the client. Chaincode performs the existence check and the write in a **single transaction against the same key**, so there is no window between "does this exist?" and "write it."

A retry of E produces the same `eventId`, hits the existing key, and returns the original commit. Exactly once, under genuinely concurrent submission.

This is tested with simultaneous identical submissions, not sequential ones. Concurrency defects do not appear in tests that run one request at a time.

---

## 7. Integrity: the hash chain

Each event commits to the hash of the previous event in that arrangement's sequence:

```
   E1 ──► E2 ──► E3 ──► E4 ──► E5
   │      │      │      │      │
   h(E1)  h(E2)  h(E3)  h(E4)  h(E5)
          ▲      ▲      ▲      ▲
          │      │      │      │
       prevHash stored in each successor
```

Alter E3 after the fact and `h(E3)` changes, so E4's `prevEventHash` no longer matches. **The chain breaks at the exact point of alteration, and either party can detect it independently.**

This is redundant with the ledger's own immutability — and deliberately so. It means the integrity guarantee survives export: a party can take their copy of the sequence out of the network and still prove it was not tampered with.

---

## 8. Application layer

The chaincode holds the rules. The application layer holds everything that does not belong on a ledger.

```
┌─────────────────────────────────────────────────────────┐
│  Next.js 14 + TypeScript                                │
│  Arrangement dashboard · dispute view · event timeline  │
└───────────────────────┬─────────────────────────────────┘
                        │ REST
┌───────────────────────▼─────────────────────────────────┐
│  Spring Boot 3 · Java 21                                │
│                                                         │
│  ├── Arrangement service    orchestration               │
│  ├── Event service          propose / endorse / query   │
│  ├── Convergence service    Para 33 deadline tracking   │
│  ├── Bureau service         validation + submission     │
│  │                          (carried over from CB-ILD)  │
│  ├── Dispute service        raise / evidence / resolve  │
│  └── Audit service          every write logged          │
│                                                         │
│  RBAC · idempotency filter · audit logging              │
└───────┬─────────────────────────────────┬───────────────┘
        │ Fabric Gateway SDK              │ JDBC
┌───────▼──────────────┐      ┌───────────▼───────────────┐
│  DRUNIX network      │      │  PostgreSQL               │
│  Java chaincode      │      │  Read models, projections │
│  4 orgs              │      │  Liquibase migrations     │
└──────────────────────┘      └───────────────────────────┘
```

**PostgreSQL holds read models, not the record.** Querying "every arrangement where classification has been in PROPOSED for more than a day" against a ledger is slow and awkward. Projecting ledger events into Postgres makes that a normal SQL query. The projection is rebuildable from the ledger at any time — if Postgres and the ledger disagree, **the ledger is right and Postgres gets rebuilt.**

---

## 9. What the bureau submission path looks like

This is where the existing CB-ILD work plugs in directly.

```
  CONVERGED classification on the ledger
              │
              ▼
  ┌───────────────────────────┐
  │ Read agreed state         │  ← NOT from either party's own system
  └───────────┬───────────────┘
              ▼
  ┌───────────────────────────┐
  │ Validate against bureau   │  ← CB-ILD validation pipeline
  │ taxonomy                  │
  └───────────┬───────────────┘
              ▼
  ┌───────────────────────────┐
  │ Submit                    │
  └───────────┬───────────────┘
              ▼
       ┌──────┴──────┐
       │             │
   accepted      rejected
       │             │
       ▼             ▼
  ┌─────────┐  ┌──────────────────────────┐
  │ Event   │  │ Rejection written to the │
  │ logged  │  │ ledger and tracked to    │
  └─────────┘  │ resolution — NOT dropped │
               └──────────────────────────┘
```

A rejected submission becoming a ledger event is the point. In the current world a rejection is a line in one institution's log that nobody reads. Here it is visible to both parties and the Regulator org until it is resolved.

---

## 10. Failure modes and what happens

| Failure | Behaviour |
|---|---|
| Counterparty peer is down when an event is proposed | Proposal is not committed; proposer retries; deadline clock still runs and will raise `DEADLINE_BREACH` if it elapses |
| Duplicate event submission | Same deterministic `eventId`; commits exactly once |
| Out-of-order arrival | Ordering service assigns `seq` at commit; proposer's clock is recorded as a claim, not as truth |
| Event for a closed arrangement | Rejected at the chaincode |
| Postgres projection drifts from the ledger | Ledger wins; projection is rebuilt from events |
| One party attempts unilateral classification change | Rejected by endorsement policy before ordering |
| Historical event altered in storage | Hash chain breaks at that point; detectable by either party independently |

---

## 11. What this architecture deliberately does not do

**It does not move money.** Escrow settlement happens on existing rails. Ekmat records that it happened and in what order.

**It does not replace either loan management system.** Para 25 requires each lender to keep its own books. They keep them. Ekmat is the shared sequence those books derive from, not a substitute for them.

**It does not hold borrower PII on the shared channel.** Private data collections and hashes only.

**It does not claim to create trust.** The two parties already have a contract. What they lack is an ordering neither of them controls.

---

## 12. Open questions

Stated plainly, because a design document that pretends to have no open questions is not a design document.

1. **Java chaincode on DRUNIX is verified in source but undocumented by NPCI.** Phase 0 exists to prove it in two days. Go fallback identified.
2. **DRUNIX's SQL state database** could make the read-model layer unnecessary, but has no demonstrated Java path. Treated as a Phase 5 optimisation, not a dependency.
3. **Multi-party arrangements** — the Directions permit more than two REs. The endorsement policy generalises, but the convergence state machine needs revisiting for n-party disagreement.
4. **Channel-per-arrangement scaling** — clean isolation, but a lender with many arrangements joins many channels. An alternative is one channel per lender-pair with arrangement-scoped private data collections. Not yet decided.
5. **Where the bureau connector sits** — currently in the application layer. There is an argument for putting submission attestation on-chain so that "we submitted X on date Y" is itself a jointly-verifiable fact.

---

*Ekmat — a shared event ledger for co-lending dispute adjudication.*
*[README](../README.md) · [WORKFLOW](WORKFLOW.md)*
