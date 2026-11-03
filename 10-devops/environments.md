# Deployment Environments

> Defines the system's environments, their purposes, access, and key configurations.
> **Rule:** If you need to access production directly, something is wrong in the process.
>
> State at 28-Oct-2026. Source: README and `deploy/deploy.sh` of `energy-monitor-infra`.

---

## Project environments

| Environment | Purpose | Update | Access | Data |
|-------------|---------|--------|--------|------|
| **local** | Development on the developer's machine: full Docker stack or development mode (`10-devops/local-setup.md`) | Manual | Each developer | Own test accounts and homes |
| **dev** (branch) | Integration: every pull request targets `dev`; CI runs the tests on every PR and push | Automatic CI, no deployment | Team | — (no `dev` server) |
| **production** | `https://energy-monitor.tech` (MQTT: `mqtt.energy-monitor.tech:8883`), Ubuntu VPS | From `main`, after CI and the approval of the GitHub `production` environment | Team | Real |

There is no separate staging environment.

---

## Environment variables per environment

| Environment | Where the values live |
|---|---|
| Full local stack | `infra/.env` (copied from `.env.example`) |
| Development mode | `infra/.env` plus each service's own `.env` and `application-local.yaml` |
| Production | `infra/.env.prod` on the server (copied from `.env.prod.example`, `chmod 600`) |
| GitHub Actions | Secrets of the `production` environment in each repository: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS` |

None of these files is versioned. The variables and how to obtain each value are in
`10-devops/local-setup.md`. Production differs from local mainly in: `DEVICE_MQTT_PUBLIC_HOST`
(`mqtt.<domain>`), `https://<domain>` as CORS origin and public URL, `Secure` session cookies,
and the JWT key pair in `JWT_KEYS_DIR`.

---

## CI/CD Pipeline

Every repository (gateway, the seven services, web, infra) has
`.github/workflows/ci-cd.yml`:

```
Developer
  │ push feat/... → pull request to dev
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ CI (every pull request and push to dev and main)                     │
│  - Java services: ./mvnw test (against a MySQL 8.4 service of the    │
│    runner where needed) and docker build                             │
│  - Web: vitest + npm run build                                       │
│  - Infra: docker compose config of the three modes, shellcheck,      │
│    nginx -t of the template                                          │
└──────────────────────────────────────────────────────────────────────┘
  │ pull request dev → main
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ CD (push to main or "Run workflow"), after the tests and the         │
│ approval of the production environment                              │
│  SSH to the VPS with a key restricted to                             │
│  infra/deploy/deploy.sh gateway|security|device|measurement|home|    │
│                         notification|alert|recommendation|frontend|  │
│                         infra                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Production deploy strategy

**Adopted strategy:** recreate one component at a time, with an automatic revert.
`deploy.sh <component>`:

1. Takes a lock, so only one deployment runs at a time.
2. Backs up MySQL (`deploy/backup-mysql.sh`).
3. Pulls `main` in that component's repository.
4. Rebuilds and recreates only that container (`docker compose ... up -d --build --no-deps`;
   `infra` recreates what changed, including Mosquitto).
5. Waits up to 300 s until the five Java services are `healthy`, the web answers 200 and
   `/api/v1/homes` answers 401 through the gateway.
6. If that fails, checks out the previous commit, rebuilds it and exits with an error (the job
   turns red).

### Production deploy checklist

- [ ] The pull request `dev` → `main` was reviewed and CI is green
- [ ] The `production` environment approval was given
- [ ] If there are database changes: the new changesets are forward-only and have a `rollback`
      block (`06-data/migration-strategy.md`)
- [ ] When a component depends on another's new version (for example a new MQTT user), the
      dependency is deployed first (the infra README lists the order used for each extraction)
- [ ] Someone checks the service after the deploy (`09-microservices/services/*/runbook.md`)

### Rollback

- Automatic: `deploy.sh` reverts to the previous commit when its health checks fail.
- On purpose: revert the change in the repository and let CD redeploy `main`. A code rollback
  does not revert an applied Liquibase changeset; the MySQL backup taken by `deploy.sh` before
  each deployment is the fallback for data.

---

## Deploy log

The versions in production are listed, with their commit, in `09-microservices/service-catalog.md`
("Version in production"). The GitHub Actions run history of each repository is the detailed log.

---

## Correlations

- Local setup → `10-devops/local-setup.md`
- Deployment architecture → `05-architecture/deployment.md`
- Post-deploy observability → `13-operations/observability.md`
- Runbook per service → `09-microservices/services/*/runbook.md`
- Branch policy → `00-governance/git-conventions.md`
