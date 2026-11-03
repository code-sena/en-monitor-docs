# Local Project Setup

> This document ensures that any developer can have a functional local environment
> in less than 1 hour. If it takes longer, the document is incomplete.
> **Rule:** If you have to guess something or look elsewhere, add what's missing here.
>
> Source of truth for the commands: the README of the `energy-monitor-infra` repository
> (state at 28-Oct-2026).

---

## Prerequisites

| Tool | Minimum version | Verify with | Notes |
|------|----------------|-------------|-------|
| Docker Engine | 24+ | `docker --version` | Tested with Docker 29 |
| Docker Compose plugin | 2.24+ | `docker compose version` | Development mode uses the `!override` tag |
| Git | — | `git --version` | |
| `openssl` | — | `openssl version` | Used by `prepare.sh` and to generate secrets |
| Java | 21 | `java -version` | Only for development mode; each service brings `./mvnw` (no Maven install needed) |
| Node.js | — | `node --version` | Only to run the web app with Vite in development mode |

---

## Repositories

Clone them side by side (other paths: set `BACKEND_CONTEXT`, `SECURITY_CONTEXT`,
`SECURITY_CONTEXT`, `DEVICE_CONTEXT`, `MEASUREMENT_CONTEXT`, `HOME_CONTEXT`, `NOTIFICATION_CONTEXT`, `ALERT_CONTEXT`, `RECOMMENDATION_CONTEXT`, `GATEWAY_CONTEXT` and `FRONTEND_CONTEXT` in `infra/.env`):

```bash
mkdir energy-monitor && cd energy-monitor
git clone https://github.com/Juan-Cortes2045/energy-monitor-infra.git infra
git clone https://github.com/Juan-Cortes2045/en-monitor-security-service.git backend/en-monitor-security-service
git clone https://github.com/Juan-Cortes2045/en-monitor-device-service.git backend/en-monitor-device-service
git clone https://github.com/Juan-Cortes2045/en-monitor-measurement-service.git backend/en-monitor-measurement-service
git clone https://github.com/angelgarcia0/en-monitor-home-service.git backend/en-monitor-home-service
git clone https://github.com/angelgarcia0/en-monitor-notification-service.git backend/en-monitor-notification-service
git clone https://github.com/angelgarcia0/en-monitor-alert-service.git backend/en-monitor-alert-service
git clone https://github.com/angelgarcia0/en-monitor-recommendation-service.git backend/en-monitor-recommendation-service
git clone https://github.com/Juan-Cortes2045/en-monitor-api-gateway.git backend/en-monitor-api-gateway
git clone https://github.com/Juan-Cortes2045/Energy-Monitor-application.git frontend/energy-monitor-web
```

The ESP32 firmware lives in the retired `energy-monitor-backend` repository (`firmware/`), which is
cloned only for it: `git clone https://github.com/Juan-Cortes2045/energy-monitor-backend.git backend/energy-monitor-backend`. Each code repository
keeps its own `Dockerfile`; infra builds the images from there.

---

## Three Compose modes

| Mode | Files | Use |
|---|---|---|
| Full local stack | `docker-compose.yml` | The whole system on this PC (demo, end-to-end tests) |
| Development | `docker-compose.yml` + `docker-compose.dev.yml` | Only MySQL, RabbitMQ and Mosquitto in Docker; the services run on the PC with `./mvnw` (IDE, tests) |
| Production | `docker-compose.prod.yml` + `.env.prod` | The server (see `05-architecture/deployment.md`) |

Development mode uses **the same database, queue and broker** (same volumes and `.env`) as the
full stack: what you do in one mode is visible in the other.

---

## Local ports in use

| Port | Service | Published in the full stack? |
|------|---------|------------------------------|
| 80 | Web (nginx), forwards `/api/` to the gateway | Yes |
| 8883 | Mosquitto, MQTT over TLS (devices) | Yes |
| 1883 | Mosquitto, MQTT without TLS | Only on `127.0.0.1` |
| 3306 | MySQL 8.4 | Only on `127.0.0.1` |
| 5672 | RabbitMQ (AMQP) | Only on `127.0.0.1` |
| 15672 | RabbitMQ management console | Only on `127.0.0.1` |
| 8080 | `en-monitor-api-gateway` | No (development mode: on the PC) |
| 8082 | `en-monitor-security-service` | No (development mode: on the PC) |
| 8084 | `en-monitor-device-service` | No (development mode: on the PC) |
| 8085 | `en-monitor-measurement-service` | No (development mode: on the PC) |
| 8086 | `en-monitor-home-service` | No (development mode: on the PC) |
| 8087 | `en-monitor-notification-service` | No (development mode: on the PC) |
| 8088 | `en-monitor-alert-service` | No (development mode: on the PC) |
| 8089 | `en-monitor-recommendation-service` | No (development mode: on the PC) |
| 5173 | Web app with Vite (development mode) | — |

**Check free ports:** `lsof -i :<port>` (macOS / Linux) or `netstat -ano | findstr ":<port>"`
(Windows).

---

## Option A — Full local stack

```bash
cd infra
cp .env.example .env && chmod 600 .env   # then fill it in: "Environment variables" below
./prepare.sh                             # RSA key pair for the JWTs and the broker's development certificates
docker compose config -q                 # names any required variable that is still missing
docker compose up -d --build
docker compose ps                        # the gateway and the seven services must be (healthy)
```

Open <http://localhost>. The API answers on the same origin (`http://localhost/api/v1/...`)
through the gateway; the mobile app on the LAN uses `http://<LAN IP>` as
`EXPO_PUBLIC_API_BASE_URL`. The RabbitMQ console is at <http://localhost:15672>
(`RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD`). The first start takes about two minutes while
Liquibase runs every changeset.

---

## Option B — Development mode (services on the PC)

```bash
cd infra
docker compose stop gateway security device measurement home notification alert recommendation frontend   # if the full stack was up
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d mysql rabbitmq mosquitto

# one terminal per service
cd ../backend/en-monitor-security-service    && ./mvnw spring-boot:run   # :8082
cd ../backend/en-monitor-device-service      && ./mvnw spring-boot:run   # :8084
cd ../backend/en-monitor-measurement-service && ./mvnw spring-boot:run   # :8085
cd ../backend/en-monitor-home-service         && ./mvnw spring-boot:run   # :8086
cd ../backend/en-monitor-notification-service && ./mvnw spring-boot:run   # :8087
cd ../backend/en-monitor-alert-service        && ./mvnw spring-boot:run   # :8088
cd ../backend/en-monitor-recommendation-service && ./mvnw spring-boot:run # :8089
cd ../backend/en-monitor-api-gateway         && ./mvnw spring-boot:run   # :8080
```

- Each service reads the `.env` of its own folder (copied from its `.env.example`) and an
  `application-local.yaml` (copied from its `.example`), with the shared values of `infra/.env`
  (table below).
- The web app with Vite (`:5173`) forwards `/api` to the gateway on `:8080` (other port:
  `VITE_DEV_API_TARGET`).
- In this mode Mosquitto reaches the device service on the host (`host-gateway`). On Linux with
  `ufw`, allow the Docker networks once:

  ```bash
  sudo ufw allow from 172.16.0.0/12 to any port 8084 proto tcp comment 'mosquitto -> device'
  ```

- Back to the full stack: `docker compose up -d` (recreates the broker without `host-gateway`).
- Swagger UI of each service in this mode: `http://localhost:8082/swagger-ui.html` (and `:8084`,
  `:8085`, `:8086`, `:8087`, `:8088`, `:8089`).

---

## Environment variables

`docker compose` reads `infra/.env`. Never commit it (it is git-ignored); protect it with
`chmod 600`. Write `KEY=value` without quotes, trailing spaces or `$`. Do not leave the
`CHANGE_ME_...` example values: the system starts, but those values are public.

Generate the random secrets that are still empty, without printing them:

```bash
gen() { grep -q "^$1=$" .env && sed -i "s|^$1=$|$1=$(openssl rand -hex "$2")|" .env; }
gen MYSQL_PASSWORD 16;    gen MYSQL_ROOT_PASSWORD 16; gen SECURITY_RESET_PEPPER 32
gen RABBITMQ_PASSWORD 24; gen INTERNAL_API_TOKEN 32;  gen MQTT_MEASUREMENT_SERVICE_PASS 24
gen MQTT_DEVICE_SERVICE_PASS 24
```

| Variable | Required | Meaning |
|---|---|---|
| `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` | Yes | Password of the `energy_app` user of the services, and of `root`. MySQL takes them only when its volume is created |
| `SECURITY_RESET_PEPPER` | Yes | Protects the recovery codes; changing it invalidates the codes already sent |
| `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` | The password | Shared by every service. RabbitMQ takes them only when its volume is created |
| `INTERNAL_API_TOKEN` | Yes | `X-Internal-Token` of `/internal/**`; the same in every service |
| `MQTT_DEVICE_SERVICE_USER/PASS`, `MQTT_MEASUREMENT_SERVICE_USER/PASS` | The passwords | MQTT users of the device and measurement services; the broker checks them against the device service |
| `DEVICE_MQTT_PUBLIC_HOST` | Yes | The PC's LAN IP (never `localhost`): the broker address written to a module when it is linked. Run `./prepare.sh` again whenever it changes |
| `DEVICE_MQTT_PUBLIC_PORT` | No | `8883` |
| `SECURITY_CORS_ALLOWED_ORIGINS`, `NOTIFICATION_WEB_URL` | No | `http://localhost` in the full stack |
| `BACKEND_UID`, `BACKEND_GID` | No | Set by `prepare.sh` (`id -u` / `id -g`) |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM` | No | SMTP account; without them no verification, recovery or alert e-mail is sent |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `VITE_GOOGLE_CLIENT_ID` | No | Google OAuth client; empty = no Google button |
| `NOTIFICATION_VAPID_PUBLIC_KEY`, `NOTIFICATION_VAPID_PRIVATE_KEY` | No | Web Push keys; empty = no browser push |
| `EXPO_ACCESS_TOKEN` | No | Expo push for the mobile app; server side only |
| `*_CONTEXT` (one per component, e.g. `HOME_CONTEXT`) | No | Only if the repositories are not cloned side by side |

Other variables the services read: the URL of each service they call (`SECURITY_SERVICE_URL`,
`HOME_SERVICE_URL`, `DEVICE_SERVICE_URL`, `MEASUREMENT_SERVICE_URL`; the gateway also
`NOTIFICATION_SERVICE_URL`, `ALERT_SERVICE_URL` and `RECOMMENDATION_SERVICE_URL`),
`MEASUREMENT_MQTT_ENABLED` (on in the infra stack) and the consumer switches of each service
(`ALERT_CONSUMERS_ENABLED`, `SECURITY_EVENTS_CONSUMER_ENABLED`, …, see each `runbook.md`).
**Removed** with the retirement of the monolith: `BACKEND_URL`, `MQTT_BACKEND_USER/PASS`,
`MQTT_CLIENT_ID` and `DEVICE_OFFLINE_AFTER` of the backend.

### Per-service `.env` files (development mode)

`MYSQL_PASSWORD` is called `SPRING_DATASOURCE_PASSWORD` there. Key paths must be absolute (Java
does not expand `~`); they may point to `infra/keys/` or to a key pair of your own.

| File | Values from `infra/.env` | Own values |
|---|---|---|
| `backend/en-monitor-security-service/.env` | `MYSQL_PASSWORD`, `SECURITY_RESET_PEPPER`, `GOOGLE_CLIENT_ID/SECRET`, `RABBITMQ_*`, `INTERNAL_API_TOKEN` | `SECURITY_JWT_PRIVATE_KEY_PATH`, `SECURITY_JWT_PUBLIC_KEY_PATH`, `SECURITY_SESSION_COOKIE_SECURE=false` |
| `backend/en-monitor-device-service/.env` | `MYSQL_PASSWORD`, `RABBITMQ_*`, `INTERNAL_API_TOKEN`, `MQTT_MEASUREMENT_SERVICE_USER/PASS`, `MQTT_DEVICE_SERVICE_USER/PASS`, `DEVICE_MQTT_PUBLIC_HOST` | `SECURITY_JWT_PUBLIC_KEY_PATH` |
| `backend/en-monitor-measurement-service/.env` | `MYSQL_PASSWORD`, `RABBITMQ_*`, `INTERNAL_API_TOKEN`, `MQTT_MEASUREMENT_SERVICE_USER/PASS` | `SECURITY_JWT_PUBLIC_KEY_PATH`, `MEASUREMENT_MQTT_ENABLED=true` |
| `backend/en-monitor-home-service/.env` | `MYSQL_PASSWORD`, `RABBITMQ_*`, `INTERNAL_API_TOKEN` | `SECURITY_JWT_PUBLIC_KEY_PATH`, `SECURITY_SERVICE_URL=http://localhost:8082` |
| `backend/en-monitor-notification-service/.env` | `MYSQL_PASSWORD`, `MAIL_*`, `RABBITMQ_*`, `INTERNAL_API_TOKEN`, VAPID, `EXPO_ACCESS_TOKEN` | `SECURITY_JWT_PUBLIC_KEY_PATH`, `NOTIFICATION_WEB_URL=http://localhost:5173`, `SECURITY_SERVICE_URL`, `HOME_SERVICE_URL=http://localhost:8086`, `DEVICE_SERVICE_URL=http://localhost:8084` |
| `backend/en-monitor-alert-service/.env` and `…/en-monitor-recommendation-service/.env` | `MYSQL_PASSWORD`, `RABBITMQ_*`, `INTERNAL_API_TOKEN` | `SECURITY_JWT_PUBLIC_KEY_PATH`, `HOME_SERVICE_URL`, `DEVICE_SERVICE_URL`, `MEASUREMENT_SERVICE_URL=http://localhost:8085` |
| `backend/en-monitor-api-gateway/.env` | — | `SECURITY_JWT_PUBLIC_KEY_PATH`, `SECURITY_CORS_ALLOWED_ORIGINS=http://localhost:5173` |
| `frontend/energy-monitor-web/.env.local` | `VITE_GOOGLE_CLIENT_ID` | — |

To check that a shared value is identical everywhere without printing it, compare its digests:

```bash
for v in RABBITMQ_PASSWORD INTERNAL_API_TOKEN MQTT_MEASUREMENT_SERVICE_PASS; do
  for f in .env ../backend/*/.env; do
    grep -q "^$v=" "$f" && echo "$v $(grep "^$v=" "$f" | cut -d= -f2- | sha256sum | cut -c1-8) $f"
  done
done
```

---

## Verify it works

```bash
docker compose ps                                                          # eight Java services (healthy)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/                 # 200 (web)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/api/v1/homes     # 401: answered by the gateway
curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' \
  http://localhost/api/v1/auth/login                                       # 400: reached the security service
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/api/v1/appliance-types     # 401: device route
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/api/v1/consumption-levels  # 401: measurement route
docker compose logs mosquitto | grep "New client connected"                # em-measurement-service and em-device-service
```

End-to-end tests over the full stack: `./e2e/device-flow.sh` (21/21) and
`./e2e/measurement-flow.sh` (25/25) on 24-Oct-2026, before the last four services joined the stack; the scripts in `infra/e2e/` are the current reference. They need `jq` and `openssl`.

**First administrator:** no account is an administrator after installing. Register an account
at <http://localhost>, then `./make-admin.sh user@example.com` from `infra/`, and sign in again.

---

## Common issues

| Symptom | Cause and fix |
|---|---|
| `required variable X is missing a value` | `X` is empty or missing in `infra/.env`; fill it and repeat `docker compose config -q` |
| `Access denied for user 'energy_app'` in a service on the PC | Its `SPRING_DATASOURCE_PASSWORD` / `application-local.yaml` does not match `MYSQL_PASSWORD` of `infra/.env` |
| A password changed in `.env` is refused by MySQL or RabbitMQ | They keep the values of their first start; change it inside (see the infra README, "Valores que guarda un volumen") or `docker compose down -v` (deletes the data) |
| The security service is `unhealthy` with RabbitMQ up | `RABBITMQ_USERNAME`/`RABBITMQ_PASSWORD` differ from the user RabbitMQ created on its first start |
| The API answers 503 through the web | The gateway cannot reach the service of that route: `docker compose ps`, `docker compose logs security device measurement home notification alert recommendation` |
| Broker logs `context deadline exceeded` calling `device:8084` (development mode) | The device service is not running on the PC, or the `ufw` rule for 8084 is missing |
| An MQTT client is refused with `rc=5` | `MQTT_*_SERVICE_USER/PASS` differ between the connecting service and the device service |
| A module does not connect: `X509 - Certificate verification failed` | The broker certificate does not cover the module's target IP: fix `DEVICE_MQTT_PUBLIC_HOST`, run `./prepare.sh`, then `docker compose up -d --force-recreate mosquitto`; reflash if the firmware was built with another CA |
| `origin_mismatch` on Google sign-in | Add `http://localhost` (and `http://localhost:5173`) to the OAuth client's authorized JavaScript origins |
| The web repeats 401 on `/api/v1/homes` | Expected without a session; sign in |

---

## Utility commands

```bash
docker compose logs -f home           # follow a service (or security, device, measurement, notification, alert, recommendation, gateway)
docker compose ps                     # state and health
docker compose up -d --build home     # rebuild one service
docker compose down                   # stop (keeps the volumes)
docker compose down -v                # stop and DELETE the data (MySQL and broker)
./mvnw test                           # tests of a service (most need the MySQL of infra: SPRING_DATASOURCE_PASSWORD=...)
```

---

## Useful local URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Web (full stack) | http://localhost | Your account |
| API through the gateway | http://localhost/api/v1/... (full stack), http://localhost:8080/api/v1/... (development mode) | Access token |
| RabbitMQ console | http://localhost:15672 | `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` of `infra/.env` |
| Swagger UI (development mode) | http://localhost:8082/swagger-ui.html, `:8084`, `:8085`, `:8086`, `:8087`, `:8088`, `:8089` | — |
| Web with Vite (development mode) | http://localhost:5173 | Your account |

---

## Correlations

- Deployment and compose modes → `05-architecture/deployment.md`
- Environments → `10-devops/environments.md`
- CI/CD → `10-devops/README.md`
- Per-service runbooks → `09-microservices/services/*/runbook.md`
