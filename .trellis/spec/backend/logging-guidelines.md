# Logging Guidelines

> Which logger to call, what the output looks like, and what must never be logged.

---

## Overview

There are **three distinct log channels**. Picking the wrong one is the most
common mistake.

| Channel | API | Use for | Context? |
|---|---|---|---|
| Process logger | `logger.LogInfo` / `LogWarn` / `LogError` / `LogDebug` | Request-scoped events in handlers, services, relay | Yes — carries request id |
| System logger | `common.SysLog` / `common.SysError` | Startup, background jobs, process-level events | No |
| Access logger | `middleware.SetUpLogger` | HTTP access lines (`[GIN]`) | Gin-managed |

Rule of thumb: **if you have a `*gin.Context` or `context.Context` for a
request, use `logger.*`.** If you are in startup/init/background code with no
request, use `common.Sys*`.

There is **no structured logging library** (no zap / slog / logrus). Everything
is formatted strings.

---

## `logger` package API

Signatures in `logger/logger.go` — note that **only `LogWarn` and `LogDebug`
accept varargs**:

```go
func LogInfo(ctx context.Context, msg string)                      // no args
func LogWarn(ctx context.Context, msg string, args ...any)          // Sprintf-style
func LogError(ctx context.Context, msg string)                      // no args
func LogDebug(ctx context.Context, msg string, args ...any)         // Sprintf-style
```

For `LogInfo` / `LogError` you must pre-format:

```go
logger.LogError(ctx, "upstream failed: "+err.Error())
logger.LogWarn(ctx, "quota saturation on consume log: %s", clamp.Error())
```

Also exported: `LogQuota`, `FormatQuota`, `SetupLogger`, `GetCurrentLogPath`,
and `LogJson` (test-only).

`relaykit/` has its own mirror, `kitutil.Debug`
(`relaykit/relayconvert/kitutil/log.go`), gated by the same debug flag.

---

## Line format

Every line is emitted as:

```text
[LEVEL] timestamp | requestID | message
```

- Format string lives in `logger/logger.go`.
- The request id is read from `ctx.Value(common.RequestIdKey)`.
  `common.RequestIdKey` is the constant `"X-Oneapi-Request-Id"`
  (`common/constants.go`).
- When the context carries no request id, the literal default is **`SYSTEM`**.

### Log levels

| Function | Level tag | Notes |
|---|---|---|
| `LogDebug` | `DEBUG` | Only emitted when `common.DebugEnabled` is true |
| `LogInfo` | `INFO` | Normal lifecycle events |
| `LogWarn` | `WARN` | Anomalies worth investigating |
| `LogError` | `ERR` | Failures needing attention |

`common.DebugEnabled` is a package variable (`common/constants.go`) set from
`DEBUG == "true"` during init (`common/init.go`). There is **no separate
log-level env var** — debug is the only gate.

---

## File output and rotation

- Flag: `--log-dir` (`common/init.go`).
- `logger.SetupLogger()` opens `oneapi-<timestamp>.log` and replaces
  `gin.DefaultWriter` / `gin.DefaultErrorWriter` with an
  `io.MultiWriter(os.Stdout, fd)` — so output goes to both console and file.
- **Rotation is hand-rolled.** There is no lumberjack or similar dependency: once
  the file accumulates 1,000,000 lines, `SetupLogger` is invoked again through
  `gopool` to start a new timestamped file.

---

## Access logs

`middleware.SetUpLogger` (`middleware/logger.go`) produces the `[GIN]` HTTP
access lines. It **masks query strings on `/oauth/` and `/api/oauth/`** before
writing — do not add OAuth routes outside those prefixes, or the masking will
not apply.

---

## What to log

- Failed upstream relay calls and their status codes, via the relay path.
- Quota conversion saturation (`common.SysError` inside `quota_math.go`, plus
  `attachQuotaSaturation` at settlement — see
  [error-handling.md](./error-handling.md)).
- Billing pre-consume / settle / refund discrepancies.
- Auth failures (login, MFA, passkey, OAuth) with non-secret context.
- DB write errors and transaction rollbacks.

Real call sites to imitate:

```go
common.SysError(clamp.Error())                         // common/quota_math.go
logger.LogError(ctx, "...")                            // controller/relay.go
logger.LogWarn(ctx, fmt.Sprintf("..."), ...)           // service/log_info_generate.go
```

---

## What must never be logged

- **Passwords, verification codes, recovery codes, private keys, and usable
  session or auth tokens.**
- OAuth/OIDC client secrets and provider credentials.
- Full upstream API keys / channel credentials.

This is enforced by design in the audit path: `model/audit_log.go` documents that
"raw URLs, query strings, credentials and response/request bodies must never
enter this table", and stores access tokens only as
`AccessTokenFingerprint` (SHA-256) — never the bearer value. Error text returned
to clients is passed through `MaskSensitiveInfo` by `ToOpenAIError()` /
`ToClaudeError()`.

When adding an auth-related log line, record the non-secret context instead:
actor id, action, outcome, and a request id for correlation.

---

## Database-backed logs are a different thing

Request/consume logs are persisted in DB tables, not written to the process log:
`model/log.go` (`Log`, `RecordConsumeLog`, `RecordErrorLog`, `RecordLog`) and
`model/audit_log.go` (`AuditLog`, `RecordAuditLog`). Note that `AuditLog` also
targets ClickHouse when the log database is ClickHouse. Do not confuse these with
the process log described above.

---

## Common mistakes

- Calling `logger.LogInfo(ctx, "x: %d", n)` — `LogInfo` takes no varargs. Use
  `LogInfo(ctx, fmt.Sprintf("x: %d", n))`.
- Using `logger.*` in startup/background code with no request context; use
  `common.SysLog` / `common.SysError`.
- Assuming a log-level environment variable exists.
- Logging full tokens or credentials in an auth audit event.
- Adding OAuth routes outside `/oauth/` or `/api/oauth/` and losing query masking.
