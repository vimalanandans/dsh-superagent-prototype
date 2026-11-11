# Engineering Backlog & User Stories (`prd/03-epics-and-engineering-backlog.md`)

**Sprint Scope:** MVP Phase 1 & Phase 2  
**Target Dev Teams:** Fleet Platform Squad (Squad A), Runtime & Security Squad (Squad B), Frontend Squad (Squad C)  
**Methodology:** INVEST criteria, Gherkin syntax (`Given/When/Then`), Dean Peters PM Skill Guidelines  

---

## 1. Backlog Overview & Epic Hierarchy

```
EPIC-100: Multi-Tenant Ingress & Webhook Demultiplexing Gateway (Squad A)
  ├── US-101: GitHub & Lark Webhook Ingress Termination & HMAC Verification
  ├── US-102: Tenant Resolution & Durable Event Forwarding
  └── US-103: Ingress Rate Limiter & Dead-Letter Queue (DLQ)

EPIC-200: Operator Fleet Control Plane UI ("Make My Job Easy") (Squad C)
  ├── US-201: Cross-Tenant Attention Triage Queue
  ├── US-202: Fleet Source Health & Quota Matrix
  └── US-203: Sub-50ms Omnibar Search Across All Customer Work

EPIC-300: Dedicated Enterprise Pod & gVisor Sandbox Runtime (Squad B)
  ├── US-301: Kubernetes CRD Operator for Tenant Pod Lifecycle
  ├── US-302: gVisor (runsc) User-Space Kernel Isolation for Worker Shells
  └── US-303: Litestream WAL Streaming & 2-Second State Recovery

EPIC-400: Enterprise Security & Credential Isolation (Squad B)
  ├── US-401: GitHub App Ephemeral Installation Access Token (IAT) Shim
  └── US-402: HashiCorp Vault Integration for Bring-Your-Own-Key (BYOK)

EPIC-500: Scoped Enterprise Customer Portal (Squad C)
  ├── US-501: Corporate SSO (OIDC/SAML) & Tenant RBAC Gateway
  ├── US-502: Tenant-Scoped Ticket & Attention View
  └── US-503: Self-Service Team Allowed List & Prompt Guidelines Editor
```

---

## 2. Sprint Backlog: Detailed User Stories

### EPIC-100: Multi-Tenant Ingress & Gateway

#### US-101: Webhook Ingress Termination & HMAC Verification
- **Priority:** P0 (Blocker) | **Story Points:** 5 | **Squad:** Squad A
- **User Story:**
  *As an Ingress Gateway,*  
  *I want to* intercept incoming webhooks from GitHub and Lark, verify their cryptographic signatures, and identify the tenant,  
  *So that* untrusted or forged requests are rejected at the edge before consuming internal compute.
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Legitimate GitHub webhook received
    Given a POST request arrives at "/webhook/github" with header "X-Hub-Signature-256"
    And the payload contains repository "acme-bank/core-banking"
    When the gateway looks up the webhook secret for "acme-bank" in Vault
    And validates the HMAC SHA-256 signature
    Then the gateway responds with HTTP 202 Accepted in < 30ms
    And pushes the payload onto the internal Kafka topic "tenant.acme-bank.ingress".

  Scenario: Unregistered or forged webhook signature
    Given a POST request arrives with an invalid "X-Hub-Signature-256"
    When the HMAC calculation fails against registered tenant secrets
    Then the gateway immediately rejects the request with HTTP 401 Unauthorized
    And logs a security metric "ingress.signature_failure".
  ```
- **Edge Cases:** Clock skew on webhook headers; sudden burst of 500 events on a bulk merge commit.

---

### EPIC-200: Operator Fleet Control Plane UI

#### US-201: Cross-Tenant Attention Triage Queue
- **Priority:** P0 (Blocker) | **Story Points:** 8 | **Squad:** Squad C + Squad A
- **User Story:**
  *As a Platform Operator ("Alex"),*  
  *I want to* view an aggregated list of all tickets currently blocked or requiring human decisions across all 100+ enterprise clients,  
  *So that* I can take action in one click without browsing multiple panels.
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Displaying blocked tickets across multiple customers
    Given Customer "Alpha" has Ticket "#T-1" in stage "person / blocked, failed"
    And Customer "Beta" has Ticket "#T-2" in stage "person / answer"
    And Customer "Gamma" has 15 tickets in stage "working"
    When the operator logs into the Fleet Control Plane
    Then the Attention Queue displays exactly 2 cards (Ticket #T-1 and #T-2)
    And displays the customer name, duration blocked, and error/question summary
    And provides a "[Retry Assignment]" button on Ticket #T-1.

  Scenario: Zero items requiring attention
    Given all customer tickets across all 100 tenants are in stage "working", "idle", or "ended"
    When the operator opens the Fleet Control Plane
    Then the Attention Queue displays a serene green status "All customer agents healthy. Zero attention required."
  ```

#### US-203: Sub-50ms Omnibar Global Search
- **Priority:** P1 | **Story Points:** 5 | **Squad:** Squad A + Squad C
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Searching by GitHub PR number
    Given Customer "FinTech-Pro" has PR "#204" tracked by Ticket "#T-882"
    When the operator types "#204" into the Fleet Omnibar
    Then search results return within 50ms
    And display "FinTech-Pro: Ticket #T-882 (PR #204) - Stage: Working"
    And clicking navigates directly to the ticket inspection view.
  ```

---

### EPIC-300: Dedicated Enterprise Pod & gVisor Sandbox Runtime

#### US-302: gVisor (runsc) User-Space Kernel Isolation
- **Priority:** P0 (Security Critical) | **Story Points:** 8 | **Squad:** Squad B
- **User Story:**
  *As an Enterprise Security Administrator ("Marcus"),*  
  *I want* worker execution sessions to run in a user-space virtualized kernel (gVisor),  
  *So that* customer-authored compilation scripts or third-party npm packages cannot execute container escapes or inspect neighboring memory.
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Worker executes shell commands in sandbox
    Given a worker session runs for Ticket "#T-900" in tenant pod "acme-bank"
    When the worker executes a shell command "prepare: npm install && make test"
    Then the system calls are handled by gVisor (RuntimeClass: runsc)
    And access to host "/proc/kcore", raw sockets, or host mounts is denied with EPERM
    And the worker directory "/work/acme-bank/T-900" is strictly isolated to its own mount namespace.
  ```

#### US-303: Litestream WAL Backup & 2-Second Pod Recovery
- **Priority:** P1 | **Story Points:** 5 | **Squad:** Squad B
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Dedicated pod crashes and recovers
    Given dedicated pod "tenant-healthcorp" experiences an OOM crash
    When Kubernetes spins up a replacement pod on a different node
    Then Litestream restores the SQLite database from S3 in < 2.0 seconds
    And "Runtime.ready" resolves with zero lost committed transactions
    And any interrupted assignment restarts automatically with a recovery note.
  ```

---

### EPIC-400: Enterprise Security & Credential Isolation

#### US-401: GitHub App Ephemeral IAT Token Shim
- **Priority:** P0 | **Story Points:** 5 | **Squad:** Squad B
- **User Story:**
  *As a Worker Session,*  
  *I want* git and GitHub CLI commands to authenticate using short-lived Installation Access Tokens,  
  *So that* no persistent personal access tokens or passwords reside on disk.
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Worker executes git push
    Given an active worker session for customer "Acme-Bank"
    When the worker runs "gh pr create --title 'Refactor auth'"
    Then the credential helper requests an IAT from the GitHub App for Acme's installation ID
    And the token is injected into memory with an expiration of 60 minutes
    And the token has zero read/write access to any other organization.
  ```

---

### EPIC-500: Scoped Enterprise Customer Portal

#### US-501 & US-503: Customer SSO & Self-Service Allowed List
- **Priority:** P1 | **Story Points:** 8 | **Squad:** Squad C
- **User Story:**
  *As an Enterprise Engineering Lead ("Elena"),*  
  *I want to* log in via corporate Okta SSO and update our team's allowed requester logins,  
  *So that* my team can start mentioning the agent without waiting for platform support.
- **Acceptance Criteria:**
  ```gherkin
  Scenario: Customer admin adds member to allowed list
    Given User "Elena" is authenticated with Okta JWT for "Acme-Bank" with role "tenant_admin"
    When Elena navigates to "Settings -> Team Permissions"
    And adds GitHub login "@sarah-dev" to the allowed list
    Then the policy for "Acme-Bank" updates its allowed list
    And the policy version increments from "v1.2" to "v1.3"
    And live tickets receive a "policyChanged" notice
    And "@sarah-dev" is immediately permitted to request work via mentions.
  ```

---

## 3. Definition of Done (DoD)

A User Story is considered **Done** when and only when:
1. **Automated Unit & Integration Tests:** 100% pass rate in Vitest, including multi-tenant isolation unit tests.
2. **Security & Sandbox Verification:** Passes automated gVisor syscall containment checks; zero filesystem traversal outside tenant directory.
3. **Audit Trail Logging:** All configuration mutations emit structured JSON audit events with `actor`, `tenantId`, `timestamp`, and `diff`.
4. **Localization:** Panel labels and error strings populated in both English (`locale/en.json`) and Chinese (`locale/zh.json`).
5. **PR Review:** Reviewed and approved by at least one Domain Lead and one Security Architect.
