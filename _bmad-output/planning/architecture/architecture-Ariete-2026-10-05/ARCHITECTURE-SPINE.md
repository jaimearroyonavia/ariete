---
status: draft
updated: 2026-10-05
---

# Ariete Architecture Spine

## Paradigm

**Event-Driven Monolith + Async Workers**

Monitoring loop detects failures and pushes incidents to a job queue. Sidekiq workers (running inside the same Rails process or separately) consume queue jobs, execute diagnostics asynchronously, and await human approval signals. Approval triggers execution workers that carry out remediation. This separation prevents tight coupling between monitoring (real-time) and remediation (high-latency, approval-gated).

## Architectural Decisions

### AD-1: Datastore Strategy

**Binds:** Single source of truth for all persistent state — accounts, domains, credentials, incidents, audit logs, RAG documents, embeddings.

**Rule:** PostgreSQL (primary). Redis only for Sidekiq queues and ephemeral session/cache data.

**Prevents:** Split-brain state (incidents in one store, RAG in another), eventual consistency gaps, complex migration paths.

**Rationale:** Keeps transactional guarantees for critical paths (approval→execution), leverages pgvector for embeddings without introducing a separate vector DB.

---

### AD-2: Agent Execution Isolation

**Binds:** How remote commands reach infrastructure and are audited.

**Rule:** 
- SSH keys stored encrypted in Rails credentials or external secrets vault.
- Agent execution happens in Sidekiq worker context, never in web request path.
- Every command execution logged with: timestamp, incident ID, approver, command, output, exit status.
- [ASSUMPTION] SSH timeouts set to 30s per command; long-running diagnostics split into separate commands.

**Prevents:** Credentials leaked in web logs, approval bypassed via direct API calls, audit trail loss.

**Adopted:** Security constraint from SPEC.md.

---

### AD-3: Incident State Machine

**Binds:** What states an incident transits through and how state changes trigger actions.

**Rule:**
```
Detected → Diagnosing → ProposedFix → AwaitingApproval → Executing → Resolved / Failed
```
- Incident record is append-only; state changes create new audit log entries.
- Only one approval action pending per incident; simultaneous approvals rejected.
- Autonomous mode skips AwaitingApproval, transitions Diagnosing → Executing (with timestamp guardrails).

**Prevents:** Race conditions on approval, lost history, unclear incident lifecycle.

---

### AD-4: RAG Integration

**Binds:** How historical incidents feed into new diagnostics.

**Rule:**
- Store incident resolution summaries + external docs (knowledge base) as text chunks in PostgreSQL with pgvector embeddings.
- [ASSUMPTION] Use OpenAI API for embedding generation (fallback: Ollama locally if offline).
- On new incident diagnosis, retrieve top-5 similar past resolutions from vector search.
- Include retrieved summaries in agent diagnostic context (prompt injection guard: sanitize summaries before LLM).

**Prevents:** Agent ignoring known solutions, knowledge silo per incident.

---

### AD-5: API and Worker Boundaries

**Binds:** Which code paths are exposed to the web, which are internal.

**Rule:**
- **Web API (public):** Admin console calls HTTPS endpoints (JSON). Endpoints: auth, domain CRUD, infrastructure config, incident viewing, approval actions.
- **Monitoring loop (internal):** Cron job or background worker polls monitored URLs; pushes failures to Sidekiq queue as `MonitoringAlert` jobs. [ASSUMPTION] No external HTTP calls; all polling internal.
- **Agent (internal):** Sidekiq worker; calls Ruby methods directly; never exposed via HTTP.

**Prevents:** Agent over-exposed to the internet, replay loops from external monitoring, authorization confusion.

---

### AD-6: Secrets and Environment

**Binds:** Where sensitive data lives and how it's accessed.

**Rule:**
- Infrastructure credentials (SSH keys, passwords) stored encrypted in Rails encrypted credentials (`config/credentials.yml.enc`). [ASSUMPTION] Rails credentials sufficient for v1; upgrade to HashiCorp Vault if > 100 domains.
- Database URL, Redis URL, API keys: Rails env vars or credentials.
- Agent workers can decrypt credentials only at execution time; never logged.

**Prevents:** Plaintext secrets in code, credentials visible in logs, oversharing of secrets to external services.

---

## Deferred

- **Agent LLM provider:** OpenAI vs. Anthropic vs. local Ollama — decision deferred to agent module (module-level AD).
- **Autonomous guardrails specifics:** Command allowlist, blast radius limits — deferred to policy engine design.
- **Multi-region deployment:** Replication, cross-region failover — deferred to v2 (v1 single-region).
- **Integration with existing incident tracking (PagerDuty, Opsgenie):** Deferred; v1 standalone.

---

## Data Model Skeleton

```
users (accounts)
  id, email, hashed_password, created_at, updated_at

domains (per-user infrastructure scopes)
  id, user_id, name, description, created_at

infrastructure (SSH connection details per domain)
  id, domain_id, hostname, port, username, encrypted_ssh_key, created_at
  # [ASSUMPTION] supports SSH key auth; password auth deferred

monitored_urls (URLs to monitor per domain)
  id, domain_id, url, expected_status, check_interval_secs, created_at

incidents (detected failures)
  id, monitored_url_id, state (detected|diagnosing|proposed_fix|awaiting_approval|executing|resolved|failed), 
  detected_at, diagnosis, proposed_fix, execution_output, resolved_at, created_at

approvals (human approval audit trail)
  id, incident_id, approver_id (user), approved_at, comment

rag_documents (knowledge base)
  id, title, content, source, created_at

rag_embeddings (pgvector)
  id, rag_document_id, embedding (vector), chunk_index

audit_logs (every action)
  id, user_id, incident_id, action, details, timestamp
```

---

## Layers (High-Level C4)

```
┌─────────────────────────────────────────┐
│  Web UI (React/Vue)                     │
│  - Admin panel (domain, infra mgmt)     │
│  - Monitoring dashboard                 │
│  - Incident view + approve button       │
└──────────────┬──────────────────────────┘
               │ HTTPS / JSON API
┌──────────────▼──────────────────────────┐
│  Rails API + Web Server                 │
│  - Authentication, domain CRUD          │
│  - Incident endpoint, approval endpoint │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴─────┬─────────┐
        │            │         │
        ▼            ▼         ▼
   PostgreSQL   Sidekiq     Redis
   (state)      (queue)     (cache)
        │
        └────────┐
                 │
        ┌────────▼──────────────┐
        │  Sidekiq Workers      │
        │  - Monitoring loop    │
        │  - Agent (diagnose)   │
        │  - Executor (fix)     │
        └────────┬──────────────┘
                 │ SSH
        ┌────────▼──────────────┐
        │  Remote Infra         │
        │  (monitored services) │
        └───────────────────────┘
```

---

## Open Questions

1. **Agent LLM:** OpenAI API vs. Anthropic vs. local Ollama? Cost, latency, data residency trade-offs?
2. **Autonomous safeguards:** Which commands are safe to execute without approval? Need an explicit allowlist or ML-based confidence score?
3. **Monitoring frequency:** Poll every 60s, 30s, or event-driven (Webhook to Ariete)? Impacts infrastructure, audit log size.
4. **Multi-region:** v1 single-region. If v2 needs multi-region, does each region run its own worker pool, or is there a shared approval queue?

