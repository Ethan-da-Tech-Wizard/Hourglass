# Unknown handling

**Problem:** most question-answering systems are built to always produce an answer; their
failure mode is a confident fabrication. Hourglass inverts this: refusal is a *correct output*,
and the system must be engineered so that refusing is easier than fabricating.

**Why it matters:** "never guess" is the founding constraint. It is also the easiest constraint
to erode accidentally — one convenience feature at a time (fuzzy matching here, a default
assumption there). This document pins down what refusal means so erosion is detectable in
review. Decision recorded in [ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md).

## The refusal taxonomy

Every query terminates in exactly one outcome class:

| Outcome | Meaning | Trigger |
|---|---|---|
| **Proven.** | Answer + verified proof chain | Proof found and checked |
| **Unknown. Insufficient evidence.** | No proof exists in the current KB | Exhaustive (terminating) search found no derivation |
| **Contradictory evidence exists.** | Derivations exist but pass through disputed grains, or competing incompatible conclusions are derivable | [Contradiction engine](contradiction-engine.md) state on the relevant grains |
| **Evidence currently incomplete.** | A proof *attempt* bottomed out at grains in `candidate` status (acquired, not yet verified) | Distinguishes "we have nothing" from "we have something unverified" — actionable for curators |
| **Question cannot be resolved.** | The question doesn't map to the knowledge model: unknown entity, unknown relationship, ill-typed pattern | Query translation failure — *before* proof search |

Each refusal is informative and specific. "Unknown" with a reason class is an honest, useful
answer; a fluent guess is neither.

## Open world for facts, closed world for search

Two classic stances:

- **Closed-world assumption (CWA):** not derivable ⇒ false. Sound for databases that are
  complete by construction (your bank knows *all* your transactions). Hourglass's KB is never
  complete — CWA would manufacture negative facts out of ignorance. **Rejected for answers.**
- **Open-world assumption (OWA):** not derivable ⇒ unknown. Matches reality for an incomplete
  KB. **Adopted for answers.**

But note the asymmetry that makes this workable: *within* the KB, search is closed and
exhaustive — Datalog termination ([ADR-0005](../ADR/ADR-0005-datalog-style-rules.md)) means "no
proof exists in this KB" is a decidable, reproducible statement, not a timeout. Hourglass never
says "false because unstated"; it says "unproven here, and that is a fact about this KB, checked
to completion." (This is also why negation-as-failure is banned from rules —
[rule-system.md](rule-system.md#negation) — it would let CWA leak in through the rule language.)

Consequence: Hourglass cannot answer "Is there no X such that…?" from absence alone. True
negative answers require explicit negative grains (future work) or exhaustively-enumerated
domains declared closed *by a source* (e.g. "the 8 planets of the Solar System" as a stated
complete enumeration — closure as evidence, not assumption).

## Where guessing is structurally excluded

Refusal-by-design only works if no component can guess. The choke points, each with its own doc:

| Temptation | Exclusion mechanism |
|---|---|
| Fuzzy entity matching at admission | Deterministic-first resolution; ambiguity → quarantine, not best-guess ([entities.md](../Knowledge_Model/entities.md)) |
| Plausible relationship mapping | Closed vocabulary; unmapped verbs don't ingest ([ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md)) |
| Default/nonmonotonic rules | Not in the rule language ([rule-system.md](rule-system.md#exceptions-and-the-penguin-problem)) |
| Confidence-weighted answers | No numeric belief anywhere in the answer path; corroboration is displayed, never aggregated into a score ([provenance.md](../Knowledge_Model/provenance.md#corroboration-not-confidence)) |
| Renderer embellishment | Renderer's only input is the checked proof object ([overview](../Architecture/overview.md#7-answer)) |
| Prover timeout treated as "no" | No timeouts needed: the language guarantees termination |

## Tradeoffs

- **Coverage suffers.** Hourglass will say "Unknown" for enormous swaths of questions an LLM
  would answer correctly. Accepted totally: that is the product. The mitigation is acquisition
  throughput, never a loosened proof standard.
- **Refusals must not leak false certainty.** "Unknown. Insufficient evidence." is a claim about
  *this KB*, and the renderer must phrase it that way — "Hourglass has no evidence for X," not
  "X is unknown to science."

## Rejected alternatives

- **Confidence-thresholded answering** ("answer if P > 0.95") — a threshold is a guess about
  guesses; also destroys reproducibility across KB versions in an unprincipled way.
- **"Likely" answer tiers** — a second-class answer channel becomes the channel everything
  flows through; forbidden by the founding constraint.

## Future work

- Explicit negative grains and source-declared closed enumerations (prerequisite for proven
  "no" answers).
- Refusal analytics: counting which entities/relations drive the most "Unknown" responses is
  the acquisition roadmap, generated by the system itself.
