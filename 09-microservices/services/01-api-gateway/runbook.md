# Runbook — API Gateway

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra` unless stated otherwise.

**Service:** `en-monitor-api-gateway` (container `gateway`)
**Repository:** https://github.com/Juan-Cortes2045/en-monitor-api-gateway
**Version:** `main` @ `0b2c164` (28-Oct-2026)
**Last updated:** 2026-11-03

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8080, inside the Docker network only (no published port) |
| Production URL | `https://energy-monitor.tech/api/v1/...` (Nginx → web container → `gateway:8080`) |
| Health | `/actuator/health` (container healthcheck) |
| Compose project | `energy-monitor-prod` (`docker-compose.prod.yml` + `.env.prod`) |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |
| State to lose on restart | Rate-limit counters only |

Shortcut used below:

```bash
emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }
```

---

## 2. Verify the service is healthy

```bash
emc ps gateway                                                            # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/homes              # 401: the gateway answers
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/appliance-types    # 401: device route
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/consumption-levels # 401: measurement route
```

These are the same checks `infra/deploy/deploy.sh` runs after every deployment (`/api/v1/homes`
must answer 401 through the gateway).

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| The API answers **503** | The gateway cannot reach the service that owns the path | `emc ps` and `emc logs --tail 100 security device measurement home notification alert recommendation`; restart or redeploy the failing service |
| The API answers **504** | The service took longer than 30 s (`GATEWAY_READ_TIMEOUT`) | Look at that service's logs and at MySQL |
| Every authenticated call answers **401** | The token cannot be validated: wrong or missing `public.pem`, or the key pair was regenerated for the security service only | Check that `JWT_KEYS_DIR` holds the `public.pem` matching the security service's `private.pem`; recreate the containers |
| The web app gets **401** while the mobile app works | The `em_access` cookie is not turned into a bearer token: the request lacks `X-Requested-With: XMLHttpRequest`, or the cookie expired | Sign in again; check the web build |
| Many **429** | More than 20 public account `POST`s per address and path per minute (users behind one NAT share it) | Wait for the window; the counters reset on restart |
| CORS errors in the browser | Origin not in `SECURITY_CORS_ALLOWED_ORIGINS` (only relevant when the web is served from another origin) | Fix the variable in `.env.prod` and recreate `gateway` |

Logs: `emc logs -f gateway`.

---

## 4. Rollback

- `deploy.sh gateway` (run by GitHub Actions after approval) backs up MySQL, pulls `main`,
  rebuilds only `gateway` and waits up to 300 s for the five Java services to be `healthy`, the
  web to answer 200 and `/api/v1/homes` to answer 401. If that fails it checks out the previous
  commit, rebuilds it and exits with an error.
- To go back to an earlier version on purpose: revert the change in the repository and let the
  CD workflow redeploy `main`.

---

## 5. Database migrations

Not applicable: the gateway has no database.

---

## 6. Alerts

No automated alerting is configured for the gateway; failures show up in `docker compose ps`, in
the deploy job and in user reports.

---

## 7. Post-incident checklist

- [ ] `emc ps` shows the five Java services `(healthy)`
- [ ] The checks of section 2 answer as expected
- [ ] Incident recorded in `13-operations/incident-management.md`
