# ADRs — Architecture Decision Records

ADRs document important architectural decisions. Each file = one decision.

## How to create an ADR

1. Copy `_template-adr.md`
2. Name it `ADR-NNN-short-title.md` (e.g.: `ADR-001-message-broker.md`)
3. Fill it in completely — especially the evaluated alternatives
4. Once accepted, the status is **permanent** (it is not deleted, it is "Superseded" by another ADR)

## Possible statuses

- `Proposed` — under discussion
- `Accepted` — approved by the team
- `Rejected` — evaluated and discarded (document why)
- `Superseded` — superseded by ADR-NNN (indicate which one)

## ADR register

| #       | Title                                     | Status     | Date       |
| ------- | ------------------------------------------ | ---------- | ---------- |
| ADR-001 | Documentation Language                     | Accepted   | 2026-09-10 |
| ADR-002 | Domain Event Transport (Kafka)             | Superseded by ADR-008 | 2026-09-11 |
| ADR-003 | Shared Physical Database                   | Accepted   | 2026-09-10 |
| ADR-004 | API Gateway Selection                      | Accepted   | 2026-09-11 |
| ADR-005 | Authentication Strategy                    | Superseded by ADR-010 | 2026-09-10 |
| ADR-006 | Database Migrations Repository Structure   | Superseded by ADR-009 | 2026-09-10 |
| ADR-007 | MQTT Broker Selection                      | Accepted   | 2026-09-11 |
| ADR-008 | Event Transport: RabbitMQ with a Transactional Outbox | Accepted | 2026-10-20 |
| ADR-009 | Liquibase Changelogs per Service, One Shared Changelog Table | Accepted | 2026-10-20 |
| ADR-010 | Session Tokens and the Web Session in HttpOnly Cookies | Accepted | 2026-10-24 |
| ADR-011 | MQTT Broker Authentication Delegated to the Device Context | Accepted | 2026-10-08 |
| ADR-012 | Internal REST API Between Services | Accepted | 2026-10-20 |
| ADR-013 | Modular Monolith First, Incremental Extraction to Microservices | Accepted (carried out 2026-10-28) | 2026-09-23 |

> Implementation status at 24-Oct-2026: every ADR in force (001, 003, 004, 007 to 013) is implemented
> and in production. ADR-002, ADR-005 and ADR-006 stay in `records/` with status "Superseded",
> unchanged below a one-line note that points to the ADR that replaced them.

> Add rows here as you create ADRs in `records/`. Keep the table in numeric order, not commit
> order — it's a lookup table, not a changelog.