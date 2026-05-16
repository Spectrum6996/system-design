# Phase 1 — TRD Analysis

**Authors:** Principal Software Architect (synthesised from Engineering Lead, Backend Architect, Mobile Lead, Security Engineer)  
**Date:** May 2026

---

## Extracted Project Profile

| Attribute | Value |
| --- | --- |
| Product purpose | Safety-first, identity-inclusive dating & community app for LGBTQ+ users in India, US, UK |
| MVP goal | Ship the PRD's Must-Have features as a single monolithic Node.js service on Railway, at < $80/month for the first 5K users |
| Primary user roles | End user (free), End user (Spectrum+ premium), Moderator, Trust & Safety Lead, Support Agent, Read-only Analyst, Super Admin |
| Functional pillars | Auth Google OAuth, Inclusive identity, Geohash discovery & matching, E2EE chat (Signal Protocol), Safety (block/report/CSAM), Privacy (incognito, contact block, face blur), Billing (IAP+Stripe), Push notifications, Admin/T&S dashboard |
| Non-functional requirements | p99 latency targets per endpoint (see [09-Performance.md](./09-Performance.md)), 99.9% uptime, GDPR + DPDP + CCPA compliance, zero plaintext PII on server beyond what auth needs, mandatory E2EE, server-side premium gating |
| Core technology stack | Node 24 LTS, TypeScript 6, Fastify 5, Prisma 7, Neon PostgreSQL 18, Upstash Redis, AWS S3, Railway.app, Flutter 3 mobile, Signal Protocol (libsignal-node), Google OAuth, FCM + APNs, Stripe + IAP |
| Third-party integrations | Google OAuth, AWS S3 (media), Stripe + Apple App Store Server API + Google Play Developer API (billing), FCM + APNs (push), Sentry (errors), BetterUptime (uptime), Resend (transactional email), Cloudflare (DNS+TLS), PhotoDNA/NCMEC (CSAM hashes) |
| Data entities (13) | users, identity_profiles, user_photos, user_locations, discovery_filters, swipes, matches, conversations, messages, blocks, reports, subscriptions, devices (+ derived: auth_sessions, consent_log, notification_log, admin_users, moderation_actions) |
| Constraints | Server must never see plaintext chat or exact GPS; client-only Signal private keys; India data resident in ap-south-1 equivalent; under-18 hard blocked at DB CHECK; moderator MUST NOT see chat content; OWASP Top 10 baseline; Railway env vars for all secrets |
| Deployment target | Railway.app (single service), Neon (PostgreSQL), Upstash (Redis), AWS S3 (ap-south-1 / us-east-1). 1 replica baseline, scales horizontally |
| Migration trigger | Any module > 200 req/sec sustained, team > 8 engineers, or 50K+ MAU → extract modules to microservices using `EXTRACT_BOUNDARY` markers |

---

## Open Questions (carried forward — not blocking)

1. **Voice/video call roadmap.** Voice calling is implied (voice notes are MVP) but real voice calling is Phase 2. Confirm no MVP requirement for in-app calling so we can skip TURN/STUN infra in v1.
2. **Boost economics.** "Profile Boost" exists in premium gates but is not priced or scoped (duration, multiplier window, cool-down). Assumed: 30-minute window, 2× scoring multiplier, 1 free boost per week for premium users.
3. **Photo verification automation.** TRD says "AWS Rekognition (Phase 2) or manual review (Phase 1)". Assumed: manual review backed by an admin queue at MVP; Rekognition deferred.
4. **Face-blur enforcement.** Blur default is opt-out in discovery feed for photo 1 only; behaviour on photos 2–9 unclear. Assumed: photos 2–9 unblurred once user explicitly opts in to show face; otherwise consistent blur.
5. **Refresh token reuse-detection.** "Single-use rotation" stated but reuse-detection / family invalidation policy not specified. Assumed: detect reuse → invalidate entire token family + force re-login (industry standard).
6. **Soft delete window for blocks/conversations.** TRD says blocked conversations are "hidden, not deleted — retained as moderation evidence" but no retention window. Assumed: 180 days retention, then hard purge unless under legal hold.
7. **Multi-region failover.** Cost-model implies single-region per market. Cross-region DR strategy is not defined. Assumed: Neon point-in-time recovery is the DR; RTO 4h / RPO 5min for MVP.
8. **Admin VPN.** Section 12 says "VPN IP allowlist". No specific VPN technology named. Assumed: Tailscale ACL (free for < 100 users) feeding a Cloudflare WAF allowlist.

---

## Architectural Style Decision (Mandated)

Per stakeholder direction and reaffirmed by the TRD: **Modular Monolith** — a single deployable Fastify process subdivided into ten domain modules (`auth`, `user`, `discovery`, `match`, `chat`, `safety`, `notification`, `billing`, `media`, `admin`) with `EXTRACT_BOUNDARY`-annotated cross-module function calls. Microservice extraction is deferred until concrete triggers (see [14-ADRs.md — ADR-001](./14-ADRs.md)) are met.

---

## Beyond-TRD Recommendations Folded In

These additions are inspired by Tinder, Bumble, Grindr, HER, Hinge, and Feeld but kept within MVP scope cost:

- **OpenTelemetry** wired from day 1 (Sentry already ingests), so distributed tracing is free if we extract modules later.
- **Feature flags** via Unleash OSS or LaunchDarkly free tier — gate Boost, premium filters, and risky migrations.
- **Outbox table** for cross-module side effects (push, billing webhooks) so internal calls can be replayed if Notification fails — critical when extracting to microservices.
- **Cloudflare Turnstile on auth endpoints** — bot-suppression widget on mobile sign-in to prevent automated account creation abuse.
- **Tombstoned soft delete with cryptographic shredding** for GDPR/DPDP: instead of `DELETE`, rotate per-user encryption envelope key in KMS so plaintext is unrecoverable, then schedule physical delete.
- **Shadow-ban primitive** in `users.status` (`shadow_banned`) so T&S can degrade abusers' reach without alerting them.
