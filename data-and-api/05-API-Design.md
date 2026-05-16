# API Design

---

## 5.1 Style

- **Primary:** REST/JSON over HTTPS for all CRUD and stateless RPC-like actions (matches Flutter Dio ergonomics, simple to cache, no schema lock-in for mobile evolution).
- **Real-time:** WebSocket Secure (WSS) for chat — exactly one persistent socket per app session via `@fastify/websocket` on the same Fastify process.
- **No GraphQL/tRPC at MVP** — endpoint surface is small (42 endpoints), and GraphQL's introspection complicates rate limiting and abuse modelling for a safety-critical app.

---

## 5.2 Base URL & Versioning

| Concern | Convention |
| --- | --- |
| REST base | `https://api.spectrum.app/v1/` |
| WSS base | `wss://api.spectrum.app/chat` |
| Versioning | URL-prefix major version (`/v1/`). Breaking changes require a `/v2/` namespace; both run in the monolith via Fastify route plugins. Deprecation requires 6-month overlap and `Deprecation`/`Sunset` headers (RFC 8594). |
| Content type | `application/json; charset=utf-8` only (no XML, no form-encoded except IAP webhooks where required). |
| Compression | `Accept-Encoding: br, gzip` honoured. |

---

## 5.3 Authentication & Authorisation

- **Auth:** Bearer JWT (RS256), 15 min TTL, in `Authorization: Bearer <token>`. Asymmetric so all modules verify with public key only.
- **Refresh:** Opaque refresh token, 30-day TTL, single-use, family-tracked (reuse → invalidate entire family + force re-login).
- **Authorization:** RBAC via JWT `role` claim (`free | premium | admin:*`). Premium gates enforced server-side in module-layer code.
- **Admin:** Separate JWT issuance flow with hardware MFA (TOTP/FIDO2) + VPN-network-bound IP allowlist.

---

## 5.4 Endpoint Reference (Phase 1 — 42 endpoints)

| Method | Path | Auth | Request Body | Response (200/2xx) | Notes |
| --- | --- | --- | --- | --- | --- |
| POST | `/v1/auth/google` | Public + Turnstile | `{ id_token }` | `{ access, refresh, user }` | Google OAuth id_token verified server-side; 10 attempts/IP/hour → 60-min lockout. |
| POST | `/v1/auth/refresh` | Refresh | `{ refresh }` | `{ access, refresh }` | Single-use rotation; family invalidation on reuse. |
| POST | `/v1/auth/logout` | JWT | `{}` | `204` | Removes refresh + device push token. |
| GET | `/v1/users/:id` | JWT | — | `User` | 404 if blocked or non-existent. |
| PUT | `/v1/users/profile` | JWT | `Profile` | `Profile` | Validates display_name, bio length. |
| PUT | `/v1/users/identity` | JWT | `Identity` | `Identity` | Custom values length-checked. |
| GET | `/v1/users/identity/options` | Public | — | `{ genders, orientations, pronouns, intents, structures }` | Driven by `identity-options.json` so additions don't need a deploy. |
| PUT | `/v1/users/settings/visibility` | JWT | `Visibility` | `Visibility` | `show_orientation`, `show_structure`, incognito (premium). |
| DELETE | `/v1/account` | JWT | `{}` | `202` | Starts GDPR/DPDP delete pipeline; 30-day soft delete. |
| GET | `/v1/account/export` | JWT | — | `202 + Location header` | Async; SSE or signed S3 download URL emailed when ready. |
| POST | `/v1/users/photos` | JWT | multipart or presign | `Photo` | Either direct multipart (small) or `presign=true` returns S3 URL. |
| DELETE | `/v1/users/photos/:id` | JWT | — | `204` | |
| POST | `/v1/users/location` | JWT | `{ geohash, city?, country? }` | `204` | Server rejects if `geohash.length != 7`. |
| POST | `/v1/chat/media/presign` | JWT | `{ file_size, mime_type }` | `{ upload_url, s3_key }` | URL valid 5 min; `Content-Length` enforced. |
| GET | `/v1/discovery/feed` | JWT | query: `limit`, `cursor?` | `{ cards: Card[], next_cursor }` | 60/h free, 120/h premium. |
| PUT | `/v1/discovery/filters` | JWT | `Filters` | `Filters` | Premium fields ignored if `is_premium=false`. |
| GET | `/v1/discovery/filters` | JWT | — | `Filters` | |
| POST | `/v1/matches/like/:id` | JWT | `{}` | `{ matched: boolean, match_id?: uuid }` | 10/day free. |
| POST | `/v1/matches/pass/:id` | JWT | `{}` | `204` | |
| DELETE | `/v1/matches/undo` | JWT + Premium | `{}` | `204` | `403` if non-premium. |
| GET | `/v1/matches` | JWT | `cursor?` | `{ matches: Match[], next_cursor }` | |
| POST | `/v1/matches/unmatch/:id` | JWT | `{}` | `204` | Hides conversation both sides. |
| GET | `/v1/chat/conversations` | JWT | `cursor?` | `{ conversations: Conv[], next_cursor }` | |
| GET | `/v1/chat/conversations/:id/messages` | JWT | `cursor?, limit` | `{ messages: Msg[], next_cursor }` | Returns ciphertext only. |
| DELETE | `/v1/chat/messages/:id` | JWT | — | `204` | Soft-delete with `deleted_at`. |
| POST | `/v1/chat/keys` | JWT | `KeyBundle` | `204` | Uploads identity+signed+one-time pre-keys. |
| GET | `/v1/chat/keys/:userId` | JWT | — | `KeyBundle` | One-time pre-key consumed per fetch. |
| POST | `/v1/safety/block/:userId` | JWT | `{}` | `204` | Refreshes Redis `block:` set within 500 ms. |
| DELETE | `/v1/safety/block/:userId` | JWT | — | `204` | |
| GET | `/v1/safety/blocks` | JWT | — | `{ blocked_users: id[] }` | |
| POST | `/v1/safety/report` | JWT | `{ reported_id, category, description }` | `202` | CRITICAL paths PagerDuty. |
| POST | `/v1/privacy/contact-block` | JWT | `{ hashed_phones: string[] }` | `204` | Hashes are HMAC-SHA256 client-side. |
| GET | `/v1/billing/subscription` | JWT | — | `Subscription` | |
| POST | `/v1/billing/iap/apple/verify` | JWT | `{ receipt }` | `Subscription` | Calls Apple SS API server-side. |
| POST | `/v1/billing/iap/google/verify` | JWT | `{ purchase_token, product_id }` | `Subscription` | Calls Play Dev API. |
| POST | `/v1/billing/cancel` | JWT | `{}` | `Subscription` | Cancel-at-period-end. |
| POST | `/v1/billing/webhooks/apple` | Apple sig | (signed payload) | `204` | App Store Server Notifications v2. |
| POST | `/v1/billing/webhooks/google` | Google sig | (signed payload) | `204` | RTDN. |
| POST | `/v1/billing/webhooks/stripe` | Stripe sig | (signed payload) | `204` | `Stripe-Signature` check. |
| POST | `/v1/devices/register` | JWT | `{ push_token, platform, app_version }` | `204` | Called every cold start. |
| POST | `/v1/devices/unregister` | JWT | `{}` | `204` | On logout. |
| GET | `/v1/health` | Public | — | `200 OK` | Health probe (no payload secrets). |

---

## 5.5 Error Envelope (universal)

```jsonc
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",     // stable machine code
    "message": "birth_date must be ≥ 18 years ago",
    "field": "birth_date",           // optional
    "details": { "min_age": 18 }     // optional
  },
  "meta": { "request_id": "uuid", "timestamp": "2026-05-13T10:00:00Z" }
}
```

**Canonical error codes:** `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `PAYMENT_REQUIRED`, `UNDERAGE`, `BLOCKED`, `INTERNAL_ERROR`, `UPSTREAM_UNAVAILABLE`.

---

## 5.6 Rate Limits

| Endpoint Group | Free | Premium | Window |
| --- | --- | --- | --- |
| `POST /auth/google` | 10 | 10 | per IP, per hour |
| `POST /matches/like` | 10 | unlimited | per user, per day |
| `GET /discovery/feed` | 60 | 120 | per user, per hour |
| `POST /chat/*` | 200 | 500 | per user, per hour |
| `POST /safety/report` | 10 | 10 | per user, per day |
| All other authed | 100 | 200 | per user, per minute |

Backed by an Upstash Lua-script token bucket. Responses include `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, `Retry-After` (RFC 9239 draft).

---

## 5.7 WebSocket Events

| Direction | Event | Payload | Notes |
| --- | --- | --- | --- |
| C→S | `message` | `{conversation_id, client_message_id, msg_type, ciphertext, media_key?, sent_at}` | Server stores ciphertext. |
| C→S | `typing` | `{conversation_id}` | TTL 5 s, throttled to 1/sec. |
| C→S | `read` | `{message_id}` | Read receipt; server only forwards if recipient is premium. |
| C→S | `ping` | `{}` | Keepalive every 30 s. |
| S→C | `message.new` | full envelope | |
| S→C | `message.delivered` | `{message_id, delivered_at}` | |
| S→C | `message.read` | `{message_id, read_at}` | Premium only. |
| S→C | `match.new` | `{match_id, user_preview}` | |
| S→C | `conversation.typing` | `{conversation_id, user_id}` | |
| S→C | `user.unmatched` | `{match_id}` | |
| S→C | `user.blocked` | `{}` | Silent — UI hides conversation. |
| S→C | `pong` | `{}` | |
| S→C | `error` | `{code, message}` | Authoritative server errors. |
