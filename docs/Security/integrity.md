# Security: integrity, tamper detection, and knowledge injection

**Problem:** a reasoning engine whose authority is "every answer is proven" concentrates risk in
its premises. If an attacker (or a bug, or bit rot) alters grains, forges provenance, or feeds
poisoned sources, the proofs remain formally valid while the conclusions become false — the
worst failure mode: *confidently wrong with a paper trail*.

**Why it matters:** integrity is not a bolt-on here; "can every answer be traced?" is vacuous if
the trace can be forged.

## Threat model (first cut)

| Threat | Vector | Primary defense |
|---|---|---|
| Data tampering at rest | Direct DB/file modification | Content-addressed ids + hash-chained audit log (below) |
| Provenance forgery | Grain claims a source that doesn't say that | Archived snapshots + offset-anchored excerpts: the claim is mechanically re-checkable |
| Malicious knowledge injection | Poisoned sources feeding the acquirer | Admission gates, corroboration policy, systematic-conflict detection |
| Compromised pipeline code | Malicious extractor/canonicalizer | Method+version in provenance (recall by toolchain); small auditable verifier |
| History rewriting | Deleting the record of what was believed when | Append-only status log; no destructive operations exist in the design |

Out of scope for this document: network security, access control/authentication, DoS — standard
concerns with standard answers, addressed at deployment time. This document covers what is
*specific* to a knowledge engine.

## Tamper detection: content addressing all the way down

Already required by the data model, so the marginal cost is near zero:

- **Grains** are identified by the hash of their canonical form
  ([grains.md](../Knowledge_Model/grains.md#what-a-grain-is)) — a mutated grain no longer
  matches its own id.
- **Sources** are identified by the hash of their snapshot
  ([provenance.md](../Knowledge_Model/provenance.md#source-model)) — re-hash the archive to
  detect silent modification.
- **Rules** hash their normalized body; **proof chains** hash their structure — a cached proof
  can be re-checked byte-for-byte.

An **integrity audit job** re-hashes stores against ids on a schedule. Cheap, boring, decisive.

## Immutable history: the hash-chained audit log

Every state transition (grain admitted, status changed, entity merged, rule retracted, conflict
resolved) appends one record to the status log
([storage schema](../Storage/storage-research.md#recommendation-sqlite-first)), and each record
includes the hash of the previous record — a hash chain, so truncation or mid-history edits
break the chain visibly. Periodic chain-head checkpoints published out-of-band (even just a
signed file elsewhere) upgrade this from tamper-*evident-internally* to
tamper-evident-against-the-administrator. This is the standard transparency-log construction
(Certificate Transparency / git commit DAG lineage) applied to belief history — prior art, not
invention.

Consequence already built into the design: *there are no destructive operations to abuse*.
Retraction is a status change; merges are recorded redirections
([entities.md](../Knowledge_Model/entities.md#entity-model)). The audit log records everything
because everything is an append.

## Signed sources

Where publishers sign content (or serve over verifiable channels), the signature is stored in
the source record and verified at acquisition. Policy hook, not a truth oracle: a signature
authenticates *who published bytes*, never that the bytes are true. Unsigned sources remain
admissible under whatever corroboration policy applies — signing raises attribution confidence,
and attribution is what poisoning response needs (below).

## Malicious knowledge injection (data poisoning)

The attack: get false grains admitted through legitimate channels — a doctored source, a
compromised mirror, a long-game "reliable then poisoned" feed.

Defenses in depth, all already-designed mechanisms doing double duty:

1. **Quarantine + verification gates** — nothing goes from acquisition to `verified` without
   the admission checks ([overview](../Architecture/overview.md#2-verify-admission)); policy can
   require N *independent* sources for sensitive relationship classes
   ([provenance.md](../Knowledge_Model/provenance.md#corroboration-not-confidence)).
2. **Contradiction engine as tripwire** — injected falsehoods that touch existing knowledge
   raise conflicts and get *both* sides suspended pending curation — the attack degrades
   availability of answers, not their truth
   ([contradiction-engine.md](../Reasoning/contradiction-engine.md)).
3. **Systematic-conflict analytics** — one source repeatedly on the losing side of conflicts is
   a poisoning signal; flag the source, and because provenance is complete, *every grain it
   ever contributed is enumerable* for re-verification or retraction. Complete provenance is
   the incident-response plan.
4. **The uncorroborated-island weakness, named honestly:** injected facts that touch nothing
   (new fake entities, no conflicts possible) evade the tripwire. Mitigation is policy
   (corroboration requirements for grains from single new sources) and the periodic
   duplicate/orphan review queue. This residual risk is accepted and documented rather than
   hidden.

## Rejected alternatives

- **Blockchain storage** — the hash-chained log + out-of-band checkpoints provide the useful
  property (tamper-evident history) without consensus machinery we have no adversary model for.
- **Reputation/trust scores for sources** — rejected for the same reason as everywhere else
  ([provenance.md](../Knowledge_Model/provenance.md#rejected-alternatives)); deterministic
  admission *policies* per source kind, yes; numeric trust, no.
- **Signing every grain individually** — grain ids are already content hashes; per-grain
  signatures add key-management weight with no added tamper evidence beyond the chained log.

## Future work

- Key management for checkpoint signing and the (future) multi-curator setting.
- Access-control design once there is a service surface (who may curate, resolve conflicts,
  accept RFCs).
- Formal audit procedure: frequency, scope, and response runbook for a failed integrity check.
