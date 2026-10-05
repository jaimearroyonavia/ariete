---
id: SPEC-ariete
companions: []
sources: []
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate.

# Ariete: Autonomous Agent for Infrastructure Error Diagnosis and Remediation

## Why

SaaS and infrastructure teams lack 24/7 on-call coverage. When services fail outside business hours, resolution is delayed, customer reputation suffers, and time is wasted on recurring, trivial problems. Ariete fills this gap by diagnosing infrastructure failures autonomously using an internal knowledge base (RAG) and proposing fixes for human approval, with optional autonomous execution during off-hours. Target users are infrastructure operators, DevOps teams, and SaaS product teams seeking faster incident resolution without hiring more support staff.

## Capabilities

- **CAP-1: Public URL monitoring**
  - **intent:** Continuously monitor a set of configured URLs and detect service failures (timeouts, connectivity issues, DNS errors, HTTP errors, app crashes).
  - **success:** Detects all monitored URL failures within 60 seconds; alerts are logged and trigger the diagnostic flow.

- **CAP-2: Remote infrastructure diagnostics**
  - **intent:** When a monitored URL fails, connect to the associated infrastructure, analyze service logs and state, and propose a root cause and remediation steps.
  - **success:** Diagnostics identify at least one actionable root cause (e.g., "PHP-FPM spare servers exhausted", "disk full, database cannot write") from logs and service state; proposal is human-readable and actionable.

- **CAP-3: Human-in-the-loop approval and execution**
  - **intent:** Operator reviews and approves (or rejects) proposed fixes via a web widget on any device without requiring direct SSH to the host.
  - **success:** Fix is applied within 30 seconds of approval; execution status and output are logged and reported back to the operator.

- **CAP-4: Autonomous remediation (off-hours mode)**
  - **intent:** During configured time windows (e.g., nights), Ariete executes approved remediation strategies without human approval, respecting predefined guardrails (e.g., no destructive changes, no scaling).
  - **success:** Autonomous fixes execute as scheduled; all actions are logged with justification and audit trail.

- **CAP-5: Internal knowledge base (RAG)**
  - **intent:** Store historical incidents, resolutions, and infrastructure knowledge; use embeddings to surface relevant past solutions when diagnosing new failures.
  - **success:** RAG retrieves at least one relevant historical resolution per incident when a matching pattern exists; operator can review retrieved context before approving fixes.

- **CAP-6: Multi-tenant administration**
  - **intent:** Users create accounts, define domains, configure infrastructure endpoints and credentials, and manage monitoring rules per domain.
  - **success:** Admin can create a domain, add infrastructure (host, credentials, monitored URLs), and start monitoring within 5 minutes; credentials are encrypted at rest and in transit.

## Constraints

- **Security is primary.** Remote command execution on infrastructure is dangerous. All agent actions must be authenticated, authorized, and audited. Credentials must never be logged in plaintext. Operators must explicitly approve (or authorize autonomous execution) before any remediation is applied.
- **Non-goals (v1): Read-only on customer data.** Ariete may read error logs but must not access or modify customer databases or persistent data.
- **Non-goals (v1): No infrastructure scaling.** Ariete does not create new VMs, containers, or scale deployments.
- **Non-goals (v1): No permanent code or config changes.** Ariete may restart services or adjust ephemeral state but not deploy code or modify configuration files.
- **Non-goals (v1): No backup management.** Backup and recovery strategies are out of scope.

## Non-goals

- Predict infrastructure failures before they occur (proactive scaling, capacity planning).
- Automatically heal all classes of errors (scope limited to known, remediable patterns).
- Replace human incident response; always requires human judgment for high-impact decisions.
- Support for on-premises or air-gapped infrastructure (assumes cloud or hybrid with outbound connectivity).
- Real-time log aggregation or replacement of existing log management systems.

## Success signal

Ariete succeeds when:
1. **MTTR improvement:** Mean time to resolution decreases by at least 50% for monitored services.
2. **Autonomous fix rate:** At least 30% of detected incidents are resolved by Ariete without human intervention (during off-hours autonomous mode or trivial failures).
3. **Uptime improvement:** Monitored services maintain ≥99.5% uptime (baseline 98.5%) due to faster detection and remediation.
4. **On-call load reduction:** Paging events for Ariete-monitored services drop by at least 40%.
5. **Knowledge base maturity:** RAG correctly surfaces a relevant historical solution for ≥70% of incidents by month 3.

## Assumptions

- Customers own or have remote access to the infrastructure Ariete monitors (SSH, API credentials, etc.).
- Monitored services follow standard conventions (error logs in `/var/log`, standard process management with systemd/supervisord, etc.).
- Operator approval for high-risk fixes is always available (assume 15-min SLA for human review during business hours).

## Open Questions

- What classes of remediation are safe for autonomous execution? (e.g., restart service yes, delete logs no, resize pool maybe?)
- Should Ariete integrate with existing incident-tracking systems (PagerDuty, Opsgenie) or start standalone?
- What is the maximum blast radius for a single autonomous fix (e.g., no more than N% CPU spike, no more than K connections)?
- Should the knowledge base auto-ingest from public sources (e.g., official docs, Stack Overflow) or start with user-curated incidents?
