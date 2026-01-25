## Why this matters
CAP is often misunderstood as a database property.
In reality, it is a *decision framework* teams are forced into during failures.

Most production incidents are not about CAP in steady state —
they are about CAP *during network partitions, replication lag, or partial outages*.

## The misconception to kill
- “We chose CP / AP” → oversimplified and misleading
- CAP choices are **per operation**, **per failure scenario**, not per system

## The real question CAP forces
When a partition happens:
- Do we block to preserve correctness?
- Or do we respond with potentially stale / inconsistent data?

There is no free option — only explicit vs accidental choices.

## Failure modes to cover
- Eventual consistency without defined bounds (“eventually” with no SLA)
- Inconsistent reads breaking user trust
- Silent data divergence across replicas
- Operational confusion during incidents (“is this expected?”)

## Trade-offs to document
- Strong consistency vs availability *for user-facing paths*
- Read-your-writes and monotonic reads as UX guarantees
- Conflict resolution strategies:
  - last-write-wins
  - domain-aware merges
  - manual reconciliation

## Questions I want answered in design reviews
- Where do we tolerate inconsistency, and for how long?
- What does the user experience during a partition?
- How do we detect and resolve conflicts?
- What is the operator playbook during a split-brain scenario?

---
- review questions
- leadership takeaways (not textbook definitions)
