# PPP Triplet and Fiduciary Office Intervener (FOI)

## AgDR-FullSystem v2.1 — Technical Specification

---

## Overview

The PPP Triplet (Provenance, Place, Purpose) is the policy core of every AgDR record. It is the minimum legally sufficient metadata that anchors an autonomous AI decision to a specific identity, jurisdiction, and fiduciary intent. Combined with the FOI (Fiduciary Office Intervener) chain, the PPP triplet transforms every AI inference from an opaque computation into a court-admissible legal instrument with a named human steward at the terminus.

The PPP triplet is constructed at **Step 4** of the 10-step decision pipeline and is sealed into the AgDR record alongside the reasoning trace, output, coherence score, zk-proof, and Merkle root.

---

## The PPP Triplet

### Definition

Every AgDR record carries exactly three policy fields:

```python
ppp = {
    "provenance": f"agent_{agent_id}@accountability.ai",
        "place": "sovereign_jurisdiction_ca",
            "purpose": "fiduciary_duty_max_coherence_for_user_and_society"
            }
            ```

            ### Provenance

            The exact identity and origin of the agent that produced the decision. Format: `agent_{agent_id}@accountability.ai`. Provenance establishes authorship — when a decision is challenged, this field answers which agent produced the output and under whose authority. In multi-agent swarms, each agent carries a distinct provenance identifier, creating a complete attribution trail.

            ### Place

            The sovereign jurisdiction and systemic boundary within which the decision was made. Format: `sovereign_jurisdiction_ca` for Canadian jurisdiction. Place establishes which legal framework governs the decision (Canada Evidence Act, PIPEDA, etc.). Designed for dynamic per-decision assignment in multi-jurisdictional deployments.

            ### Purpose

            The pre-defined fiduciary intent governing the decision. Format: `fiduciary_duty_max_coherence_for_user_and_society`. Purpose transforms an AI output from computation into an accountable act by declaring the duty of care at capture time. Must be set before inference, not after. Immutable once sealed.

            ---

            ## Fiduciary Office Intervener (FOI)

            ### Definition

            The FOI is the terminal human anchor in the AgDR accountability chain. It ensures that there is always a named human who bears ultimate legal and fiduciary responsibility.

            ### Implementation

            ```python
            self.foi_chain = ["human_fiduciary_root"]  # Terminal human anchor
            ```

            The `foi_chain` is a list maintaining the chain of human stewards. The terminal element (`foi_chain[-1]`) is recorded in every AgDR record as the `foi_anchor` field. The chain supports nested delegation (append-only).

            ### FOI Escalation

            When the Deviation Critic detects drift, FOI escalation is triggered:

            ```python
            proposed_output = (
                f"[AgDR v2.1 CORRECTED via sensory spine] "
                    f"{proposed_output} | FOI ESCALATION"
                    )
                    ```

                    This is sealed into the immutable record, creating an auditable trail of the escalation event.

                    ---

                    ## AgDR Record Schema

                    ```python
                    @dataclass
                    class AgDRRecord:
                        timestamp_us: int              # Microsecond-precision capture time
                            ppp_triplet: Dict[str, str]    # {"provenance", "place", "purpose"}
                                reasoning_trace: str           # Full model reasoning
                                    output: Any                    # Decision output
                                        signature: str                 # BLAKE3 + Ed25519 signature
                                            merkle_root: str               # SMT root after insert
                                                zk_proof: str                  # Zero-knowledge proof
                                                    coherence_score: float         # Deviation Critic score (0.0-1.0)
                                                        foi_anchor: str                # Terminal human fiduciary
                                                        ```

                                                        ### SQLite Storage

                                                        ```sql
                                                        CREATE TABLE records (
                                                            key TEXT PRIMARY KEY,
                                                                timestamp_us INTEGER,
                                                                    ppp_json TEXT,              -- JSON-serialized PPP triplet
                                                                        reasoning_trace TEXT,
                                                                            output TEXT,
                                                                                signature TEXT,
                                                                                    merkle_root TEXT,
                                                                                        zk_proof TEXT,
                                                                                            coherence_score REAL,
                                                                                                foi_anchor TEXT
                                                                                                )
                                                                                                ```

                                                                                                ---

                                                                                                ## Legal Admissibility

                                                                                                The PPP + FOI combination satisfies requirements for electronic record admissibility under the Canada Evidence Act and equivalent frameworks. Provenance establishes authorship, Place establishes jurisdiction, Purpose establishes intent, FOI establishes human accountability, the Merkle chain establishes integrity, the BLAKE3 + Ed25519 signature establishes authentication, and the zk-proof provides verifiable computation guarantees.

                                                                                                ---

                                                                                                ## For Implementors

                                                                                                - **Set PPP before inference.** Retroactive PPP assignment defeats accountability.
                                                                                                - **Customize Place per jurisdiction.** Set dynamically for multi-jurisdictional systems.
                                                                                                - **FOI is non-optional.** Every deployment needs at least one named human steward.
                                                                                                - **Purpose must be specific.** Vague purpose weakens legal value.
                                                                                                - **Never modify sealed PPP.** Create correction records instead.

                                                                                                ---

                                                                                                Built on the AgDR v0.2 + AKI standard from [accountability.ai](https://accountability.ai)
