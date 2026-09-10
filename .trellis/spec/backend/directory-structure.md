# Directory Structure

> Where backend code lives and which layer owns what.

---

## Overview

This is an **AI API gateway/proxy** written in Go. It aggregates 40+ upstream AI
providers (OpenAI, Claude, Gemini, Azure, AWS Bedrock, …) behind a unified API,
with user management, billing, rate limiting, and an admin dashboard.

- Root module: `github.com/QuantumNous/new-api` (see `go.mod`)
- Go version: 1.25.1
- `relaykit/` is a **second, independent module**: `github.com/QuantumNous/new-api/relaykit`
- Frontend `web/` is a separate Bun/React project — see `web/AGENTS.md`

## Top-level layout

| Directory | Owns |
|-----------|------|
| `router/` | HTTP route registration + middleware wiring |
| `middleware/` | Gin middleware: auth, rate limit, body limits, audit, CORS, gzip |
| `controller/` | HTTP handlers — parse request, call services, write response |
| `service/` | Business logic spanning multiple models (billing, sessions, authz, logging) |
| `model/` | GORM models, schema definition, and the bulk of data access |
| `relay/` | Upstream provider relay: channel adaptors, helpers, common handlers |
| `relaykit/` | Independent module: relay DTOs, protocol conversion, kit utilities |
| `setting/` | Feature-config domain packages (`setting/<feature>_setting/`) |
| `types/` | Shared value types (e.g. `PriceData`) used across layers |
| `dto/` | Request/response transfer objects for the public API |
| `constant/` | Shared constants (channel types, request-id keys, limits) |
| `common/` | Shared helpers: JSON, quota math, crypto, DB setup, i18n bridge |
| `oauth/` | OAuth/OIDC provider implementations |
| `pkg/` | Self-contained libraries: `billingexpr`, `cachex`, `ionet`, `jsplugin`, `perf_metrics` |
| `plugins/tasks/` | JavaScript task plugins executed by Sobek |
| `logger/` | Custom Gin-based process logger |
| `i18n/` | Backend translations (YAML locale files) |
| `web/`, `electron/` | Frontend + desktop wrapper (separate builds) |
| `custom/` | **Fork-only** custom business logic (see [custom-development.md](./custom-development.md)) |
| `migrations/` | **Fork-only** custom schema changes and data migrations |
| `scripts/` | **Fork-only** deployment, upgrade, backup, rollback, maintenance |

Do **not** invent other new top-level directories for a feature. Put the code in
the layer that owns it (see "Where new code goes").

---

## Layer ownership

### `relay/` must not touch the database

`relay/` is transport and protocol conversion only. It must not import `model/`
or issue GORM queries. Verified: no non-test file under `relay/` references the
DB layer.

### `model/` owns the schema, but is NOT an exclusive data-access gate

`model/` defines every GORM model and the migration logic. It also holds most
queries. However, **in practice `controller/` and `service/` do issue GORM
queries directly** — the "only `model/` touches the DB" rule is aspirational,
not enforced. Real examples of direct access outside `model/`:

- `controller/channel.go:87` — `model.DB.Model(&model.Channel{})`
- `controller/access_token.go`, `controller/oauth.go`, `controller/setup.go`
- `service/security_verification.go`, `service/user_notify.go`, `service/authz/*`, `service/passkey/*`

Guidance for new work:

- Prefer putting new queries in `model/`. It keeps dialect handling in one place.
- Do **not** refactor existing direct queries out of `controller/`/`service/`
  as a side effect of an unrelated change. That inflates the diff and hurts
  upstream merges.
- `relay/` is the one hard boundary — never add DB access there.

### `router/` only wires

Route groups are registered from `SetRouter` in `router/main.go`, which calls
`SetApiRouter`, `SetRelayRouter`, `SetDashboardRouter`, `SetVideoRouter`,
`SetTaskRouter`, `SetPluginRouter`, `SetTaskPluginProtocolRouter`,
`SetAuthzRouter`, and `SetWebRouter`. Each lives in its own `router/*.go`
file: `api-router.go`, `relay-router.go`, `channel-router.go`,
`dashboard.go`, `video-router.go`, `task-router.go`, `plugin-router.go`,
`task-plugin-protocol-router.go`, `authz-router.go`, `web-router.go`.

Add a route by extending the matching group function — not by registering
handlers anywhere else.

### `relaykit/` is an independent module

`relaykit/` must never import root-module packages (`common`, `model`,
`controller`, …). Its JSON helpers live in
`relaykit/relayconvert/kitutil/` (`json.go`, `log.go`, `mask.go`, `value.go`).
Verify isolation with:

```bash
cd relaykit && GOWORK=off go build ./...
```

---

## Naming conventions

- **`model/` files are flat domain names — never kebab-case.** Real names:
  `user.go`, `channel.go`, `task.go`, `log.go`, `user_session.go`. There are no
  hyphenated filenames in `model/`.
- **Router files** use kebab-case and the `*-router.go` suffix when they map to a
  route group (`channel-router.go`, `api-router.go`), but some groups use a bare
  domain name (`dashboard.go`). Follow the file you are extending.
- **Tests** sit next to the source in the same package with a `_test.go` suffix
  (e.g. `model/option_auto_group_test.go`,
  `controller/relay_count_tokens_test.go`). 259 test files exist repo-wide.
- **Setting features**: `setting/<feature>_setting/`, e.g.
  `setting/billing_setting/`, `setting/console_setting/`.

---

## Where new code goes

| You are adding… | Put it in |
|---|---|
| A **fork-specific** feature | `custom/` — not an upstream layer |
| A **fork-specific** schema change | `migrations/` |
| A **fork-specific** ops script | `scripts/` |
| A new HTTP endpoint | handler in `controller/<resource>.go` + route in `router/<group>-router.go` |
| Logic spanning several models | `service/<domain>.go` |
| A new table | model + queries in `model/<entity>.go` |
| A user-tunable feature flag | `setting/<feature>_setting/` |
| A new upstream provider | `relay/channel/<provider>/` |
| Reusable leaf library | `pkg/<name>/` |
| A relay protocol DTO or conversion | `relaykit/` (respect module isolation) |

**Fork-specific work goes in the fork boundary instead.** `custom/`,
`migrations/`, and `scripts/` are reserved for custom business logic, custom
schema changes, and deployment/maintenance scripts respectively. Do not add
fork-specific code to an upstream layer when `custom/` would do. See
[custom-development.md](./custom-development.md).

---

## Reference files

- Layer wiring: `router/main.go`, `router/relay-router.go`
- Thin-ish handlers: `controller/token.go`, `controller/channel.go`
- Data access: `model/main.go`, `model/locking.go`, `model/user_session.go`
- Independent module: `relaykit/go.mod`, `relaykit/relayconvert/kitutil/json.go`
- Shared types: `types/price_data.go`
