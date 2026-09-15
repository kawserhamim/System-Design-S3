# Distributed Systems Decision Record

A one-page reference that combines Labs 1, 2, and 3 into a single set of design rules.
Order follows the design flow: **transport** (A) → **retry policy** (B) → **consistency level** (C).

> Lab-measurement numbers have been removed. Only protocol-intrinsic facts (e.g. gRPC's 5-byte framing) remain.

---

## A. Wire Format → Use Case (from Lab 1)

Qualitative summary of what we observed comparing REST/JSON against gRPC/protobuf on the same payload.

| Dimension                  | REST (HTTP/JSON)                              | gRPC (protobuf over HTTP/2)                       |
| -------------------------- | --------------------------------------------- | ------------------------------------------------- |
| Wire format                | Human-readable text                           | Compact binary (field tags, no names on the wire) |
| Per-message framing        | None (length is in HTTP body)                 | Fixed small header per message                    |
| Payload size for same data | Larger (field names, whitespace, JSON syntax) | Smaller (binary encoding)                         |
| Latency tail               | Wider: more variance under load              | Tighter: HTTP/2 multiplexing + binary framing     |
| Throughput under load      | Lower                                         | Higher                                            |
| Browser / curl friendliness | Native                                        | Needs a gRPC client                               |

**Decision rule:** pick **gRPC** for service-to-service traffic on a fixed schema. Pick **REST** when the client is a browser, a third-party tool that can't speak gRPC, or a human who needs to read the JSON directly.

---

## B. Failure Mode → Retry Policy (from Lab 2)

Each row maps a failure mode to the **symptom the client actually sees** and the **retry policy that mitigates it**.

| Failure   | Symptom in client log                          | Fast / slow fail | Retry policy that wins           | Why it wins                                                                 |
| --------- | ---------------------------------------------- | ---------------- | -------------------------------- | --------------------------------------------------------------------------- |
| **Crash**     | `FAIL err=ECONNRESET`                          | Fast fail (kernel RST) | **Bounded retry + timeout** (default); **no retry** for latency-optimal | Node is gone. Cap total wait, then give up cleanly. No retry is fastest because the kernel typically surfaces the RST almost immediately in well-behaved networks. |
| **Omission**  | `FAIL err=ECONNREFUSED` or `FAIL err=ECONNRESET` (intermittent) | Fast fail (kernel reports quickly) | **Fast retry, no backoff** (bounded) for isolated loss; add small jittered backoff if omission looks load-related | Link is healthy but one packet vanished. Retrying immediately gives the link its best chance to heal on the next try. |
| **Temporal**  | `FAIL err=timeout` at every call               | Slow fail (waits full timeout) | **Retry with longer timeout** (or fail fast if SLA forbids long waits) | Server is consistently slow, not flaky. Backoff cannot fix constant slowness. Only a longer timeout can. |

**Decision rule:** match the **signature**, not the symptom. A fast fail (`ECONNRESET` / `ECONNREFUSED`) means the network already told you the truth: retry quickly or not at all. A slow fail (`timeout`) means the network kept you waiting: only a longer timeout or a different node helps.

---

## C. Consistency Level → Latency Trade-off (from Lab 3)

Qualitative summary of what we observed between eventual and strong consistency on the Redis master-replica pair.

| Mode                          | Staleness behaviour                           | Latency character                                | Pick when                                              |
| ----------------------------- | --------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| **Eventual** (read replica)   | Non-zero; replica lags the master by a few writes | Lower: just the read round-trip                 | Stale reads are tolerable: counters, feeds, catalogues |
| **Strong** (`WAIT` + read)    | Zero for the waiting writer (read-your-writes) | Higher: adds the cost of waiting for replication | Writer must see its own write: money, inventory, seats  |

**Trade-off in one line:** strong consistency **removes staleness by adding latency**; eventual consistency **removes latency by allowing staleness**.

**Direction of the gap:** in the lab, strong added a small but measurable amount of latency on top of the eventual read. In real systems (with cross-AZ replicas, network jitter, and load) that gap grows substantially. Treat the lab result as the *direction* of the trade-off, not its magnitude.

**Important caveat:** `WAIT` gives the *waiting client* read-your-writes. Other clients reading from the replica in parallel can still see stale data until replication catches up. It is not full linearizability.

---

## D. Cross-cutting decision rules

These combine all three labs into rules you can apply at design time.

| Scenario                                        | Transport | Consistency                                       | Retry policy                                      |
| ----------------------------------------------- | --------- | ------------------------------------------------- | ------------------------------------------------- |
| Browser-facing public API                       | REST      | Strong for read-after-write; eventual elsewhere   | Bounded retry + timeout                           |
| Internal microservice, fixed schema             | gRPC      | Per-key: strong for money/inventory, eventual for feeds | Bounded retry + timeout for crash, fast retry for omission |
| Read-heavy feed / catalogue / like-count        | gRPC      | Eventual                                          | Fast retry, no backoff                            |
| Payment / seat reservation / inventory write    | gRPC      | Strong                                            | Longer timeout, fail fast (use idempotency keys for retry safety) |
| Third-party integration you don't control       | REST      | Match their SLA                                   | Bounded retry + longer timeout                    |

**The single principle across all three labs:** every distributed-system choice trades one thing for another: bytes for readability (REST vs gRPC), latency for freshness (eventual vs strong), and waiting time for success rate (retry policy). Pick the trade-off that matches the cost of being wrong.


## Conclusion

Every distributed-system choice trades one thing for another. Pick the trade-off that matches the cost of being wrong in your specific scenario. There is no globally correct answer.
