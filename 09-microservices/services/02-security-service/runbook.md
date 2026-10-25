# Runbook — Security Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> If this service is down, nobody can sign in or refresh a session.

**Service:** `en-monitor-security-service` (container `security`)
**Repository:** https://github.com/Juan-Cortes2045/en-monitor-security-service
**Version:** `main` @ `334691a` (24-Oct-2026)
**Last updated:** 2026-10-24

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8082, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/auth/...` (through the gateway) |
| Health | `/actuator/health`, includes MySQL and RabbitMQ |
| Runs as | `BACKEND_UID:BACKEND_GID` (10001), to read `private.pem` |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps security                       # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' \
  https://energy-monitor.tech/api/v1/auth/login     # 400: the request reached the security service
```

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| `unhealthy` with RabbitMQ up | `RABBITMQ_USERNAME`/`RABBITMQ_PASSWORD` do not match the user RabbitMQ created when its volume was first started | Change the password inside RabbitMQ (`rabbitmqctl change_password`, see the infra README) or recreate the `rabbitmq-data` volume (pending messages are lost) |
| Does not start, names a key path | `private.pem` / `public.pem` missing, unreadable or of the wrong kind | Check `JWT_KEYS_DIR`: `private.pem` readable by uid 10001, `public.pem` 644 |
| Does not start, pepper missing | `SECURITY_RESET_PEPPER` empty | Set it in `.env.prod` (keep the same value as before, or issued codes stop working) |
| Account deletion answers 503 | The backend did not answer the deletion check (or `INTERNAL_API_TOKEN` differs) | Check `emc ps backend` and that `INTERNAL_API_TOKEN` is the same in every service |
| Verification / recovery e-mails do not arrive | RabbitMQ down (messages wait in `security_outbox`) or the backend's mail settings | `emc logs security backend`; the backend is the one that sends e-mail |
| Google sign-in answers 503 | `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` empty | Fill them in `.env.prod` |
| Google answers `origin_mismatch` | The production origin is not in the OAuth client's authorized JavaScript origins | Add it in Google Cloud Console |
| Sign-in answers 403 "Verify your email address" | The account has not verified its e-mail | Expected; the web asks for a new code |

---

## 4. Rollback

`infra/deploy/deploy.sh security` (GitHub Actions, after approval) backs up MySQL, pulls `main`,
rebuilds only `security` and waits for every Java service to be healthy; if that fails it
rebuilds the previous commit and exits with an error. To return to an earlier version on purpose,
revert the change in the repository and let CD redeploy `main`. A rollback of the code does not
roll back an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/security/` (`security-001` …
`security-027`). Never rename or move a changelog file: its path is part of its identity in the
shared `DATABASECHANGELOG`.

---

## 6. Maintenance operations

**First administrator** (no account is administrator after installing): register the account in
the web, then from `/opt/energy-monitor/infra`:

```bash
./make-admin.sh --prod user@example.com
```

---

## 7. Alerts

No automated alerting is configured; failures show up in `emc ps`, in the deploy job and in user
reports.
