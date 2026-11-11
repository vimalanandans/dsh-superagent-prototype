# Non-Functional Requirements & Security Guardrails (`prd/04-compliance-security-nfr.md`)

**Document Version:** 1.0  
**Target Compliance:** SOC 2 Type II, ISO/IEC 27001:2022, GDPR, HIPAA  
**Applicability:** All Production Deployments of Enterprise dsh-superagent  

---

## 1. Security & Data Isolation Invariants

### 1.1 Zero Cross-Tenant Leakage Guarantee
- **Invariant:** No trigger, prompt text, repository file, or memory buffer belonging to Customer Alpha shall ever be transmitted to, stored in, or readable by Customer Beta.
- **Enforcement Mechanism:**
  1. *Transport Layer:* Ingress requests are routed strictly to tenant-scoped Kafka/gRPC channels.
  2. *Process Layer:* Worker execution is constrained within dedicated gVisor (`runsc`) user-space kernel containers.
  3. *Storage Layer:* Database state is segregated into customer-dedicated SQLite databases or PostgreSQL Row-Level Security (RLS) tables.

### 1.2 Ephemeral Credentials & Secret Zeroization
- All GitHub tokens must be short-lived Installation Access Tokens (IAT) with a Maximum Lifetime of 60 minutes.
- Worker processes must never store personal access tokens, private keys, or API tokens in environment variables or on persistent disk. Tokens are mounted via in-memory `tmpfs` volumes and wiped immediately upon ticket conclusion.

### 1.3 LLM Data Governance & Zero Data Retention (ZDR)
- For Bring-Your-Own-Key (BYOK) customers, all inference calls must route directly to the customer's enterprise endpoint (e.g., Azure OpenAI with Zero Data Retention enabled).
- The platform core must never transmit customer code to public consumer endpoints.

---

## 2. Service Level Agreements (SLAs) & Performance Targets

| Area | Metric | Target SLA | Monitoring Method |
|---|---|---|---|
| **Ingress Gateway** | Webhook Acceptance Latency (p95) | **< 30 ms** | CloudWatch / Prometheus APM |
| **Ingress Gateway** | Webhook Uptime | **99.99%** | Synthetic health check every 10s |
| **Search Engine** | Cross-Tenant Omnibar Latency (p95) | **< 50 ms** | API Gateway response telemetry |
| **Worker Engine** | Worker Slot Launch Latency (p95) | **< 2.5 s** | Time from accept to `assignmentStarted` |
| **Disaster Recovery** | Recovery Point Objective (RPO) | **< 1.0 s** | Litestream continuous WAL replication |
| **Disaster Recovery** | Recovery Time Objective (RTO) | **< 3.0 s** | Cold container restart with S3 restore |

---

## 3. Concurrency & Fair-Share Rate Limiting

### 3.1 Tier-1 Dedicated Pods (Model B)
- Each dedicated customer pod has a hard concurrency quota of **4 worker slots + 1 priority slot** for human wake events (`docs/003-execution.md`).
- A customer cannot consume resources from another tenant's pod.

### 3.2 Tier-2 Shared Pools (Model A)
- The global worker allocator implements **Weighted Fair Queuing (WFQ)**.
- A single customer's burst (e.g., 50 CI failures) is throttled to at most 2 concurrent slots in the shared pool, ensuring other standard tenants are never starved.

---

## 4. Compliance, Audit Trails & Observability

### 4.1 Immutable Mandate Audit Log
- Every code commit, PR creation, comment, and tool execution must record:
  - `tenantId`
  - `ticketId`
  - `mandateVersion` and `policyVersion`
  - `actor` (GitHub/Lark identity that authorized the action)
  - `timestamp` (ISO 8601 UTC)
  - `gitDiff` SHA-256 hash
- Logs are exported to write-once, read-many (WORM) S3 buckets with Object Lock enabled for 7 years compliance retention.

### 4.2 OpenTelemetry (OTel) Distributed Tracing
- All requests carry a `traceparent` header linking:
  ```
  Webhook Ingress ──► Engine Intake ──► Policy Gate ──► Orchestrator Eval ──► Worker Session ──► Outbox Reply
  ```
- Traces are exported to central Jaeger / Datadog collectors tagged with `tenant.id` and `ticket.stage`.
