# AD-044 — Tenant-Evaluation-Config Table Hardened

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-44 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-64, AD-36, AD-18

## Context

The `{env}-tenant-evaluation-config` table lets tenants override quality thresholds. That makes it a privilege-escalation target: a compromised workload identity that can write it could lower evaluation thresholds to near-zero and silently disable quality enforcement (MITRE ATLAS "Persist — Modify Agent Config"). Without hardening, a single compromised IAM role produces a silent, persistent governance bypass.

## Decision

Harden the table with four controls: a resource-policy write-deny for all agent IAM roles (only a break-glass MFA admin may write), CloudTrail data-event logging retained for 365 days, a 60-second write-anomaly alarm publishing to a security-critical SNS topic, and a DynamoDB Streams Lambda that reverts any write setting a threshold below its governance floor.

## Alternatives Considered

- **Rely on IAM role scoping alone.** Rejected: agent roles require read access to this table; scoping them to read-only still leaves the write surface open to role compromise or misconfiguration.
- **Detect-and-alert only, no automated revert.** Rejected: detection latency means a threshold change could persist through one or more negotiations before human intervention, silently degrading quality enforcement.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Escalation surface closed at the IAM layer — agents structurally cannot write the table | Legitimate threshold changes now require a break-glass path, adding operational friction for routine config updates |
| Self-healing — the Streams Lambda reverts sub-floor writes within seconds, with full audit and near-real-time alerting | The floor-validator Lambda is an extra moving part that must itself be correct, deployed, and available |

The operational friction of the break-glass path is accepted because the alternative — silent quality-gate disablement — defeats the entire evaluation system.

## Results

This hardening pattern mirrors what was later applied to `{env}-system-config` after the AppConfig→DynamoDB migration (platform v1.7.0). The floor-revert behavior is owned by PRD-010 §5.1 / REQ-C304 and cross-references AD-64's threshold configuration model. The table sits within the Application layer of the six-layer stack (AD-36); its hardening closes the ATLAS "Modify Agent Config" escalation surface that node-level governance code (AD-18) protects at the graph level.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
