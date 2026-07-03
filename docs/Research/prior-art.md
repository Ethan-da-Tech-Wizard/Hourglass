# Prior art survey

**Problem:** almost every subsystem Hourglass needs has decades of research behind it under a
different name. Ignoring that history means re-inventing worse versions; copying any one system
wholesale means importing philosophy that conflicts with ours.

**Why it matters:** this survey is the project's defense against both failure modes. Per field:
what it is, what we borrow, what we reject and why. Each section links to the design doc where
the borrowed idea lands.

---

## Symbolic AI and expert systems

MYCIN, DENDRAL, XCON and the 1970s–80s expert-systems wave: knowledge as explicit rules,
inference by chaining, explanations by rule trace.

- **Borrow:** the core loop (facts + rules + chaining ⇒ traceable conclusions); the insight that
  *explanation is the rule trace* — our proof chain is exactly this, formalized
  ([proof-engine.md](../Reasoning/proof-engine.md)).
- **Reject:** certainty factors (MYCIN's 0–1 confidence arithmetic) — the ancestor of exactly
  the confidence-score reasoning we forbid. Also the monolithic "knowledge engineer types in
  rules" acquisition model as the *only* path — acquisition must be pipeline-driven with
  provenance ([provenance.md](../Knowledge_Model/provenance.md)).
- **Lesson from their failure:** knowledge-acquisition cost killed expert systems more than
  inference did. Hourglass's bet is that structured-source ingestion + strict canonicalization
  amortizes that cost; the bet is stated honestly in [parsing.md](parsing.md).

## Cyc

The 40-year project to hand-encode common sense in higher-order logic (CycL), with microtheories
for context.

- **Borrow:** proof that canonicalization at scale is *possible*; microtheories prefigure our
  qualifier scoping (facts true within a context/time).
- **Reject:** higher-order expressiveness (undecidable inference — we take Datalog's guarantees
  instead, [ADR-0005](../ADR/ADR-0005-datalog-style-rules.md)); closed manual curation as the
  primary acquisition strategy.
- **Lesson:** expressiveness is a cost center. Cyc's power made every downstream tool harder.
  KISS is a survival strategy, not an aesthetic.

## Logic programming: Prolog, Datalog, XSB

- **Borrow:** nearly everything in the proof engine. SLD resolution (backward chaining), SLG
  tabling for termination + memoization (XSB), semi-naive forward evaluation, magic sets. The
  Datalog fragment *is* our rule language.
- **Reject:** Prolog's extra-logical features (cut, assert/retract mid-query,
  negation-as-failure) — each one makes outcomes depend on evaluation order or on absence of
  evidence. Details: [rule-system.md](../Reasoning/rule-system.md).

## Production rule engines: OPS5, CLIPS, Drools, Rete

- **Borrow:** the **Rete algorithm** — incremental pattern matching that avoids re-evaluating
  all rules on every new fact — is directly relevant to running admission/contradiction checks
  incrementally at the Link stage ([contradiction-engine.md](../Reasoning/contradiction-engine.md#detection-at-the-write-path-incrementally)).
- **Reject:** actions/side effects in rule consequents; conflict-resolution strategies (recency,
  specificity ordering) — order-dependent firing is nondeterminism of exactly the kind Datalog's
  confluence removes.

## Truth maintenance systems

Doyle's JTMS (justifications, in/out labeling), de Kleer's ATMS (tracking all assumption sets).

- **Borrow:** justification networks as the dependency index — grain → conclusions built on it —
  powering both proof-cache invalidation and contradiction blame assignment
  ([proof-engine.md](../Reasoning/proof-engine.md#proof-caching-and-invalidation)).
- **Reject:** autonomous belief revision (the TMS choosing which belief to abandon). Hourglass
  surfaces conflicts, humans/policies resolve them
  ([contradiction-engine.md](../Reasoning/contradiction-engine.md)). AGM belief-revision theory:
  borrowed as *vocabulary*, rejected as engine (needs an entrenchment ordering = trust ranking).

## Automated theorem proving and proof assistants

Resolution provers (Vampire, E), SAT/SMT (Z3), proof assistants (Coq, Lean, Isabelle; LCF
lineage).

- **Borrow:** the **de Bruijn criterion** — the checker, not the prover, is the trusted
  component; keep it tiny. This is the single most important idea imported into Hourglass's
  trust model ([proof-engine.md](../Reasoning/proof-engine.md#verification)). Also: proof
  objects as first-class serializable artifacts.
- **Reject:** full FOL/HOL as the reasoning substrate — semi-decidability turns "no answer yet"
  into a resource-dependent nondeterministic outcome.

## Semantic web: RDF, RDFS, OWL

- **Borrow:** triples as the atomic shape; global identifiers; domain/range declarations;
  property characteristics (transitive, symmetric, inverse, functional) — our relationship
  declarations are a curated subset of this
  ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md)).
- **Reject:** OWL DL reasoning (tableau complexity, and its open-world *inference* semantics
  differ from our "open world for answers, exhaustive search within the KB" stance —
  [unknown-handling.md](../Reasoning/unknown-handling.md#open-world-for-facts-closed-world-for-search));
  RDF reification for statement metadata (four extra triples per statement; our
  qualifiers-on-the-grain do this natively); the full SPARQL/ontology tooling stack (weight
  without benefit at our stage).

## Wikidata

The closest living relative: statements = property–value + qualifiers + **references**, opaque
Q/P identifiers, aliases, ranks.

- **Borrow:** the statement model almost exactly (grain = statement + mandatory provenance);
  opaque ids ([entities.md](../Knowledge_Model/entities.md)); the observation that a closed,
  community-governed property vocabulary works at 100M-entity scale — empirical support for
  [ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md).
- **Reject:** ranks (preferred/normal/deprecated) as soft resolution of conflicts — a mild
  authority score; our contradiction engine is stricter. Wikidata also does no inference; we
  add the rule/proof layer it lacks.

## Databases and compilers

- **Borrow:** normalization theory (store each fact once; our grain dedup is normalization
  applied to knowledge); WAL/immutability discipline for the audit trail; from compilers — IR
  design (grains are the IR between surface language and reasoning; canonicalization = lowering),
  and recording the toolchain version that produced each artifact
  ([provenance.md](../Knowledge_Model/provenance.md#per-grain-provenance-entry)).
- **Reject:** closed-world query semantics for answers (fine inside the engine, wrong as an
  epistemic claim — [unknown-handling.md](../Reasoning/unknown-handling.md)).

## Provenance systems: W3C PROV

- **Borrow:** the Entity/Activity/Agent decomposition and derivation vocabulary
  ([provenance.md](../Knowledge_Model/provenance.md#prior-art-borrowed)).
- **Reject:** RDF serialization as our storage format (we store provenance relationally).

## Entity resolution and ontology engineering

Fellegi–Sunter record linkage, blocking techniques; ontology-design lore (Brachman's IS-A
analysis, OntoClean).

- **Borrow:** blocking for candidate-pair generation in duplicate *review* queues; the
  INSTANCE_OF/SUBCLASS_OF discipline
  ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md#why-there-is-no-is_a)).
- **Reject:** probabilistic match scores deciding merges
  ([entities.md](../Knowledge_Model/entities.md#rejected-alternatives)).

## Information theory

- **Borrow:** the framing that *structure shared is redundancy removed* — rule inheritance is
  dictionary compression where the dictionary is the ontology. Also the discipline of measuring:
  compression claims need a defined metric
  ([grains.md future work](../Knowledge_Model/grains.md#future-work)).
- **Reject:** lossy compression anywhere near evidence.

---

## Summary table

| Field | Single most important import | Lands in |
|---|---|---|
| Expert systems | explanation = rule trace | proof-engine.md |
| Cyc | expressiveness is a cost center | ADR-0005 |
| Logic programming | Datalog + tabling | rule-system.md, proof-engine.md |
| Rete engines | incremental matching | contradiction-engine.md |
| TMS | justification networks | proof-engine.md (caching) |
| Proof assistants | de Bruijn criterion | proof-engine.md (verifier) |
| Semantic web | property declarations | canonical-relationships.md |
| Wikidata | statement+qualifier+reference model | grains.md |
| Databases | normalization, immutable logs | grains.md, Security |
| W3C PROV | provenance decomposition | provenance.md |
| Entity resolution | blocking (for review only) | entities.md |
