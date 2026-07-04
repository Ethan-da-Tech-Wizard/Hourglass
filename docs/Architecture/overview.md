# Architecture overview

**Problem:** we need a system that answers questions only when the answer is derivable from
explicit evidence, and that can show the derivation. No existing off-the-shelf architecture does
this end-to-end: LLMs generate without proof; databases store without reasoning; theorem provers
reason without knowledge acquisition or human-language output.

**Why it matters:** every downstream design choice (data model, storage, algorithms) follows from
the shape of this pipeline. Getting the boundaries right first keeps each subsystem simple.

Vocabulary used here is defined once in the [glossary](../Glossary/README.md).

## The seven-stage pipeline

```
        raw sources                      questions
             │                               │
             ▼                               ▼
      ┌────────────┐                  ┌────────────┐
      │  ACQUIRE   │                  │   PROVE    │
      │ (acquirer) │                  │(proof eng.)│
      └─────┬──────┘                  └─────┬──────┘
            │ candidate grains              │ proof chain
            ▼                               ▼
      ┌────────────┐                  ┌────────────┐
      │   VERIFY   │                  │   VERIFY   │  (same verifier,
      │ (verifier) │                  │ (verifier) │   second duty)
      └─────┬──────┘                  └─────┬──────┘
            │ admitted grains               │ checked proof
            ▼                               ▼
      ┌────────────┐                  ┌────────────┐
      │CANONICALIZE│                  │   ANSWER   │
      └─────┬──────┘                  │ (renderer) │
            │ canonical grains        └────────────┘
            ▼
      ┌────────────┐     ┌────────────┐
      │  COMPRESS  │────▶│    LINK    │
      └────────────┘     └─────┬──────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │ KNOWLEDGE GRAPH │◀── rules, constraints
                      │    (storage)    │
                      └─────────────────┘
```

The left column is the **write path** (knowledge in). The right column is the **read path**
(answers out). They meet only at the knowledge graph. This separation is deliberate: ingestion
can be slow, batched, and heavily checked; querying must be fast and side-effect-free.

## Stage responsibilities

Each stage has exactly one responsibility (KISS). A stage that grows a second responsibility gets
split.

### 1. Acquire

Input: a source (document, dataset, table, structured record).
Output: *candidate* grains — proposed facts with attached evidence excerpts and source metadata.
Candidates carry no authority; they are quarantined until verified.
Research: [parsing](../Research/parsing.md).

### 2. Verify (admission)

Input: candidate grains.
Output: admitted grains, or rejections with reasons.
Checks: schema validity (known relationship, typed literal, resolvable entities), source
integrity (hash matches), constraint compatibility (does not instantly violate a functional
constraint without going through the contradiction engine), and — where policy requires —
corroboration by independent sources.

### 3. Canonicalize

Input: admitted grains in surface form.
Output: canonical grains — entities resolved to canonical identifiers, verbs mapped to canonical
relationships, literals normalized (ISO dates, SI units).
This is where "the Earth revolves around the Sun" and "our planet circles the Sun" become the
same grain with two pieces of evidence.
Research: [entities](../Knowledge_Model/entities.md),
[canonical relationships](../Knowledge_Model/canonical-relationships.md).

### 4. Compress

Input: canonical grains.
Output: deduplicated structure. Identical canonical grains merge (evidence lists concatenate).
Facts derivable from an existing rule are *not* stored as base grains — they are recomputed on
demand (rule inheritance is the main compression mechanism).
Constraint: compression must be lossless with respect to proof — nothing needed to reconstruct a
proof chain or provenance may be discarded.
Research: [structural compression](../Knowledge_Model/grains.md#structural-compression).

### 5. Link

Input: the compressed grain set.
Output: an updated knowledge graph — indexes rebuilt or incrementally updated, affected proof
caches invalidated, contradiction checks run against the new neighborhood.
Research: [contradiction engine](../Reasoning/contradiction-engine.md),
[performance](../Performance/performance.md).

### 6. Prove

Input: a question, translated into a goal pattern over the graph.
Output: one or more proof chains, or a definite failure classified as
`Unknown / Insufficient evidence / Contradictory evidence exists / Cannot be resolved`.
Research: [proof engine](../Reasoning/proof-engine.md),
[unknown handling](../Reasoning/unknown-handling.md).

### 7. Answer

Input: a *verifier-checked* proof chain (or a failure classification).
Output: human-readable text plus the machine-readable proof chain.
The renderer is a template-driven translator: every sentence it emits maps to a grain or rule
application in the proof. It has no access to any knowledge outside the proof object it is
handed — this is enforced by interface, not by convention.

## Subsystem boundaries and dependencies

```
acquirer ──▶ verifier ──▶ canonicalizer ──▶ knowledge graph ◀── proof engine ◀── query front-end
                                              ▲        ▲              │
                              contradiction ──┘        └── rules      ▼
                              engine                            verifier ──▶ renderer
```

Dependency rules:

- The **renderer** depends only on the proof-chain data structure. It cannot read the graph.
- The **proof engine** reads the graph but never writes it (proof caches live in a separate
  cache store).
- The **verifier** is the only component trusted on both paths, so it must stay small enough to
  audit by hand (see [de Bruijn criterion](../Reasoning/proof-engine.md#verification)).
- The **contradiction engine** is the only component allowed to change a grain's status
  (verified → disputed), and no component may delete a grain
  ([ADR-0001](../ADR/ADR-0001-atomic-grains.md); grains are retracted by status change, keeping
  history).

## What Hourglass explicitly is not

- Not a language model. There is no statistical text prediction anywhere in the answer path.
- Not a search engine. It does not return "relevant documents"; it returns proven statements or
  a refusal.
- Not a belief ranker. There are no confidence scores in answers
  ([ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md)). Corroboration is counted and shown;
  it is never converted into a probability of truth.

## Related documents

- The module/repository shape implementing this: [codebase-layout.md](codebase-layout.md)
- Where uploaded files (PDF, Excel, Word, any format) physically live: [file-intake.md](file-intake.md)
- Showing users where each fact came from (highlights on originals): [evidence-viewer.md](evidence-viewer.md)
- Decisions: [ADR-0001 atomic grains](../ADR/ADR-0001-atomic-grains.md),
  [ADR-0002 closed relationship vocabulary](../ADR/ADR-0002-closed-relationship-vocabulary.md),
  [ADR-0003 unknown over guessing](../ADR/ADR-0003-unknown-over-guessing.md),
  [ADR-0004 storage engine](../ADR/ADR-0004-sqlite-first-storage.md),
  [ADR-0005 Datalog-style rules](../ADR/ADR-0005-datalog-style-rules.md)
- Prior art informing this shape: [Research/prior-art.md](../Research/prior-art.md)
- End-to-end walk-through: [Examples/worked-example.md](../Examples/worked-example.md)
