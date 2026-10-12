# Rule syntax — normative

**Problem this spec fixes:** rules are authored by humans, hashed for identity, matched by the
proof engine, and displayed in proofs. One concrete syntax must serve all four without
translation drift.

Semantics are fixed by [ADR-0005](../ADR/ADR-0005-datalog-style-rules.md) (Datalog: no function
symbols, no negation, range-restricted); this spec only pins the surface form and normalization.

## Grammar (EBNF)

```ebnf
rule        = "rule" ws name ws? ":" ws body ws "then" ws atom ws "why" ws ":" ws string ;
body        = "if" ws atom { ws "and" ws atom } ;
atom        = term ws relname [ "*" ] ws term ;
term        = variable | entityref | literal ;
variable    = "?" ident ;                        (* ?x, ?cls *)
entityref   = "E" digits                         (* E42 — canonical id *)
            | "[" label "]" ;                    (* [Mammal] — resolved to an id at parse time *)
relname     = upperident ;                       (* INSTANCE_OF, ORBITS *)
literal     = number | quantity | date | string | boolean ;   (* canonical forms per
                                                  grain-serialization.md *)
name        = ident ;                            (* author-chosen; display only, see below *)
```

Example, exactly as authored:

```
rule inherit-properties:
  if  ?x INSTANCE_OF ?c
  and ?c SUBCLASS_OF* ?d
  and ?d HAS_PROPERTY ?p
  then ?x HAS_PROPERTY ?p
  why: "Class properties hold for all instances of all subclasses."
```

## Static validity — rejected at parse, not at runtime

1. **Range restriction:** every variable in `then` MUST appear in some `if` atom.
2. **Known vocabulary:** every `relname` MUST be a declared canonical relationship.
3. **Type sanity:** literal positions MUST satisfy the relationship's declared range; entity
   positions its domain/range classes where declared.
4. **No negation, no disjunction, no function symbols** — the grammar simply cannot express
   them; that is the point of fixing the grammar
   ([unknown-handling.md](../Reasoning/unknown-handling.md#where-guessing-is-structurally-excluded)).
5. `[label]` entity references MUST resolve to exactly one entity at parse time; ambiguity is a
   parse error (name the id explicitly instead). The stored normalized form always carries ids,
   never labels — rules must not silently re-bind when labels change.

## The `*` closure sugar

`R*` is valid only where `R` is declared transitive. `?a R* ?b` means "?b is reachable from ?a
via zero or more R steps" — i.e. the reflexive-transitive closure. It is sugar for a reference
to the closure relation that the declaration of `transitive(R)` already induces
([rule-system.md](../Reasoning/rule-system.md#property-derived-rules)); the proof engine answers
it from the materialized closure when one is declared hot, and the *proof chain shows the
underlying R-step grains*, not the sugar (proofs bottom out in real grains, always).

Zero-step inclusion is deliberate: `?c SUBCLASS_OF* ?d` with `?c = ?d` lets one rule cover
"direct property" and "inherited property" — without it, every inheritance rule needs a twin.

## Normalization (the hashed form)

Rule identity is the SHA-256 of the **normalized text**
([grain-serialization.md](grain-serialization.md#identifier-formats)), produced by:

1. Resolve every `[label]` to its entity id.
2. Rename variables to `?v0, ?v1, …` in order of first appearance, reading atoms left-to-right
   as authored.
3. Collapse whitespace to single spaces; lowercase keywords; strip the `why` string and the
   author-chosen name (they are metadata — two authors writing the same logic MUST collide on
   the same rule id).
4. Serialize as netstrings: format tag `hourglass-rule-v1`, atom count, then each atom
   (term-kind, term, relname, closure-flag, term-kind, term), then the `then` atom.

**Known, accepted limitation:** atom *order* is preserved, so logically identical rules with
reordered `if` atoms get different ids. Full logical canonicalization is graph canonicalization
— complexity disproportionate to the harm (a duplicate rule is redundant, not wrong; confluence
means it cannot change any answer, only waste a little work). Duplicate-rule *detection* is
future work in [rule-system.md](../Reasoning/rule-system.md#future-work); this is the KISS
trade, made explicitly.

## Display in proofs

Proof chains reference rules by id; the renderer shows the author-chosen name and the `why`
string, and prints atoms with the *current preferred labels* of the ids (labels are display,
ids are truth — consistent with [entities.md](../Knowledge_Model/entities.md#entity-model)).

## Conformance

`conformance/rules-v1/` (committed when Phase 2 starts) holds: valid rules with their expected
normalized bytes and ids; invalid rules each violating exactly one static-validity rule, with
the expected error class. Parser and normalizer MUST pass both sets bit-exactly.
