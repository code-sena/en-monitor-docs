# ADR-011 — MQTT Broker Authentication Delegated to the Device Context

- **ID:** ADR-011
- **Date:** 2026-10-08
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

`ADR-007-mqtt-broker-selection.md` chose Eclipse Mosquitto but did not decide how the broker knows
who may connect and which topics each client may use. Every measuring module receives its own
credentials when it is linked to a home (`device_code` and an API key), keys are reissued when a
module changes Wi-Fi network, and they are revoked when it is unlinked or its home is deleted. The
broker has to follow those changes at once, and a module must never publish as another one.

Besides the modules, platform services connect to the broker to read telemetry and to send
commands to the modules.

**Known constraints:**
- Credentials are created and revoked by the Devices context at runtime; the broker cannot be
  reloaded every time a module is linked
- API keys are stored only as a SHA-256 hash
- The broker runs in the same Docker network as the services (`energy-monitor-infra`)

---

## Decision

**We decided:** the broker stores no users. Mosquitto runs with the `mosquitto-go-auth` plugin in
HTTP mode and asks the Devices context, for every connection and topic access, whether it is
allowed.

- Endpoints: `POST /internal/mqtt/auth/user` and `POST /internal/mqtt/auth/acl`, answered first by
  `energy-monitor-backend` (2026-10-08) and, since the device extraction (2026-10-22), by
  `en-monitor-device-service` on port 8084. Only the Docker network and loopback may call them.
- The plugin caches each decision 5 minutes, so a module publishing every minute does not cost one
  HTTP call per message.
- Clients and what they may do:

| Client | Credentials | Access |
|---|---|---|
| `en-monitor-device-service` | `MQTT_DEVICE_SERVICE_USER/PASS` | Read telemetry and status of every module; the only client that writes commands |
| `en-monitor-measurement-service` | `MQTT_MEASUREMENT_SERVICE_USER/PASS` | Read telemetry of every module, nothing else |
| Each ESP32 module | `device_code` + API key | Publish its own `telemetry` and `status`; read its own `command` |

**Justification:** the context that issues and revokes the credentials is the only one that can
answer, without delay, whether a key is still valid; asking it keeps one source of truth and
makes a revocation effective as soon as the cached decision expires. The ACL by client type is
what prevents a module from publishing on another module's topics.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **go-auth HTTP backend answered by the Devices context (chosen)** | One source of truth; immediate effect of linking and revocation; per-client ACL in code with tests | The broker depends on the device service being up | — (chosen) |
| Mosquitto password and ACL files | No dependency at runtime | Files must be rewritten and the broker reloaded on every link, reissue and unlink | Does not follow runtime changes |
| go-auth reading the database directly | No HTTP hop | Couples the broker to the schema and the hashing scheme of the device tables, against table ownership (`ADR-003`) | Breaks ownership of the device tables |
| Anonymous access with topic conventions | Simplest | Any client could publish as any module | Insecure |

---

## Consequences

**Positive:**
- Linking, reissuing and unlinking a module take effect without touching the broker
- The ACL rules live in code with unit tests (`BrokerAccessService`)
- The backend stopped being a broker client once the measurement service took the readings

**Negative / Trade-offs:**
- While the device service is down, new MQTT logins are refused; connected modules keep publishing
  only while their cached decisions last (5 minutes)
- A revoked key may keep working for up to 5 minutes (the cache)

**Impact on the system:**
- Affected services: `en-monitor-device-service`, `en-monitor-measurement-service`, Mosquitto in
  `energy-monitor-infra`
- Documents updated: `05-architecture/overview.md`, `09-microservices/services/03-device-service/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| The device service is down and modules cannot reconnect | Low | High | Health checks and automatic rollback in `deploy.sh`; the decision cache bridges short outages |
| Someone outside the Docker network calls the auth endpoints | Low | High | Allowed networks only; the gateway never forwards `/internal/**` |

---

## References

- Broker selection → `ADR-007-mqtt-broker-selection.md`
- Internal endpoints between services → `ADR-012-internal-service-api.md`
- Code: `BrokerAccessService` and README of `en-monitor-device-service`;
  `mosquitto/mosquitto.conf` of `energy-monitor-infra`
