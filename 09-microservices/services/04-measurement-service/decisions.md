# Design Decisions — Measurement Service

> Decisions taken for the extraction of the Monitoring and Measurements context (source:
> `README.md` of `en-monitor-measurement-service` and `docs/Architecture.md` of
> `energy-monitor-backend`). The service's first commits are from 24 October 2026.

---

## [DEC-001] The only subscriber that stores telemetry

**Status:** Active

**Decision:** this service subscribes to the telemetry with a read-only MQTT user and stores every
reading; `energy-monitor-backend` is no longer an MQTT client and hears readings as
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
- ✅ The backend's alerts see every stored reading, even after a broker outage
- ⚠️ The outbox is the highest-volume outbox of the system, hence the purge

---

## [DEC-003] Home access and limits asked to the backend, devices to the device service

**Status:** Active

**Decision:** membership is asked on every request and never cached (a removed member loses
access at once); devices are cached 30 s with a stale copy up to 1 h.

---

## [DEC-004] Same paths, same table

**Status:** Active

**Decision:** the gateway routes the existing paths here (`/api/v1/measurements/**`,
`/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**`), and the changelogs keep
their monolith paths, so the clients and the reading history did not change.
