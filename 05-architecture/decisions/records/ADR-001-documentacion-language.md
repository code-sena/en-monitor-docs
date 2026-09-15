# ADR-001 — Documentation Language

| Field | Value |
|-------|-------|
| **ID** | ADR-001 |
| **Date** | 2026-09-10 |
| **Status** | Accepted |
| **Authors** | Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |
| **Reviewers** | N/A — two-person team, decided jointly |

---

## Context

The software industry standard — Stack Overflow, library documentation, technical articles,
frameworks, and open-source tooling — operates in English. Mixing languages across artifacts
(Spanish docs, English code, Spanish API descriptions) creates a cognitive translation
boundary that slows onboarding, increases errors in naming, and makes it harder to search
for help online.

A single, clear rule established from day one prevents inconsistencies: mismatched variable
names, Spanish comments in English code, and conflicting terminology between documentation
and implementation.

**This is a separate decision from the interface's own language support (RF3.3, up to 4 UI
languages for end users).** That is a product feature the system offers to the people using it.
This ADR only covers the language the team itself writes documentation and code in — the two
do not need to match, and neither one determines the other.

**Known constraints:**
- Two-person team, both fluent enough in English to work in it day to day
- SENA academic evaluation may involve reviewers more comfortable in Spanish

---

## Evaluated alternatives

### Alternative A — Everything in English (CHOSEN)
- **Pros:** Industry standard; libraries and frameworks are in English; technical reference documentation is in English; eliminates the translation boundary between docs and code; GitHub, Stack Overflow, and AI tools all work best with English content
- **Cons:** May require slightly more effort from team members who are not native English speakers

### Alternative B — Everything in Spanish
- **Pros:** Natural communication for a project whose end users and academic evaluators are Spanish-speaking; business domain names preserved exactly
- **Cons:** Uncomfortable mix with language keywords (if, for, return, etc.); inconsistent with the library ecosystem; harder to search for help online

### Alternative C — Split by layer (discarded)
- **Pros:** Each artifact uses the most natural language for its audience
- **Cons:** Requires strict discipline and explicit rules; creates a permanent translation boundary between docs and code; domain terms accumulate two canonical names — this project already saw this risk firsthand with terms like `Home` (from "hogar") before the glossary was established

---

## Decision

**Alternative A:** Use English for all documentation and code.

| Artifact | Language | Reason |
|----------|----------|--------|
| Variables, functions, classes in code | English | Consistency with libraries and frameworks |
| Table and column names in DB | English | Coherence with the code that maps them |
| Commits (Conventional Commits) | English | Established standard, readable on GitHub |
| Git branch names | English | Consistent with commits |
| Markdown documentation | English | Eliminates the translation boundary; searchable |
| End-user error messages | Localized (up to 4 languages, per RF3.3) | The localization layer handles language at runtime — unrelated to this decision |
| Internal system logs | English | Facilitates search in library documentation and alerts |
| ADRs and technical documentation | English | Single source of truth, no translation boundary |

---

## Consequences

**Positive:**
- A single language across all artifacts eliminates the cognitive translation boundary
- External contributors and AI tools work without friction
- Domain terms have one canonical form (the English one in the code), tracked in the glossary

**Negative / Trade-offs:**
- Some business terms require a deliberate translation decision instead of using the Spanish
  term directly (e.g. `Home` for "hogar", `Threshold` for "umbral")
- Source material produced in Spanish before this framework existed creates a translation
  boundary against it — mitigated by treating that material as historical reference once it
  conflicts with a documented decision, not as an ongoing source of truth to keep translating

**Impact on the system:**
- Affected: every service, every documentation folder
- Documents that must stay aligned with this decision: `01-context/glossary.md` (canonical
  English term for each business concept)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A term gets used with two different English translations in different documents | Medium | Medium | Check the glossary before introducing a new term; update it immediately if a gap is found |
---

## References

- Domain term glossary → `01-context/glossary.md`
- Team documentation conventions → `00-governance/documentation-rules.md` *(path not yet
  confirmed to exist in this repository — verify before relying on it)*