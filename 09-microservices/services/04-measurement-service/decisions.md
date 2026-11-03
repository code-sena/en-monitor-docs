# Design Decisions — Measurement Service

> Decisions taken for the extraction of the Monitoring and Measurements context (source:
> `README.md` of `en-monitor-measurement-service` and `docs/Architecture.md` of
> `energy-monitor-backend`). The service's first commits are from 24 October 2026.

> **Update (28-Oct-2026):** these decisions describe the extraction from the modular monolith as it
> was done on 24-Oct-2026. The monolith's other modules have since become services of their own
> (`05-home-service` … `08-recommendation-service`) and `energy-monitor-backend` was retired, so every
> mention of "the backend" below means what the backend was at that time; its proxy packages no
> longer exist.

---

## [DEC-001] The only subscriber that stores telemetry

**Status:** Active

**Decision:** this service subscribes to the telemetry with a read-only MQTT user and stores every
reading; the alert service hears readings as
`measurement.recorded`.

**Consequences:**
- ✅ A reading is stored exactly once in the shared table
- ⚠️ The subscription is off by default (`MEASUREMENT_MQTT_ENABLED=false`) and must be turned on
  in exactly one running instance

---

## [DEC-002] Every reading announced through the outbox

**Status:** Active

**Decision:** one `measurement_outbox` row per reading, written in the reading's transaction;
delivered rows purged after 7 days.

**Consequences:**
- ✅ The alerts see every stored reading, even after a broker outage
- ⚠️ The outbox is the highest-volume outbox of the system, hence the purge

---

## [DEC-003] Home access and limits asked to the home service, devices to the device service

**Status:** Active

**Decision:** membership is asked on every request and never cached (a removed member loses
access at once); devices are cached 30 s with a stale copy up to 1 h.

---

## [DEC-004] Same paths, same table

**Status:** Active

**Decision:** the gateway routes the existing paths here (`/api/v1/measurements/**`,
`/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**`), and the changelogs keep
their monolith paths, so the clients and the reading history did not change.
