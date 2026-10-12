# Specifications

Everything else in `docs/` is **explanatory** — it tells you why the design is what it is. The
documents in this directory are **normative** — they tell an implementer exactly what to build,
byte-for-byte and call-for-call. When an explanatory doc and a spec disagree, the spec wins and
the disagreement is a bug to fix.

Normative language: **MUST** / **MUST NOT** / **MAY** carry their usual (RFC 2119) force.

## The contract set

| Spec | Fixes in place | Consumed by |
|---|---|---|
| [v1-scope.md](v1-scope.md) | What v1 is and is not; change control | Everyone, before anything |
| [requirements-traceability.md](requirements-traceability.md) | Every requirement → spec → phase → test | Project reviews, phase gates |
| [grain-serialization.md](grain-serialization.md) | Canonical byte form and identity of grains, entities, sources, rules, proofs | Storage, verifier, security audit |
| [storage-interface.md](storage-interface.md) | The only operations any subsystem may perform on stored knowledge | Every subsystem |
| [rule-syntax.md](rule-syntax.md) | The concrete textual rule language and its normalization | Rule authoring, admission, proof engine |
| [answer-envelope.md](answer-envelope.md) | Query input shape and answer/refusal output shape | Proof engine, renderer, any client |

## Why these six and no more

Each spec pins one seam where two subsystems meet. If the seams are exact, the subsystems can be
built independently and will assemble; if a seam is vague, the vagueness surfaces as a mid-build
redesign. There is no seam in the v1 architecture not covered by one of these six — that claim
is checkable against the [architecture overview](../Architecture/overview.md#subsystem-boundaries-and-dependencies)
dependency diagram, and keeping it true is a review obligation for any architectural change.

## Change policy

Specs are frozen for v1 once Phase 1 begins ([v1-scope.md](v1-scope.md#change-control)).
A spec change after that point is a versioned event: the spec's version tag increments, the
change ships with its migration note, and anything hashed under the old version says so in its
bytes (see [grain-serialization.md](grain-serialization.md#format-versioning)).
