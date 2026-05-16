# Infrastructure & Deployment

---

## 8.1 Target Environment

- **Primary:** Railway.app (US-East). Single service. WebSocket-native.
- **DB:** Neon Serverless PostgreSQL 18 — `spectrum-prod-us` (default), `spectrum-prod-eu` (EU/UK), `spectrum-prod-ap` (India / DPDP residency).
- **Cache:** Upstash Redis — region-pinned per Neon region.
- **Storage:** AWS S3 — `us-east-1`, `eu-west-1`, `ap-south-1` buckets; same KMS key policy.
- **Edge:** Cloudflare (DNS, TLS, WAF, Turnstile).

---

## 8.2 Infrastructure Diagram

```
            ┌─────────────────────────────────────────┐
            │            Cloudflare (global)          │
            │   DNS · TLS 1.3 · WAF · Turnstile       │
            └─────────────────────┬───────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌────────────┐      ┌────────────┐      ┌────────────┐
      │  Railway   │      │  Railway   │      │  Railway   │
      │  US-East   │      │  EU-West   │      │  AP-South  │
      │ (default)  │      │            │      │  (DPDP)    │
      └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
            │                   │                   │
   ┌────────┴────────┐ ┌────────┴────────┐ ┌────────┴────────┐
   │ Neon us-east-1  │ │ Neon eu-west-1  │ │ Neon ap-south-1 │
   │ Upstash us-east │ │ Upstash eu-west │ │ Upstash ap-south│
   │ S3 us-east-1    │ │ S3 eu-west-1    │ │ S3 ap-south-1   │
   └─────────────────┘ └─────────────────┘ └─────────────────┘

   Shared services (no PII): Sentry · BetterUptime · Resend · Stripe ·
   Apple SS · Google Play Dev API · FCM · APNs · Google OAuth
```

---

## 8.3 Environment Matrix

| Environment | Purpose | Compute | Data | Trigger |
| --- | --- | --- | --- | --- |
| `local` | Dev laptop | Docker Compose (Postgres+Redis) | Synthetic seed | manual `npm run dev` |
| `preview` | Per-PR ephemeral | Railway preview env | Neon branch, anonymised | open PR |
| `staging` | Pre-prod integration | Railway staging service | Anonymised prod copy nightly | merge to `main` |
| `prod-us`, `prod-eu`, `prod-ap` | Live | Railway 1×0.5vCPU/512MB → autoscale | Real PII (residency-correct) | manual approval gate |

---

## 8.4 Containerisation

- **Single Dockerfile**, multi-stage (Node 24 Alpine). `railway.toml` declares the start command + health check.
- **No Kubernetes at MVP.** Railway supplies process supervision, auto-restart, and rolling deploys. The architecture supports moving to AWS ECS-Fargate when scaling triggers fire.
- **Image registry:** GHCR; images signed with cosign; SBOM via syft attached to release.

---

## 8.5 CI/CD Pipeline (GitHub Actions)

```
PR opened
 ├── lint (eslint, prettier --check)
 ├── typecheck (tsc --noEmit)
 ├── unit tests (Jest)
 ├── integration tests (testcontainers Postgres + Redis)
 ├── SAST (Semgrep w/ ts-node-security ruleset)
 ├── dep scan (Snyk — high/critical block merge)
 ├── secret scan (gitleaks)
 ├── docker build + cosign sign
 ├── deploy preview env on Railway (per-PR Neon branch)
 └── E2E smoke (Patrol against preview env)

merge to main
 ├── prisma migrate deploy (staging)
 ├── Railway auto-deploy to staging
 ├── DAST (OWASP ZAP baseline) against staging
 ├── E2E full suite (Patrol)
 ├── manual approval (Release Manager + CTO via GitHub Environments)
 ├── Railway deploy to prod (rolling)
 ├── post-deploy smoke (/health, /v1/users/identity/options, sample login)
 └── auto-rollback if error rate > 1% within 10 minutes
```

---

## 8.6 Secrets Management

- All credentials live in Railway encrypted env vars (per environment).
- JWT keys rotated every 90 days via overlap window: new `kid` introduced, both keys honoured for 24h, then old key retired.
- KMS-managed envelope key per Neon region for cryptographic delete.

---

## 8.7 Rollback Strategy

- **Tactical (instant):** Railway "Redeploy previous" — single click, rolls back to last green Docker image (≤ 2 min).
- **DB migrations:** Every migration must be backward-compatible across one deploy. If a forward migration ships with a code change, deploy migration **first**, code **second**. Down-migrations are written but only applied as last resort with DBA approval.
- **Feature flags:** Unleash flags wrap risky paths so we can disable a feature without a rollback (preferred over redeploying).
- **Automatic rollback:** GH Action triggers `railway redeploy --previous` when Sentry error rate > 1% sustained 10 min after deploy.
