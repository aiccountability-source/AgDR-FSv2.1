# Live Sparse Merkle Tree Sensory Spine

## AgDR-FullSystem v2.1 — Technical Specification

---

## Overview

The **Live Sparse Merkle Tree (SMT) Sensory Spine** is the persistent, tamper-evident memory substrate of the AgDR-FullSystem v2.1. It functions as the nervous system's spinal column — a cryptographically chained, forward-secret data structure through which every autonomous decision must pass before it is permitted to fire.

Unlike conventional logging systems that append records after the fact, the Sparse Merkle Spine is **alive**: it participates in inference by feeding the most recent 100 Merkle-chained records directly into the model's context window at every step. The spine is not merely a record of what happened — it is an active participant in what happens next.

---

## Architecture

### Data Structure: 256-Bit Depth Sparse Merkle Tree

The spine is implemented as a Sparse Merkle Tree with a depth of 256 bits, instantiated via the `AgDRSparseMerkleChain` class. This depth corresponds to the full address space of BLAKE2b hashes, ensuring that every possible record key maps to a unique leaf position without collision risk within any computationally feasible timeframe.

The SMT is backed by a production-grade SQLite database (`agdr_fullsystem_v2_1.db`) operating in `IMMEDIATE` isolation mode. This guarantees that all write operations acquire an exclusive lock at transaction start, preventing concurrent writes from corrupting the chain — a critical property when multiple agents in a swarm share the same spine.

### Schema

The underlying SQLite table captures the complete AgDR record:

```
key             TEXT PRIMARY KEY
timestamp_us    INTEGER
ppp_json        TEXT
reasoning_trace TEXT
output          TEXT
signature       TEXT
merkle_root     TEXT
zk_proof        TEXT
coherence_score REAL
foi_anchor      TEXT
```

Each column maps directly to a field on the `AgDRRecord` dataclass, ensuring a one-to-one correspondence between the in-memory record and its persistent representation.

---

## Forward-Secret Chain Verification

### Startup Integrity Check

On every system initialization, the `_verify_chain()` method performs an exhaustive forward walk of the entire record history. Beginning from the `"genesis"` sentinel root, it reconstructs each expected Merkle root by computing:

```
BLAKE2b(previous_root + JSON(record, sort_keys=True))
```

If any computed root fails to match the stored `merkle_root` for that record, the assertion fails immediately and the system refuses to start. This is not a soft warning — it is a hard abort. The philosophy is unambiguous: a corrupted chain is a dead chain. No inference may proceed on a tampered history.

This verification is the cryptographic backbone that transforms ordinary database rows into an immutable, court-admissible audit trail.

### Why Forward Secrecy Matters

Forward secrecy in this context means that knowledge of any single Merkle root reveals nothing about prior roots. Each root is derived from the previous root concatenated with the current record's full content, then hashed through BLAKE2b. An adversary who obtains root `N` cannot reverse-engineer root `N-1` without possessing the exact content of record `N`. This property ensures that even partial chain exposure does not compromise the integrity of the historical audit trail.

---

## Sensory Context Window

### The Last 100 Records

The `get_last_n(100)` method retrieves the most recent 100 records from the spine and formats them into a structured sensory context string via `_format_sensory()`. This string is prepended to every model inference call, giving the model direct visibility into its own recent decision history.

The sensory context follows this format:

```
[AGDR SENSORY SPINE v2.1 — Sparse Merkle + Embeddings + Wisdom]
t={timestamp}µs | PPP={ppp_json} | Output: {truncated_output}...
t={timestamp}µs | PPP={ppp_json} | Output: {truncated_output}...
...
[END SPINE — Drift only intentional, provable, and human-anchored]
```

This is not window dressing. The sensory spine transforms a stateless model into a stateful agent with cryptographic memory. The model does not merely generate — it generates in the context of everything it has done before, as witnessed by an unbroken Merkle chain.

### Latency Enforcement: The 3.94 µs Bound

Every read from the spine is instrumented with nanosecond-precision timing via `time.perf_counter_ns()`. If a read exceeds 3,940 nanoseconds (3.94 µs), the system raises a `RuntimeError` and halts. This is the same latency bound enforced by the core AKI atomic capture layer.

The rationale: if the spine cannot be read within the time it takes light to travel approximately 1.2 kilometers, something is fundamentally wrong with the system state. The bound serves as both a performance guarantee and a canary — if it trips, the system is either under resource contention, disk degradation, or adversarial delay, and no decision should be trusted.

---

## Atomic Insert

### Transaction Semantics

The `insert()` method operates within a SQLite context manager (`with self.conn`), which guarantees atomic transaction semantics. Either the entire insert (SMT update + SQLite row write) succeeds, or nothing is committed. There is no intermediate state where the SMT has been updated but the database has not, or vice versa.

The insert flow:

1. Compute the new SMT root by inserting the serialized record at the given key
2. Write the full record (with the new root) to the SQLite `records` table
3. If either operation fails, the transaction rolls back entirely

This atomicity is critical in the swarm context, where multiple agents may be inserting concurrently. The `IMMEDIATE` isolation level prevents write starvation and ensures serializable execution of inserts.

---

## Integration with the Decision Pipeline

The spine occupies steps 1 and 8 of the 10-step `AgDRFullSystem.step()` pipeline:

**Step 1 — Sensory Read:** The last 100 records are retrieved and formatted into the sensory context, which is injected into the model's input alongside the new observation.

**Step 8 — Atomic SMT Insert:** After BFT consensus validation passes, the sealed record is atomically inserted into the spine. Only after this insert succeeds does the decision "fire" — that is, only after the record is permanently and verifiably committed to the chain does the system return the decision to the caller.

This ordering is deliberate: no decision exists until its record exists. The record is the decision. The spine is the witness.

---

## Design Rationale

The Sparse Merkle Tree was chosen over simpler append-only chains for several reasons:

**Efficient membership proofs.** Any individual record's inclusion (or non-inclusion) can be proven with a logarithmic-depth path from leaf to root, without revealing any other records. This is essential for the public audit API, where a verifier may need to confirm that a specific decision was recorded without accessing the full history.

**Deterministic structure.** Unlike traditional Merkle trees whose shape depends on insertion order, the SMT's structure is determined entirely by the key space. This makes verification reproducible across independent implementations — a critical property for legal admissibility.

**Forward compatibility with zk-proof circuits.** The 256-bit depth aligns with standard zero-knowledge proof systems (zk-STARKs, zk-SNARKs), enabling future on-chain verification of spine integrity without revealing record contents. The `zk_proof` field on each record is the hook for this capability.

---

## For Implementors

If you are extending or integrating with the Sparse Merkle Spine:

- **Never bypass the spine.** Every decision must flow through `AgDRFullSystem.step()`. Direct model calls that skip the spine produce unrecorded, unaccountable decisions.
- **Respect the latency bound.** If your deployment environment cannot consistently achieve sub-4µs SQLite reads, consider deploying the database on a RAM-backed filesystem or NVMe storage.
- **Chain verification is non-negotiable.** Do not disable or weaken the startup `_verify_chain()` assertion. If verification is slow on large chains, optimize the verification — do not skip it.
- **The spine is shared in swarm mode.** All agents in a `swarm_step()` call share the same `AgDRSparseMerkleChain` instance. Plan your concurrency model accordingly.

---

*The spine is alive. Trust the record.*

*Built on the AgDR v0.2 + AKI standard from [accountability.ai](https://accountability.ai)*
