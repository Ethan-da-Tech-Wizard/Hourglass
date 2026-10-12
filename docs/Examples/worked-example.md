# Worked example: the whole pipeline on a small knowledge base

**Purpose:** make every abstraction in the design concrete on one tiny KB — ingestion,
canonicalization, compression, rules, proof, refusal, and contradiction, end to end. If a design
document and this example ever disagree, one of them has a bug.

## 1. Sources arrive

Three text snippets from three sources (already snapshotted and hashed —
[provenance.md](../Knowledge_Model/provenance.md)):

- **S1** (astronomy textbook, retrieved 2026-05-01): "The Earth revolves around the Sun."
- **S2** (encyclopedia, retrieved 2026-05-02): "Our planet circles the Sun. Earth is a planet."
- **S3** (children's science site, retrieved 2026-05-03): "The Earth orbits the Sun."

Plus one curated ontology source **S0** (manual entry by curator, 2026-04-30) asserting:
"A dog is a mammal. Mammals have lungs. Rex is a dog."

## 2. Acquire → candidate grains

Tier-3 pattern extractors ([parsing.md](../Research/parsing.md)) fire. From S1, pattern
`X revolves around Y` proposes:

```
candidate: (Earth?, "revolves around", Sun?)   evidence: "The Earth revolves around the Sun." [S1, offsets 0–37]
```

From S2, "Our planet circles the Sun" — the subject is the referring phrase "our planet."
The deterministic extractor does **not** resolve this pronoun-like reference in general prose;
however S2's second sentence "Earth is a planet" fires the copular pattern. Result: S2 yields
one candidate (`Earth?, "is a", planet?`) and the "our planet" sentence yields a candidate
*only because* this source's style sheet declares "our planet" a registered alias of Earth
(alias with provenance — [entities.md](../Knowledge_Model/entities.md)). Had it not, the
sentence would be skipped: recall sacrificed, no guess made.

## 3. Verify + Canonicalize

Entity resolution: "Earth", "our planet" → `E-earth`; "Sun" → `E-sun` (alias match, unambiguous
within ORBITS's astronomical domain/range). Relationship normalization
([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md)): "revolves
around", "circles", "orbits" → `ORBITS`; "is a" with (individual, class) operands → `INSTANCE_OF`;
with (class, class) → `SUBCLASS_OF`.

## 4. Compress + Link

Three sources produced the *same canonical grain* — content hashing merges them into one grain
with three provenance entries:

```
G1: E-earth ORBITS E-sun            status: verified
    provenance: [S1 "The Earth revolves around…", S2 "Our planet circles…", S3 "The Earth orbits…"]
```

The KB now holds (with S0 and S2's contributions):

```
G1: Earth ORBITS Sun                 [S1, S2, S3]
G2: Earth INSTANCE_OF Planet         [S2]
G3: Dog SUBCLASS_OF Mammal           [S0]
G4: Mammal HAS_PROPERTY lungs        [S0]
G5: Rex INSTANCE_OF Dog              [S0]
```

Five grains. Note what is *not* stored: "Rex has lungs", "Dog has lungs" — derivable, therefore
not stored ([grains.md](../Knowledge_Model/grains.md#structural-compression)). The rule set
contains the general inheritance rule ([rule-system.md](../Reasoning/rule-system.md#rule-form)):

```
R-INHERIT: IF ?x INSTANCE_OF ?c AND ?c SUBCLASS_OF* ?d AND ?d HAS_PROPERTY ?p
           THEN ?x HAS_PROPERTY ?p
```

(`SUBCLASS_OF*` includes the zero-step case, ?c = ?d.)

## 5. Query: "Does Rex have lungs?"

Goal: `Rex HAS_PROPERTY lungs`. Backward chaining
([proof-engine.md](../Reasoning/proof-engine.md)): no base grain matches; R-INHERIT's THEN
unifies with bindings `{x→Rex, p→lungs}`. Sub-goals: `Rex INSTANCE_OF ?c` → G5 (`c→Dog`);
`Dog SUBCLASS_OF* ?d` with `?d HAS_PROPERTY lungs` → G3 + G4 (`d→Mammal`). Proof chain:

```
Rex HAS_PROPERTY lungs
└── R-INHERIT {x→Rex, c→Dog, d→Mammal, p→lungs}
    ├── G5: Rex INSTANCE_OF Dog          [S0]
    ├── G3: Dog SUBCLASS_OF Mammal       [S0]
    └── G4: Mammal HAS_PROPERTY lungs    [S0]
```

The verifier re-checks the chain (grains exist and are verified; rule premises match bindings;
root matches goal). The renderer — given *only* this object — emits:

> **Yes — Rex has lungs.** Rex is a dog (source S0); dogs are mammals (S0); mammals have lungs
> (S0); by the inheritance rule R-INHERIT, Rex has lungs.

Every sentence maps to a proof node. Nothing else may appear.

## 6. Query: "Does Rex have gills?"

Goal: `Rex HAS_PROPERTY gills`. No base grain; R-INHERIT requires some superclass of Dog with
`HAS_PROPERTY gills` — the exhaustive, terminating search finds none. Outcome class: **Unknown**
([unknown-handling.md](../Reasoning/unknown-handling.md)):

> **Unknown. Insufficient evidence.** Hourglass has no evidence that Rex has gills, and no
> evidence that he does not. (Absence of a stored fact is not a "no" —
> [open world](../Reasoning/unknown-handling.md#open-world-for-facts-closed-world-for-search).)

An LLM would cheerfully answer "no." Hourglass cannot prove "no" without a negative grain or a
source-declared closed enumeration of mammalian respiratory organs — and says so.

## 7. A contradiction arrives

A new source **S4** (a scraped low-quality page) yields `Earth ORBITS Proxima-Centauri`.
`ORBITS` is declared functional over (subject, qualifiers)
([contradiction-engine.md](../Reasoning/contradiction-engine.md#what-counts-as-a-contradiction)).
At the Link stage the functional check on `(E-earth, ORBITS, ∅)` finds G1 ≠ G-new:

- Both grains → `disputed`; conflict record created (constraint, grains, provenance of each).
- Query "What does Earth orbit?" now answers: **Contradictory evidence exists** — presenting
  G1 (three independent sources) versus G-new (one source), *with counts displayed, not
  scored*. A curator retracts G-new (rationale logged, append-only); G1 returns to `verified`;
  the incident makes S4 a candidate for the systematic-conflict review described in
  [Security](../Security/integrity.md#malicious-knowledge-injection).

## 8. What this example pins down

| Claim in the design | Where it showed up |
|---|---|
| Equivalent language → one canonical grain, evidence preserved | Step 4, G1 with 3 provenance entries |
| Derivable facts are never stored | "Rex has lungs" absent from the KB |
| Every answer carries a recoverable proof | Step 5 |
| Renderer adds nothing beyond the proof | Step 5 rendering |
| Refusal is a correct, specific output | Step 6 |
| Conflicts suspend, never overwrite | Step 7 |
| Corroboration is counted, not scored | Steps 4 and 7 |

## Future work

- Extend the example with a time-qualified fact (population) exercising supersession
  ([contradiction-engine.md](../Reasoning/contradiction-engine.md#outdated-information)).
- A machine-readable version of this KB as the first conformance fixture once implementation
  starts ([roadmap](../Roadmap/roadmap.md)).
