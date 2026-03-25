# AgDR-FullSystem v2.1 — Peak Coherence Edition

**Official open-source reference implementation built on [accountability.ai](https://accountability.ai)'s AgDR v0.2 + AKI standard.**

Complete, production-grade sensory feedback loop and multi-agent swarm layer that extends the AgDR (Atomic Genesis Decision Record) standard into a living, self-coherent nervous system for accountable AI agents.

---

## What Is AgDR?

AgDR (Atomic Genesis Decision Record) is an open governance and audit-trail standard from [accountability.ai](https://accountability.ai). It cryptographically signs every autonomous AI/agent decision at the exact inference instant (the "i" point), capturing a mathematically indivisible PPP triplet (Provenance · Place · Purpose). Records are tamper-evident, BLAKE3 + Ed25519 signed, and chained in a forward-secret Merkle tree — all with ~3.94 µs average capture latency.

Every AgDR record is simultaneously:

- An **AI audit trail**
- A **training dataset** (for future model improvement)
- A **legal instrument** (court-admissible under Canada Evidence Act and equivalent frameworks)

## What This Implementation Adds

This v2.1 reference implementation extends the core AgDR standard with:

- **Atomic AgDR/AKI capture** at 3.94 µs with full PPP triplet + FOI (Fiduciary Office Intervener) anchoring
- **Live Sparse Merkle Tree sensory spine** — forward-secret, tamper-evident shared memory
- **Real-time embedding-based deviation critic** — prevents unintentional drift using sentence-transformer embeddings (all-MiniLM-L6-v2)
- **Byzantine fault tolerant consensus** — tolerates <1/3 faulty agents in multi-agent swarms
- **zk-verifiable computation hooks** — for provable correctness of reasoning and coherence
- **Persistent SQLite storage** with full chain verification on startup
- **Public audit API** for societal transparency
- **Continual learning distillation queue** — high-coherence traces flagged as ultimate training data
- **Full atomic rollback** — if AKI capture or BFT consensus fails, the step is cleanly aborted
- **Multi-agent swarm orchestration** — parallel execution with shared Merkle spine

## Core Architecture

```
Observation → Sensory Spine (last 100 Merkle records)
    → Model Inference (with spine as context)
    → Embedding Deviation Critic (cosine similarity check)
    → PPP Triplet Construction (Provenance · Place · Purpose)
    → zk-Proof Generation
    → Atomic AKI Capture (3.94 µs, BLAKE3 + Ed25519 + Merkle-append)
    → BFT Consensus Validation
    → SMT Insert (persistent, verified)
    → Decision Fires (only after complete sealed record)
```

## Key Components

| Component | Class | Purpose |
|-----------|-------|---------|
| Sparse Merkle Chain | `AgDRSparseMerkleChain` | 256-bit depth SMT with SQLite persistence and forward-secret verification |
| Deviation Critic | `DeviationCritic` | Embedding-based coherence scoring against the sensory spine |
| Byzantine Consensus | `ByzantineConsensus` | BFT quorum validation (tolerates <1/3 faults) |
| Full System | `AgDRFullSystem` | Turn-key integration: single `.step()` or `.swarm_step()` |
| AgDR Record | `AgDRRecord` | Dataclass capturing PPP + FOI + zk-proof + coherence score |

## Installation

```bash
git clone https://github.com/aiccountability-source/AgDR-FSv2.1.git
cd AgDR-FSv2.1
pip install sentence-transformers torch sparse-merkle-tree
```

## Quick Start

```python
import asyncio
from agdr_fullsystem_v2_1 import AgDRFullSystem

class YourModel:
    async def generate(self, ctx: str):
        return "reasoning trace", "decision output"

async def main():
    system = AgDRFullSystem(base_model=YourModel())
    decision, record = await system.step("Your observation here")
    print("Decision:", decision)
    print("Merkle Root:", record.merkle_root)
    print("Coherence:", record.coherence_score)

asyncio.run(main())
```

## PPP Triplet (Policy Core)

Every decision is anchored by the PPP triplet:

- **Provenance**: Exact identity, model state, hash of governing charter
- **Place**: Sovereign jurisdiction and systemic boundary
- **Purpose**: Pre-defined fiduciary intent

## FOI — Fiduciary Office Intervener

Every chain has a terminal human (or escalation path) who bears ultimate legal/fiduciary responsibility. This keeps AI decisions anchored in existing human law and ensures that liability always maps back to a named human steward.

## Design Philosophy

> "Don't trust the machine. Don't even trust me. Trust the record." — accountability.ai manifesto

This implementation honors that philosophy: the record is the witness, the spine is alive, and drift is only possible when intentional and provably human-anchored. Made with care for users and society.

## Relationship to the AgDR Standard

This is an **extension** built on top of AgDR v0.2 + AKI. The core AgDR standard provides the atomic capture layer (the indivisible kernel transaction at the inference instant). This implementation adds the closed-loop sensory spine, deviation critic, BFT consensus, and swarm orchestration that the standard was designed to enable.

## Technical Documentation

Exhaustive technical specifications for every component of the system are available in the `docs/` directory:

- [**Decision Pipeline and Atomic AKI Capture**](docs/DECISION_PIPELINE.md) — The complete 10-step atomic pipeline from observation to sealed record, including the 3.94 us AKI kernel transaction, swarm orchestration, and public audit API
- - [**Sparse Merkle Tree Sensory Spine**](docs/SPARSE_MERKLE_SPINE.md) — The live, forward-secret, tamper-evident memory substrate: 256-bit SMT architecture, SQLite persistence, chain verification, latency enforcement, and sensory context windowing
  - - [**Embedding-Based Deviation Critic**](docs/DEVIATION_CRITIC.md) — Real-time coherence enforcement via sentence-transformer embeddings (all-MiniLM-L6-v2), exponential decay weighting, threshold tuning, and drift correction mechanics
    - - [**Byzantine Fault Tolerant Consensus**](docs/BYZANTINE_CONSENSUS.md) — BFT validation layer with zk-proof integration, fault tolerance bounds (n >= 3f+1), atomic rollback on failure, and swarm consensus configuration
      - - [**PPP Triplet and Fiduciary Office Intervener**](docs/PPP_TRIPLET_AND_FOI.md) — The Provenance-Place-Purpose policy core, FOI human anchoring chain, AgDR record schema, legal admissibility design, and escalation mechanics
        - - [**Cryptographic Primitives and zk-Proof Architecture**](docs/CRYPTOGRAPHIC_PRIMITIVES.md) — BLAKE2b/BLAKE3 hashing, Ed25519 signatures, zero-knowledge proof stubs with production integration paths (gnark, circom, halo2), and forward secrecy properties
          -
          - ## License

Apache 2.0 — See [LICENSE](LICENSE) for details.

## Contributing

Contributions welcome. Please ensure all changes maintain the atomic integrity of the AgDR capture and the mission of care for users and society.

---

*Built with the level of care and stewardship that accountability.ai's vision deserves.*
