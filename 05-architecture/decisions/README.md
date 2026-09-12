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
| ADR-002 | Domain Event Transport                     | Accepted   | 2026-09-11 |
| ADR-003 | Shared Physical Database                   | Proposed   | 2026-09-10 |
| ADR-004 | API Gateway Selection                      | Accepted   | 2026-09-11 |
| ADR-005 | Authentication Strategy                    | Proposed   | 2026-09-10 |
| ADR-006 | Database Migrations Repository Structure   | Proposed   | 2026-09-10 |
| ADR-007 | MQTT Broker Selection                      | Accepted   | 2026-09-11 |

> Add rows here as you create ADRs in `records/`. Keep the table in numeric order, not commit
> order — it's a lookup table, not a changelog.