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
Step 1  — Sensory Spine Read (last 100 Merkle records)
Step 2  — Model Inference (with spine context)
Step 3  — Embedding Deviation Critic  ◄ THIS LAYER
Step 4  — PPP Triplet Construction
Step 5  — zk-Proof Generation
Step 6  — Atomic AKI Capture (3.94 µs)
Step 7  — Record Sealing (BLAKE3 + Ed25519 signing)
Step 8  — BFT Consensus Validation
Step 9  — Atomic SMT Insert
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

The critic uses **all-MiniLM-L6-v2** from the `sentence-transformers` library. This model produces 384-dimensional dense vector representations of text, optimized for semantic similarity tasks.

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

### Mathematical Formulation

Let $\mathbf{e}_{\text{new}} \in \mathbb{R}^{384}$ be the embedding of the proposed reasoning trace, and let $\mathbf{e}_i \in \mathbb{R}^{384}$ be the embedding of the $i$-th record in the history window $\mathcal{H} = \{h_1, h_2, \ldots, h_N\}$ where $N \leq 60$.

**Step 1 — Cosine Similarity per Record:**

$$s_i = \cos(\mathbf{e}_{\text{new}},\, \mathbf{e}_i) = \frac{\mathbf{e}_{\text{new}} \cdot \mathbf{e}_i}{\|\mathbf{e}_{\text{new}}\| \cdot \|\mathbf{e}_i\|}$$

**Step 2 — Exponential Decay Weight:**

Each similarity score is weighted by recency. Let $d_i$ denote the number of positions from the most recent record (i.e., $d_1 = 0$ for the most recent):

$$w_i = \lambda^{d_i}, \quad \lambda = 0.96$$

This yields the following effective weights across history depth:

| Distance $d_i$ | Weight $w_i = 0.96^{d_i}$ |
|:-:|:-:|
| 0 (most recent) | 1.000 |
| 10 | 0.665 |
| 30 | 0.294 |
| 60 | 0.086 |

**Step 3 — Weighted Coherence Score:**

$$C = \frac{\displaystyle\sum_{i=1}^{N} w_i \cdot s_i}{\displaystyle\sum_{i=1}^{N} w_i}$$

where $C \in [0, 1]$, with $C = 1$ indicating perfect semantic alignment with recent history and $C = 0$ indicating maximal divergence.

**Step 4 — Status Classification:**

$$\text{status} = \begin{cases} \texttt{COHERENT} & \text{if } C \geq \tau \\ \texttt{UNINTENTIONAL\_DRIFT} & \text{if } C < \tau \end{cases}$$

where $\tau = 0.85$ is the global coherence threshold.

### `score()` Method

The `score()` method computes the weighted coherence score $C$ between a proposed reasoning trace and the most recent 60 entries from the Merkle spine history.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `proposed_trace` | `str` | The reasoning trace produced by the model at Step 2 |
| `history` | `List[Dict]` | Recent AgDR records from the Sparse Merkle Spine (typically last 100) |

**Returns:** `Tuple[float, str]` — The coherence score $C \in [0, 1]$ and a status string (`"COHERENT"` or `"UNINTENTIONAL_DRIFT"`).

**Computation Steps:**

1. **Empty History Check:** If `history` is empty (cold start), return `(1.0, "COHERENT")`. A system with no prior decisions cannot have drifted.
2. 2. **Proposed Embedding:** Encode the `proposed_trace` string into $\mathbf{e}_{\text{new}} \in \mathbb{R}^{384}$.
   3. 3. **History Iteration:** For each of the last 60 records in `history`, concatenate `reasoning_trace` and `output` fields and encode to obtain $\mathbf{e}_i$.
      4. 4. **Similarity Computation:** Compute $s_i = \cos(\mathbf{e}_{\text{new}}, \mathbf{e}_i)$ using `torch.nn.functional.cosine_similarity()`.
         5. 5. **Decay Weighting:** Apply $w_i = 0.96^{d_i}$ to each $s_i$.
            6. 6. **Weighted Average:** Compute $C = \sum w_i s_i \,/\, \sum w_i$.
               7. 7. **Threshold Classification:** Return `"UNINTENTIONAL_DRIFT"` if $C < 0.85$, otherwise `"COHERENT"`.
                 
                  8. ---
                 
                  9. ## Coherence Threshold
                 
                  10. The global coherence threshold is defined in the module configuration:
                 
                  11. ```python
                      COHERENCE_THRESHOLD = 0.85
                      ```

                      ### Threshold Tuning Guidance

                      | Threshold $\tau$ | Behaviour | Use Case |
                      |:-:|---|---|
                      | $\tau \geq 0.90$ | Very strict; flags most novel reasoning | High-stakes legal/medical domains |
                      | $\tau = 0.85$ | Default; balanced strictness | General accountability workloads |
                      | $\tau = 0.75$ | Permissive; allows moderate domain shifts | Exploratory research agents |
                      | $\tau < 0.70$ | Not recommended | May allow semantic drift to accumulate |

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
                      2. 2. **Escalation Signal:** The `FOI ESCALATION` tag alerts the Fiduciary Office Intervener that human review is warranted.
                         3. 3. **Preservation:** The original reasoning is preserved in the `reasoning_trace` field. The correction only modifies the `output` field, maintaining full provenance.
                           
                            4. The corrected output then continues through the remaining pipeline steps and is permanently sealed in the spine with its drift-corrected status visible.
                           
                            5. ---
                           
                            6. ## Exponential Decay Rationale
                           
                            7. The decay factor $\lambda = 0.96$ per position creates a recency-biased comparison window. The practical effect of this weighting is that the **effective window** of comparison is approximately 25–30 records (where $w_i > 0.3$), with a long tail of diminishing influence out to 60.
                           
                            8. Without decay, an agent that gradually shifts its reasoning domain over hundreds of decisions would appear coherent at each step but could be dramatically different from its behaviour 500 decisions ago. The decay weighting partially mitigates this by ensuring very old records contribute negligibly to $C$ while maintaining enough historical context to detect multi-step drift patterns.
                           
                            9. The choice $\lambda = 0.96$ satisfies:
                           
                            10. $$\sum_{d=0}^{59} 0.96^d = \frac{1 - 0.96^{60}}{1 - 0.96} \approx 22.7$$
                           
                            11. meaning the effective denominator of the weighted average is approximately 22.7, concentrating 95% of the scoring influence in the 60 most recent records while still acknowledging the earliest entries in the window.
                           
                            12. ---
                           
                            13. ## Integration with the Sensory Spine
                           
                            14. The Deviation Critic receives its `history` parameter from the Sensory Spine read at Step 1:
                           
                            15. ```python
                                history = self.chain.get_last_n(100)
                                ```

                                The critic uses only `history[-60:]` for scoring. The remaining 40 records are available for sensory context formatting but are not factored into $C$. The 100-record read is subject to the 3.94 µs latency bound enforced by the Sparse Merkle Spine.

                                ---

                                ## Performance Characteristics

                                | Metric | Value | Notes |
                                |--------|-------|-------|
                                | Embedding latency (per string) | ~2–5 ms | Depends on text length and hardware |
                                | Cosine similarity computation | < 0.1 ms | Single tensor operation |
                                | Total critic overhead (60 comparisons) | ~120–300 ms | Dominated by embedding computation |
                                | Memory footprint (model loaded) | ~80 MB | all-MiniLM-L6-v2 with PyTorch |

                                For latency-sensitive deployments, consider pre-computing and caching embeddings for spine records (they are immutable once committed), reducing the critic to a single embedding computation (the proposed trace) plus 60 cached cosine similarity lookups.

                                ---

                                ## For Implementors

                                - **Do not disable the critic.** It is the primary defence against semantic drift in the decision record. Disabling it is equivalent to removing the brakes from a vehicle.
                                - - **Cache spine embeddings.** Since Merkle spine records are immutable, their embeddings never change. Pre-compute and cache them to reduce critic latency from ~200 ms to ~5 ms.
                                  - - **Monitor coherence distributions.** Track the distribution of $C$ over time. A downward trend indicates systematic drift that may require model retraining or threshold adjustment.
                                    - - **Respect the classification.** A status of `"UNINTENTIONAL_DRIFT"` means the system has detected a semantic shift that was not explicitly authorised. Do not ignore this signal.
                                      - - **The model matters.** Replacing `all-MiniLM-L6-v2` with a different sentence-transformer will change coherence scores. Recalibrate $\tau$ if you change the embedding model.
                                       
                                        - ---

                                        *Built on the AgDR v0.2 + AKI standard from [accountability.ai](https://accountability.ai)*
