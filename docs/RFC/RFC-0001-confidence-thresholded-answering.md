# RFC-0001: Confidence-thresholded answering

- **Status:** Rejected
- **Author:** project owner (proposed in design discussion), recorded by the notebook
- **Date:** 2026-07-04
- **Related:** [ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md),
  [unknown-handling.md](../Reasoning/unknown-handling.md),
  [answer-envelope.md](../Specifications/answer-envelope.md)

## Problem

Coverage pressure is real: a proof-only engine says "Unknown" for enormous swaths of questions,
especially while the KB is young. The proposal under consideration: let the engine read
documents and answer when an internal confidence value clears a threshold (e.g. > 0.95), so
useful answers flow before full verification coverage exists.

## Proposal (as considered)

When no proof exists, compute a confidence score for the best candidate answer from extraction
signals and corpus statistics; if the score exceeds a fixed threshold, return the answer
(optionally labeled), else refuse as today.

## Why it was rejected

1. **Silent wrongness at a known rate.** A 0.95 threshold ships roughly one wrong answer in
   twenty with no way to identify which one. The project's first success criterion — every
   answer traceable — becomes unsatisfiable, because a score is not a trace.
2. **Proof chains become decoration.** If the answer was actually decided by a score, the
   attached "proof" is a rationalization. The verifier's guarantee
   ([proof-engine.md](../Reasoning/proof-engine.md#verification)) only means something if the
   proof is the *whole* basis of the answer.
3. **Determinism breaks structurally.** Confidence values derive from corpus statistics, which
   change as the KB grows; the same question crosses the threshold on different days with no
   underlying fact changing. The determinism contract — equal kb_version + equal query ⇒
   byte-identical response ([answer-envelope.md](../Specifications/answer-envelope.md#kb_version))
   — cannot be maintained, and it is the project's central promise.
4. **Product identity.** A confidence-thresholded document answerer already exists as a mature,
   crowded product category (LLM + retrieval). Hourglass's reason to exist is being the system
   that cannot be wrong silently. Adopting the threshold rebuilds the incumbent with extra
   steps.

This is a rejection of confidence *deciding truth*, in any subsystem — the same line drawn by
ADR-0003, now with the strongest form of the proposal considered and turned down explicitly.

## What is adopted instead (the accepted alternatives)

- **Deterministic admission policies** — rules, not scores: "grains from source kind X require
  N independent sources," "OCR-derived grains require human confirmation against the page
  image." Same inputs, same outcome, every run
  ([provenance.md](../Knowledge_Model/provenance.md#corroboration-not-confidence)).
- **Confidence as work-ranking only.** Statistical tools (OCR, fuzzy matchers, future
  extractors) MAY sort the quarantine queue so curators see the most promising candidates
  first. Ranking human attention is harmless; the score dies at the quarantine boundary and
  never touches grain status or answers
  ([parsing.md](../Research/parsing.md#where-probabilistic-methods-may-be-required--and-the-containment-rule)).
- **Coverage through acquisition throughput** — bulk tier-1 ingestion (spreadsheets, databases)
  where determinism is natural and no threshold is needed, plus refusal analytics steering what
  to ingest next ([unknown-handling.md](../Reasoning/unknown-handling.md#future-work)).

## Impact of the rejection

None on code (pre-implementation). The answer-envelope schema rule — no field named
confidence/score/probability/rank — stands, and this RFC is the recorded precedent to cite when
the proposal recurs under a new name (it will: "certainty factor," "reliability weighting,"
"soft answer tier" are the historical aliases — see
[MYCIN, prior art](../Research/prior-art.md#symbolic-ai-and-expert-systems)).

## Resolution

Rejected 2026-07-04 by owner decision after design discussion, in favor of deterministic
admission policies, contained work-ranking, and throughput-first coverage. Recorded per the
RFC process rule that rejected proposals are kept forever
([RFC/README.md](README.md#process-deliberately-lightweight)).
