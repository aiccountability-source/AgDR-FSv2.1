# Decision Pipeline and Atomic AKI Capture

## AgDR-FullSystem v2.1 — Technical Specification

---

## Overview

The AgDR-FullSystem v2.1 decision pipeline is a 10-step atomic sequence that transforms a raw observation into a cryptographically sealed, court-admissible decision record. No step can be skipped, reordered, or partially executed. The pipeline is implemented in the `AgDRFullSystem.step()` async method and enforces the core principle: no decision exists until its record exists.

The Atomic AKI (Accountability Kernel Instant) capture at Step 6 is the indivisible kernel transaction at the exact inference instant — the "i" point — with a target latency of 3.94 microseconds.

---

## The 10-Step Pipeline

### Step 1 — Sensory Spine Read

Retrieves the 100 most recent Merkle-chained records and formats them into sensory context. Subject to the 3.94 us latency bound. See `SPARSE_MERKLE_SPINE.md`.

### Step 2 — Model Inference

The base model receives full context (spine + observation) and returns `(reasoning_trace, proposed_output)`. Interface: `async generate(ctx: str) -> Tuple[str, str]`.

### Step 3 — Embedding Deviation Critic

Computes weighted cosine similarity (all-MiniLM-L6-v2) against last 60 spine records. Threshold: 0.85. Drift triggers correction + FOI escalation. See `DEVIATION_CRITIC.md`.

### Step 4 — PPP Triplet Construction

Constructs Provenance-Place-Purpose policy triplet anchoring the decision to identity, jurisdiction, and fiduciary intent. See `PPP_TRIPLET_AND_FOI.md`.

### Step 5 — zk-Proof Generation

Generates BLAKE2b digest proof attesting to coherence score and PPP integrity. Interface stub for production zk-STARK/zk-SNARK circuits.

### Step 6 — Atomic AKI Capture (3.94 us)

```python
record_obj = aki_capture(
    ctx=full_context,
        reasoning_trace=reasoning_trace,
            output=proposed_output,
                ppp_triplet=ppp,
                    human_delta_chain=self.foi_chain
                    )
                    ```

                    The indivisible kernel transaction. Seals context, reasoning, output, PPP, and FOI chain with BLAKE3 hash + Ed25519 signature at the exact inference instant. This is where computation becomes record.

                    ### Step 7 — Record Sealing

                    All fields assembled into the sealed record dictionary. Key is BLAKE2b hash of `agent_id:timestamp_ns`. Signature from AKI capture (Step 6).

                    ### Step 8 — BFT Consensus Validation

                    Validates sealed record via Byzantine consensus. Single-agent: zk-proof verification. Swarm: 2/3 supermajority quorum. Failure triggers atomic rollback. See `BYZANTINE_CONSENSUS.md`.

                    ### Step 9 — Atomic SMT Insert

                    Record atomically inserted into Sparse Merkle Tree + SQLite within single transaction. All or nothing. See `SPARSE_MERKLE_SPINE.md`.

                    ### Step 10 — Decision Fires + Public Audit Hook

                    Decision returns to caller. High-coherence traces (>0.95) flagged for distillation. Public audit key + Merkle root logged.

                    ---

                    ## Atomicity Guarantees

                    **AKI Capture Atomicity (Step 6):** Indivisible kernel transaction at 3.94 us. No intermediate state between inference and record.

                    **SQLite Transaction Atomicity (Step 9):** IMMEDIATE isolation. Full commit or full rollback.

                    **Pipeline Atomicity (Steps 1-10):** Any step failure rolls back the entire pipeline. Merkle root unchanged. Decision never fires.

                    ---

                    ## Swarm Orchestration

                    ```python
                    async def swarm_step(self, observations):
                        tasks = [self.step(obs, aid) for obs, aid in observations]
                            return await asyncio.gather(*tasks)
                            ```

                            Parallel execution via asyncio.gather(). Shared AgDRSparseMerkleChain instance. SQLite IMMEDIATE isolation serializes concurrent inserts. Failed agents excluded from results.

                            ---

                            ## Public Audit API

                            ```python
                            def public_audit_report(self) -> Dict:
                                return {
                                        "merkle_root": self.chain.smt.root,
                                                "total_records": self.chain.conn.execute(
                                                            "SELECT COUNT(*) FROM records"
                                                                    ).fetchone()[0]
                                                                        }
                                                                        ```

                                                                        External verifiers can query Merkle root and total records for third-party verification.

                                                                        ---

                                                                        ## Configuration

                                                                        | Constant | Value | Purpose |
                                                                        |----------|-------|---------|
                                                                        | `DB_PATH` | `agdr_fullsystem_v2_1.db` | SQLite spine database |
                                                                        | `MAX_HISTORY` | `1000` | Memory record limit |
                                                                        | `COHERENCE_THRESHOLD` | `0.85` | Drift detection threshold |

                                                                        ---

                                                                        ## For Implementors

                                                                        - **Implement base_model interface:** `async generate(ctx) -> Tuple[str, str]`
                                                                        - **Never call steps out of order.** Guarantees depend on sequential execution.
                                                                        - **Decision does not exist until Step 10.** Unrecommitted data is unaccountable.
                                                                        - **Use swarm_step() for parallel agents.** Do not share databases across instances.

                                                                        ---

                                                                        Built on the AgDR v0.2 + AKI standard from [accountability.ai](https://accountability.ai)
