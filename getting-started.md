---
layout: default
title: Getting Started
nav_order: 2
description: "Get up and running with AgDR Eternal Witness v3.0 in minutes. Install, configure, and seal your first cryptographically-verified AI decision record."
permalink: /getting-started/
---

# Getting Started

AgDR Eternal Witness v3.0 wraps any AI model and turns every inference into a cryptographically sealed, court-admissible record — atomically, in under 100 µs.

This guide takes you from zero to your first sealed decision record in under 5 minutes.

---

## Prerequisites

- Python 3.9 or higher
- pip
- git

No cloud account, no API key, no sign-up required. Fully local and open-source.

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/aiccountability-source/AgDR-FSv2.1.git
cd AgDR-FSv2.1
```

---

## Step 2 — Install Dependencies

```bash
pip install -r requirements.txt
```

This installs all required packages including the sentence-transformer embedding model, BLAKE3, Ed25519 signing, and the SQLite-backed Sparse Merkle Tree.

---

## Step 3 — Run Your First Sealed Inference (v3.0)

Create a file called `my_first_record.py` and paste the following:

```python
import asyncio
from agdr_eternal_witness_v3_0 import AgDREternalWitness

# 1. Define your model — wrap any AI inference function
class MyModel:
    async def generate(self, ctx):
        # Replace with your real model call
        reasoning = "I analysed the input and determined the safest course."
        decision  = "Proceed with action A."
        return reasoning, decision

# 2. Initialise the Eternal Witness
#    fo_i = Fiduciary Office Intervener — the responsible human anchor
system = AgDREternalWitness(
    model=MyModel(),
    fo_i="Your Name (Toronto)"
)

# 3. Step — observe, infer, seal, record
async def main():
    observation = "High-stakes input requiring an accountable decision."
    decision, record = await system.step(observation)

    print(f"Decision : {decision}")
    print(f"Record ID: {record['aki_id']}")
    print(f"Merkle   : {record['merkle_root']}")
    print(f"Latency  : {record['capture_latency_us']:.1f} µs")

asyncio.run(main())
```

Run it:

```bash
python my_first_record.py
```

**Expected output:**

```
Decision : Proceed with action A.
Record ID: agdr-<blake3-hash>
Merkle   : <root-hash>
Latency  : 72.3 µs
```

Every run appends a new, immutable record to your local Sparse Merkle Tree — tamper-evident, BLAKE3 + Ed25519 signed, and ready for audit.

---

## Step 4 — Use v2.1 (Stable Reference)

If you prefer the stable v2.1 implementation:

```bash
git checkout v2.1.0
```

```python
import asyncio
from agdr_fullsystem_v2_1 import AgDRFullSystem

class MyModel:
    async def generate(self, ctx):
        return "My reasoning.", "My decision."

system = AgDRFullSystem(model=MyModel(), fo_i="Your Name (Toronto)")

async def main():
    decision, record = await system.step("My observation.")
    print(f"Decision : {decision}")
    print(f"Record ID: {record['aki_id']}")

asyncio.run(main())
```

---

## What Happens Under the Hood

Every call to `system.step()` executes the following 10-step atomic pipeline:

| Step | Component | Description |
|---|---|---|
| 1 | Sensory Spine | Loads the last 500 Merkle-chained records as context |
| 2 | Model Inference | Calls your model with the spine as context |
| 3 | Deviation Critic | Cosine-similarity check against historical embeddings |
| 4 | PPP Triplet | Constructs Provenance, Place, Purpose record |
| 5 | NANOZK Proof | Generates zero-knowledge reasoning proof |
| 6 | AKI Capture | BLAKE3 + Ed25519 atomic kernel transaction |
| 7 | BFT Consensus | WeightedQuorumBFT validation across swarm |
| 8 | Merkle Insert | JellyfishSMT persistent append |
| 9 | Verification | Full chain integrity check |
| 10 | Decision Fires | Released only after complete sealed record |

If any step fails, the transaction rolls back atomically — the Merkle root stays untouched.

---

## Verify Your Audit Chain

After running several inferences, verify the full chain integrity:

```python
from agdr_eternal_witness_v3_0 import AgDREternalWitness

system = AgDREternalWitness(model=None, fo_i="Your Name (Toronto)")
result = system.verify_chain()
print(f"Chain valid: {result['valid']}")
print(f"Records    : {result['record_count']}")
print(f"Root hash  : {result['merkle_root']}")
```

---

## Configuration Options

| Parameter | Default | Description |
|---|---|---|
| `fo_i` | Required | Fiduciary Office Intervener — human accountability anchor |
| `coherence_threshold` | `0.85` | Minimum embedding similarity before deviation alert |
| `bft_quorum` | `0.67` | Minimum quorum fraction for BFT consensus |
| `spine_window` | `500` | Number of past records loaded as sensory context |
| `db_path` | `agdr.db` | SQLite path for the persistent Merkle spine |

---

## Next Steps

- Read the [full technical documentation](https://github.com/aiccountability-source/AgDR-FSv2.1/tree/main/docs) for deep-dives into each component
- Review the [v3.0 release notes](https://github.com/aiccountability-source/AgDR-FSv2.1/releases/tag/v3.0.0) for all new capabilities
- Understand the [AgDR standard](https://github.com/aiccountability-source/AgDR) and the PPP policy core
- Visit [accountability.ai](https://accountability.ai) — the governance standard this is built on

---

## License

Dual licensed under [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/) OR [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) — open, royalty-free, dedicated to the ecosystem.
