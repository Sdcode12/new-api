# Security & Authentication

> OWASP-mandatory authentication work — and where this codebase differs from the ideal.

---

## The mandate

`AGENTS.md` requires that **any** implementation, modification, or review touching
authentication comply with the latest stable **OWASP ASVS** and the relevant
**OWASP Cheat Sheet Series** (Authentication, Session Management, Password
Storage, Forgot Password, MFA, OAuth, CSRF).

Scope: registration, login/logout, password change and recovery, email
verification, MFA, WebAuthn/Passkeys, OAuth/OIDC, account linking and unlinking,
sessions, JWTs, API credentials, and re-authentication for sensitive actions.

Non-negotiables:

- **Enforce on the server.** Frontend checks never substitute for server-side
  enforcement, and recovery or alternative login paths must not bypass the
  required assurance.
- **Existing code is not a justification** for retaining or introducing an
  insecure pattern.
- **Audit events must exclude** passwords, verification codes, recovery codes,
  private keys, and usable session/auth tokens — while still recording enough
  non-secret context to investigate.
- Verify with **focused regression tests** covering failure, expiry, replay, and
  bypass. Record OWASP references (ASVS version + requirement IDs) and any
  unresolved gaps in the change summary. **Do not claim compliance while an
  applicable requirement is unmet.**

---

## Reality map

| Control | Where it lives |
|---|---|
| Dashboard auth | `middleware/auth.go` — `UserAuth` / `AdminAuth` / `RootAuth` → `authHelper` → `classifyDashboardCredential` → `service.ParseDashboardAccessToken`, `service.ValidateLoginSession`, `model.ValidateAccessToken` |
| Relay auth | `middleware/auth.go` — `TokenAuth` / `TokenOrUserAuth` / `TokenAuthReadOnly` |
| Sessions | `model/user_session.go` (status `active`/`revoking`/`revoked`, HMAC `RefreshHash`, rotation, reuse detection, revoke, deny-fence tombstones) |
| Session issuance | `service/auth_session.go` — `CreateLoginSession`, `ValidateLoginSession`, `RefreshCookieName` |
| Session endpoints / cookie | `controller/auth_session.go`, `common/session_cookie.go` |
| Session invalidation | Per-user `AuthVersion`; password change advances it (`model/account_security.go`) |
| Relay tokens | `model/token.go` — `Token.Key`, `GetTokenByKey`, `ValidateUserToken` |
| Personal access tokens | `model/user.go` — `ValidateAccessToken` |
| Password hashing | `common/crypto.go` (`Password2Hash`, `ValidatePasswordAndHash`, bcrypt) and `common/account_password.go` (argon2id, `HashAccountPassword`, env `ACCOUNT_PASSWORD_HASH_ALGORITHM`, 8–128 chars) |
| Browser-side password transport | `common/password_crypto.go` (RSA-OAEP / AES-GCM), `model/password_crypto.go` |
| MFA / TOTP | `model/twofa.go` (`ValidateTOTPAndUpdateUsage`, `ValidateBackupCodeAndUpdateUsage`), `service/twofa.go`, `controller/twofa.go`, `model/twofa_enrollment.go`; routes `/login/2fa`, `/login/verify` |
| WebAuthn / Passkeys | `service/passkey/` (`service.go` enforces RPID/origin/HTTPS, `session.go`, `user.go`), `controller/passkey.go`, `model/passkey.go`, library `go-webauthn/webauthn` |
| OAuth / OIDC providers | `oauth/` — `github.go`, `discord.go`, `linuxdo.go`, `telegram.go`, `oidc.go`, `generic.go`, `provider.go`, `registry.go`, `types.go` |
| Account linking | `controller/oauth.go` (`HandleOAuth`, `handleOAuthBind`, `findOrCreateOAuthUser`), `model/user_oauth_binding.go`, `model/account_security.go` (`UnbindUserOAuthForSession`) |
| Authorization | `service/authz/` (Casbin) — `permission.go`, `registry.go`, `role.go` (built-in `root`/`admin`), `seed.go`, `enforcer.go`, `adapter.go`, `assignment.go`, `resolver.go`, `override.go`, `resources_*.go` |
| Route protection | `middleware/auth.go` → `RequirePermission`, with per-route `authz.*` constants in `router/` |
| Re-authentication | "security proof": `service/auth_token.go` (`IssueSecurityProof`, `verifySecurityProof`), `service/security_verification.go` (`ConsumeOperationProof`), `middleware/secure_verification.go` (`SecureVerificationRequired`, `RequireSecurityProof`, `X-Security-Proof` header) |
| Auth audit | `middleware/audit.go` (`TokenOperationAudit`, `AccessTokenAudit`), `model/audit_log.go` (`RecordAuditLog`), test `controller/access_token_audit_test.go` |

Password verification accepts **both** bcrypt and argon2id hashes — dispatch is by
the `$argon2id$` prefix. Do not remove bcrypt support; existing accounts depend
on it.

### Token storage — read before touching credentials

Relay token keys (`model/token.go`) and dashboard PATs (`model/user.go`) are
stored as the **opaque secret value itself**, in a uniquely-indexed column, and
looked up by direct equality. They are **not** salted-hashed. The only derived
value is a SHA-256 fingerprint used for audit correlation
(`model/audit_log.go` → `AccessTokenFingerprint`), which never stores the bearer.

This is workable for high-entropy random secrets, but it means a database read
exposes usable credentials. If you are adding a new credential type, prefer
storing a hash and comparing digests. Do not silently change the storage scheme of
existing tokens — that is a migration.

---

## Known gaps

These are real. Do not write code that assumes the stronger control exists, and do
not claim ASVS compliance in these areas.

1. **No CSRF-token middleware.** There is no double-submit or synchronizer-token
   implementation. Protection relies on `middleware/SessionCookieOriginGuard`
   (registered in `router/api-router.go`), an HttpOnly/SameSite refresh cookie,
   and Cloudflare **Turnstile** (`middleware/turnstile-check.go`) on `/login`,
   `/register`, `/reset_password`, and check-in. Origin checking is not the same
   control as a CSRF token — treat this as an open item for any new
   state-changing endpoint reached by a cookie.
2. **No per-account lockout or failed-attempt counter.** `/login` is protected by
   a *global* `middleware.CriticalRateLimit()` (20 requests / 20 min,
   `common/constants.go`) plus Turnstile. There is no per-account throttling, so a
   distributed attacker is not slowed per account. Email/SMS verification codes
   (`common/verification.go`) are in-memory with a 10-minute TTL and a capped map
   — that is not a lockout either.
3. **API tokens and PATs are not hashed at rest** (see above).
4. **Server-side error localization is partial.** Many `service/` and `model/`
   returns still use hardcoded strings (e.g. `model/token.go`,
   `common/verification.go`) rather than i18n keys. See
   [i18n-guidelines.md](./i18n-guidelines.md).

---

## Rules for new auth work

- Put server-side enforcement in `middleware/` + `service/`. A frontend guard is
  never the control.
- Reuse the existing primitives rather than inventing parallels: session
  rotation via `service/auth_session.go`, permission checks via
  `RequirePermission` + `service/authz/`, re-auth via the security-proof path.
- Any new cookie-authenticated state-changing endpoint must consider CSRF —
  check that `SessionCookieOriginGuard` actually covers its route.
- Rate-limit and add an upper bound to every value that gates a sensitive action.
  Remember the `*uint` hazard from `AGENTS.md`: a `>= 0` check is not enough.
- Advance `AuthVersion` for credential changes so existing sessions die.
- Log non-secret context only (actor, action, outcome, request id). Never log the
  token, code, recovery code, or private key.
- Add focused regression tests for failure, expiry, replay, and bypass cases.
  Follow the test-scatter rules in [quality-guidelines.md](./quality-guidelines.md).
- If a required control is missing, say so in the change summary instead of
  implying it is handled.

---

## Reference files

- `middleware/auth.go`, `middleware/audit.go`, `middleware/secure_verification.go`
- `service/auth_session.go`, `service/auth_token.go`, `service/security_verification.go`, `service/twofa.go`
- `service/authz/enforcer.go`, `service/authz/registry.go`
- `model/user_session.go`, `model/token.go`, `model/twofa.go`, `model/audit_log.go`
- `common/account_password.go`, `common/crypto.go`, `common/verification.go`
- `controller/access_token_audit_test.go`
