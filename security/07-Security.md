# Security Design

---

## 7.1 Threat Model Summary

| Trust boundary | Attack surface | Top threats |
| --- | --- | --- |
| Mobile client ↔ Internet | TLS, JWT, refresh token | Token theft (root/jailbreak), MITM, replay |
| Internet ↔ Cloudflare | TLS termination, WAF | DDoS, scraping, OWASP Top 10 |
| Cloudflare ↔ Railway | TLS, IP allowlist (admin) | Origin bypass, unauthorised admin access |
| Railway ↔ Neon/Upstash/S3 | Connection strings in env vars | Credential theft, lateral movement |
| User ↔ User | Block, report, chat, profile views | Harassment, doxxing, outing, CSAM, catfishing, contact-stalking |
| Admin ↔ System | VPN + MFA + RBAC | Insider abuse, moderator over-reach into chat |

---

## 7.2 Authentication Flow (step-by-step)

```
1. App initiates Google OAuth → receives id_token from Google
   └─ App → /v1/auth/google { id_token }
      ├─ Cloudflare Turnstile token verified
      ├─ Upstash rate limit: 10/h/IP (else 429)
      ├─ Google id_token verified server-side (audience + expiry check)
      ├─ If new user: insert users + identity (status=registering) + consent_log
      └─ Mint access JWT (RS256, 15min) + refresh (opaque, 30d) bound to a new family_id

2. App stores access in memory + refresh in flutter_secure_storage (Keychain / Keystore)

3. Subsequent API calls → Authorization: Bearer <access>
   ├─ JWT middleware verifies RS256 signature against rotating public key (kid header)
   ├─ Validates exp, nbf, iat, sub
   └─ Attaches { userId, role } to request.context

4. Access expires → App → /v1/auth/refresh { refresh }
   ├─ Look up devices.refresh_token_hash (sha256)
   ├─ If found: rotate (insert new hash, mark old as used)
   ├─ If old token reused: invalidate entire family_id (security incident)
   └─ Return new pair

5. Logout → /v1/auth/logout → device row + family invalidated; push token unregistered
```

---

## 7.3 Authorisation Matrix

| Role | Endpoints | Notes |
| --- | --- | --- |
| **anonymous** | `/auth/google`, `/users/identity/options`, `/health`, billing webhooks | Public surfaces; rate-limited heavily. |
| **user (free)** | All `/v1/*` except `/admin/*`, `/matches/undo`, premium gates | Limits enforced (likes/day, filters). |
| **user (premium)** | Same + premium gated endpoints | Server checks `users.is_premium`. |
| **moderator** | `/admin/reports/*`, `/admin/users/search`, `/admin/users/:id`, `/admin/actions/*` | NEVER message content endpoints. |
| **trust_safety_lead** | All moderator perms + policy config + SLA dashboards | |
| **support_agent** | `/admin/users/:id` (no photos, no full PII) + `/admin/users/:id/force-logout` | |
| **readonly_analyst** | `/admin/metrics/*` (anonymised) | Read-only. |
| **super_admin** | All admin endpoints + `/admin/team/*` | Hardware FIDO2 required. |

---

## 7.4 Data Protection

- **In transit:** TLS 1.3 minimum (Cloudflare enforces). HSTS preload list submission. mTLS between Railway and Stripe/Apple/Google not required (signature verification suffices).
- **At rest:** Neon Postgres AES-256 (managed). Upstash AES-256. S3 SSE-S3 (or SSE-KMS for India region if regulator audit demands). E2EE messages — only ciphertext stored.
- **Secrets:** Railway encrypted env vars. **Zero secrets in code or Git.** Pre-commit hook + `git-secrets` scan. KMS-backed envelope keys for cryptographic shredding on GDPR delete.
- **PII handling:**
  - Google `sub` (subject identifier) stored for account linkage; contact-block phones are **HMAC-SHA256 client-side** before upload (server holds hashes only).
  - Birth date stored as `DATE` (not exact timestamp) — age is computed; year-only display option in UI.
  - Exact GPS coordinates **never** leave device — only precision-7 geohash.

---

## 7.5 Input Validation & Injection Prevention

- All routes use **Zod schemas** at handler entry. Reject unknown keys (`strict()` parse).
- **Prisma ORM** — all DB access parameterised; no raw SQL strings allowed (lint rule).
- **Length caps** at schema layer: bio 500, display_name 50, custom identity 100, ciphertext 64 KB.
- **JSON depth limits**: Fastify `bodyLimit: 1mb`; custom schema validation rejects nesting > 8.
- **SVG/HTML uploads:** rejected — only JPEG/PNG/HEIC photos.
- **URL fetching:** the only outbound URLs the server fetches are S3 presigned URLs it minted itself. No user-supplied URL is fetched (SSRF prevention).

---

## 7.6 OWASP Top 10 (2021) Mitigation Checklist

| Risk | Control |
| --- | --- |
| A01 Broken Access Control | JWT RS256 verified every request; userId from token only (never body/header); premium gates server-side only; admin VPN+MFA+RBAC; chat-content read denied to admin by API design. |
| A02 Cryptographic Failures | TLS 1.3, AES-256 at rest, Signal Protocol E2EE for chat, HMAC-SHA256 for contact hashes, RS256 JWTs. |
| A03 Injection | Prisma parameterised; Zod validation; output encoding via Fastify JSON serializer; no eval, no shell exec. |
| A04 Insecure Design | Threat model + STRIDE pass during architecture review; privacy-by-default; under-18 hard CHECK; geohash > exact coords. |
| A05 Security Misconfiguration | Terraform IaC for AWS; Railway env-vars; CSP, HSTS, X-Content-Type-Options, Referrer-Policy headers set globally; no defaults left. |
| A06 Vulnerable & Outdated Components | Renovate bot bumps deps weekly; Snyk SCA blocks high/critical CVEs in CI; SBOM generated by syft. |
| A07 Auth Failures | 15-min JWT, single-use refresh, family invalidation on reuse, per-IP lockout after 10 failed Google OAuth attempts, Turnstile on `/auth/google`, optional biometric step-up on app open. |
| A08 Software & Data Integrity | Signed Docker images (cosign), GitHub branch protection, mandatory PR review, deploy approval gate. |
| A09 Logging & Monitoring Failures | Sentry + structured Pino logs + immutable audit trail in `moderation_actions` + PagerDuty escalations. |
| A10 SSRF | Server only fetches its own presigned S3 URLs; outbound URL allowlist enforced by `undici` interceptor. |

---

## 7.7 Compliance Plan

| Regime | Obligations | Implementation |
| --- | --- | --- |
| **GDPR** (EU/UK) | Right to access, delete, rectify, portability; DPO; lawful basis; DPA | `/account/export` returns JSON within 72h; `/account` DELETE with 30-day soft → hard purge across all modules + S3 + Neon WAL purge; consent_log with policy version; DPO appointed; DPA template under legal review. |
| **DPDP** (India) | Notice + granular consent; Grievance Officer; data localisation | Onboarding consent screen with per-category opt-in; Grievance Officer email in app settings; India users routed to `spectrum-prod-ap` Neon region. |
| **CCPA** (California) | Right to know, delete, opt-out of sale | We don't sell data — "Do Not Sell" link surfaced anyway; deletion via the unified pipeline. |
| **Apple/Google Store** | LGBTQ+ content 17+, content declaration | iOS App Tracking Transparency (we don't track), 17+ rating, content declaration in App Store Connect. |
| **NCMEC** | CSAM reporting | CyberTipline account active; PhotoDNA hash check on every photo upload before persistence. |
