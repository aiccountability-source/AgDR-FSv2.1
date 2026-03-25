# Embedding-Based Deviation Critic

## AgDR-FullSystem v2.1 — Technical Specification

---

## Overview

The Deviation Critic is the real-time coherence enforcement engine of AgDR-FullSystem v2.1. It uses sentence-transformer embeddings to measure semantic similarity between a proposed reasoning trace and the agent's recent decision history stored on the Sparse Merkle Spine. Its purpose is singular and non-negotiable: prevent unintentional drift from entering the immutable record.

The Deviation Critic occupies **Step 3** of the 10-step decision pipeline. Every proposed inference must pass through this gate before it can proceed to PPP triplet construction, zk-proof generation, BFT consensus, and ultimately the atomic Merkle append. If the critic detects unintentional drift, the output is corrected in place and flagged for FOI (Fiduciary Office Intervener) escalation.

Drift is permitted only when it is intentional, provable, and human-anchored.

---

## Position in the Decision Pipeline

```
Step 1 — Sensory Spine Read (last 100 Merkle records)
Step 2 — Model Inference (with spine context)
Step 3 — Embedding Deviation Critic  ◄ THIS LAYER
Step 4 — PPP Triplet Construction
Step 5 — zk-Proof Generation
Step 6 — Atomic AKI Capture (3.94 µs)
Step 7 — Record Sealing (BLAKE3 + Ed25519 signing)
Step 8 — BFT Consensus Validation
Step 9 — Atomic SMT Insert
Step 10 — Decision Fires + Public Audit Hook
```

---

## Implementation: `DeviationCritic` Class

### Class Signature

```python
class DeviationCritic:
    def __init__(self):
            self.embedder = SentenceTransformer('all-MiniLM-L6-v2')
                    self.embedder.eval()
                            self.threshold = COHERENCE_THRESHOLD  # 0.85

                                def score(
                                        self,
                                                proposed_trace: str,
                                                        history: List[Dict]
                                                            ) -> Tuple[float, str]
                                                            ```

                                                            ### Embedding Model

                                                            The critic uses **all-MiniLM-L6-v2** from the `sentence-transformers` library. This model produces 384-dimensional dense vector representations of text, optimized for semantic similarity tasks. Key characteristics:

                                                            | Property | Value |
                                                            |----------|-------|
                                                            | Model | `all-MiniLM-L6-v2` |
                                                            | Embedding Dimension | 384 |
                                                            | Max Sequence Length | 256 tokens |
                                                            | Similarity Metric | Cosine similarity |
                                                            | Inference Mode | `eval()` (no gradient computation) |
                                                            | Source | Hugging Face `sentence-transformers` |

                                                            The model is set to evaluation mode (`self.embedder.eval()`) at initialization. This disables dropout and batch normalization updates, ensuring deterministic embeddings for identical inputs — a critical property for reproducible coherence scoring.

                                                            ---

                                                            ## Scoring Algorithm

                                                            ### `score()` Method

                                                            The `score()` method computes a weighted coherence score between a proposed reasoning trace and the most recent 60 entries from the Merkle spine history.

                                                            **Parameters:**

                                                            | Parameter | Type | Description |
                                                            |-----------|------|-------------|
                                                            | `proposed_trace` | `str` | The reasoning trace produced by the model at Step 2 |
                                                            | `history` | `List[Dict]` | Recent AgDR records from the Sparse Merkle Spine (typically last 100) |

                                                            **Returns:** `Tuple[float, str]` — A coherence score between 0.0 and 1.0, and a status string (`"COHERENT"` or `"UNINTENTIONAL_DRIFT"`).

                                                            ### Computation Steps

                                                            1. **Empty History Check:** If `history` is empty (cold start), return `(1.0, "COHERENT")`. A system with no prior decisions cannot have drifted.

                                                            2. **Proposed Embedding:** Encode the `proposed_trace` string into a 384-dimensional vector using `self.embedder.encode()`. Wrap the resulting numpy array in a PyTorch tensor and add a batch dimension via `unsqueeze(0)`.

                                                            3. **History Iteration:** For each of the last 60 records in `history`:
                                                               - Concatenate `reasoning_trace` and `output` fields into a single comparison string: `"{reasoning_trace} | {output}"`
                                                                  - Encode this string into a 384-dimensional embedding
                                                                     - Compute cosine similarity against the proposed embedding using `torch.nn.functional.cosine_similarity()`

                                                                     4. **Exponential Decay Weighting:** Apply an exponential decay weight of `0.96^(distance)` to each similarity score, where `distance` is the number of positions from the most recent record. This gives recent decisions significantly more influence than older ones:
                                                                        - Most recent record: weight = 1.0 (0.96^0)
                                                                           - 10 records ago: weight ≈ 0.665
                                                                              - 30 records ago: weight ≈ 0.294
                                                                                 - 60 records ago: weight ≈ 0.086

                                                                                 5. **Weighted Average:** Compute the final coherence score as:
                                                                                    ```
                                                                                       coherence = sum(score_i * weight_i) / sum(weight_i)
                                                                                          ```
                                                                                             This normalizes the weighted sum to account for the variable number of history records.

                                                                                             6. **Status Classification:** If `coherence < self.threshold` (default 0.85), return `"UNINTENTIONAL_DRIFT"`. Otherwise, return `"COHERENT"`.

                                                                                             ---

                                                                                             ## Coherence Threshold

                                                                                             The global coherence threshold is defined in the module configuration:

                                                                                             ```python
                                                                                             COHERENCE_THRESHOLD = 0.85
                                                                                             ```

                                                                                             This value was selected to balance sensitivity against false positives. At 0.85:

                                                                                             - Decisions that are semantically aligned with the recent history (cosine similarity > 0.85 on average) pass through cleanly.
                                                                                             - Decisions that introduce novel reasoning or shift domain focus trigger the drift flag, which does not block the decision but corrects the output and escalates to FOI.
                                                                                             - The threshold is deliberately strict because the system philosophy holds that it is better to flag a potentially coherent decision for human review than to silently commit a drifting one to the immutable record.

                                                                                             ### Threshold Tuning Guidance

                                                                                             | Threshold | Behavior | Use Case |
                                                                                             |-----------|----------|----------|
                                                                                             | 0.90+ | Very strict; flags most novel reasoning | High-stakes legal/medical domains |
                                                                                             | 0.85 | Default; balanced strictness | General accountability workloads |
                                                                                             | 0.75 | Permissive; allows moderate domain shifts | Exploratory research agents |
                                                                                             | < 0.70 | Not recommended | May allow semantic drift to accumulate |

                                                                                             ---

                                                                                             ## Drift Correction Mechanism

                                                                                             When the Deviation Critic returns `"UNINTENTIONAL_DRIFT"`, the `AgDRFullSystem.step()` method does not halt — it corrects the output in place:

                                                                                             ```python
                                                                                             if status == "UNINTENTIONAL_DRIFT":
                                                                                                 proposed_output = (
                                                                                                         f"[AgDR v2.1 CORRECTED via sensory spine] "
                                                                                                                 f"{proposed_output} | FOI ESCALATION"
                                                                                                                     )
                                                                                                                     ```
                                                                                                                     
                                                                                                                     This correction serves three purposes:
                                                                                                                     
                                                                                                                     1. **Transparency:** The correction is visible in the final record. Anyone auditing the chain can see that this decision was drift-corrected.
                                                                                                                     2. **Escalation Signal:** The `FOI ESCALATION` tag alerts the Fiduciary Office Intervener that human review is warranted.
                                                                                                                     3. **Preservation:** The original reasoning is preserved in the `reasoning_trace` field. The correction only modifies the `output` field, maintaining full provenance.
                                                                                                                     
                                                                                                                     The corrected output then continues through the remaining pipeline steps (PPP construction, zk-proof, BFT consensus, Merkle append) and is permanently sealed in the spine with its drift-corrected status visible.
                                                                                                                     
                                                                                                                     ---
                                                                                                                     
                                                                                                                     ## Exponential Decay Rationale
                                                                                                                     
                                                                                                                     The decay factor of `0.96` per position creates a recency-biased comparison window. This is not arbitrary — it reflects the design principle that an agent's coherence should be measured primarily against its recent behavior, not its distant past.
                                                                                                                     
                                                                                                                     Without decay, an agent that gradually shifts its reasoning domain over hundreds of decisions would appear coherent at each step (high similarity with the immediately prior decision) but could be dramatically different from its behavior 500 decisions ago. The decay weighting partially mitigates this by ensuring that very old records contribute negligibly to the score, while maintaining enough historical context (60 records at 0.96 decay) to detect multi-step drift patterns.
                                                                                                                     
                                                                                                                     The practical effect: the "effective window" of comparison is approximately 25-30 records (where weights are still above 0.3), with a long tail of diminishing influence out to 60.
                                                                                                                     
                                                                                                                     ---
                                                                                                                     
                                                                                                                     ## Integration with the Sensory Spine
                                                                                                                     
                                                                                                                     The Deviation Critic receives its `history` parameter from the Sensory Spine read at Step 1:
                                                                                                                     
                                                                                                                     ```python
                                                                                                                     history = self.chain.get_last_n(100)
                                                                                                                     ```
                                                                                                                     
                                                                                                                     This means the critic has access to the 100 most recent Merkle-chained records, but only uses the most recent 60 for scoring (via `history[-60:]`). The remaining 40 records are available for the sensory context formatting but are not factored into the coherence calculation.
                                                                                                                     
                                                                                                                     The 100-record read is subject to the 3.94 µs latency bound enforced by the Sparse Merkle Spine. If the spine read exceeds this bound, the entire step aborts before the critic is even invoked.
                                                                                                                     
                                                                                                                     ---
                                                                                                                     
                                                                                                                     ## Performance Characteristics
                                                                                                                     
                                                                                                                     | Metric | Value | Notes |
                                                                                                                     |--------|-------|-------|
                                                                                                                     | Embedding latency (per string) | ~2-5 ms | Depends on text length and hardware |
                                                                                                                     | Cosine similarity computation | < 0.1 ms | Single tensor operation |
                                                                                                                     | Total critic overhead (60 comparisons) | ~120-300 ms | Dominated by embedding computation |
                                                                                                                     | Memory footprint (model loaded) | ~80 MB | all-MiniLM-L6-v2 with PyTorch |
                                                                                                                     
                                                                                                                     The critic is the most computationally expensive step in the pipeline (by wall-clock time), but this cost is justified by the guarantee it provides: no unintentional drift enters the immutable record.
                                                                                                                     
                                                                                                                     For latency-sensitive deployments, consider pre-computing and caching embeddings for spine records (they are immutable once committed), reducing the critic to a single embedding computation (the proposed trace) plus 60 cached cosine similarity lookups.
                                                                                                                     
                                                                                                                     ---
                                                                                                                     
                                                                                                                     ## For Implementors
                                                                                                                     
                                                                                                                     - **Do not disable the critic.** It is the primary defense against semantic drift in the decision record. Disabling it is equivalent to removing the brakes from a vehicle.
                                                                                                                     - **Cache spine embeddings.** Since Merkle spine records are immutable, their embeddings never change. Pre-compute and cache them to reduce critic latency from ~200 ms to ~5 ms.
                                                                                                                     - **Monitor coherence distributions.** Track the distribution of coherence scores over time. A downward trend indicates systematic drift that may require model retraining or threshold adjustment.
                                                                                                                     - **Respect the classification.** A status of `"UNINTENTIONAL_DRIFT"` means the system has detected a semantic shift that was not explicitly authorized. Do not ignore this signal.
                                                                                                                     - **The model matters.** Replacing `all-MiniLM-L6-v2` with a different sentence-transformer will change coherence scores. Recalibrate the threshold if you change the embedding model.
                                                                                                                     
                                                                                                                     ---
                                                                                                                     
                                                                                                                     Built on the AgDR v0.2 + AKI standard from [accountability.ai](https://accountability.ai)
