# Observability Plan

---

## 10.1 Logging

- **Format:** JSON via Pino. Required fields: `ts`, `level`, `request_id`, `user_id` (when authenticated), `module`, `event`, `latency_ms`, `status_code`.
- **PII rules:** never log Google id_tokens, JWTs, ciphertext, S3 keys with user identifiers, or geohashes longer than 5 chars.
- **Levels:** `trace` (dev only), `debug` (preview only), `info` (default), `warn`, `error`, `fatal`.
- **Pipelines:** stdout → Railway log drain → Sentry Logs (free tier).

---

## 10.2 Metrics

Instrumented via `prom-client` → OpenTelemetry exporter.

| Metric | Type | Labels |
| --- | --- | --- |
| `http_request_duration_ms` | histogram | method, route, status |
| `ws_message_dispatch_ms` | histogram | direction |
| `db_query_duration_ms` | histogram | module, operation |
| `outbox_pending` | gauge | kind |
| `auth_google_attempts_total` | counter | result |
| `safety_reports_total` | counter | severity |
| `match_likes_total` | counter | premium |
| `notif_dispatch_total` | counter | platform, result |
| `chat_active_ws_connections` | gauge | — |

---

## 10.3 Tracing

OpenTelemetry SDK in Fastify, exporting OTLP → Sentry Performance. Every request has a trace; spans for DB, Redis, S3, Google OAuth, FCM/APNs, Stripe. Trace IDs propagated to Sentry errors so a single `request_id` joins logs + traces + errors.

---

## 10.4 Alerting

| Alert | Threshold | Channel |
| --- | --- | --- |
| `http_5xx_rate` > 0.5% for 5 min | warn → PagerDuty | P1 page |
| `p99_latency` > 2× SLO for 5 min | warn | Sentry alert → Slack |
| `db_connections` > 80% | warn | Slack |
| `auth_failure_rate` > 100/min | warn | Slack + Security on-call |
| `chat_active_ws_connections` > 80% capacity | warn | Slack |
| CRITICAL report created | immediate | PagerDuty page (no threshold) |
| Stripe webhook failures > 3 / 10 min | warn | Slack |
| `app_crash_free_rate` < 99% / 1 h | warn | Sentry alert |
| `/health` non-200 > 3 min | page | BetterUptime → PagerDuty |
| Auto-rollback fired | info | Slack |

---

## 10.5 Health Checks

- `GET /health` — returns 200 if process up, DB ping < 200 ms, Redis ping < 200 ms. Used by Railway liveness + BetterUptime probe.
- `GET /ready` (internal-only) — returns 200 only once warm caches loaded; Railway readiness probe.
- `/metrics` exposed on a separate internal-only port (`9091`) for future Prometheus scrape (currently OTel push to Sentry).
