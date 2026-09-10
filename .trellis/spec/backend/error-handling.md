# Error Handling

> Error types, the quota-safety contract, and how errors become HTTP responses.

---

## Overview

Errors surface to clients as **Gin JSON responses** with explicit HTTP status
codes. Business errors carry a user-readable message; internal failures are
logged server-side; relay errors are mapped to an OpenAI-compatible envelope so
clients written against OpenAI see consistent failures.

---

## Typed errors: quota saturation

The billing layer has one typed error, defined in `common/quota_math.go`:

```go
type QuotaClamp struct {
	Op       string
	Kind     QuotaClampKind
	Original float64
	Clamped  int
}
```

- `Error() string` — makes it usable as a fail-fast error.
- `AuditMap() map[string]any` — makes the same value usable as an audit marker.
- `QuotaClampKind` has constants `QuotaClampOverflow`, `QuotaClampUnderflow`,
  `QuotaClampNaN`.

This dual role is the point: a saturated conversion is both an error *and* a
record of what happened, so billing can fail closed without losing the evidence.

### Conversion helper surface

The real exported API in `common/quota_math.go`:

| Plain | `*Checked` (returns clamp) | `*Strict` (fails closed) |
|---|---|---|
| `QuotaFromFloat` | `QuotaFromFloatChecked` | `QuotaFromFloatStrict` |
| `QuotaRound` | `QuotaRoundChecked` | `QuotaRoundStrict` |
| `QuotaFromDecimal` | `QuotaFromDecimalChecked` | `QuotaFromDecimalStrict` |
| — | — | `WalletQuotaFromDecimalStrict` |

Plus `ValidateWalletQuota` and the bounds `MaxQuota`, `MinQuota`,
`MaxWalletQuota`.

**There are no `*Generic` variants.** Anything referencing those is stale.

`common/quota_math.go` uses bare numeric casts internally — that is by design.
Call sites must not.

---

## The quota-safety contract

Billing must never produce a negative charge from arithmetic overflow. Three
layers enforce this:

1. **Bound user-controlled multipliers at the edge.** Reject out-of-range
   quantities with **HTTP 400** before they reach the quota math:

   | Bound | Identifier | Location |
   |---|---|---|
   | Image count | `dto.MaxImageN` (= 128) | `relaykit/dto/openai_image.go` |
   | Video/task duration | `relaycommon.MaxTaskDurationSeconds` (= 3600) | `relay/common/relay_utils.go` |
   | `max_tokens` | `maxTokensLimit` (= `math.MaxInt32/2`, unexported) | `relay/helper/valid_request.go` |

   Note: `common/validate.go` only declares `var Validate *validator.Validate`.
   It does **not** hold these constants — a common mis-citation.

2. **Convert with saturating helpers**, never `int(float64(x) * ratio)` or
   `int(math.Round(x))`. Use `*Checked` when the clamp should be observable, and
   `*Strict` when the caller must fail closed.

3. **Audit every clamp.** `service/log_info_generate.go` →
   `attachQuotaSaturation()` calls
   `other.SetAdmin("quota_saturation", clamp.AuditMap())`, which lands at the
   JSON path `other.admin_info.quota_saturation`, and emits
   `logger.LogWarn(ctx, fmt.Sprintf("quota saturation on consume log: ..."))`.

   `SetAdmin` nests under `admin_info`, and `model.formatUserLogs` strips the
   entire `admin_info` object for non-admin readers — so the clamp detail is
   admin-only by construction.

---

## Relay errors

The relay error type is `types.NewAPIError`, defined in
`relaykit/types/error.go` (package `relaykit/types`, **not** the controller
package — it is not named `newAPIError`).

Its fields include `Err`, `RelayError`, `StatusCode`, `Metadata json.RawMessage`,
and unexported `errorType` / `errorCode` / `skipRetry` / `recordErrorLog`.

The OpenAI-compatible envelope is a **separate** struct, `OpenAIError`
(`Message`, `Type`, `Param`, `Code`, `Metadata`), produced by
`newAPIError.ToOpenAIError()`. Response sites:

```go
// controller/relay.go, controller/playground.go
c.JSON(newAPIError.StatusCode, gin.H{"error": newAPIError.ToOpenAIError()})
```

`ToOpenAIError()` / `ToClaudeError()` run error text through
`MaskSensitiveInfo` — keep upstream errors routed through these so credentials
never reach the client.

---

## Controller response shapes

```go
// success
c.JSON(http.StatusOK, gin.H{"input_tokens": inputTokens})

// client error — reject bad input before it reaches billing
c.JSON(http.StatusBadRequest, gin.H{
	"error": gin.H{"message": "invalid request: ...", "type": "invalid_request_error"},
})

// internal failure
c.JSON(http.StatusInternalServerError, gin.H{
	"error": gin.H{"message": "internal server error", "type": "internal_error"},
})
```

Conventions:

- **Success** — `c.JSON(http.StatusOK, gin.H{...})`.
- **Client error** — `StatusBadRequest` / `StatusNotFound` / `StatusNotImplemented`
  with a descriptive `error` object.
- **Relay** — mirror the OpenAI `{"error": {...}}` envelope.
- Use early returns; keep handlers flat (`AGENTS.md` "Common Code Quality").

---

## Common mistakes

- Swallowing errors from DB writes or billing math.
- Bare float→int quota casts on unbounded input. The real offenders outside
  `common/quota_math.go` are all legitimate (`common/utils.go` byte→MB
  formatting, `service/token_counter.go` token geometry) — do not add a new one.
- Citing `common/validate.go` as the home of the billing bounds. It is not.
- Referencing `*Generic` quota helpers that do not exist.
- Returning internal error detail to clients instead of logging it server-side.
- Ignoring a `*Checked` clamp instead of forwarding it to the audit path.
- Editing `relaykit/types/error.go` from the host module as if it were shared —
  it is a separate module.
