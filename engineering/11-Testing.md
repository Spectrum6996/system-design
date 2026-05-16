# Testing Strategy

---

## 11.1 Test Pyramid

```
                  ┌────────────────┐
                  │ Manual QA +    │  (T&S drills, panic-button device matrix)
                  │ exploratory    │
                  └────────────────┘
                ┌────────────────────┐
                │ E2E (Patrol)       │  Critical user journeys
                └────────────────────┘
            ┌──────────────────────────┐
            │ Contract (Pact)          │  Inter-module event schemas
            └──────────────────────────┘
        ┌──────────────────────────────────┐
        │ Integration (Jest + testcontain.)│  REST + WS + DB + Redis
        └──────────────────────────────────┘
   ┌──────────────────────────────────────────────┐
   │ Unit (Jest + Supertest, flutter_test)        │  80% backend / 70% mobile
   └──────────────────────────────────────────────┘
```

---

## 11.2 Unit Test Targets

| Area | Critical cases |
| --- | --- |
| `auth.googleSignIn` | invalid id_token, expired id_token, duplicate google_sub (returning user), per-IP rate limit boundary |
| `auth.verifyJWT` | expired token, wrong kid, tampered signature, family invalidation on refresh reuse |
| `match.like` | self-like, blocked target, simultaneous reciprocal, daily quota boundary, premium unlimited |
| `discovery.feed` | block-set exclusion, contact-block exclusion, incognito exclusion, age boundary, intent intersection, no-results path |
| `safety.createReport` | CRITICAL → immediate suspend + PagerDuty stub asserted, severity classification |
| `billing.gates.isPremium` | grace period, refund, cross-platform, expired |
| `chat.deliverEnvelope` | recipient blocked sender, recipient offline → outbox enqueue |
| `media.uploadPipeline` | CSAM match rejection, pHash duplicate, > 10 MB rejection |
| age + birth-date | `<18` rejection, leap-year edge, time-zone boundary |

---

## 11.3 Integration Boundaries

- **Real:** Postgres (testcontainers), Redis (testcontainers), Prisma — these are the system's truth.
- **Mocked:** Google OAuth (stub id_token with fixed `sub`), FCM/APNs (record-only), S3 (LocalStack), Apple/Google IAP (fixture receipts), PhotoDNA (offline hash list).

---

## 11.4 E2E Critical Journeys (Patrol)

1. Sign-up with Google OAuth → identity → first photo → feed loads.
2. Like a profile → mutual match → push notification (silent assert via mock) → open conversation.
3. Send and receive E2EE text + image messages on 2 emulators.
4. Block from chat → conversation disappears both sides within 500 ms.
5. Report CRITICAL → target suspended → audit row written.
6. Panic button 3-finger long-press → decoy screen within 500 ms.
7. Subscribe via Stripe sandbox → premium gate flips.
8. GDPR delete → re-login attempt fails → export available.

---

## 11.5 Performance Testing

- **k6** load script, run pre-launch and weekly thereafter against staging.
- Scenarios: `feed_browse` (60 rph/user × 1 K users), `match_flow` (likes + push), `chat_burst` (100 msg/min × 200 users), `auth_storm` (Google OAuth 10/h/IP cap stress).
- **Pass criteria:** all SLOs in [01-System-Overview.md](./01-System-Overview.md) met at 2× expected Day-1 load.

---

## 11.6 Test Data Management

- **Synthetic only** in non-prod. No production data ever copied to dev.
- **Staging:** nightly anonymised snapshot — names, emails, phones HMAC-replaced; geohashes shifted by a fixed seed; photos replaced with placeholders.
- **Seed fixtures:** versioned in `prisma/seed/` — covers all gender/orientation combinations + 3 admin roles + IAP fixture receipts.
