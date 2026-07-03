# ADR-0003: No confidence scores in the answer path; refusal is a correct output

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [unknown-handling.md](../Reasoning/unknown-handling.md),
  [provenance.md](../Knowledge_Model/provenance.md#corroboration-not-confidence),
  [MYCIN prior art](../Research/prior-art.md#symbolic-ai-and-expert-systems)

## Problem

Every layer of a knowledge system generates candidate places to "helpfully" quantify belief:
extractor confidence, entity-match scores, source reliability weights, majority counts among
conflicting claims, rule plausibility. Each is individually defensible; composed, they turn the
system into a belief ranker whose answers are un-provable — the exact failure of the systems
Hourglass exists to differ from. A bright-line rule was needed, and needed *before*
implementation, because confidence creeps in one convenience at a time.

## Options considered

1. **Full probabilistic stance** (Bayesian KB / Markov logic / MYCIN-style certainty factors) —
   graceful with messy evidence, one uniform mechanism for conflict and doubt; but answers
   become "0.87 true," which is not a proof, not reproducible across KB versions in any
   principled way, and MYCIN's certainty-factor arithmetic is the canonical historical warning
   (its numbers had no coherent probabilistic semantics).
2. **Hybrid: proofs, plus confidence-thresholded fallback** ("if no proof, answer if score >
   0.95") — preserves coverage; but the fallback channel inevitably becomes the main channel
   under coverage pressure, and a threshold is a guess about guesses.
3. **Binary provability with a refusal taxonomy** — an answer is proven (with chain) or it is a
   classified refusal (Unknown / Contradictory / Incomplete / Unresolvable). Numeric belief is
   banned from the answer path entirely; corroboration counts may be *displayed*, never
   aggregated into scores that decide anything.

## Decision

Option 3, applied system-wide — not only to rendered answers but to every admission-adjacent
judgment: entity merges need evidence, not similarity scores; conflicts suspend rather than
resolve by weighted vote; statistical tools may *propose* (quarantined candidates, merge
suggestions) but never decide.

## Reasoning

- This is the founding constraint of the project restated as an enforceable engineering rule.
  Its value is that it is checkable in review: any patch introducing a numeric belief that
  influences an answer is wrong by definition, no judgment call required.
- Determinism requires it: scores derived from corpus statistics change as the KB grows, making
  yesterday's "0.96, answered" today's "0.94, refused" with no fact having changed.
- The de Bruijn-style trust argument ([proof-engine.md](../Reasoning/proof-engine.md#verification))
  only works if what the verifier checks — the proof — is the *whole* basis of the answer.

## Tradeoffs

- Coverage is dramatically lower than probabilistic systems; "Unknown" will be the most common
  answer for a long time. Accepted: coverage is bought with acquisition, never with epistemic
  discount.
- Some genuinely graded real-world matters (measurement error, disputed science) must be
  represented as explicit facts *about* uncertainty (a source states a confidence interval —
  that interval is a literal in a grain, which is fine) rather than as system-level belief.
- Curators do more work (quarantine queues, conflict resolution) because the system refuses to
  auto-resolve.

## Consequences

- Refusal taxonomy is API surface: every query returns proven-or-classified-refusal
  ([unknown-handling.md](../Reasoning/unknown-handling.md#the-refusal-taxonomy)).
- No field named `confidence`, `score`, `weight`, or `trust` may appear in grain, rule, proof,
  or answer schemas; extractor-internal scores die at the quarantine boundary.
- Negative answers require explicit negative evidence (future negative grains / declared closed
  enumerations) — the system says "Unknown," not "no," from absence.
