# Ariete Data Model

## Core Tables

### users
Account holders (SaaS tenants).
```sql
id BIGINT PRIMARY KEY
email VARCHAR NOT NULL UNIQUE
encrypted_password VARCHAR NOT NULL
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()
```

### domains
Infrastructure scopes per user (e.g., "production", "staging").
```sql
id BIGINT PRIMARY KEY
user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE
name VARCHAR NOT NULL
description TEXT
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()
UNIQUE(user_id, name)
```

### infrastructure
SSH connection endpoints per domain.
```sql
id BIGINT PRIMARY KEY
domain_id BIGINT NOT NULL REFERENCES domains(id) ON DELETE CASCADE
hostname VARCHAR NOT NULL
port INTEGER DEFAULT 22
username VARCHAR NOT NULL
encrypted_ssh_key TEXT NOT NULL  -- encrypted at rest using Rails credentials
ssh_key_algorithm VARCHAR DEFAULT 'rsa'  -- rsa, ed25519, etc.
connection_timeout_secs INTEGER DEFAULT 30
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()
```

### monitored_urls
URLs to poll per domain (e.g., https://api.example.com/health).
```sql
id BIGINT PRIMARY KEY
domain_id BIGINT NOT NULL REFERENCES domains(id) ON DELETE CASCADE
url VARCHAR NOT NULL
expected_status_code INTEGER DEFAULT 200
check_interval_secs INTEGER DEFAULT 60  -- min 30s [ASSUMPTION]
enabled BOOLEAN DEFAULT TRUE
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()
```

### incidents
Detected failures and their resolution state.
```sql
id BIGINT PRIMARY KEY
monitored_url_id BIGINT NOT NULL REFERENCES monitored_urls(id) ON DELETE CASCADE
state VARCHAR NOT NULL  -- detected, diagnosing, proposed_fix, awaiting_approval, executing, resolved, failed
detected_at TIMESTAMP NOT NULL
diagnosis TEXT  -- root cause analysis (LLM output or agent notes)
proposed_fix TEXT  -- remediation steps (LLM output)
execution_output TEXT  -- stdout/stderr from executed command(s)
resolved_at TIMESTAMP
failed_reason TEXT  -- if state = failed, why
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()
autonomous BOOLEAN DEFAULT FALSE  -- true if executed without human approval

-- Indexes for queries
INDEX idx_state (state)
INDEX idx_created_at (created_at DESC)
```

### approvals
Audit trail of human approvals.
```sql
id BIGINT PRIMARY KEY
incident_id BIGINT NOT NULL REFERENCES incidents(id) ON DELETE CASCADE
approver_id BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT
comment TEXT
approved_at TIMESTAMP DEFAULT NOW()
executed_at TIMESTAMP  -- when fix actually ran (post-approval)
execution_failed BOOLEAN DEFAULT FALSE
failure_reason TEXT  -- if execution_failed = true

-- Only one pending approval per incident (constraint enforced in app)
INDEX idx_incident_id (incident_id)
```

### rag_documents
Knowledge base entries (historical incident resolutions, docs, runbooks).
```sql
id BIGINT PRIMARY KEY
domain_id BIGINT REFERENCES domains(id) ON DELETE CASCADE  -- [ASSUMPTION] org-wide KB if NULL
title VARCHAR NOT NULL
content TEXT NOT NULL
source VARCHAR  -- "internal_incident", "runbook", "public_docs", etc.
created_at TIMESTAMP DEFAULT NOW()
updated_at TIMESTAMP DEFAULT NOW()

INDEX idx_domain_id (domain_id)
```

### rag_embeddings (pgvector)
Vector embeddings for semantic search.
```sql
id BIGINT PRIMARY KEY
rag_document_id BIGINT NOT NULL REFERENCES rag_documents(id) ON DELETE CASCADE
embedding vector(1536)  -- [ASSUMPTION] OpenAI embedding dimension (ada-002)
chunk_index INTEGER  -- if documents are chunked, which chunk
chunk_text TEXT  -- the actual text that was embedded
created_at TIMESTAMP DEFAULT NOW()

-- Vector index for similarity search
INDEX idx_embedding_hnsw ON rag_embeddings USING hnsw (embedding vector_cosine_ops)
```

### audit_logs
Immutable log of all sensitive actions.
```sql
id BIGINT PRIMARY KEY
user_id BIGINT REFERENCES users(id) ON DELETE SET NULL  -- NULL if system action
incident_id BIGINT REFERENCES incidents(id) ON DELETE SET NULL
action VARCHAR NOT NULL  -- "approve_fix", "execute_fix", "create_domain", "add_infrastructure", etc.
details JSONB  -- context-specific (e.g., {"command": "systemctl restart nginx", "exit_code": 0})
ip_address INET
created_at TIMESTAMP DEFAULT NOW()

INDEX idx_user_id (user_id)
INDEX idx_incident_id (incident_id)
INDEX idx_created_at (created_at DESC)
```

---

## Data Flow Examples

### Incident Detection → Approval → Execution

```
1. Monitoring loop (cron or background job) polls monitored_urls.url
2. If URL fails (timeout, 5xx, DNS error):
   a. Insert incident row (state='detected', detected_at=now)
   b. Enqueue MonitoringAlert job to Sidekiq
3. MonitoringAlert job (Sidekiq worker):
   a. Update incident state to 'diagnosing'
   b. Fetch infrastructure creds for domain, decrypt
   c. SSH to infrastructure, run diagnostics (grep logs, check service status)
   d. Query RAG: vector_search(diagnosis_context) → top-5 similar resolutions
   e. Call LLM (OpenAI) with logs + RAG context → proposed fix
   f. Update incident (state='proposed_fix', diagnosis=..., proposed_fix=...)
   g. Emit event: incident ready for approval (webhook or poll by UI)
4. UI displays incident to operator; operator clicks "Approve"
5. API endpoint POST /incidents/{id}/approve:
   a. Verify approver is authorized (domain owner or admin)
   b. Insert approval row (approver_id, approved_at, comment)
   c. Update incident state to 'awaiting_approval' → 'executing'
   d. Enqueue ExecuteFix job to Sidekiq
6. ExecuteFix worker:
   a. Decrypt infrastructure creds
   b. SSH, execute proposed_fix commands
   c. Capture stdout/stderr
   d. Update incident (state='resolved'/'failed', execution_output=...)
   e. Log to audit_logs (action='execute_fix', details={command, exit_code})
```

### Autonomous Mode (Off-Hours)

```
1-3. Same as above (detection, diagnosis, proposal)
4. Check incident.detected_at; if within off-hours window AND state='proposed_fix':
   a. Check autonomous policies (allowlist, blast radius) for proposed command
   b. If approved for autonomous execution:
      i.  Update incident state to 'executing', set autonomous=true
      ii. Enqueue ExecuteFix immediately (no approval step)
5. ExecuteFix proceeds as above
6. Operator reviews audit trail next morning (all autonomous actions logged)
```

### RAG Knowledge Base Update

```
1. Operator resolves an incident manually (outside Ariete)
2. Creates rag_document entry: "Nginx out of file descriptors — increase ulimit"
3. Backend generates embedding via OpenAI API
4. Inserts rag_embeddings row
5. Next incident with similar symptoms: vector_search retrieves this doc
```

---

## Constraints & Indexes

- **users.email**: UNIQUE (login key)
- **monitored_urls.url, domains.name**: Combinations should be unique per domain (enforced in app validation)
- **incidents**: Separate index on `state` for filtering dashboard queries
- **audit_logs**: Append-only; never update/delete (audit compliance)
- **rag_embeddings**: Vectorized index (pgvector HNSW) for fast similarity search

---

## Assumptions & Deferred

- [ASSUMPTION] SSH key encryption uses Rails.application.credentials; upgrade to Vault if scaling > 1000 domains.
- [ASSUMPTION] pgvector uses OpenAI ada-002 embeddings (1536 dims); if switching to local Ollama, adjust dim and re-embed.
- [ASSUMPTION] check_interval_secs minimum 30s to avoid DoS-ing monitored services.
- **Deferred:** Sharding/partitioning strategy for incidents table (v1: single instance, partition by domain later if needed).
- **Deferred:** Backup and retention policies (audit_logs archive to S3 after 90 days?).
