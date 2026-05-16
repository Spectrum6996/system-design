# Development Guidelines

---

## 12.1 Folder Structure (monorepo)

```
spectrum/
├─ backend/
│  ├─ src/
│  │  ├─ index.ts                     # Fastify bootstrap
│  │  ├─ app.ts                       # plugin registration
│  │  ├─ modules/
│  │  │  ├─ auth/
│  │  │  │  ├─ routes.ts
│  │  │  │  ├─ service.ts
│  │  │  │  ├─ repository.ts
│  │  │  │  ├─ oauth.ts
│  │  │  │  ├─ jwt.ts
│  │  │  │  ├─ types.ts
│  │  │  │  └─ __tests__/
│  │  │  ├─ user/
│  │  │  ├─ discovery/
│  │  │  ├─ match/
│  │  │  ├─ chat/
│  │  │  │  ├─ ws.ts
│  │  │  │  └─ ...
│  │  │  ├─ safety/
│  │  │  ├─ notification/
│  │  │  ├─ billing/
│  │  │  ├─ media/
│  │  │  └─ admin/
│  │  ├─ shared/
│  │  │  ├─ db/prisma.ts
│  │  │  ├─ redis/upstash.ts
│  │  │  ├─ s3/client.ts
│  │  │  ├─ logger.ts
│  │  │  ├─ telemetry.ts
│  │  │  ├─ outbox/dispatcher.ts
│  │  │  ├─ middleware/
│  │  │  │  ├─ jwt.ts
│  │  │  │  ├─ rateLimit.ts
│  │  │  │  ├─ errorHandler.ts
│  │  │  │  └─ requestId.ts
│  │  │  └─ config/
│  │  │     ├─ env.ts                 # zod-validated env
│  │  │     └─ identity-options.json
│  │  └─ types/
│  ├─ prisma/
│  │  ├─ schema.prisma
│  │  ├─ migrations/
│  │  └─ seed/
│  ├─ Dockerfile
│  ├─ railway.toml
│  ├─ package.json
│  └─ tsconfig.json
├─ mobile/
│  ├─ lib/
│  │  ├─ main.dart
│  │  ├─ features/
│  │  │  ├─ auth/  discovery/  chat/  match/  safety/  billing/  profile/
│  │  ├─ core/
│  │  │  ├─ api/  storage/  signal/  router/  theme/
│  │  └─ shared/ widgets/
│  ├─ ios/
│  ├─ android/
│  └─ test/
├─ infra/
│  ├─ terraform/                       # only AWS (S3, IAM, KMS, Route53)
│  └─ railway/                          # railway.toml templates
├─ .github/workflows/
│  ├─ pr.yml
│  ├─ main.yml
│  └─ release.yml
└─ docs/
   ├─ ARCHITECTURE.md
   ├─ RUNBOOKS/
   └─ ADRs/
```

---

## 12.2 Naming Conventions

| Element | Rule | Example |
| --- | --- | --- |
| Files (TS) | kebab-case for files, PascalCase for components | `match-service.ts`, `MatchService` |
| Files (Dart) | snake_case | `match_card.dart` |
| Variables | camelCase | `swiperId` |
| Constants | UPPER_SNAKE | `MAX_LIKES_FREE` |
| DB tables | snake_case plural | `user_locations` |
| DB columns | snake_case | `incognito_since` |
| API routes | kebab-case under `/v1/` | `/v1/safety/contact-block` |
| Env vars | UPPER_SNAKE prefixed by domain | `JWT_PRIVATE_KEY`, `S3_BUCKET` |
| Module boundary tags | `// EXTRACT_BOUNDARY: <target_module> ← <caller_module>` | as in TRD |

---

## 12.3 Branching: Trunk-Based with Short-Lived Branches

- `main` is always deployable.
- Feature branches: `feat/<ticket>-short-slug`. Max lifetime 3 days.
- Hotfixes: `hotfix/<ticket>` → merged directly via expedited approval.
- Release tags: `vYYYY.MM.DD-N` on every prod deploy.

---

## 12.4 Commit Messages — Conventional Commits

```
<type>(<scope>): <imperative summary>

[optional body]

[optional footers: BREAKING CHANGE:, Refs: TICKET-123]
```

Types: `feat | fix | chore | refactor | perf | test | docs | ci | build | security`.

Scopes: module names (`auth`, `match`, `chat`, ...) or `infra`, `mobile`, `db`.

---

## 12.5 Code Review Checklist (mandatory)

1. New endpoint? Added to OpenAPI + has Zod schema + integration test.
2. New DB column? Migration is backward-compatible + indexed if read-path.
3. Cross-module call? Tagged `EXTRACT_BOUNDARY`.
4. PII added? Confirmed not logged + retention defined.
5. Secret introduced? Lives in Railway env vars + documented in `env.ts` zod schema.
6. Rate-limited path? Limit defined + Upstash key naming consistent.
7. Premium feature? Server-side enforcement present + test asserts 403 for free users.
8. Security review tag (`security:`) requires +1 from Security Engineer.

---

## 12.6 Tooling

- **Lint:** ESLint + `@typescript-eslint` + `eslint-plugin-security` + `eslint-plugin-import` (no relative `../../..` beyond 2 levels). Dart: `flutter_lints`.
- **Format:** Prettier (2-space, semi, single-quote). `dart format`.
- **Pre-commit (Husky + lint-staged):** lint, format, typecheck (changed files), gitleaks scan, run unit tests for changed module.
- **Commit lint:** commitlint with Conventional Commits config.
- **Codeowners:** `CODEOWNERS` enforces module owners for review.
