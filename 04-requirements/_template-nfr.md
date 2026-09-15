# RNF[N]: [Category Name]

> **ID convention in this project:** `RNF[N]` / `ERNF[N].x`, matching the numbering in `SRS.docx`,
> section "Requerimientos No Funcionales." Each NFR is tagged with the Historia de Usuario (HU)
> that tracks it, if one exists — see `04-requirements/non-functional.md` for the current list,
> including RNF5–RNF7, which exist in the SRS but have no HU yet.
>
> **Worked example below** uses a real requirement from this project (`RNF1` — Performance) so the
> template shows a fully specified NFR, not just bracketed placeholders. Copy this file, replace
> the content with your own non-functional requirement, and delete this note.

---

## Requirement

| ID      | Requirement                              | Metric                                                                                                               | How to verify                                                                              |
| ------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| ERNF1.1 | Response time, load capacity, throughput | Data/graph loading response time < 2 seconds; supports multiple concurrent users while maintaining stable throughput | Load tests, concurrent-user simulations (per SRS, "Documentos de visualización asociados") |

A metric that cannot be measured is not a non-functional requirement yet — it is a wish. If you
cannot fill in "Metric" with a number, threshold, or pass/fail condition, the requirement needs
more work before it belongs in this folder. See `04-requirements/README.md`'s "Common mistakes to
avoid": _"The system must be fast"_ is not verifiable; _"p95 < 200ms"_ is.

---

## Applies to

Which functional requirements or services this NFR constrains — an NFR with no stated scope
silently ends up applied (or ignored) inconsistently across services.

**Example:** RF2 (Project Management), RF5 (Device Management), per the SRS's own
"Especialización" field for this requirement.

---

## Abnormal-situation handling

What the system does when this NFR is violated at runtime — not just the target, but the
fallback behavior when the target is missed.

**Example (from the SRS):**

- Performance degradation → activate load balancing or scale services
- Congestion → notify the user and retry automatically

---

## Architectural backing

Which decision in `05-architecture/` actually delivers this NFR. **An NFR with nothing in this
section is a requirement no design decision backs yet** — flag it in
`04-requirements/traceability-matrix.md`'s "Identified gaps" table rather than leaving the
connection implicit. (The worked example below is left blank on purpose: as of this writing, no
document in `05-architecture/` specifies load balancing, autoscaling, or a concurrency target for
`ERNF1.1` — this gap is real, not merely illustrative.)

---

## Source HU and priority

| Field     | Value                    |
| --------- | ------------------------ |
| Source HU | HU7 (System Performance) |
| Priority  | High                     |
| Status    | Backlog                  |

---

## Correlations

- Full NFR list this template's entries join → `04-requirements/non-functional.md`
- Coverage tracking, including the "no architectural backing" gap type this template's
  "Architectural backing" section exists to prevent → `04-requirements/traceability-matrix.md`
- Functional counterparts → `04-requirements/functional.md`
- User Stories this NFR is tracked under → `04-requirements/user-stories.md`
