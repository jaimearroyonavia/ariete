---
status: in-progress
stepsCompleted: [step-01-validate-prerequisites]
inputDocuments: [/Users/jaimearroyo/Proyectos/Personales/Ariete/_bmad-output/specs/spec-ariete/SPEC.md, /Users/jaimearroyo/Proyectos/Personales/Ariete/_bmad-output/planning/architecture/architecture-Ariete-2026-10-05/ARCHITECTURE-SPINE.md, /Users/jaimearroyo/Proyectos/Personales/Ariete/_bmad-output/planning/architecture/architecture-Ariete-2026-10-05/DATA-MODEL.md, /Users/jaimearroyo/Proyectos/Personales/Ariete/_bmad-output/planning/architecture/architecture-Ariete-2026-10-05/STACK.md]
---

# Ariete - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Ariete, decomposing requirements from the Specification, Architecture, and Data Model into implementable user stories with acceptance criteria for developers.

---

## Requirements Inventory

### Functional Requirements

- **FR1:** Continuously monitor configured URLs for service failures (timeouts, DNS errors, connectivity issues, HTTP errors, app crashes) with detection within 60 seconds
- **FR2:** Diagnose infrastructure failures remotely via SSH by analyzing error logs and service state to identify root cause
- **FR3:** Propose remediation steps based on diagnostics using LLM analysis of logs and RAG knowledge base
- **FR4:** Provide web UI widget for operators to review and approve proposed fixes without direct SSH access
- **FR5:** Execute approved remediation commands remotely via SSH with command output capture and status reporting
- **FR6:** Execute remediation autonomously during configured off-hours windows without human approval (with guardrails)
- **FR7:** Store, retrieve, and integrate historical incident resolutions into diagnostics via RAG + vector embeddings
- **FR8:** Multi-tenant administration: users create accounts, define infrastructure domains, add endpoints and credentials, manage monitoring rules
- **FR9:** Audit logging of all sensitive actions (approvals, executions, credential access) with user, timestamp, command, and output

### Non-Functional Requirements

- **NFR1:** Security - all agent actions authenticated, authorized, and audited; no unauthorized command execution
- **NFR2:** Credentials encrypted at rest (Rails credentials or Vault) and in transit; never logged plaintext
- **NFR3:** Remediation execution within 30 seconds of approval
- **NFR4:** Monitor URL checks complete within 60 seconds of failure occurrence
- **NFR5:** Mean time to resolution (MTTR) improvement: 50% reduction for monitored services
- **NFR6:** Autonomous fix rate: 30% of detected incidents resolved without human intervention
- **NFR7:** Uptime guarantee: maintain 99.5% uptime (baseline 98.5%) for monitored services
- **NFR8:** On-call load reduction: 40% fewer paging events for Ariete-monitored services
- **NFR9:** RAG knowledge base accuracy: 70% of incidents retrieve relevant historical solutions
- **NFR10:** Incident state machine with append-only audit trail (never lose historical state changes)

### Additional Requirements (Architecture + Stack)

- **AR1:** Implement event-driven monolith pattern: monitoring failures → job queue → async diagnosis/execution
- **AR2:** PostgreSQL as primary datastore; Redis for Sidekiq job queue only
- **AR3:** pgvector extension in PostgreSQL for RAG embedding storage and vector similarity search
- **AR4:** Rails 8+ framework with Sidekiq 7.1+ for async background job processing
- **AR5:** SSH keys stored encrypted via Rails credentials (upgrade to Vault for >100 domains)
- **AR6:** Agent execution isolated to Sidekiq worker context; never execute in web request path
- **AR7:** Incident lifecycle state machine: Detected → Diagnosing → ProposedFix → AwaitingApproval → Executing → Resolved/Failed
- **AR8:** Append-only incident and approval records; state changes generate separate audit log entries
- **AR9:** OpenAI API integration for LLM diagnostics and embedding generation [ASSUMPTION]
- **AR10:** Web UI (React or Vue) with dashboard for monitoring + incident management

### Requirements Coverage Map

| Requirement | Epic | Story | Notes |
|-------------|------|-------|-------|
| FR1 | E1: Monitoring & Detection | S1.1, S1.2 | URL monitoring, detection pipeline |
| FR2 | E2: Diagnostics & Analysis | S2.1, S2.2, S2.3 | SSH connection, log analysis, LLM analysis |
| FR3 | E2: Diagnostics & Analysis | S2.3 | LLM fix proposal with RAG context |
| FR4 | E3: Approval & Execution | S3.1, S3.2 | Web UI, approval workflow |
| FR5 | E3: Approval & Execution | S3.3 | Execute remediation via SSH |
| FR6 | E4: Autonomous Mode | S4.1, S4.2 | Off-hours execution, guardrails |
| FR7 | E5: Knowledge Base (RAG) | S5.1, S5.2, S5.3 | Store docs, generate embeddings, vector search |
| FR8 | E6: Admin & Multi-Tenancy | S6.1, S6.2, S6.3, S6.4 | User accounts, domains, infrastructure, credentials |
| FR9 | E7: Audit & Compliance | S7.1, S7.2 | Audit logging, compliance reporting |
| NFR1-2 | E8: Security Hardening | S8.1, S8.2, S8.3 | Auth, secrets, encryption |
| NFR3-9 | Cross-cutting | All stories | Performance, reliability targets |

---

## Epic List (Restructured via First Principles Analysis)

### Foundation (v0 - Prerequisite)
- **E0: Multi-Tenant Administration & Infrastructure Setup**

### Core MVP (v1 - Focused delivery)
1. **E1: URL Monitoring & Incident Detection**
2. **E2: Infrastructure Diagnostics & Analysis**
3. **E3: Human Approval & Remediation Execution (including off-hours autonomous variant)**

### Enhancements (v2 - Deferred)
4. **E4: RAG Knowledge Base & Vector Search** *(improves diagnosis quality, not blocking)*
5. **E5: Audit Reporting Dashboard** *(audit logging built into each epic's DoD)*
6. **E6: Advanced Security** *(RBAC, MFA; hardening built into each epic's DoD)*

**Rationale:**
- Merged E4→E3: Autonomous execution is variant of approval logic, not separate epic
- Integrated RAG into E2: Only matters during diagnostics
- E6 Admin as Foundation: Prerequisite for all other epics
- Audit+Security as DoD: Every epic includes logging/security; no standalone needed for v1

---

## Epic 1: URL Monitoring & Incident Detection

**Goal:** Establish continuous monitoring of customer-configured URLs and reliably detect service failures within 60 seconds, triggering the diagnostic pipeline.

### Story 1.1: Set Up Monitoring Loop (Polling Infrastructure)

As a **DevOps operator**,
I want the system to continuously poll monitored URLs on a configurable interval,
So that failures are detected promptly and triggering diagnostics.

**Acceptance Criteria:**

**Given** a domain with 1+ monitored URLs configured
**When** the monitoring loop runs (cron job or Sidekiq worker)
**Then** each URL is polled with a GET request + timeout
**And** the poll completes within 5 seconds per URL
**And** results (status code, latency, error) are recorded

**Given** a URL has been failing for >60 seconds
**When** the monitoring loop polls it
**Then** an `IncidentAlert` job is enqueued to Sidekiq
**And** an incident record is created with state='detected'

**Given** the monitoring loop encounters a transient network error
**When** retrying the same URL
**Then** the system retries up to 2 times with exponential backoff
**And** only creates an incident if all retries fail

---

### Story 1.2: Classify URL Failures & Alert

As a **system**,
I want to categorize detected failures (timeout, DNS error, 5xx, connection refused, etc.),
So that operators understand failure type and diagnostics can target the right systems.

**Acceptance Criteria:**

**Given** a URL request times out after 30 seconds
**When** the monitoring loop records this result
**Then** the incident is tagged as failure_type='timeout'

**Given** a URL returns DNS resolution error
**When** the monitoring loop logs this
**Then** the incident is tagged as failure_type='dns_error'

**Given** a URL returns HTTP 500
**When** the monitoring loop records this
**Then** the incident is tagged as failure_type='http_5xx' with status_code=500

**Given** a URL is unreachable (connection refused)
**When** the monitoring loop encounters this
**Then** the incident is tagged as failure_type='connection_refused'

---

## Epic 2: Infrastructure Diagnostics & Analysis

**Goal:** When a failure is detected, automatically connect to the associated infrastructure, gather diagnostic information, and propose root causes and remediation steps.

### Story 2.1: Establish SSH Connection to Infrastructure

As a **diagnostics agent**,
I want to securely connect to remote infrastructure via SSH,
So that I can run diagnostic commands and gather system information.

**Acceptance Criteria:**

**Given** an incident with a monitored URL linked to an infrastructure endpoint
**When** the diagnostics worker starts
**Then** it retrieves the encrypted SSH key from PostgreSQL secrets
**And** it decrypts the key using Rails credentials
**And** it connects to the infrastructure via SSH (host, port, username)
**And** the SSH connection succeeds with a 10-second timeout

**Given** the SSH connection fails (authentication error, host unreachable)
**When** the diagnostics worker logs the error
**Then** it records the failure in the audit log with timestamp and error message
**And** the incident state transitions to state='failed'
**And** no plaintext credentials are logged

---

### Story 2.2: Gather Diagnostic Information from Logs & Services

As a **diagnostics agent**,
I want to query error logs and check service status on remote infrastructure,
So that I can identify root causes (disk full, service crash, resource exhaustion, etc.).

**Acceptance Criteria:**

**Given** an active SSH connection to an infrastructure endpoint
**When** the diagnostics worker runs diagnostic commands
**Then** it executes: `tail -100 /var/log/error.log` (or similar per service type)
**And** it executes: `systemctl status <service>` (or similar) to check service state
**And** it executes: `df -h` to check disk usage
**And** each command completes within 30 seconds or times out

**Given** the diagnostics worker collects error log output
**When** it processes the output
**Then** it extracts the last 100 lines of logs
**And** it stores raw log content in incident.diagnosis field

**Given** service status checks show a crashed service (e.g., nginx inactive)
**When** diagnostics logs this
**Then** it records service_name='nginx', status='inactive' in the incident diagnosis

---

### Story 2.3: Generate Proposed Fix via LLM + RAG

As a **diagnostics agent**,
I want to use LLM (with RAG-retrieved historical solutions) to propose a remediation strategy,
So that operators receive actionable recommendations for fixing the issue.

**Acceptance Criteria:**

**Given** an incident with collected log data and service state information
**When** the diagnostics worker calls the RAG service
**Then** it queries: "similar incidents with these error patterns"
**And** it retrieves top-5 past incident summaries ranked by similarity
**And** the retrieved summaries are included in the LLM prompt context

**Given** the diagnostics worker calls the LLM (OpenAI gpt-4o)
**When** the LLM receives logs + RAG context + failure_type
**Then** the LLM generates a structured response with:
  - Root cause analysis (e.g., "Nginx out of file descriptors")
  - Proposed remediation steps (e.g., "Increase ulimit -n to 65536, restart nginx")
  - Confidence level (high/medium/low)

**Given** the LLM generates a proposed fix
**When** the diagnostics worker receives the response
**Then** it stores the proposal in incident.proposed_fix field
**And** it transitions the incident state to state='proposed_fix'
**And** it emits an event to notify operators that incident is ready for approval

---

## Epic 3: Human Approval & Remediation Execution

**Goal:** Enable operators to review proposed fixes via a web UI and approve execution without direct SSH access; then execute approved fixes remotely.

### Story 3.1: Build Web UI for Incident Viewing & Approval

As an **infrastructure operator**,
I want to view detected incidents with diagnosis and proposed fixes in a web dashboard,
So that I can review and approve remediation from any device (mobile, laptop, etc.).

**Acceptance Criteria:**

**Given** an operator is logged into the Ariete web UI
**When** they visit the incidents dashboard
**Then** they see a list of incidents filtered by state (e.g., proposed_fix, awaiting_approval)
**And** each incident shows: URL, failure_type, diagnosis, proposed_fix, timestamp

**Given** an operator clicks on an incident
**When** the incident detail view loads
**Then** it displays:
  - Full error logs captured from the infrastructure
  - RAG context (similar past incidents)
  - LLM-generated diagnosis and remediation steps
  - Approval action button

**Given** an operator clicks "Approve" button
**When** the UI submits the approval
**Then** an approval record is created with:
  - incident_id, approver_id (user), approved_at (timestamp)
  - Optional comment field (for operator notes)
**And** the incident state transitions to state='awaiting_approval'

---

### Story 3.2: Approval Workflow & State Transitions

As a **system**,
I want to enforce approval state machine transitions and prevent race conditions,
So that multiple operators cannot accidentally approve the same incident twice.

**Acceptance Criteria:**

**Given** an incident in state='proposed_fix'
**When** an operator approves the fix
**Then** an approval record is created and linked to the incident
**And** the incident state transitions to state='awaiting_approval'

**Given** an approval is created for an incident
**When** the execution worker picks up the job
**Then** it verifies that exactly one approval exists for this incident
**And** it retrieves the approval.approver_id and approval.approved_at for the audit log
**And** the incident state transitions to state='executing'

**Given** an operator attempts to approve an incident that is already in state='executing'
**When** the approval API receives the request
**Then** the API returns a 409 Conflict error
**And** no second approval is created

---

### Story 3.3: Execute Approved Remediation via SSH

As a **remediation executor (Sidekiq worker)**,
I want to execute approved remediation commands on remote infrastructure,
So that proposed fixes are applied and the failure is resolved.

**Acceptance Criteria:**

**Given** an incident with a single approval and state='awaiting_approval'
**When** the execution worker starts
**Then** it retrieves the proposed remediation steps from incident.proposed_fix
**And** it re-establishes SSH connection to the infrastructure
**And** it parses the proposed steps into individual commands
**And** each command is executed in sequence with 30-second timeout

**Given** a remediation command executes (e.g., `systemctl restart nginx`)
**When** the command completes
**Then** the executor captures:
  - Command string
  - Exit code (0 for success, non-zero for failure)
  - Stdout and stderr output
  - Execution timestamp and duration

**Given** all remediation commands execute successfully (exit code 0)
**When** the executor completes
**Then** it stores the execution_output in the incident record
**And** it transitions incident state to state='resolved'
**And** it creates an audit log entry: action='execute_fix', details={command, exit_code, duration}

**Given** a remediation command fails (non-zero exit code)
**When** the executor detects the failure
**Then** it stops executing further commands
**And** it transitions incident state to state='failed'
**And** it logs the failure reason in incident.failed_reason

---

## Epic 4: Autonomous Off-Hours Remediation

**Goal:** Enable automatic remediation during configured off-hours windows without human approval, subject to guardrails (safe command allowlist, blast radius limits).

### Story 4.1: Define Autonomous Mode Configuration & Guardrails

As an **infrastructure operator**,
I want to configure off-hours time windows and command allowlist,
So that safe remediation (e.g., service restart) happens autonomously without waking me up.

**Acceptance Criteria:**

**Given** an operator configures autonomous mode settings in the admin panel
**When** they set autonomous_enabled=true, autonomous_hours='22:00-06:00', and command_allowlist=['systemctl restart *', 'systemctl reload *']
**Then** the settings are stored in the domain configuration
**And** the system validates that command_allowlist is non-empty and commands are well-formed

**Given** the autonomous_hours configuration is '22:00-06:00'
**When** the system evaluates whether to autonomously execute a fix
**Then** it checks if current_time is within the off-hours window
**And** only autonomous execution is permitted during 22:00-06:00

**Given** a proposed remediation step is 'systemctl restart nginx'
**When** the system checks if this command is in the allowlist
**Then** it matches against the allowlist pattern 'systemctl restart *'
**And** the command is approved for autonomous execution

**Given** a proposed remediation step is 'rm -rf /data'
**When** the system checks if this is allowed
**Then** it does NOT match any safe patterns in the allowlist
**And** the fix is NOT autonomously executed (requires approval)

---

### Story 4.2: Autonomous Execution During Off-Hours

As a **remediation executor**,
I want to bypass the approval requirement during off-hours for whitelisted commands,
So that safe fixes apply immediately without human delay.

**Acceptance Criteria:**

**Given** an incident detected during off-hours (22:00-06:00)
**When** diagnostics complete and propose a fix (e.g., 'systemctl restart nginx')
**Then** the system checks if autonomous mode is enabled for this domain
**And** it checks if the proposed command matches the allowlist
**And** if both conditions pass, it skips AwaitingApproval state
**And** it transitions incident state directly: Diagnosing → Executing

**Given** an incident is executed autonomously
**When** the execution completes
**Then** the incident record is marked with autonomous=true
**And** a full audit log entry is created with action='autonomous_execute_fix'
**And** the details include: command, exit_code, reason='autonomous_off_hours'

**Given** autonomous execution completes
**When** the operator reviews the incident the next morning
**Then** they see full audit trail (diagnosis, proposed fix, execution output, autonomous flag)
**And** they can optionally override/rollback the autonomous fix if needed (deferred to v2)

---

## Epic 5: RAG Knowledge Base & Vector Search

**Goal:** Build an internal knowledge base of past incident resolutions and make that knowledge accessible during diagnostics via vector similarity search.

### Story 5.1: Ingest Historical Incidents into RAG

As an **operator**,
I want to store summaries of resolved incidents in a knowledge base,
So that future similar incidents can learn from past solutions.

**Acceptance Criteria:**

**Given** an incident is resolved
**When** the operator navigates to the incident detail view
**Then** they see an "Add to Knowledge Base" button
**And** they can click to open a form with pre-filled fields: title, incident_summary, root_cause, solution_applied

**Given** the operator fills the form and clicks "Save to Knowledge Base"
**When** the form submits
**Then** a rag_documents record is created with:
  - title (operator input)
  - content (incident summary)
  - source='internal_incident'
  - domain_id (optional, for domain-specific KB)

**Given** a rag_documents record is created
**When** the backend processes it
**Then** it automatically calls OpenAI embedding API with the content
**And** it receives a 1536-dim embedding vector
**And** it stores the embedding in rag_embeddings table with pgvector

---

### Story 5.2: Vector Similarity Search for RAG Retrieval

As a **diagnostics agent**,
I want to query the knowledge base for similar past incidents,
So that the LLM can include relevant solutions in its remediation proposal.

**Acceptance Criteria:**

**Given** a new incident with collected diagnostics (logs, error messages, service state)
**When** the diagnostics worker prepares to call the LLM
**Then** it generates a query embedding from the incident's collected data
**And** it performs a pgvector similarity search: `SELECT * FROM rag_embeddings ORDER BY embedding <-> query_embedding LIMIT 5`
**And** it retrieves the top-5 most similar past incidents

**Given** the similarity search returns 5 past incident summaries
**When** the diagnostics worker constructs the LLM prompt
**Then** it includes: "Similar past incidents:\n- [incident 1 summary]\n- [incident 2 summary]\n..."
**And** the LLM uses this context to inform its remediation proposal

**Given** the similarity search returns no matches (similarity score < 0.5)
**When** the diagnostics worker logs this
**Then** it notes in the incident.diagnosis that "no similar past incidents found"
**And** the LLM proceeds without RAG context (still generates a proposal based on raw diagnostics)

---

### Story 5.3: Manage Knowledge Base & Bulk Import

As an **infrastructure operator**,
I want to bulk import runbooks and documentation into the knowledge base,
So that the system can recommend solutions for complex problems.

**Acceptance Criteria:**

**Given** an operator navigates to the knowledge base admin section
**When** they click "Import Documents"
**Then** they can upload a CSV or JSON file with columns: title, content, source

**Given** the operator uploads 10 runbooks
**When** the import processes
**Then** each row creates a rag_documents entry
**And** the backend automatically generates embeddings for each document via OpenAI API
**And** rag_embeddings records are created for all documents

**Given** documents are imported
**When** the import completes
**Then** the operator sees a summary: "10 documents imported, embeddings generated"
**And** the knowledge base immediately becomes available for future incident diagnostics

---

## Epic 6: Multi-Tenant Administration

**Goal:** Enable SaaS multi-tenancy with per-user account management, infrastructure domains, credentials storage, and monitoring rules.

### Story 6.1: User Registration & Authentication

As a **new user**,
I want to create an account and log into Ariete,
So that I can manage my infrastructure monitoring.

**Acceptance Criteria:**

**Given** an unauthenticated user visits the Ariete sign-up page
**When** they enter email and password and click "Sign Up"
**Then** a user record is created with hashed_password (bcrypt or similar)
**And** they receive a confirmation email [ASSUMPTION: email verification deferred]
**And** they can log in with email/password
**And** the system creates a session token (JWT or Rails session)

**Given** a user logs in
**When** they are authenticated
**Then** all subsequent API requests must include a valid session/auth token
**And** endpoints return 401 Unauthorized if token is missing or invalid

---

### Story 6.2: Create Infrastructure Domains

As an **infrastructure operator**,
I want to create named domains (e.g., "production", "staging"),
So that I can organize monitoring for different infrastructure environments.

**Acceptance Criteria:**

**Given** a logged-in user navigates to the "Domains" section
**When** they click "Create New Domain"
**Then** a form appears for domain name and optional description

**Given** the user enters domain name='production' and clicks "Create"
**When** the form submits
**Then** a domains record is created with user_id (current user), name='production'
**And** the domain is now visible in the user's domain list
**And** only this user can view/edit this domain (RBAC: owner)

**Given** a user attempts to access another user's domain via direct URL
**When** the request is processed
**Then** the system checks user_id != domain.user_id
**And** returns 403 Forbidden

---

### Story 6.3: Add Infrastructure Endpoints & SSH Credentials

As an **infrastructure operator**,
I want to register SSH connection details for a domain,
So that Ariete can connect and run diagnostics.

**Acceptance Criteria:**

**Given** a user has created a domain
**When** they navigate to the domain and click "Add Infrastructure"
**Then** a form appears for: hostname, port, username, SSH private key (textarea)

**Given** the user enters: hostname='api.example.com', port=22, username='root', and pastes their SSH private key
**When** they click "Add Infrastructure"
**Then** the backend:
  - Validates the hostname is a valid domain/IP
  - Validates the port is 1-65535
  - Encrypts the SSH private key using Rails credentials
  - Creates an infrastructure record with encrypted_ssh_key stored in PostgreSQL

**Given** the SSH key is encrypted
**When** an admin or developer inspects the database
**Then** they see only the encrypted blob, not the plaintext key
**And** the key is only decrypted at execution time inside a Sidekiq worker

**Given** a user attempts to add an infrastructure endpoint
**When** they enter an invalid SSH key format
**Then** the backend validates the key format (PEM, OpenSSH, etc.)
**And** returns a validation error: "Invalid SSH key format"

---

### Story 6.4: Configure Monitored URLs per Domain

As an **infrastructure operator**,
I want to add URLs to monitor for a domain,
So that Ariete detects failures and diagnoses issues.

**Acceptance Criteria:**

**Given** a user has created a domain and added infrastructure
**When** they navigate to "Monitored URLs"
**Then** they see a form to add new URLs

**Given** the user enters: url='https://api.example.com/health', expected_status_code=200, check_interval_secs=60
**When** they click "Add URL"
**Then** a monitored_urls record is created with domain_id, url, expected_status_code=200, check_interval_secs=60

**Given** the check_interval_secs is set to 60
**When** the monitoring loop runs
**Then** this URL is polled approximately every 60 seconds
**And** the polling frequency respects the interval (±5 seconds acceptable)

**Given** a user adds multiple URLs to a domain
**When** they view the domain dashboard
**Then** they see a list of all monitored URLs with current status (healthy, failed, checking)

---

## Epic 7: Audit Logging & Compliance

**Goal:** Maintain immutable audit logs of all sensitive actions (approvals, executions, credential access) for compliance and forensics.

### Story 7.1: Audit Log Data Model & Append-Only Logging

As a **security/compliance officer**,
I want an immutable record of every sensitive action in Ariete,
So that I can audit who did what and when for compliance purposes.

**Acceptance Criteria:**

**Given** the system processes a sensitive action (approve fix, execute command, view credentials, etc.)
**When** the action completes
**Then** an audit_logs entry is created with:
  - user_id (who performed the action)
  - action (e.g., 'approve_fix', 'execute_fix', 'create_domain', 'add_infrastructure')
  - incident_id (if applicable)
  - details (JSON: command executed, exit code, timestamp, etc.)
  - created_at (immutable timestamp)
**And** the audit_logs entry is never updated or deleted (append-only)

**Given** multiple approvers approve the same incident (shouldn't happen, but if race condition)
**When** the second approval is attempted
**Then** the system detects the conflict
**And** creates an audit log: action='concurrent_approval_attempted', details={incident_id, first_approver, second_approver}

---

### Story 7.2: Audit Log Querying & Reporting

As a **security officer**,
I want to query audit logs to understand incident histories and approver actions,
So that I can verify compliance and detect anomalies.

**Acceptance Criteria:**

**Given** a security officer navigates to "Audit Logs" section
**When** they apply filters (date range, user, action type, incident)
**Then** the system queries audit_logs with WHERE clauses
**And** returns paginated results matching the filters

**Given** a security officer views audit logs for an incident
**When** they click on an incident ID
**Then** they see all related audit entries in chronological order:
  - approval created (user, timestamp)
  - execution started (timestamp)
  - command executed (command, exit code, output)
  - execution completed (status)

**Given** an audit log entry contains sensitive data (e.g., command output with credentials [if leaked])
**When** the log is displayed to an unauthorized user
**Then** the system masks/redacts the sensitive fields
**And** only security admins can view unredacted logs

---

## Epic 8: Security Hardening

**Goal:** Implement security controls to prevent unauthorized access, credential theft, and misuse of remote execution privileges.

### Story 8.1: Authentication & Authorization

As a **security officer**,
I want strong authentication (session management, token expiry) and authorization checks,
So that only authorized users can access infrastructure monitoring.

**Acceptance Criteria:**

**Given** a user logs in
**When** they authenticate successfully
**Then** the system creates a session token (JWT with exp claim or Rails session)
**And** the token expires after 8 hours (configurable)

**Given** a user's session token expires
**When** they attempt an API call with an expired token
**Then** the API returns 401 Unauthorized
**And** the UI redirects to login

**Given** a user attempts to access another user's domain
**When** the API processes the request
**Then** it verifies user_id matches domain.user_id
**And** returns 403 Forbidden if authorization fails
**And** logs the unauthorized attempt in audit_logs

**Given** an operator approves a fix
**When** they click "Approve"
**Then** the backend verifies that current_user is an admin OR domain owner
**And** only then creates the approval record

---

### Story 8.2: Credential Encryption & Secrets Management

As a **system**,
I want SSH keys and passwords encrypted at rest and never logged in plaintext,
So that stolen database dumps do not expose infrastructure credentials.

**Acceptance Criteria:**

**Given** a user adds an SSH private key to an infrastructure endpoint
**When** the key is stored
**Then** Rails.application.credentials.key is used to encrypt the key
**And** the encrypted_blob is stored in PostgreSQL infrastructure.encrypted_ssh_key column

**Given** the diagnostics worker needs to connect via SSH
**When** it retrieves the infrastructure.encrypted_ssh_key
**Then** it decrypts using Rails.application.credentials.key
**And** the decryption happens in-memory only
**And** the plaintext key is never written to disk or logs

**Given** an SSH connection fails with an authentication error
**When** the error is logged
**Then** the log entry does NOT include the plaintext SSH key or password
**And** the log includes only: "SSH auth failed: hostname=api.example.com, reason=permission_denied"

**Given** command execution output contains sensitive data (passwords in error messages)
**When** the output is stored in incident.execution_output
**Then** the system should ideally redact obvious patterns (password=..., token=...) [ASSUMPTION: v2 enhancement]

---

### Story 8.3: Rate Limiting & DoS Protection

As a **system**,
I want to prevent abuse of monitoring and remediation endpoints,
So that bad actors cannot spam diagnostics or execute many commands.

**Acceptance Criteria:**

**Given** a user repeatedly polls for incident status (e.g., >100 requests/minute)
**When** the API detects excessive requests from the same IP/user
**Then** the system applies rate limiting: 429 Too Many Requests after 100 req/min
**And** the user must wait before retrying

**Given** a user attempts to execute 10 remediation commands in 1 minute
**When** the execution worker detects this
**Then** it applies a rate limit on execution: max 1 execution/minute per domain
**And** excess requests are queued or rejected

**Given** the monitoring loop polls URLs
**When** a single domain has 100+ URLs
**Then** the system spreads polling across multiple workers
**And** no single worker polls faster than 1 URL/second (to avoid DoS-ing the monitored services)

