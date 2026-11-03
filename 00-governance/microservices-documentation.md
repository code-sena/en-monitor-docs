# Per-Microservice Documentation Standard

> Defines exactly what documents each bounded context/module must have, who writes them,
> when they are created, and when they must be updated. Applicable to every microservice of the system
> (it was written for the modular monolith and guided the migration to microservices). Non-compliance
> blocks the merge.

> **Note (28-Oct-2026):** every bounded context runs as a microservice behind the API gateway
> (`09-microservices/services/01-api-gateway/` to `08-recommendation-service/`); the modular
> monolith `energy-monitor-backend` was retired.

---

## Required structure for each bounded context/module

For each bounded context (microservice), the following documentation should be maintained:

```
09-microservices/services/NN-module-name/  (or module-specific docs under 02-domain/05-architecture as appropriate)
├── README.md         ⭐ REQUIRED
├── data-model.md     ⭐ REQUIRED if data model exists
├── events.md         ⭐ REQUIRED if the module emits/consumes events
├── decisions.md      🔵 RECOMMENDED — internal technical decisions
└── runbook.md        🟢 REQUIRED before first deploy to staging
```

And its OpenAPI contract in:
```
07-api/contracts/openapi/module-name.yaml   ⭐ REQUIRED if it exposes REST endpoints
```

---

## README.md — Service technical sheet

**When to create it:** When the module is defined/documented
**Owner:** Developer assigned to the module
**Update when:** Responsibility, ports, or dependencies between modules change

Minimum content:

| Section | What it must say |
|---------|-----------------|
| Responsibility | One sentence: what it does and what data it is the authoritative owner of |
| Architecture location | Hexagonal layers (domain/application/adapter/api), DB engine, who it communicates with |
| Responsibilities (what it DOES) | List of concrete responsibilities |
| Out of scope (what it does NOT do) | What it delegated and to whom |
| How to run locally | Exact commands, must work |
| Related documents | Links to the other files of the module |

---

## data-model.md — Service data model

**When to create it:** Before the first migration script
**Owner:** Developer assigned to the module
**Update when:** A table/collection is created or modified

Minimum content:
- ER diagram (Mermaid) of the module's tables
- Description of each table with its columns, types, constraints, and purpose
- Justification of the chosen DB engine (MySQL in current implementation)
- Migration strategy (Liquibase in current implementation)

**Rule:** A field whose reason for existing is not obvious MUST have a comment in the diagram.

---

## events.md — Service event catalog

**When to create it:** When the module publishes or consumes its first domain event
**Owner:** Developer assigned to the module
**Update when:** An event is added, modified, or removed

Minimum content:
- Table of published events: name, topic/exchange, when it is emitted, schema
- Table of consumed events: name, which module it comes from, what action it triggers
- Payload schema (can reference the OpenAPI contract)

See standard event structure: `02-domain/domain-events.md`

---

## decisions.md — Service technical decisions

**When to create it:** When the team makes a non-obvious technical decision about the module
**Owner:** Whoever made the decision
**Update when:** A new decision is made or a previous one is revoked

Recommended format: miniADR (aligned with ADR-001 in `05-architecture/decisions/records/`):
```markdown
### Decision: [short name]
**Date:** [date]
**Context:** [what problem was being solved]
**Decision:** [what was decided]
**Consequences:** [known trade-offs]
```

---

## runbook.md — Service operations manual

**When to create it:** Before the first deploy to staging
**Owner:** Responsible developer + DevOps
**Update when:** A new operational issue is discovered or a procedure changes

Minimum content:
- How to verify the module/service is healthy (health check, key metrics)
- Known symptoms and their causes: "If you see X, the problem is Y, the solution is Z"
- How to perform a service/module rollback
- How to run DB migrations (Liquibase)
- Configured alerts and what to do when they fire

---

## OpenAPI Contract

**When to create it:** Before implementing the module's first endpoint (API-first)
**Owner:** Developer assigned to the module
**Update when:** An endpoint is added, modified, or removed

**API-First Rule:** The contract is written BEFORE the code. Contract tests validate
that the code fulfills the contract, not the other way around.

Use the template: `07-api/contracts/openapi/_template-service.yaml`

---

## How to add a new bounded context/module (or microservice)

1. Document the bounded context in `02-domain/` (ubiquitous language, boundaries, responsibilities)
2. For transition to microservice: create structure under `09-microservices/services/NN-module-name/`
3. Update `09-microservices/service-catalog.md` with the module's entry
4. Update `09-microservices/dependency-map.md` (or create it if it doesn't exist)
5. Create OpenAPI contract: `07-api/contracts/openapi/module-name.yaml`
6. Create a PR with at least the README.md and the sketched API contract

---

## Correlations

- Service template → `09-microservices/_template/service/` (if exists)
- Service catalog → `09-microservices/service-catalog.md`
- API contracts → `07-api/contracts/openapi/`
- General documentation rules → `00-governance/documentation-rules.md`
