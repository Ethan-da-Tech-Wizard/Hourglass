# Entities and entity resolution

**Problem:** the same real-world object appears under many names ("Earth", "the Earth", "our
planet", "Terra"), and different objects share names ("Mercury" the planet, the element, the
god). If entities do not resolve to canonical identifiers, deduplication fails, the graph
fragments, and proofs silently reason about the wrong thing.

**Why it matters:** entity resolution errors are *knowledge corruption*. A wrong merge
(planet-Mercury = element-Mercury) manufactures false facts; a missed merge hides true ones.
This is the highest-stakes canonicalization step.

## Entity model

```
entity id      : opaque canonical identifier (e.g. E00042), never reused, never renamed
kind           : class | individual
labels         : preferred label per language
aliases        : list of (alias string, source id)   — every alias is itself provenance-tracked
description    : one-sentence disambiguator ("Mercury — first planet from the Sun")
status         : active | merged-into(E…) | split
```

Design points:

- **Opaque ids, human labels.** Ids carry no meaning (no `PLANET_MERCURY`), because meaningful
  ids break when knowledge changes (Pluto). Labels and descriptions are data, not identity.
  Wikidata's Q-numbers demonstrate this works at 100M+ entity scale.
- **Merges are recorded, not destructive.** When two entities are found to be the same, one
  becomes `merged-into` the other; grains are re-pointed, but the merge event and both prior
  ids are kept so any historical proof can be replayed.
- **Classes are entities too.** `Mammal`, `Planet` are entities of kind `class`, connected by
  `SUBCLASS_OF`, so the rule system needs no separate type machinery.

## Resolution strategy (deterministic first)

Resolution runs at canonicalization ([pipeline stage 3](../Architecture/overview.md#3-canonicalize)).
Order of methods, strictest first:

1. **Exact identifier match** — the source provides a stable external id (ISBN, DOI, Wikidata
   Q-id, database primary key). Highest confidence; structured sources are prioritized for
   exactly this reason ([parsing.md](../Research/parsing.md)).
2. **Exact alias match, unambiguous** — the surface string matches exactly one entity's
   label/alias set within the expected class ("the object of ORBITS must be an astronomical
   body" narrows "Mercury" to the planet). Class expectations come from relationship
   domain/range declarations ([canonical-relationships.md](canonical-relationships.md)).
3. **Exact alias match, ambiguous** — multiple candidates survive. The grain is **held in
   candidate status** with the ambiguity recorded. It is not admitted on a guess. Ambiguities
   are resolved by more context (another grain from the same source disambiguates) or by human
   review.
4. **No match** — a new entity is created, flagged `newly-minted`, and queued for periodic
   duplicate review.

There is deliberately no fuzzy/statistical matching in the admission path. String-similarity and
embedding matching may *propose* merges for human or corroboration-based review, but a proposal
is not a merge ([ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md) applies to identity claims
too: an entity merge is itself a fact and needs evidence).

## Tradeoffs

- **Cost:** deterministic-first resolution leaves more grains parked in candidate status than an
  aggressive matcher would. Accepted: a smaller, correct graph beats a larger, corrupted one.
  The quarantine queue is a visible, workable backlog; silent mis-merges are invisible damage.
- **Class-constrained disambiguation depends on relationship declarations** being right. That
  coupling is acceptable because both live in the same closed, reviewed vocabulary.

## Rejected alternatives

- **Names as identifiers** — collapses homonyms, breaks on renames.
- **Probabilistic record-linkage in the admission path** (Fellegi–Sunter style scoring) —
  excellent recall in the entity-resolution literature, but admission-by-threshold is a
  confidence score deciding truth, which the core philosophy forbids. Used only as a *proposer*.
- **One global namespace shared with relationships** — entities and relationships have different
  lifecycles (open vs. closed vocabulary); separate spaces keep both simple.

## Future work

- Split handling (one entity later found to be two) — rarer and harder than merging; needs a
  procedure for re-attributing existing grains, likely with human review per grain.
- Cross-language labels and transliteration policy.
- Periodic duplicate-detection job over `newly-minted` entities (candidate-pair generation by
  blocking on shared labels/classes — standard entity-resolution technique, used here only to
  generate review proposals).
