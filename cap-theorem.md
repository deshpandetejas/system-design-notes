# CAP Theorem (Practical View)

## Why this matters
CAP is often misunderstood as a database property.  
In reality, it is a *decision framework* teams are forced into during failures.

Most production incidents are not about CAP in steady state —  
they are about CAP *during network partitions, replication lag, or partial outages*.

---

## The misconception to kill
- "We chose CP / AP" → oversimplified and misleading  
- CAP choices are **per operation**, **per failure scenario**, not per system  

You don’t build a CP system or an AP system.  
You make **localized CAP decisions** — often implicitly.

---

## The real question CAP forces
When a partition happens:
- Do we block to preserve correctness?
- Or do we respond with potentially stale / inconsistent data?

There is no free option — only **explicit vs accidental trade-offs**.

---

## Failure modes to watch for
- Eventual consistency without defined bounds (“eventually” with no SLA)
- Inconsistent reads breaking user trust
- Silent data divergence across replicas
- Operational confusion during incidents (“is this expected or a bug?”)

---

## Trade-offs that actually matter
- Strong consistency vs availability *for user-facing paths*
- Read-your-writes and monotonic reads as UX guarantees
- Conflict resolution strategies:
  - last-write-wins
  - domain-aware merges
  - manual reconciliation

---

## Questions I ask in design reviews
- Where do we tolerate inconsistency, and for how long?
- What does the user experience during a partition?
- Is partial success acceptable — or dangerous?
- How do we detect and resolve conflicts?
- What is the operator playbook during a split-brain scenario?

---

## Practical application from systems I owned

CAP became real for me not through theory, but while owning production systems
where user experience, downstream dependencies, and failure behavior mattered.

Below are two systems where I intentionally made **opposite CAP choices**.

---

## Case 1: Real-time event delivery (Availability > Consistency)

### Context
I owned an API acting as a middleware between WhatsApp Business (WABA) events
and client systems.

Flow (simplified):
- Receive real-time user message events from WABA
- Forward events to client APIs in near real time

This was a **user-facing, real-time delivery path**.

### Decision
- Continued accepting events even when downstream client APIs
  were partially unavailable.
- Attempted real-time delivery first.
- **On delivery failure, persisted events into a queue for retry** instead of blocking ingestion.
- Did not block the upstream flow waiting for downstream consistency.

### Why
- Dropped or delayed ingestion would immediately impact user experience.
- Clients preferred *eventual delivery* over *missed delivery*.
- Blocking ingestion would cascade failures back into WABA and amplify outages.

### Trade-offs accepted
- Events could arrive late or out of order
- Temporary divergence between WABA state and client systems
- Additional complexity in retry handling and deduplication
- Increased reliance on downstream idempotency

### Failure framing
For real-time systems, **liveness becomes correctness**.

Accepting the event and guaranteeing *eventual delivery* was more important
than enforcing immediate consistency across systems.

---

## Case 2: MSISDN whitelisting across multiple accounts (Consistency > Availability)

### Context
I owned an API responsible for whitelisting MSISDNs across multiple client accounts.

Problem:
- If a number was not consistently whitelisted across *all required accounts*,
  message delivery would silently fail.
- Partial success was worse than failure.

### Decision
- Accepted the request and immediately returned a **Transaction ID**.
- Performed whitelisting asynchronously:
  - Request placed into a queue
  - Background workers processed pending entries
  - Per-account state persisted at each step
- Generated reports later using the same Transaction ID.

### Why
- A fast but incorrect “success” would break the product.
- Delayed correctness was safer than immediate inconsistency.

### Trade-offs accepted
- Slower user feedback
- Asynchronous completion instead of synchronous confirmation
- Additional operational complexity (queues, workers, reconciliation tables)

### Failure framing
When **partial correctness breaks the business**, blocking or delaying
is safer than being available but wrong.

---

## The real CAP insight (beyond theory)

CAP is not a system-wide label.

Mature systems:
- mix AP and CP paths intentionally
- make CAP decisions per workflow
- document expected behavior during failure

Most CAP-related outages happen not because teams chose wrong —
but because they **never realized they were choosing at all**.

---

## My takeaway

- CAP decisions should be **explicit and documented**
- Async workflows are often a way to *escape false CAP binaries*
- User experience during failure matters more than theoretical purity
- Accidental CAP decisions are a reliability risk

CAP is not about picking sides.  
It is about **owning the consequences of your choices**.
