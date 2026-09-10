# Discovery Brief — Energy Monitor

## 1. Research methodology

| Instrument                                     | Target audience                             | Purpose                                                                           | Status                                               |
| ---------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Survey ("Formulario de encuesta del proyecto") | Prospective end users (household residents) | Quantify how people currently track electricity use and which features they value | Completed — 31 valid responses                       |
| Interview guide — "Entrevista a profesional"   | A technical/software professional           | Validate technical feasibility (architecture, ingestion, alerting, scalability)   | Guide designed; responses/transcripts not yet logged |
| Interview guide — "Entrevista a aprendiz"      | An apprentice-level end user                | Explore day-to-day electricity habits and UX expectations                         | Guide designed; responses/transcripts not yet logged |

**Fieldwork window:** June 9 – June 11, 2025 (survey)
**Sample size:** 31 valid responses (4 additional rows in the raw export were
spreadsheet-generated percentage summaries, not respondents, and were excluded). Self-selected
sample of prospective household users; not segmented by home role (Owner vs. Member).

## 2. Findings

### 2.1 Housing type and current monitoring habits

| Answer                                     | %     |
| ------------------------------------------ | ----- |
| Lives in a house                           | 80.6% |
| Lives in an apartment                      | 19.4% |
| Currently monitors consumption in some way | 45.2% |
| Does not monitor consumption at all        | 54.8% |

**What this means:** more than half of respondents do not track their consumption today, even
though (see 2.2) the large majority consider it very important — the gap is a tooling gap, not
an interest gap.

### 2.2 Perceived importance of monitoring

| Answer             | %     |
| ------------------ | ----- |
| Very important     | 80.6% |
| Somewhat important | 12.9% |
| Not interested     | 6.5%  |

### 2.3 How people who already monitor do it

Of the respondents who said they currently track consumption, the dominant method is reading
the monthly electricity bill, with a smaller number observing the physical meter directly. Two
free-text answers illustrate the pattern well:

- _"Honestly I only check the bill if it comes out expensive or not."_
- _"I check the bill monthly and I've also looked at the meter."_

**What this means:** today's "monitoring" is reactive and after-the-fact, not an ongoing habit.

### 2.4 Desired features (multi-select)

| Feature                               | %     |
| ------------------------------------- | ----- |
| Real-time consumption visualization   | 83.9% |
| High-consumption alerts               | 83.9% |
| Recommendations to reduce consumption | 64.5% |
| Comparison with previous months       | 61.3% |
| Monthly cost estimate                 | 58.1% |
| Personalized notifications            | 35.5% |
| Remote device control                 | 16.1% |

### 2.5 Preferred data format (multi-select)

| Format                 | %     |
| ---------------------- | ----- |
| Charts / graphs        | 64.5% |
| Numbers and statistics | 48.4% |
| Visual comparisons     | 45.2% |
| No preference          | 12.9% |

### 2.6 Access device

| Device       | %     |
| ------------ | ----- |
| Mobile phone | 87.1% |
| Computer     | 6.5%  |
| Tablet       | 6.5%  |

### 2.7 Expected check-in frequency

| Frequency                  | %     |
| -------------------------- | ----- |
| Several times a day        | 12.9% |
| Once a day                 | 32.3% |
| A few times a week         | 38.7% |
| Only when an alert arrives | 16.1% |

## 3. Interviews conducted

**"Entrevista a profesional"** — technical professional. Planned topics: perceived usefulness
of the idea, recommended architecture for hourly multi-device ingestion, database design for
per-hour/per-user/per-device data, real-time alerting mechanisms, variables that should drive
personalized recommendations, and scaling considerations.
Status: guide designed, not yet conducted / transcripts not included in shared files.

**"Entrevista a aprendiz"** — end user. Planned topics: usefulness in daily life, current
bill-checking habits, preference for push notifications vs. on-demand checking, preferred data
presentation, desired type of energy-saving advice, and what makes an app frustrating vs. easy
to use.
Status: guide designed, not yet conducted / transcripts not included in shared files.

## 4. What we discovered

1. The electricity bill, not an app, is today's de facto "monitoring tool" — an after-the-fact,
   low-resolution signal, not an active habit.
2. There is a clear latent-need vs. current-behavior gap: 80.6% say monitoring matters a lot,
   yet 54.8% do not monitor at all today.
3. Visualization and alerts are the two most wanted features (~84% each), far ahead of
   "nice-to-have" features like remote device control (16.1%).
4. Mobile is the primary, near-exclusive access channel (87.1%).
5. Engagement will mostly be alert-driven, not dashboard-driven — most respondents expect to
   check the app a few times a week or when notified, not continuously.
6. Charts are the preferred way to consume the data (64.5%), ahead of raw numbers.

## 5. Assumptions confirmed vs. invalidated

### 5.1 Confirmed by the evidence

| Assumption                                                        | Evidence                                   |
| ----------------------------------------------------------------- | ------------------------------------------ |
| Users care about understanding their electricity consumption      | 80.6% rate it very important               |
| Real-time visualization is a core, expected feature               | 83.9% selected it                          |
| Consumption alerts are a core, expected feature                   | 83.9% selected it                          |
| The product must be mobile-first                                  | 87.1% would access it from a mobile phone  |
| Users want to compare current vs. historical consumption          | 61.3% selected month-over-month comparison |
| Charts are preferred over plain numbers                           | 64.5% chose charts vs. 48.4% numbers       |
| Recommendations to reduce consumption are wanted, not just alerts | 64.5% selected this feature                |

### 5.2 Invalidated or weaker than assumed

| Original assumption                                                  | What the data actually shows                                      |
| -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Users already actively monitor their consumption in some way         | False for the majority — 54.8% do not monitor it at all today     |
| Remote control of connected devices is a priority feature            | Weak signal — only 16.1% selected it, the lowest-ranked option    |
| Users will open the app frequently throughout the day                | Not supported — only 12.9% expect to check it several times a day |
| Personalized per-channel notification preferences are a top priority | Moderate, not top — 35.5%, well behind visualization/alerts       |

## 6. Implications for the product definition

- Prioritize real-time visualization and consumption alerts as the MVP core (confirmed as the
  two top-ranked features).
- Treat month-over-month comparison as part of the MVP, not a later iteration.
- Deprioritize remote device control relative to visualization, alerts, and recommendations.
- Design mobile-first; treat web as a secondary surface for the same functionality.
- Invest as much in alert accuracy and delivery as in the dashboard itself, since most
  engagement will be alert-driven.

## 7. Open items

- [ ] Log the "profesional" and "aprendiz" interviews (notes or transcripts) and update Sections
      3 and 5 with the actual answers.
- [ ] Run a follow-up survey specifically for Home Owner profiles (multi-device/household
      administrators), since the current sample mostly reflects a general resident perspective.
- [ ] Re-validate the "alert-driven engagement" finding once the product is in beta, using real
      usage analytics instead of self-reported frequency.

---

## Correlations

- Problem this research validates → `problem-framing.md`
- Vision this research informs → `vision.md`
- Backlog prioritization affected by these findings → `product-backlog.md`
