# Traceability Matrix

> Connects every functional (ERF) and non-functional (RNF) requirement from the SRS to the HU
> that carries it and the test case that will verify it.

---

## How to use this matrix

```
Requirement (SRS) → HU → Test Case → Implementation → Service

If a requirement has no HU: it is not planned
If a HU has no test case: it has no completeness criterion
If a test case has no implementation: there is test technical debt
If there is code without an HU: possible gold-plating or a bug introduced without a story
```

---

## FR (ERF) → HU → Test matrix

| FR ID    | FR Description                               | HU  | Tests that verify it | Status  |
| -------- | -------------------------------------------- | --- | -------------------- | ------- |
| ERF1.1   | Registration and authentication (web/mobile) | HU1 | TC-HU1-01            | Pending |
| ERF1.1.1 | Social login                                 | HU1 | TC-HU1-02            | Pending |
| ERF1.2   | Log in                                       | HU1 | TC-HU1-03            | Pending |
| ERF1.3.1 | Update profile information                   | HU1 | TC-HU1-04            | Pending |
| ERF1.3.2 | Recover password                             | HU1 | TC-HU1-05            | Pending |
| ERF1.3.3 | Delete account                               | HU1 | TC-HU1-06            | Pending |
| ERF2.1   | Register a project or home                   | HU2 | TC-HU2-01            | Pending |
| ERF2.2   | User permission management by role           | HU2 | TC-HU2-02            | Pending |
| ERF2.3   | Link devices and users to a project          | HU2 | TC-HU2-03            | Pending |
| ERF2.4   | Responsible user per project                 | HU2 | TC-HU2-04            | Pending |
| ERF2.5   | Mark projects/homes as favorites             | HU2 | TC-HU2-05            | Pending |
| ERF3.1   | Configure consumption thresholds             | HU3 | TC-HU3-01            | Pending |
| ERF3.2   | Configure colors                             | HU3 | TC-HU3-02            | Pending |
| ERF3.3   | Configure language                           | HU3 | TC-HU3-03            | Pending |
| ERF3.4   | Configure alert channel                      | HU3 | TC-HU3-04            | Pending |
| ERF3.5   | Global system configuration                  | HU3 | TC-HU3-05            | Pending |
| ERF4.1   | Automatic consumption recording              | HU4 | TC-HU4-01            | Pending |
| ERF4.2.1 | Date range selection                         | HU4 | TC-HU4-02            | Pending |
| ERF4.2.2 | Consumption comparison                       | HU4 | TC-HU4-03            | Pending |
| ERF4.2.3 | Consumption trend charts                     | HU4 | TC-HU4-04            | Pending |
| ERF4.3   | Automatic alert generation                   | HU5 | TC-HU5-01            | Pending |
| ERF4.3.1 | Alert visualization and resolution           | HU5 | TC-HU5-02            | Pending |
| ERF4.4   | Automatic recommendation generation          | HU5 | TC-HU5-03            | Pending |
| ERF4.4.1 | Recommendation visualization / mark as read  | HU5 | TC-HU5-04            | Pending |
| ERF5.1   | Device data transmission (60s / MQTT)        | HU6 | TC-HU6-01            | Pending |
| ERF5.2   | Device linking and authentication            | HU6 | TC-HU6-02            | Pending |
| ERF5.3   | Device connection status                     | HU6 | TC-HU6-03            | Pending |
| ERF5.4   | Automatic reconnection                       | HU6 | TC-HU6-04            | Pending |

---

## NFR (RNF) → HU → Validation matrix

| NFR ID  | Description                                                       | HU          | How it is validated                        | Status               |
| ------- | ----------------------------------------------------------------- | ----------- | ------------------------------------------ | -------------------- |
| ERNF1.1 | Response time < 2s, stable throughput under concurrent load       | HU7         | Load test, concurrent-user simulation      | Pending              |
| ERNF2.1 | Horizontal scaling under user/device/data growth                  | HU8         | Autoscaling / task redistribution test     | Pending              |
| ERNF3.1 | Encryption in transit/at rest, secure auth, regulatory compliance | HU9         | Security review, penetration/lockout test  | Pending              |
| RNF4.1  | Usability, accessibility (WCAG), contextual help                  | HU10        | Manual accessibility and usability review  | Pending              |
| RNF5.1  | Availability (e.g. 99.9%)                                         | _No HU yet_ | Uptime monitoring                          | ⏸ Not yet in backlog |
| ERNF6.1 | Ease of updating/fixing the system                                | _No HU yet_ | Manual review of deployment/change process | ⏸ Not yet in backlog |
| ERNF7.1 | Cross-platform portability                                        | _No HU yet_ | Manual verification on target OS/browsers  | ⏸ Not yet in backlog |

> RNF5–RNF7 have no HU in the current document — see the open item in
> `04-requirements/non-functional.md`. They are listed here so the gap is visible in the matrix,
> not silently dropped.

---

## Inverse traceability: HU → FR/NFR

| HU   | Title                                          | Requirements it covers                                 |
| ---- | ---------------------------------------------- | ------------------------------------------------------ |
| HU1  | Registration and Account Management            | ERF1.1, ERF1.1.1, ERF1.2, ERF1.3.1, ERF1.3.2, ERF1.3.3 |
| HU2  | Project and Home Management                    | ERF2.1, ERF2.2, ERF2.3, ERF2.4, ERF2.5                 |
| HU3  | System Configuration and Personalization       | ERF3.1, ERF3.2, ERF3.3, ERF3.4, ERF3.5                 |
| HU4  | Electricity Consumption Query and Analysis     | ERF4.1, ERF4.2.1, ERF4.2.2, ERF4.2.3                   |
| HU5  | Alert Management and Automatic Recommendations | ERF4.3, ERF4.3.1, ERF4.4, ERF4.4.1                     |
| HU6  | Measuring Device Management                    | ERF5.1, ERF5.2, ERF5.3, ERF5.4                         |
| HU7  | System Performance                             | ERNF1.1                                                |
| HU8  | System Scalability                             | ERNF2.1                                                |
| HU9  | Information Security                           | ERNF3.1                                                |
| HU10 | Usability and Accessibility                    | RNF4.1                                                 |

---

## Identified gaps

| Gap type                  | Description                                                                                                 | Required action                                                                                | Owner |
| ------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ----- |
| RNF without HU            | RNF5 (Availability), RNF6 (Maintainability), RNF7 (Portability) are in the SRS but have no corresponding HU | Decide whether to add HU11–HU13, or treat them as implicit quality bars not tracked as stories | Team  |
| HU without test           | All 10 HUs have a planned test case ID but no implementation yet                                            | Write tests alongside each HU as development starts                                            | Team  |
| RNF without detail in SRS | RNF5–RNF7 only have a one-line description in the SRS, unlike RNF1–RNF4's full tables                       | Detail them in the SRS first if they are going to be built against, not just documented        | Team  |

---

## Correlations

- User Stories → `04-requirements/user-stories.md`
- Functional Requirements → `04-requirements/functional.md`
- Non-Functional Requirements → `04-requirements/non-functional.md`
- Source SRS → `SRS.docx`
