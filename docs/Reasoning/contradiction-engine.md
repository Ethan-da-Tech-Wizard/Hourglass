# Contradiction engine

**Problem:** independent sources disagree — because one is wrong, outdated, less precise, or
talking about something subtly different. A system that silently overwrites, silently ignores,
or averages disagreements is fabricating a consensus that does not exist.

**Why it matters:** contradiction handling is where "truth through proof" meets messy reality.
The success criterion is explicit: *can contradictions be explained?* — not "resolved
automatically."

## What counts as a contradiction

Contradiction is not free-floating; it is defined by declared constraints. Two `verified` grains
conflict when:

1. **Functional-property violation.** A relationship declared functional (at most one object per
   subject + qualifiers) has two: `Earth ORBITS Sun` vs `Earth ORBITS Proxima` — same subject,
   same qualifiers, different objects.
2. **Declared disjointness.** `?c SUBCLASS_OF A` and `?c SUBCLASS_OF B` where
   `A DISJOINT_WITH B` is declared (e.g. `Planet DISJOINT_WITH Star`).
3. **Explicit negation** (future, once negative grains exist): `X R Y` vs `X NOT R Y`.
4. **Derived conflict.** Any of the above where one or both sides are rule-derived — the
   conflict then implicates the *premise* grains and rules, found via the proof chains.

No constraint declared ⇒ no contradiction detected. This is honest: the system flags exactly the
conflicts it has been told how to recognize, and the constraint vocabulary grows by RFC like
everything else. (Same-triple-different-qualifiers is *not* a conflict —
[grains.md](../Knowledge_Model/grains.md#why-triples--qualifiers-not-paragraphs-and-not-n-ary-tuples).)

## Detection: at the write path, incrementally

Checks run at [pipeline stage 5 (Link)](../Architecture/overview.md#5-link), scoped to the
neighborhood of each newly admitted grain: for a new grain `(s, R, o, q)`, check functional
constraints over `(s, R, q)`, disjointness over the affected classes, and re-run derived-conflict
checks only for conclusions reachable from the new grain (the same reverse index used for proof
cache invalidation — [performance](../Performance/performance.md#proof-caching)). Full-graph
sweeps are a batch audit job, not the primary mechanism.

## Response: mark, preserve, explain — never overwrite

When a conflict is detected:

1. Both (all) participating grains move `verified → disputed`, with a **conflict record**:
   the constraint violated, the grains involved, their provenance, and — for derived conflicts —
   the proof chains implicating premises.
2. Nothing is deleted. Both sides keep full provenance.
3. Proof caches touching the disputed grains are invalidated.
4. Queries needing a disputed grain answer **"Contradictory evidence exists"** and present both
   sides with sources ([unknown-handling.md](unknown-handling.md#the-refusal-taxonomy)).
5. Resolution is a *curated event*: a human (or a documented deterministic policy — below)
   retracts one side or refines the model, and the conflict record keeps the resolution
   rationale. Resolution history is append-only.

This "mark and suspend" stance is deliberately conservative next to the prior art:

- **JTMS/ATMS (Doyle, de Kleer):** we adopt the *justification bookkeeping* (which conclusions
  depend on which premises — that is our proof/dependency index) but not automatic
  belief-state flipping: a TMS picks a consistent belief set on its own; Hourglass surfaces the
  inconsistency instead.
- **AGM belief revision:** gives postulates for *minimal* change when accepting new conflicting
  information — principled, but it requires an entrenchment ordering ("which beliefs do we give
  up first"), and any automatic entrenchment is a trust/confidence ranking in disguise. Used as
  vocabulary for thinking about resolution, not as an engine.
  ([Prior art](../Research/prior-art.md#truth-maintenance-systems).)

## Outdated information

Most apparent contradictions are time, not falsehood ("population 8.0M" vs "8.4M"). Handling, in
order:

1. **Model time explicitly.** Relationships whose values change get a required `as_of`/
   `valid_during` qualifier; differently-qualified grains coexist without conflict. This
   dissolves the majority class of conflicts *at the schema level*.
2. **Source dating as evidence for qualifiers.** A source snapshot's publication date can
   justify an `as_of` qualifier during extraction — that's evidence-based, not a guess.
3. **Supersession policy** (deterministic, declared per relationship): for a functional,
   time-qualified relationship, the grain with the latest `as_of` is the *current* value;
   older grains remain verified as historical facts. "Latest wins" is only ever a *selection*
   among verified time-scoped facts, never a retraction of the older ones.

## What is deliberately not built

- **Automatic majority voting** (3 sources vs 1 ⇒ majority wins) — count-as-truth is a
  confidence score; counts are *displayed* in the conflict record for the curator.
- **Source-authority overrides** (encyclopedia beats blog) — same rejection; authority policies
  may gate *admission* ([provenance.md](../Knowledge_Model/provenance.md)) but never silently
  settle conflicts between already-admitted grains.
- **Paraconsistent logic** (reason on despite contradiction) — mathematically respectable,
  operationally opaque; quarantining the conflict is simpler and more explainable.

## Future work

- Conflict-record schema and curator workflow (queue, resolution actions, audit trail —
  overlaps [Security audit logging](../Security/integrity.md)).
- `DISJOINT_WITH` and `functional` coverage review for the seed vocabulary.
- Detection of *systematic* conflict (one source disagreeing with many others across grains) as
  a poisoning signal — [Security](../Security/integrity.md#malicious-knowledge-injection).
