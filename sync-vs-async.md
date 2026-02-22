# Sync vs Async Systems

“Sync vs async” is often treated as an implementation choice.

In practice, it is a **guarantee choice**:
- ordering vs throughput
- blocking vs decoupling
- delivery certainty vs capacity protection
- immediate correctness vs eventual handling

This note captures two real systems I owned where I intentionally made opposite choices.

---

# Case 1: SFTP Ingestion — Sync by Design (Ordering > Throughput)

## Problem context

- Files dropped via SFTP
- File names contained timestamps
- Files must be processed strictly in incremental order:
  - T1 → T2 → T3 …
- Incorrect timestamp → file rejected
- Client requirement: ordering correctness mandatory

If T2 was processed before T1, downstream state would be invalid.

---

## Design

- Cron job ran every minute
- Each run:
  - listed files
  - sorted by timestamp
  - picked **only one file**
- Cron executions did not overlap
- Processing time:
  - small files < 1 minute
  - large (GB-sized) files: several minutes
- Failure handling:
  - retry twice within same run
  - if still failing → move file to invalid folder
- “Last processed timestamp” stored in DB (checkpoint)

This created a **strictly serialized processing pipeline**.

---

## What this optimized for

- Deterministic ordering
- Controlled state progression
- Predictable downstream behavior

Throughput was deliberately limited.

---

## Trade-offs accepted

- Head-of-line blocking  
  Large T1 delays all subsequent files.
- Throughput constrained by single-thread execution.
- Increased operational overhead (invalid folder management).

These were acceptable because correctness was non-negotiable.

---

## Failure modes this design embraced (intentionally)

- A failing file halts progression.
- Backlog accumulates during large file processing.
- Latency for later files increases.

This system preferred **blocking over corruption**.

---

# Case 2: UPI Keyword Processing — Async by Design (Throughput > Delivery Guarantee)

## Problem context

- High sustained inbound keyword requests
- Used for UPI authentication
- Continuous traffic flow
- Primary risk: **thread pool exhaustion**
- Client API assumed highly available
- No retry requirement

---

## Design

- API immediately returned **200 OK**
- Work decoupled via queue:
  - accept request
  - enqueue
  - worker forwards to client API
- No retry on failure
- No blocking on downstream latency

This created an **at-most-once delivery model**.

---

## What this optimized for

- Thread pool protection
- Stable request acceptance under load
- Isolation from downstream API latency

This system protected ingestion capacity above all else.

---

## Trade-offs accepted

- At-most-once semantics (possible loss)
- Caller sees success before actual delivery
- Queue lag can grow under downstream slowness

Throughput was prioritized over delivery certainty.

---

## Failure modes this design introduced

- Silent drops if worker fails without observability
- Backlog growth during downstream slowness
- Mismatch between “accepted” and “delivered”

This system preferred **capacity stability over strict delivery guarantees**.

---

# Decision Comparison

| Dimension | SFTP (Sync) | UPI Keywords (Async) |
|------------|-------------|----------------------|
| Primary Goal | Ordering correctness | Thread pool protection |
| Blocking Behavior | Yes (intentional) | No |
| Throughput | Limited | High |
| Delivery Guarantee | Deterministic | At-most-once |
| Failure Style | Stop on error | Isolate & continue |
| Backpressure | Head-of-line blocking | Queue buffering |

---

# What Sync vs Async Actually Means

It is not about:
- using a queue
- using threads
- using callbacks

It is about answering:

- Is partial progress acceptable?
- Can we tolerate out-of-order execution?
- What happens when dependencies are slow?
- Is blocking safer than parallelism?
- What guarantee does the caller believe they received?

---

# Review Questions I Now Ask

- What is the caller’s mental model of completion?
- Are we protecting correctness or protecting capacity?
- What is the acceptable failure mode: block, drop, retry, or delay?
- Where does backpressure live?
- What guarantee are we actually providing:
  - at-most-once?
  - at-least-once?
  - exactly-once (rare)?

---

# My Takeaway

Sync vs async is not about style.

It is about choosing:

- determinism vs elasticity
- correctness under sequence vs resilience under volume
- visible blocking vs hidden backlog

In one system, I blocked to prevent corruption.  
In another, I decoupled to prevent collapse.

Both were correct — because the **risk being mitigated was different**.
