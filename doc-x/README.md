# Enterprise Superagent Architecture & Multi-Customer Strategy (`doc-x`)

## 1. Executive Summary & Purpose

This directory (`doc-x/`) contains the architectural blueprint, trade-off analysis, user interface specifications, and operational guides for scaling **dsh-superagent** from a single-operator/single-repository assistant to an **enterprise-grade, multi-customer platform** serving hundreds of enterprise clients.

The core philosophy of dsh-superagent remains immutable:
> **Take as little human attention as possible, and give as much value as possible.**

When scaling to serve hundreds of enterprise customers, this principle must be protected. Without rigorous architecture, scaling an AI agent platform across enterprise organizations leads to attention exhaustion: alert fatigue, cross-tenant noise, security breaches, noisy neighbor starvation, and administrative gridlock.

This documentation series provides the technical specifications to prevent those failure modes while making life as effortless as possible for both the platform operator and enterprise customer administrators.

---

## 2. The Core Problem Statement

An organization operating dsh-superagent faces a fundamental architectural inflection point when expanding from internal use to serving **hundreds of external enterprise customers**:

1. **Enterprise Customer Expectations:**
   - Enterprise customers are not casual users. They demand strict data segregation, compliance (SOC 2, ISO 27001, GDPR), custom security policies, dedicated repository integrations, private VPC access, and deterministic audit trails.
   - Premium enterprise customers require customized prompts, bespoke toolchains, dedicated agent identities, and strict SLAs.

2. **The Architectural Dichotomy:**
   - **Archetype A (Centralized Shared Multi-Tenant Superagent):** A single centralized agent cluster/pool acting in a stateless, session-isolated manner across all 100+ customers.
   - **Archetype B (Dedicated Superagent per Customer / Premium Superagent):** An autonomous, dedicated Superagent instance provisioned per enterprise customer, acting exclusively on their behalf with isolated state and credentials.
   - **Archetype C (The Hybrid Tiered Architecture):** A unified multi-tenant ingress, triage, and control plane combined with dynamic, sandboxed, customer-isolated worker execution environments.

3. **Operational Feasibility ("Make My Job Easy"):**
   - The platform operator cannot manually manage 100+ disconnected tmux sessions or local CLI profiles.
   - The user interface must provide a centralized "single pane of glass" for the operator to immediately spot blocked/failed tickets and source outages across all tenants, while offering a scoped, branded, self-service experience to individual enterprise admins.

---

## 3. Document Navigation

The documentation in `doc-x/` is organized into logical, sequential specifications:

| Document | Focus & Scope | Target Audience |
|---|---|---|
| [**01. Enterprise Tenancy Models**](01-enterprise-tenancy-models.md) | Architectural specifications for Centralized Shared (Model A), Dedicated Siloed (Model B), and Hybrid Tiered (Model C). Deep mapping of domain entities (`Source`, `Trigger`, `Thread`, `Eval`, `Mandate`, `Ticket`, `Assignment`, `Executor`, `Policy`) to multi-tenancy. | Architects, Lead Engineers |
| [**02. Multi-Customer Trade-Off Analysis**](02-multi-customer-tradeoff-analysis.md) | 8-dimensional trade-off matrix comparing Centralized Shared vs. Dedicated Siloed vs. Hybrid. Detailed evaluation of security, noisy neighbors, LLM token attribution, SRE complexity, blast radius, and TCO. Decision framework for tiering customers. | Architects, Product Managers, CTO/SRE Leads |
| [**03. Enterprise Panel & UI Specification**](03-enterprise-panel-and-ui-specification.md) | Redesign and extension of the Superagent Panel. Dual-persona architecture: Operator Fleet Control Plane ("Make my job easy") vs. Enterprise Customer Portal. Wireframes, API routes, RBAC, and telemetry. | Full-stack Engineers, UI/UX Designers |
| [**04. Enterprise Deployment & Operations Guide**](04-enterprise-deployment-and-operations-guide.md) | Production infrastructure blueprints: Kubernetes/Container fleet, storage & SQLite/Postgres strategies, secret management, CI/CD provisioning, backup/restore, and SRE runbooks. | DevOps, SREs, Platform Engineers |

---

## 4. Architectural Alignment with dsh-superagent Core

All specifications in `doc-x/` preserve the strict invariants established in `docs/`:

1. **Domain Purity:** The core domain (`src/core/`) remains agnostic of transport and hosting mechanics. Tenancy is modeled cleanly without contaminating the mathematical elegance of `Eval`, `Mandate`, and `Ticket`.
2. **Deterministic Gate before Stochastic Assessment:** Unadmitted requests from unauthorized customer tenants are filtered deterministically before reaching any LLM context.
3. **Accountability & Mandates:** Every action taken by a worker must trace back to a cryptographically validated, policy-governed `Mandate` tied to a verified customer actor.
4. **Attention Invariant:** High-noise, broadcast notifications are rejected. The system only alerts operators or customer engineers when human intervention is strictly blocking forward progress.
