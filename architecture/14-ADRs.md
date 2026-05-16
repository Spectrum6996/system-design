# Open Questions & Architecture Decision Records

---

## Open Questions (pending stakeholder input)

1. **Boost economics** — duration, frequency, and multiplier window not yet specified.
2. **Photo-verification automation timeline** — Rekognition vs. manual review Phase 2 commit date.
3. **Face-blur policy for photos 2–9** — current assumption: unblurred once user explicitly opts in.
4. **Refresh-token family-invalidation policy** — needs security team sign-off on proposed behaviour.
5. **Soft-delete retention window for blocked conversations** — proposed: 180 days.
6. **Multi-region DR posture** — RTO/RPO formalisation needed.
7. **Admin VPN technology and ACL ownership** — proposed: Tailscale + Cloudflare WAF.
8. **iCloud/Google Drive backup of Signal identity key** — Phase 1 or Phase 1.5 decision pending.

### Items Requiring Stakeholder Input

1. **Product:** Boost product spec (duration, frequency, free allotment per tier).
2. **Trust & Safety:** photo verification timeline — Rekognition Phase 2 commit date + interim moderator headcount sizing.
3. **Privacy/Legal:** confirm soft-delete retention window for blocked conversations (proposed: 180 days).
4. **Security:** approve refresh-token family-invalidation policy as written; approve Tailscale + Cloudflare WAF combo for admin.
5. **DevOps:** approve multi-region production stack rollout calendar tied to EU + India launches.
6. **Mobile:** decide on Signal identity-key backup story (iCloud/Drive encrypted blob) — Phase 1 or Phase 1.5.
7. **Finance:** confirm budget envelope and credit-programme application status (Neon, Railway, Upstash, Sentry).
8. **Legal:** DPO + Grievance Officer appointment dates locked to market launch dates.

---

## Architecture Decision Records

### ADR-001: Monolithic-Modular over Microservices for MVP

**Status:** Decided

**Context:** Choice between the original 10-service AWS topology and a single Fastify process on Railway, before the product has PMF.

**Options considered:** (a) 10 microservices on ECS Fargate, (b) Modular monolith on Railway, (c) Lambda-per-route serverless.

**Decision:** Modular monolith on Railway.

**Consequences:** Save ~$2,479/mo; lose independent scaling per module; require disciplined `EXTRACT_BOUNDARY` annotation to retain extraction optionality; single point of failure during the 1-replica phase.

---

### ADR-002: PostgreSQL GIN over Elasticsearch for Discovery

**Status:** Decided

**Context:** Original TRD specified a 3-node ES cluster (~$773/mo).

**Options considered:** Elasticsearch, Meilisearch, PostgreSQL GIN+pg_trgm, OpenSearch.

**Decision:** PostgreSQL GIN with `jsonb_path_ops`.

**Consequences:** Free; sufficient to 50K profiles; complex synonym/typo handling deferred (not needed at MVP); Meilisearch remains Plan B at scale trigger.

---

### ADR-003: Signal Protocol for Chat E2EE (mandatory)

**Status:** Decided

**Context:** Privacy is a brand pillar; user safety in LGBTQ+ context demands true E2EE.

**Options considered:** Signal Protocol, Matrix/Olm, custom AES-GCM with key exchange.

**Decision:** Signal Protocol via libsignal.

**Consequences:** Forward + future secrecy; server can't comply with content subpoenas (intentional); reinstall loses message history (acceptable, documented UX); higher mobile complexity (offset by mature SDK).

---

### ADR-004: REST over GraphQL/tRPC for API

**Status:** Decided

**Context:** Need a Flutter-friendly, cache-friendly, abuse-modellable API.

**Options considered:** REST, GraphQL, tRPC.

**Decision:** REST/JSON.

**Consequences:** Simple Dio client; per-route rate limiting trivial; small endpoint surface; if BFF needs grow, GraphQL can be added on top later.

---

### ADR-005: Geohash-Only Location Storage

**Status:** Decided

**Context:** LGBTQ+ users face disproportionate physical-safety risk if precise location data leaks.

**Options considered:** Exact GPS, geohash 7, geohash 5, city-only.

**Decision:** Geohash precision 7 computed client-side; server never sees coordinates.

**Consequences:** ~150 m accuracy (enough for "nearby" feel); zero server liability for GPS data; bucketed distance display; geohash adjacency requires 9-cell scan logic.

---

### ADR-006: RS256 JWT with 15-min Access + Single-Use Refresh

**Status:** Decided

**Context:** Cross-module verification with rotation safety.

**Options considered:** HS256 shared-secret, RS256, opaque session tokens.

**Decision:** RS256 with `kid` rotation + single-use refresh + family invalidation.

**Consequences:** Public-key verification anywhere (future microservices); short blast radius if access token leaks; client implements refresh interceptor.

---

### ADR-007: Outbox Table for Cross-Module Side Effects

**Status:** Decided (extension to TRD)

**Context:** Notification dispatch must survive process restart and eventual module extraction.

**Options considered:** Direct call, in-memory queue, outbox+poll, external queue (NATS/SQS).

**Decision:** Postgres outbox + 250 ms polling worker in-process.

**Consequences:** Atomic with business mutation (same TX); replayable; transitions cleanly to NATS/SQS when chat module extracts.

---

### ADR-008: Railway Single Region (US-East) at MVP, Per-Region Production Stacks Post-Launch

**Status:** Decided

**Context:** DPDP and GDPR residency obligations vs. cost of triplicating infra.

**Options considered:** Single global region, three production stacks day 1, three stacks lazily provisioned.

**Decision:** Provision EU + AP stacks before EU/UK/India launch, not before US-only launch.

**Consequences:** Faster initial launch; DPDP/GDPR not in scope until those market launches; clear runbook to spin up regional stacks.

---

### ADR-009: Cryptographic Shredding for GDPR/DPDP Delete

**Status:** Decided (extension to TRD)

**Context:** "Hard delete" of all rows referencing a user is expensive and fragile across modules + S3 + backups.

**Options considered:** Hard delete cascade, soft delete + scheduled purge, cryptographic shredding via KMS.

**Decision:** Per-user envelope key in KMS; on delete, rotate/destroy the key so plaintext PII becomes unrecoverable, then run a 30-day delayed physical purge across modules.

**Consequences:** Compliant within 30 days; backups become safely unreadable once key destroyed; small added complexity in S3 upload path (envelope-encrypt photos with user key).

---

### ADR-010: VPN-bound Admin via Tailscale + Cloudflare WAF

**Status:** Decided (extension to TRD)

**Context:** TRD says "VPN IP allowlist" but doesn't pick a vendor.

**Options considered:** Self-hosted WireGuard, OpenVPN, Tailscale, AWS Client VPN.

**Decision:** Tailscale (zero-trust, free tier, ACLs as code) → fixed egress IP → Cloudflare WAF allowlist on `/admin/*`.

**Consequences:** Cheap, fast to set up, easy revocation; depends on Tailscale uptime — backup IP allowlist documented for break-glass.

---

## Documentation Completeness Checklist

| § | Section | Status | Notes |
| --- | --- | --- | --- |
| 1 | System Overview | Fully addressed | See [01-System-Overview.md](./01-System-Overview.md) |
| 2 | High-Level Architecture | Fully addressed | Diagrams given as structured text. See [02-Architecture.md](./02-Architecture.md) |
| 3 | Technology Stack | Fully addressed | Versions taken from TRD where stated; mobile lib pins are current LTS. See [03-Technology-Stack.md](./03-Technology-Stack.md) |
| 4 | Data Architecture | Fully addressed | DDL is canonical for primary tables; full schema lives in `prisma/schema.prisma`. See [04-Data-Architecture.md](./04-Data-Architecture.md) |
| 5 | API Design | Fully addressed | 42 MVP endpoints catalogued; WS events listed. See [05-API-Design.md](./05-API-Design.md) |
| 6 | System Components | Fully addressed | All 10 modules documented. See [06-Module-Design.md](./06-Module-Design.md) |
| 7 | Security Design | Fully addressed | Threat model + OWASP + compliance covered. See [07-Security.md](./07-Security.md) |
| 8 | Infrastructure & Deployment | Fully addressed | Railway-centric with multi-region plan. See [08-Infrastructure.md](./08-Infrastructure.md) |
| 9 | Performance & Scalability | Fully addressed | Bottleneck/trigger table extended. See [09-Performance.md](./09-Performance.md) |
| 10 | Observability | Fully addressed | OTel + Sentry + Pino + PagerDuty. See [10-Observability.md](./10-Observability.md) |
| 11 | Testing Strategy | Fully addressed | Pyramid + journeys + k6 thresholds. See [11-Testing.md](./11-Testing.md) |
| 12 | Development Guidelines | Fully addressed | Full directory tree, conventions, commit format. See [12-Development-Guidelines.md](./12-Development-Guidelines.md) |
| 13 | Risk Register | Fully addressed | 15 concrete risks identified. See [13-Risk-Register.md](./13-Risk-Register.md) |
| 14 | Open Questions & ADRs | Partially addressed | 8 open questions need stakeholder input — see list above. |
