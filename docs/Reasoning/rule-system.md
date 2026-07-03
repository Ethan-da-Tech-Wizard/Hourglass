# Rule system

**Problem:** storing every derivable fact explicitly (`Dog has lungs`, `Cat has lungs`, `Horse
has lungs`, …) bloats storage and, worse, turns an update ("actually, lungfish complicate this")
into a mass edit. We need deterministic inference: store the general fact once, derive instances
on demand.

**Why it matters:** rules are simultaneously the compression mechanism, the reasoning mechanism,
and — because every rule application appears in proof chains — part of the explanation shown to
humans. One mechanism, three duties: it must stay simple.

## Rule form

A rule is a stored, versioned, provenance-tracked object:

```
rule id      : R-xxxx (content hash of the normalized rule body)
IF           : conjunction of grain patterns (variables allowed)
THEN         : one grain pattern (variables bound by the IF side)
why          : documented justification + source, like any grain
status       : candidate | verified | disputed | retracted   (same lifecycle as grains)
```

Example (the directive's running example):

```
R-0001:
IF    ?x INSTANCE_OF ?c
AND   ?c SUBCLASS_OF* Mammal          # * = transitive closure of SUBCLASS_OF
THEN  ?x HAS_PROPERTY lungs
```

Better, the *general inheritance rule* stated once, so no per-property rule is needed:

```
R-INHERIT:
IF    ?x INSTANCE_OF ?c
AND   ?c SUBCLASS_OF* ?d
AND   ?d HAS_PROPERTY ?p
THEN  ?x HAS_PROPERTY ?p
```

With `Mammal HAS_PROPERTY lungs`, `Dog SUBCLASS_OF Mammal`, `Rex INSTANCE_OF Dog` stored as three
grains, `Rex HAS_PROPERTY lungs` is derivable — and `Dog has lungs`, `Cat has lungs`, `Horse has
lungs` are never stored individually.

## Formal basis: Datalog, restricted

The rule language is **Datalog**: Horn clauses, no function symbols, range-restricted (every
variable in THEN appears in IF). Decision recorded in
[ADR-0005](../ADR/ADR-0005-datalog-style-rules.md). Why this and not more power:

| Property of Datalog | Consequence for Hourglass |
|---|---|
| No function symbols | The derivable universe is finite → every query **terminates**. Determinism includes "always halts." |
| Declarative fixpoint semantics | The set of derivable facts is unique regardless of rule evaluation order → same KB, same answers, always |
| Decades of evaluation research | Semi-naive evaluation, magic sets, incremental maintenance are known, documented algorithms — no invention required ([proof engine](proof-engine.md)) |
| Simple enough to explain | A rule application in a proof chain reads as an IF/THEN a non-specialist can follow |

Full first-order logic and higher-order systems were rejected: semi-decidable or worse, and
proof search becomes an open-ended theorem-proving problem
([prior art](../Research/prior-art.md#automated-theorem-proving)).

## Property-derived rules

Relationship declarations ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md))
generate generic rules mechanically:

- `transitive(R)` → `IF ?a R ?b AND ?b R ?c THEN ?a R ?c`
- `inverse(R, S)` → `IF ?a R ?b THEN ?b S ?a`
- `symmetric(R)` → `IF ?a R ?b THEN ?b R ?a`

These are real rules with real ids (provenance: "derived from declaration of R"), so they appear
in proof chains like any other — no hidden inference.

## Exceptions and the penguin problem

`Bird HAS_PROPERTY can_fly` + inheritance derives `Penguin can fly`. Classic fix is default
logic / nonmonotonic reasoning ("birds fly *unless* stated otherwise") — **rejected**:
nonmonotonic conclusions can be invalidated by *adding* knowledge, which breaks the contract
that a verified proof stays valid, and "unless stated otherwise" is negation-as-failure smuggled
into the semantics ([unknown handling](unknown-handling.md) explains why that's dangerous here).

Hourglass's position: **the taxonomy must be honest instead of the logic being clever.**

- Don't assert over-broad class properties. `can_fly` belongs on `FlyingBird` (or equivalently,
  only on the classes where it is actually universal), not on `Bird`.
- If a source genuinely asserts "birds can fly," that grain is admitted with its provenance, and
  the contradiction engine will surface the conflict with `Penguin NOT can_fly` when negative
  grains land — conflicts are *reported*, not resolved by default-reasoning magic
  ([contradiction-engine.md](contradiction-engine.md)).

Cost: modeling burden shifts to ontology quality. Accepted — that burden is visible and
reviewable; nonmonotonic semantics is neither.

## Negation

v1 rules are negation-free. `NOT` in rule bodies ("IF ?x has no known predator THEN …") means
negation-as-failure, which converts *absence of evidence* into a premise — precisely what the
project forbids. If negation is ever needed, the only candidate is **stratified negation over
explicit negative grains** (NOT-facts stated by sources, not inferred from silence), and it gets
its own RFC. Tracked in [grains.md future work](../Knowledge_Model/grains.md#future-work).

## Rule lifecycle and safety

- Rules go through admission like grains: schema check (range-restriction, known relationships,
  domain/range compatibility), justification review, then `verified`.
- Retracting a rule invalidates every cached proof that used it
  ([performance](../Performance/performance.md#proof-caching)).
- A rule whose conclusions participate in contradictions gets flagged `disputed` exactly like a
  grain — bad generalizations are first-class suspects, not just bad facts.

## Rejected alternatives

- **Production systems (OPS5/CLIPS-style) with actions in THEN** — THEN-sides that mutate state
  make evaluation order observable, destroying confluence. Hourglass THEN-sides only conclude
  facts. (Rete-style matching is still relevant as an *algorithm* — see
  [proof engine](proof-engine.md).)
- **Prolog as the rule language** — cut, negation-as-failure, and unbounded terms trade
  determinism-of-outcome for expressive power; wrong trade here.
- **Probabilistic rules (Markov logic, weighted rules)** — a weighted rule outputs degrees of
  belief; the entire answer path is built on not doing that.

## Future work

- Rule subsumption detection (new rule generalizes an existing one → flag for consolidation).
- Qualifier-aware rules (inheritance that respects `as_of` time scoping).
- Authoring ergonomics: a concrete textual syntax for rule files, with a parser that round-trips
  to the normalized hashed form.
