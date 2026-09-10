# Backend Development Guidelines

> Coding guidelines for the Go gateway in this repository.

These specs describe **this repository as it exists now**. Where a stated rule and
the current code diverge, the divergence is called out explicitly so you do not
write code that assumes a protection or convention is in force.

---

## Guidelines index

| Guide | Covers |
|---|---|
| [Directory Structure](./directory-structure.md) | Top-level layout, layer ownership, naming, where new code goes |
| [Database Guidelines](./database-guidelines.md) | GORM, dialect compatibility, migrations, locking, JSON wrappers |
| [Error Handling](./error-handling.md) | Quota-clamp typed errors, billing safety bounds, relay error envelope |
| [Logging Guidelines](./logging-guidelines.md) | Which logger to call, line format, secrets that must never be logged |
| [Quality Guidelines](./quality-guidelines.md) | Modern Go conventions, forbidden patterns, testing rules, governance |
| [Custom Development](./custom-development.md) | The `custom/` · `migrations/` · `scripts/` fork boundary and upstream preservation |
| [Security & Authentication](./security-guidelines.md) | OWASP mandate, auth reality map, known control gaps |
| [Internationalization](./i18n-guidelines.md) | Backend YAML locales, frontend i18next conventions |

---

## How these specs are used

Every AI coding task spawns `trellis-implement` (writes code) and `trellis-check`
(verifies quality). Each task's `implement.jsonl` / `check.jsonl` manifests decide
which of the files above get loaded. The platform hook injects the selected specs
plus the task's `prd.md` into the sub-agent prompt automatically.

That means the specs here are the only conventions a sub-agent will see. Keeping
them accurate — including the "this rule is aspirational" notes — is what stops
sub-agents from writing code that looks out of place.

---

## Primary source

`AGENTS.md` at the repository root is the authoritative rule set. These files
extract and organize it for backend work; if they ever conflict, `AGENTS.md` wins.
Frontend-specific conventions live in `web/AGENTS.md`.

---

**Language**: All documentation in this directory is written in **English**.
