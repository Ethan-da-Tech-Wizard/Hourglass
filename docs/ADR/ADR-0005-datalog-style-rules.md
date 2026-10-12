# ADR-0005: The rule language is Datalog — no function symbols, no negation, range-restricted

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [rule-system.md](../Reasoning/rule-system.md),
  [proof-engine.md](../Reasoning/proof-engine.md),
  [logic programming prior art](../Research/prior-art.md#logic-programming-prolog-datalog-xsb)

## Problem

Rules are Hourglass's inference and compression mechanism, and rule applications appear in
human-facing proof chains. The rule language determines whether queries always terminate,
whether answers depend on evaluation order, and whether a non-specialist can read a proof. The
expressiveness/guarantees frontier had to be chosen deliberately.

## Options considered

1. **Full first-order logic with a resolution prover** — maximal expressiveness; but FOL is
   semi-decidable: proof search may not terminate, so practical systems bolt on timeouts, and a
   timeout makes "unprovable" depend on hardware and load — nondeterminism at the epistemic
   core.
2. **Prolog** — mature, expressive; but cut, negation-as-failure, and evaluation-order
   sensitivity mean the *program*, not the knowledge, determines outcomes.
3. **Answer set programming** — elegant defaults/exceptions; but multiple stable models present
   multiple incompatible world-views, and nonmonotonicity itself was rejected
   ([rule-system.md](../Reasoning/rule-system.md#exceptions-and-the-penguin-problem)).
4. **Probabilistic rule systems (MLNs, ProbLog)** — excluded outright by
   [ADR-0003](ADR-0003-unknown-over-guessing.md).
5. **Datalog** (Horn clauses, no function symbols, range-restricted, negation-free) — strictly
   less expressive; in exchange: guaranteed termination (finite derivable universe), confluence
   (unique fixpoint — evaluation strategy provably cannot affect answers), and forty years of
   evaluation algorithms (tabling, semi-naive, magic sets, incremental maintenance) ready to
   use.

## Decision

Option 5. Rules are Datalog clauses over grain patterns; transitivity/inverse/symmetry arrive as
declaration-derived generic rules; negation is excluded from v1 entirely, with stratified
negation over *explicit negative grains* as the only admissible future extension (own RFC
required).

## Reasoning

- Termination is not a performance nicety here — it is what makes "Unknown: no proof exists in
  this KB" a *decidable, reproducible statement* rather than a timeout
  ([unknown-handling.md](../Reasoning/unknown-handling.md#open-world-for-facts-closed-world-for-search)).
  The refusal taxonomy is only honest because the language guarantees exhaustive search
  completes.
- Confluence delivers determinism by construction: caching, parallelism, and strategy changes
  are provably answer-preserving, which converts a whole class of would-be correctness reviews
  into performance reviews.
- Negation-as-failure would convert absence of evidence into a premise — the precise thing the
  project forbids; excluding it at the language level is stronger than policing it in review.
- Datalog rules read as IF/THEN over facts: proofs stay explainable to non-specialists, which
  is a stated success criterion.
- Cyc is the cautionary tale for the opposite choice: expressiveness as a permanent cost center
  ([prior art](../Research/prior-art.md#cyc)).

## Tradeoffs

- No arithmetic/aggregation in rules initially ("population density = population/area" is not
  expressible); such derived quantities need either safe built-in predicates (a controlled
  future extension with its own termination argument) or precomputation at acquisition.
- No existential invention (no "every person has a birthplace" creating unknown entities) —
  consistent with the philosophy, but a real modeling limitation to be aware of.
- Exceptions/defaults must be modeled in the ontology (honest taxonomy) rather than the logic —
  modeling burden accepted knowingly.

## Consequences

- The proof engine can commit to SLG-style tabled backward chaining plus selective semi-naive
  materialization, all standard for exactly this language
  ([proof-engine.md](../Reasoning/proof-engine.md)).
- Rule admission checks enforce range-restriction and vocabulary membership mechanically.
- Any future extension (stratified negation, built-ins, aggregation) must arrive with its own
  termination and confluence argument in an RFC — the guarantees are the contract, and
  extensions inherit the burden of preserving them.
