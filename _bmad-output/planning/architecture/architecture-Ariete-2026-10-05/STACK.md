# Ariete Technology Stack

## Backend

| Component | Tech | Version | Purpose | Notes |
|-----------|------|---------|---------|-------|
| Framework | Ruby on Rails | 8.0+ | Monolith: web API, background jobs, auth | [ASSUMPTION] Rails 8 current stable |
| Ruby | Ruby | 3.3+ | Language | |
| Database | PostgreSQL | 16+ | Primary datastore | pgvector extension for RAG embeddings |
| Cache/Queue | Redis | 7+ | Sidekiq job queue, session store | Separate instance from DB |
| Background Jobs | Sidekiq | 7.1+ | Async processing (monitoring, agent, execution) | |
| Secrets | Rails credentials | native | Encrypted SSH keys, API keys | Upgrade to HashiCorp Vault if scaling >100 domains |

## Agent / Execution

| Component | Tech | Version | Purpose | Notes |
|-----------|------|---------|---------|-------|
| SSH Client | Net::SSH (Ruby gem) | 7.2+ | Remote command execution | Supports key-based auth, timeouts |
| LLM | OpenAI API | gpt-4o or gpt-4-turbo | Diagnostics, fix proposal | [ASSUMPTION] OpenAI; fallback to Anthropic or Ollama if cost/latency issues |
| Embeddings | OpenAI API | text-embedding-3-small | RAG vector search | [ASSUMPTION] 1536-dim embeddings; pgvector stores + searches |
| RAG Lib | LangChain Ruby (ruby-openai) | TBD | Embedding generation, vector search orchestration | [ASSUMPTION] Direct pgvector queries, no separate RAG framework if complexity low |

## Frontend (Admin UI)

| Component | Tech | Version | Purpose | Notes |
|-----------|------|---------|---------|-------|
| Framework | React or Vue 3 | latest | Single-page app | [ASSUMPTION] React; choose Vue if team preference |
| HTTP Client | Fetch API or axios | native/latest | API calls | |
| UI Framework | Tailwind CSS | latest | Styling | [ASSUMPTION] Tailwind; or Bootstrap if preference |
| State | TanStack Query (React) | latest | Server state sync | Handles polling for incident updates |

## Deployment & Operations

| Component | Tech | Version | Purpose | Notes |
|-----------|------|---------|---------|-------|
| Container | Docker | latest | Containerize Rails app + Sidekiq | Separate images for web/worker or single image with `ps` switch |
| Orchestration | Kubernetes or Docker Compose | latest | Prod: K8s; dev: Docker Compose | [ASSUMPTION] Kubernetes for v1 (Helm chart) |
| Monitoring | Prometheus + Grafana | latest | App metrics, alerting | [ASSUMPTION] Standard setup; logs to CloudWatch/ELK/Loki |
| Logging | ELK Stack or CloudWatch | latest | Centralized logs | [ASSUMPTION] CloudWatch if AWS; ELK if self-hosted |

## Development Tools

| Component | Tech | Version |
|-----------|------|---------|
| Linter | RuboCop | latest |
| Testing | RSpec + FactoryBot | latest |
| API Docs | OpenAPI (Swagger) | 3.1 |

---

## Gem Dependencies (Core)

```ruby
# Web framework
rails 8.0+
puma                    # application server

# Database
pg                      # PostgreSQL adapter
pgvector                # Vector search in PostgreSQL

# Authentication & Secrets
devise                  # [ASSUMPTION] or custom auth for simplicity
rails-security          # [ASSUMPTION] OWASP compliance helpers
secretsmanagerclient    # [ASSUMPTION] if using Vault

# Background jobs
sidekiq 7.1+            # job processing
sidekiq-pro             # [ASSUMPTION] for reliability (retry, rate limiting)

# SSH & remote execution
net-ssh 7.2+            # SSH client library
net-scp                 # SCP for file transfer [DEFERRED]

# LLM integration
ruby-openai             # OpenAI API wrapper

# RAG / Embeddings
rails-vectorize         # [ASSUMPTION] or direct pgvector queries

# JSON & serialization
active_model_serializers # API response formatting
jsonapi-serializer      # [ASSUMPTION] JSON:API compliance

# Monitoring & logging
prometheus-client       # Prometheus metrics
ruby-datadog            # [ASSUMPTION] Datadog integration optional

# Testing
rspec-rails
factory_bot_rails
faker                   # fake data generation
webmock                 # HTTP mocking
```

---

## Environment Variables (Example)

```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/ariete_prod
REDIS_URL=redis://localhost:6379/0

# Secrets
RAILS_MASTER_KEY=<your-key>
RAILS_ENV=production

# LLM
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=<optional>

# Feature flags
AUTONOMOUS_MODE_ENABLED=true
AUTONOMOUS_MODE_HOURS=22-06  # 10pm-6am

# Monitoring
PROMETHEUS_PUSHGATEWAY_URL=http://localhost:9091
DATADOG_ENABLED=false
```

---

## Assumptions & Deferred

- [ASSUMPTION] Docker Compose for local dev, Kubernetes (Helm) for production.
- [ASSUMPTION] OpenAI API for LLM + embeddings; consider Anthropic or self-hosted Ollama if cost/latency becomes issue.
- [ASSUMPTION] Devise for auth; roll custom if minimal scope.
- **Deferred:** API versioning strategy (URL `/v1/`, header, or content negotiation).
- **Deferred:** RBAC (role-based access control) — starts simple (domain owner, admin), scales to fine-grained later.
- **Deferred:** WebSocket support for real-time incident updates (v1: polling, v2: WebSocket if UX requires).
