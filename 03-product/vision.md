# Product Vision

> The vision is the team's north star. All sprints, design decisions,
> and trade-offs are evaluated against this vision.
> It must be ambitious yet achievable, inspiring but specific.

---

## Vision statement

> Geoffrey Moore template:

**For** residents of Colombian households who consume electricity,
**who** lack clear, timely information about how, when, and where that consumption happens,
**the** Electricity Consumption Monitor
**is a** web and mobile energy-monitoring platform
**that** shows hour-by-hour consumption, raises alerts when it goes above the household's own thresholds, and gives personalized recommendations to reduce it.
**Unlike** relying on the monthly electricity bill or the meter as the only source of truth,
**our product** turns consumption into an at-a-glance, continuously updated picture the household can act on before the bill arrives, not after.

---

## Team mission

We exist to close the gap between how much energy people use and how much they actually
understand about that usage. Discovery showed that 8 out of 10 people consider monitoring
their consumption very important, yet more than half do not track it in any way today — our
mission is to remove that gap with a tool simple enough to check in passing and precise enough
to trust.

---

## Strategic pillars

| Pillar                       | Description                                                                                                                                       | Success metrics                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Clarity**                  | Consumption data is presented as charts and trends first, not raw numbers, so any household member — technical or not — understands it in seconds | ≥ 80% of surveyed users rate the dashboard as easy to understand (WCAG-compliant, RNF4)                      |
| **Timely alerting**          | Because most users will engage through notifications rather than by opening the app on their own, alerts must be accurate and fast                | Alerts generated and delivered in real time, with response/load time < 2s (RNF1)                             |
| **Mobile-first reliability** | 87% of surveyed users would access the product from a mobile phone; the mobile experience must never be a second-class citizen to web             | Feature parity between web and mobile; < 2s load time on both                                                |
| **Trust and privacy**        | Consumption data is sensitive household information                                                                                               | Encryption in transit and at rest, secure authentication, compliance with data-protection regulations (RNF3) |
| **Grows with the household** | A project/household can have multiple linked users, devices, and a responsible user managing them (RF2)                                           | System scales horizontally as users, households, and devices grow (RNF2)                                     |

---

## High-level roadmap

> Horizon 1 (0–3 months): high certainty, detailed in HUs.
> Horizon 2 (3–6 months): medium certainty, epics.
> Horizon 3 (6–12 months): low certainty, focus areas.

```
Horizon 1 ──────────── Horizon 2 ──────────── Horizon 3
     │                       │                       │
   [MVP]              [Personalization]        [Ecosystem]
 Accounts, projects,   Smarter alerts &         Expand device
 devices, consumption  recommendations,         support, deeper
 visualization, basic  favorites, themes,       automation and
 alerts (RF1–RF4)      multi-language (RF3)     integrations (RF5+)
```

| Horizon                    | Objective                                                                            | Epics / Features                                                                                                                                                                                                                                 | Uncertainty |
| -------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| **H1 — MVP**               | Prove the core loop: register a project, link a device, see consumption, get alerted | User registration & profile (RF1); project/household registration and device linking (RF2); consumption visualization with date ranges, historical comparison, and trend charts (RF4.1–RF4.2); threshold configuration and alerts (RF3.1, RF4.3) | Low         |
| **H2 — Personalization**   | Make the experience feel tailored to each household                                  | Automatic energy-saving recommendations (RF4.4); alert channel and theme/language configuration (RF3.2–RF3.5); favorites (RF2.5)                                                                                                                 | Medium      |
| **H3 — Scale & ecosystem** | Support growth in users, households, and connected hardware                          | Device connection-state monitoring and auto-reconnection at scale (RF5); horizontal scaling and multi-region readiness (RNF2); broader device/manufacturer support                                                                               | High        |

---

## Product principles

1. **Alerts over dashboards:** discovery shows people won't watch a live dashboard all day —
   design for "tell me when something matters," not "make me check constantly."

2. **Charts before numbers:** when in doubt about how to present consumption data, default to a
   trend chart; raw tables and numbers are a secondary, not primary, view.

3. **Mobile is the default surface:** every feature is designed mobile-first and ported to web,
   never the other way around.

4. **One responsible user per household, full transparency for the rest:** permissions follow
   the _usuario responsable_ / _usuario regular_ model defined in RF2 — administration is
   concentrated, but consumption visibility is shared.

---

## Product Definition of Done

> The product is "done" for this delivery cycle when it achieves these OKRs:

**Objective:** Give Colombian households a trustworthy, mobile-first way to understand and act
on their electricity consumption.

| Key Result                                                     | Baseline                             | Target                                                 | Date      |
| -------------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------ | --------- |
| KR1: Data/graph load time                                      | Not measured (no product exists yet) | < 2 seconds (RNF1)                                     | End of H1 |
| KR2: Users who currently monitor consumption in any active way | 45.2% (survey baseline)              | ≥ 70% of onboarded users check the app at least weekly | End of H2 |
| KR3: System availability                                       | Not measured                         | ≥ 99.9% uptime (RNF5)                                  | End of H2 |
| KR4: Accessibility compliance                                  | Not measured                         | WCAG-compliant UI across web and mobile (RNF4)         | End of H1 |

---

## Correlations

- Problem framing (the why) → `03-product/problem-framing.md`
- User research backing this vision → `03-product/discovery-brief.md`
- Backlog that implements the vision → `04-requirements/user-stories.md`
- KPIs in operations → `13-operations/README.md`
