# Design Decisions — Device Service

> Decisions taken for the extraction of the Devices context (source: `docs/Architecture.md` of
> `en-monitor-device-service`). Platform decisions (shared schema, event transport, gateway, MQTT
> broker) are the ADRs in `05-architecture/decisions/records/`. The extraction ran from 22 to
> 24 October 2026.

> **Update (28-Oct-2026):** these decisions describe the extraction from the modular monolith as it
> was done on 24-Oct-2026. The monolith's other modules have since become services of their own
> (`05-home-service` … `08-recommendation-service`) and `energy-monitor-backend` was retired, so every
> mention of "the backend" below means what the backend was at that time; its proxy packages no
> longer exist.

---

## [DEC-001] Full migration, with a cached proxy left in the backend

**Status:** Active

**Decision:** endpoints, broker authentication, commands, connectivity and their tables moved
here; `energy-monitor-backend` keeps a `device` proxy package (`device::api`) answered by this
service's internal API.

**Consequences:**
- ✅ The backend modules that use devices (`alert`, `recommendation`, `notification`) did not change
- ⚠️ Device reads cross the network (backend cache: 30 s, unknown devices included, stale copy up to 1 h)

---

## [DEC-002] Connectivity from its own MQTT subscriptions

**Status:** Active

**Decision:** this service subscribes to telemetry and status with its own broker user, so device
state does not depend on any other service.

---

## [DEC-003] Home deletion as an event

**Status:** Active

**Decision:** `home.deleted` arrives from the backend's outbox; there is no synchronous call to
unlink a home's modules.

**Consequences:**
- ✅ Deleting a home never waits for this service
- ⚠️ The modules are unlinked shortly after, not in the same transaction

---

## [DEC-004] Mosquitto depends on this service

**Status:** Active (accepted trade-off)

**Decision:** the broker stores no users; go-auth asks this service for every login and ACL check.

**Consequences:**
- ✅ One source of truth for device credentials (hashed API keys)
- ⚠️ While this service is down, new MQTT logins are refused and connected modules publish only
  while the broker's cached decisions last (5 minutes)
