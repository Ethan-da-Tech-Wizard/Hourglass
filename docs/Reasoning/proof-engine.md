# Proof engine

**Problem:** given a question (a goal pattern) and a knowledge graph of grains and rules, find a
derivation of the answer — or establish, definitively and reproducibly, that none exists. Then
make that derivation independently checkable.

**Why it matters:** this is the heart of Hourglass. "Truth through proof" is only as real as the
algorithm that builds proofs and the verifier that checks them.

## The proof chain object

A proof chain is a directed acyclic graph:

- **Leaves:** verified grains (each carrying provenance — so every proof bottoms out in sources).
- **Internal nodes:** rule applications — rule id + the variable bindings used + the premise
  nodes consumed.
- **Root:** the conclusion grain (possibly derived, never stored as base fact).

```
Rex HAS_PROPERTY lungs
└── R-INHERIT  {x→Rex, c→Dog, d→Mammal, p→lungs}
    ├── grain: Rex INSTANCE_OF Dog          [source S1]
    ├── R-TRANS(SUBCLASS_OF) {a→Dog, c→Mammal}
    │   └── grain: Dog SUBCLASS_OF Mammal   [source S2]
    └── grain: Mammal HAS_PROPERTY lungs    [source S3]
```

Proof chains are serializable, content-hashable (cacheable, tamper-evident), and the *only*
input the renderer receives.

## Query evaluation: backward chaining, with selective forward materialization

Two classic strategies, both needed, for different jobs:

| | Backward chaining (goal-driven) | Forward chaining (data-driven) |
|---|---|---|
| What it does | Start from the question, recurse into rules whose THEN matches, down to grains | Start from grains, apply rules to fixpoint, materialize all derivable facts |
| Advantages | Touches only relevant facts; no storage of derived facts; natural proof-tree construction | Answers become lookups; incremental maintenance is well understood (semi-naive) |
| Disadvantages | Repeated sub-goal work without memoization; recursion needs cycle handling | Materializes facts nobody asks about; derived storage fights the compression goal |

**Recommended direction:**

- **Backward chaining (SLD-resolution over Datalog) is the primary query mechanism**, with
  **tabling/memoization** (as in SLG resolution / XSB) — tabling both eliminates repeated
  sub-goal work and guarantees termination on recursive rules (left-recursive `SUBCLASS_OF*`
  loops terminate because a repeated sub-goal hits the table instead of recursing).
- **Forward materialization only for declared hot closures** — e.g. the `SUBCLASS_OF` transitive
  closure, which nearly every inheritance query needs, maintained incrementally
  (semi-naive on insert; DRed or backward/forward counting on delete —
  [performance](../Performance/performance.md#incremental-materialization)). Materialized facts
  are marked derived, carry their generating rule id, and are cache, not knowledge: deleting
  them loses nothing.
- The **magic sets** transformation (rewriting rules so forward evaluation only derives facts
  relevant to a goal) is the known escape hatch if pure tabling proves too slow; noted, not
  planned.

Because Datalog is confluent ([rule-system.md](rule-system.md#formal-basis-datalog-restricted)),
strategy choice affects speed only, never answers — the evaluation strategy is an implementation
detail *by construction*, which is exactly what determinism requires.

## Multiple proofs and minimal proofs

One conclusion may have many derivations. Policy:

- **Existence first:** the engine returns the first proof found under a *deterministic* search
  order (ordered rule ids, ordered grain ids — no hash-map iteration order leaking into
  answers).
- **Minimal proof on request:** "minimal" must be defined to be computable — we define it as
  fewest rule applications, ties broken by fewest distinct leaf grains, then by lexicographic
  grain ids (total order → unique winner → deterministic). Found by iterative-deepening over
  proof depth; acceptable because proofs in a Datalog setting are shallow relative to the graph.
- **All proofs on request, bounded:** enumeration capped by count/depth with the cap stated in
  the response — "here are 3 of ≥3 proofs" is honest; silently picking one is not. Multiple
  independent proofs are also how corroboration manifests structurally.

## Verification

The engine that *finds* proofs is complex (search, tabling, caches). Complexity invites bugs.
Following the **de Bruijn criterion** from proof-assistant design (LCF/Coq/Lean lineage: trust
only a small kernel), Hourglass separates:

- **Proof search** (proof engine): arbitrarily clever, untrusted.
- **Proof checking** (verifier): a small function that walks a proof chain and checks, per node:
  the grain exists and is `verified`; the rule exists, is `verified`, and its premises under the
  given bindings are exactly the child nodes; the root matches the question asked.

Checking is linear in proof size and needs no search. Every answer is checked before rendering
([pipeline stage 7](../Architecture/overview.md#7-answer)). The verifier is the component we
keep small enough to audit by hand — the trusted computing base of the whole system.

## Failure classification

When no proof exists, the engine must say *why* in one of the documented refusal forms —
distinguishing "no evidence," "contradictory evidence," and "malformed question" is itself
information. Full taxonomy: [unknown-handling.md](unknown-handling.md).

## Proof caching and invalidation

Checked proofs are cached by (goal, KB-version) with their content hash. A cached proof is
invalidated when any leaf grain or rule it references changes status — the reverse index
(grain id → proofs using it) is exactly a **justification network as in Doyle's truth
maintenance systems**; TMS is prior art for cache invalidation here, not just for belief
revision ([prior art](../Research/prior-art.md#truth-maintenance-systems)). Details and cost
analysis: [performance.md](../Performance/performance.md#proof-caching).

## Contradiction interaction

The proof engine never reasons *through* disputed grains. If a goal is provable only via
disputed grains, the answer is "Contradictory evidence exists," accompanied by the competing
proof chains — see [contradiction-engine.md](contradiction-engine.md).

## Rejected alternatives

- **Resolution theorem proving over full FOL** — semi-decidable; a timeout is a
  nondeterministic answer ("unknown today, provable tomorrow on a faster machine"), which
  violates determinism. Datalog restriction removes the problem at the language level.
- **Probabilistic inference (Bayesian nets, MLNs, PSL)** — outputs beliefs, not proofs.
- **Answer set programming** — powerful for defaults/exceptions, but multiple stable models =
  multiple incompatible world-views, and we rejected nonmonotonicity itself
  ([rule-system.md](rule-system.md#exceptions-and-the-penguin-problem)).
- **Trusting the search engine's proof without a separate checker** — rejected on de Bruijn
  grounds; the checker is cheap and the trust reduction is enormous.

## Future work

- Deterministic parallel search: parallel sub-goal evaluation is safe for speed, but "first
  proof found" must stay order-defined — likely: parallel search, deterministic *selection*
  among completed candidates ([performance](../Performance/performance.md)).
- Proof presentation levels (full chain vs. leaf-sources-only summary) — a renderer concern,
  but the chain format must support slicing.
- Query language surface for humans (the goal-pattern syntax) — deferred until the model
  stabilizes.
