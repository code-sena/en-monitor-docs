# Discovery Brief — Electricity Consumption Monitor

> **What is this?** A record of the user research carried out before and during the
> definition of the SRS: who we talked to, what we asked, what we found, and which
> assumptions behind the product were confirmed or proven wrong by that evidence.

---

## 1. Research methodology

Two research instruments were designed for this discovery phase:

| Instrument                                         | Target audience                             | Purpose                                                                                 | Status                                                   |
| -------------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **Survey** ("Formulario de encuesta del proyecto") | Prospective end users (household residents) | Quantify how people currently track their electricity use and which features they value | **Completed** — 31 valid responses                       |
| **Interview guide** — "Entrevista a profesional"   | A software/technical professional           | Validate technical feasibility (architecture, data ingestion, alerting, scalability)    | Guide designed; **responses/transcripts not yet logged** |
| **Interview guide** — "Entrevista a aprendiz"      | An apprentice-level end user                | Explore day-to-day electricity habits and UX expectations                               | Guide designed; **responses/transcripts not yet logged** |

The survey was distributed as a Google Form and collected **31 valid responses** between
**June 9 and June 11, 2025**. Four additional rows in the raw export were spreadsheet-generated
percentage summaries, not respondents, and were excluded from the analysis below.

> **Open item:** the two interview guides (`FORMATO_ENTREVISTAS.docx`) only contain the
> question scripts — no recorded answers or notes were included in the files shared with us.
> Section 3 below documents the _intended_ scope of those conversations. Once transcripts or
> notes are available, this brief should be updated with the actual answers and the
> confirmed/invalidated lists in Section 5 re-checked against them.

---

## 2. Survey findings (n = 31)

### 2.1 Who answered

- **80.6%** live in a house (_casa_); **19.4%** live in an apartment.
- **45.2%** already monitor their electricity consumption in some way; **54.8%** do not monitor it at all today.

### 2.2 How the ones who already monitor do it

Of the 14 respondents who said they currently track consumption:

| Method                                                   | Respondents |
| -------------------------------------------------------- | ----------- |
| Check the monthly electricity bill                       | 16\*        |
| Observe the meter directly                               | 1           |
| Only checks the bill "if it looks expensive" (free-text) | 1           |

_\*Some respondents selected this option even though they had answered "no" to actively monitoring, which suggests the bill is treated as a passive, after-the-fact source rather than active monitoring._

Two free-text answers are worth quoting for context (translated from Spanish):

- _"Honestly I only check the bill if it comes out expensive or not."_
- _"I check the bill monthly and I've also looked at the meter."_

Both point to the same pattern: **today's "monitoring" is reactive and retrospective** — people
look at a number after the spending already happened, not while it is happening.

### 2.3 How important consumption monitoring is perceived to be

| Answer             | %     |
| ------------------ | ----- |
| Very important     | 80.6% |
| Somewhat important | 12.9% |
| Not interested     | 6.5%  |

### 2.4 Features people want in a monitoring app (multi-select)

| Feature                               | % of respondents who selected it |
| ------------------------------------- | -------------------------------- |
| Real-time consumption visualization   | 83.9%                            |
| High-consumption alerts               | 83.9%                            |
| Recommendations to reduce consumption | 64.5%                            |
| Comparison with previous months       | 61.3%                            |
| Monthly cost estimate                 | 58.1%                            |
| Personalized notifications            | 35.5%                            |
| Remote device control                 | 16.1%                            |

### 2.5 Preferred format for the information (multi-select)

| Format                 | %     |
| ---------------------- | ----- |
| Charts / graphs        | 64.5% |
| Numbers and statistics | 48.4% |
| Visual comparisons     | 45.2% |
| No preference          | 12.9% |

### 2.6 Device used to access the app

| Device       | %     |
| ------------ | ----- |
| Mobile phone | 87.1% |
| Computer     | 6.5%  |
| Tablet       | 6.5%  |

### 2.7 How often people would check an app like this

| Frequency                  | %     |
| -------------------------- | ----- |
| Several times a day        | 12.9% |
| Once a day                 | 32.3% |
| A few times a week         | 38.7% |
| Only when an alert arrives | 16.1% |

---

## 3. Interview scope (planned)

Two informant profiles were designed for qualitative follow-up on top of the survey:

**"Entrevista a profesional"** — aimed at a technical professional, to sanity-check the
product idea against real engineering constraints. Planned topics: perceived usefulness of the
idea, recommended architecture for hourly multi-device ingestion, database design for
per-hour/per-user/per-device data, real-time alerting mechanisms, variables that should drive
personalized recommendations, and what to consider when scaling to thousands of users/devices.

**"Entrevista a aprendiz"** — aimed at an end user, to go deeper than the survey on habits and
expectations. Planned topics: usefulness of the idea in daily life, current bill/consumption
checking habits, preference for push notifications vs. checking on demand, preferred
presentation of data (charts / numbers / simple alerts / a mix), the kind of energy-saving
advice they'd want, and what makes an app frustrating vs. easy to use.

As noted in Section 1, no recorded answers for either guide were part of the shared files.
Recommendation: log these interviews (even brief written notes per question) and fold the
findings into Section 5 below.

---

## 4. What we discovered

1. **The bill, not the app, is today's "monitoring tool."** Even among people who claim to
   monitor consumption, the dominant method is reading the monthly bill — an after-the-fact,
   low-resolution signal. Nobody described an active, ongoing tracking habit prior to using a
   tool like this.
2. **There is a clear latent-need vs. current-behavior gap.** 80.6% say monitoring is _very
   important_ to them, yet 54.8% do not monitor at all today. The barrier looks like a lack of
   a convenient tool, not a lack of interest.
3. **Visualization and alerts are the two features people want most**, both at ~84%, and by a
   wide margin over "nice-to-have" features like remote device control (16.1%).
4. **Mobile is the primary, near-exclusive access channel** (87.1%), which should drive
   design and QA priority even where a web client also exists.
5. **Engagement will mostly be alert-driven, not dashboard-driven.** Most respondents expect to
   check the app a few times a week or when notified, not continuously — reinforcing that
   alerting quality (correct thresholds, timely delivery) matters as much as the dashboard
   itself.
6. **Charts are the preferred way to consume the data** (64.5%), ahead of raw numbers, which
   supports prioritizing the trend/graph views over tabular-only views in the visualization
   feature.

---

## 5. Assumptions confirmed vs. invalidated

### 5.1 Confirmed by the evidence

| Assumption                                                        | Evidence                                                                                |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Users care about understanding their electricity consumption      | 80.6% rate it _very important_                                                          |
| Real-time visualization is a core, expected feature               | 83.9% selected it — the top feature, tied with alerts                                   |
| Consumption alerts are a core, expected feature                   | 83.9% selected it                                                                       |
| The product must be mobile-first                                  | 87.1% would access it from a mobile phone                                               |
| Users want to compare current vs. historical consumption          | 61.3% selected month-over-month comparison — this directly supports ERF4.2.2 in the SRS |
| Charts are preferred over plain numbers                           | 64.5% chose charts vs. 48.4% numbers/statistics                                         |
| Recommendations to reduce consumption are wanted, not just alerts | 64.5% selected this feature                                                             |

### 5.2 Invalidated or weaker than assumed

| Original assumption                                                            | What the data actually shows                                                                                                                                                                                            |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Users already actively monitor their consumption in some way                   | **False for the majority** — 54.8% do not monitor it at all today; the ones who do rely on the monthly bill, not active tracking                                                                                        |
| Remote control of connected devices is a priority feature                      | **Weak signal** — only 16.1% selected it, the lowest-ranked option by far; it should not be prioritized ahead of visualization, alerts, or recommendations                                                              |
| Users will open the app frequently throughout the day, similar to a social app | **Not supported** — only 12.9% expect to check it several times a day; most expect weekly/daily or alert-triggered use, which argues for a "glanceable + notify" experience rather than a constant-engagement dashboard |
| Personalized per-channel notification preferences are a top priority           | **Moderate, not top** — 35.5% wanted personalized notifications, well behind visualization/alerts, useful context for prioritizing ERF3.4 relative to ERF4.2/ERF4.3                                                     |

---

## 6. Implications for the product definition

- Prioritize **real-time visualization** and **consumption alerts** as the MVP core — both the
  survey's top-ranked features and already reflected as high-priority items in the SRS (RF4.2,
  RF4.3) and in the updated user stories.
- Treat **month-over-month/historical comparison** (RF4.2.2) as part of the MVP, not a later
  iteration — it is a top-5 requested feature, not a stretch goal.
- **Deprioritize remote device control** relative to other RF4 features; it is out of scope for
  the current SRS, which the survey data supports.
- Design the mobile experience first, and treat web as a secondary surface for the same
  functionality.
- Because most users will engage through alerts rather than by opening the app on their own,
  the accuracy and clarity of alerts (ERF4.3, ERF4.3.1) is as important to get right as the
  dashboard itself.

---

## 7. Open items

- [ ] Log the "profesional" and "aprendiz" interviews (notes or transcripts) and update
      Sections 3 and 5 with the actual answers.
- [ ] Consider a short follow-up survey specifically for _usuario responsable_ profiles
      (multi-device/household administrators), since the current sample mostly reflects a
      general household-resident perspective.
- [ ] Re-validate the "alert-driven engagement" finding once the product is in beta, using real
      usage analytics instead of self-reported frequency.
