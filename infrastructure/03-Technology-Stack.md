# Technology Stack

| Layer | Technology | Version | Justification |
| --- | --- | --- | --- |
| Mobile app | Flutter / Dart | 3.41 / 3.11 | One codebase for iOS+Android keeps a 2-engineer mobile team viable; Impeller-by-default on iOS + Android (API 29+); UIScene lifecycle support. |
| State mgmt | Riverpod | 3.x | Major version bump from 2.x — unified API, code-gen is the preferred path; compile-time safety vs. Provider; testable; widely adopted. |
| Routing | GoRouter | 14.x | Still the latest major; maintained by the Flutter team; deep links + nested routes required for panic-mode decoy screen + push deep links. |
| Mobile HTTP | Dio | 5.x | Interceptors for JWT refresh, retry, request signing. |
| Mobile secure store | flutter_secure_storage | 9.x | Keychain (iOS) / Android Keystore for JWTs and Signal private keys. |
| Mobile local DB | sqflite | 2.x | Encrypted message cache; large message history without re-fetch. |
| Geo | geolocator | 13.x | Bumped from 12.x; approximate accuracy supported; battery-efficient. |
| Push (mobile) | firebase_messaging | 15.x | One SDK handles APNs + FCM. |
| Crypto (mobile) | libsignal-client (Dart bindings) | latest | Official Signal implementation; battle-tested. |
| Backend lang | Node.js | 24 LTS | Current Active LTS (until Oct 2026, then maintenance to Apr 2028); strong async I/O fit for WebSocket + REST; same language across team. |
| Type system | TypeScript | 6.0 | Released 23 Mar 2026 — last release on the JS-based compiler before TS 7 (Go-based); strict-mode on by default; compile-time guarantees critical for cross-module calls. |
| HTTP + WS framework | Fastify | 5.8.5 | v5 requires Node 20+, aligns with Node 24 LTS; fastest Node framework benchmark; first-class plugin model; @fastify/websocket. |
| ORM | Prisma | 7.x | Rust-free WASM-based query compiler; ~70% faster type checks; smaller generated types; parameterized queries by default; great migration tooling. Note: MongoDB not yet supported in v7. |
| Validation | Zod | 4.4.3 | ~14× faster string parsing, ~57% smaller core bundle, new z.registry() metadata API; inferred TS types from schemas; consider zod/mini (~1.9KB gzipped) for client-side validation paths. |
| Primary DB | Neon (PostgreSQL) | 18 | Serverless Postgres with PgBouncer; scale-to-zero saves cost; per-branch databases ideal for previews. |
| Cache + queues | Upstash Redis | — | HTTP API, no TCP connections to manage; serverless free tier covers MVP. |
| Object storage | AWS S3 | — | Presigned URLs let client upload directly; SSE-S3 at rest; region selection for data residency. |
| Discovery search | PostgreSQL GIN + pg_trgm | 18 | Replaces Elasticsearch (~$773/mo) at < 50K profiles. Plan B: Meilisearch self-host. |
| Auth | JWT RS256 + Google OAuth | — | Asymmetric so all modules verify with public key only; Google OAuth for frictionless, passwordless sign-in without SMS dependency. |
| E2EE | Signal Protocol (libsignal-node) | latest | Forward + future secrecy; industry standard; non-negotiable per TRD. |
| Push (server) | firebase-admin + node-apn | latest | FCM v1 (Android) + APNs HTTP/2 (iOS). |
| Payments | Stripe + StoreKit 2 + Play Billing 7 | latest | Web (Stripe), iOS, Android — all three required; Play Billing 6 reached deadline, Billing 7 is now required for new Play submissions. |
| Email | Resend | — | Free 3K/month; clean DX. |
| Deploy | Railway.app | — | Dockerfile auto-build, WSS native, env-var secrets, ~2-min deploys. |
| IaC | Terraform | 1.15.x | Ephemeral resources + provider-defined functions now stable; only AWS resources are Terraformed; Railway uses railway.toml. Consider OpenTofu 1.9.x as a license-clean alternative if BUSL is a concern. |
| CI/CD | GitHub Actions + Railway auto-deploy | — | Free tier covers PR checks + main-branch deploys. |
| Errors | Sentry | — | Free Developer tier; OpenTelemetry-compatible. |
| Uptime | BetterUptime | — | 3-min probes on `/health`. |
| Tracing/metrics | OpenTelemetry → Sentry Performance | — | Future-proofs distributed tracing for microservice split. |
| Feature flags | Unleash OSS or LaunchDarkly free | — | Gate Boost, premium filters, risky migrations. |
| WAF + DNS | Cloudflare | — | DDoS, TLS, IP allowlists for admin. |
| Anti-abuse | Cloudflare Turnstile | — | Suppresses bot and abuse attacks on auth endpoints. |
