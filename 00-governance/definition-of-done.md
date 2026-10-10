# Definition of Done (DoD)

> A User Story is **DONE** when it meets ALL criteria on this checklist.
> If even one is missing, the story is NOT done — it goes back to In Progress.

## Mandatory checklist

### Code
- [ ] Code implements all acceptance criteria of the user story
- [ ] Code was reviewed and approved by at least 1 team member (PR review)
- [ ] Code follows project standards (linting and formatting pass in CI)
- [ ] No technical debt introduced without registering it in `15-project-control/technical-backlog.md`

### Tests
- [ ] Unit tests written for new business logic
- [ ] Test coverage does not decrease from the project baseline
- [ ] All tests pass locally and in CI
- [ ] Acceptance criteria verified (manual or automated)

### Integration
- [ ] Changes do not break other bounded contexts/modules (integration/modulith tests pass)
- [ ] If API changes: OpenAPI contract updated in `07-api/contracts/` (or per module documentation)
- [ ] If data model changes: bounded context/module documentation updated (e.g., `data-model.md`)
- [ ] If new/modified events: event catalog updated

### Deployment
- [ ] Code is mergeable to `dev` (no conflicts)
- [ ] Deployed to staging environment (when applicable)
- [ ] Basic smoke test passing on staging (when applicable)

### Documentation
- [ ] Module/bounded context `README.md` or relevant documentation updated if the public interface changed
- [ ] If a significant technical decision was made: ADR created or updated in `05-architecture/decisions/records/`

---

## Allowed exceptions

The following exceptions must be explicitly agreed to by the Tech Lead:
- E2E tests omitted due to environment limitations (document the risk)
- Documentation deferred for urgent delivery (create a tech-debt ticket)

---

## What is NOT a Done criterion

- "The code is on my machine" — it must be in the repository
- "It works on my local environment" — it must work on staging
- "The PM/PO approved it" — that is the product Definition of Done, not the code's
