# Hourglass

**A deterministic knowledge reasoning engine. Truth through proof, not statistical prediction.**

Hourglass is not a language model. It does not predict the next word. It is designed around one
central principle:

> **Nothing may be stated unless it can be proven from known evidence and explicitly documented
> reasoning.**

If sufficient evidence does not exist, Hourglass answers:

> Unknown. Insufficient evidence.

Guessing is forbidden. Probability is not proof. Confidence scores are not proof. Language
generation is secondary. Truth is primary.

## The pipeline

```
Acquire → Verify → Canonicalize → Compress → Link → Prove → Answer
```

Every fact the system holds is a **grain**: the smallest atomic verified unit of knowledge, with
full source provenance. Every answer carries a **proof chain**: the exact sequence of grains and
rules that derives the conclusion. Every proof is checked by an independent **verifier** before a
**renderer** turns it into human language. The renderer may never introduce information that is
not present in the verified proof.

## Project status

**Phase 0 — Design and research.** This repository is the engineering notebook for the project.
The architecture is being documented before any implementation begins. See the
[roadmap](docs/Roadmap/roadmap.md) for the phased plan and **[CHECKLIST.md](CHECKLIST.md)** for
the live, check-off-as-you-go progression list that every working session updates. Contributor
and agent working rules live in [CLAUDE.md](CLAUDE.md); all documentation is governed by the
[preservation policy](docs/README.md#documentation-preservation-policy) — nothing is ever
deleted or silently rewritten.

## Where to start reading

| Document | What it answers |
|---|---|
| [Architecture overview](docs/Architecture/overview.md) | What the subsystems are and how data flows between them |
| [Glossary](docs/Glossary/README.md) | The project vocabulary (grain, entity, rule, proof chain, …) |
| [Grains](docs/Knowledge_Model/grains.md) | What the atomic unit of knowledge looks like |
| [Canonical relationships](docs/Knowledge_Model/canonical-relationships.md) | How equivalent language collapses into one canonical fact |
| [Proof engine](docs/Reasoning/proof-engine.md) | How conclusions are derived and verified |
| [Unknown handling](docs/Reasoning/unknown-handling.md) | Why and how the system refuses to guess |
| [Prior art](docs/Research/prior-art.md) | What existing fields we borrow from, and what we reject |
| [Worked example](docs/Examples/worked-example.md) | The whole pipeline on a small concrete knowledge base |
| [ADR index](docs/ADR/README.md) | Every significant engineering decision, with reasoning |
| [Specifications](docs/Specifications/README.md) | The normative contracts code will be built against — byte formats, interfaces, frozen v1 scope, requirements traceability |

The full documentation map is in [docs/README.md](docs/README.md).

## Design values

- **Determinism.** The same knowledge base and the same question always produce the same answer
  and the same proof.
- **Traceability.** Every grain knows where it came from. Every answer can be traced to grains.
- **Honesty about ignorance.** Absence of evidence yields "Unknown," never a fabricated answer.
- **Structural compression.** Compress structure (shared entities, inherited rules, deduplicated
  relationships), never the information required for proof.
- **KISS.** Every subsystem has one responsibility. Every algorithm is explainable. No hidden
  behavior, no magic.

## Contributing to the design

Significant decisions are recorded as [ADRs](docs/ADR/README.md). Proposals that change the
knowledge model or the canonical vocabulary go through the [RFC process](docs/RFC/README.md).
Documentation standards — what every document must answer — are described in
[docs/README.md](docs/README.md#documentation-standards).
