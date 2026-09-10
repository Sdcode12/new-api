# Bootstrap Task: Fill Project Development Guidelines

**You (the AI) are running this task. The developer does not read this file.**

The developer just ran `trellis init` on this project for the first time.
`.trellis/` now exists with empty spec scaffolding, and this bootstrap task
exists under `.trellis/tasks/`. When they want to work on it, they should start
this task from a session that provides Trellis session identity.

**Your job**: help them populate `.trellis/spec/` with the team's real
coding conventions. Every future AI session — this project's
`trellis-implement` and `trellis-check` sub-agents — auto-loads spec files
listed in per-task jsonl manifests. Empty spec = sub-agents write generic
code. Real spec = sub-agents match the team's actual patterns.

Don't dump instructions. Open with a short greeting, figure out if the repo
has any existing convention docs (CLAUDE.md, .cursorrules, etc.), and drive
the rest conversationally.

---

## Status (update the checkboxes as you complete each item)

- [x] Fill backend guidelines
- [x] Add code examples

### What was done (2026-09-10, redo)

Every spec file was re-derived from the codebase rather than carried over. Each
rule was checked against a real file; claims that no longer matched the code were
corrected and the divergence recorded in the spec itself.

Corrections applied to the pre-existing drafts:

- `directory-structure.md` — the "`model/` is the only DB layer" claim was false
  (controllers and services issue GORM queries directly). Documented as
  aspirational with real counterexamples. Removed the false "kebab-case in
  `model/`" rule. Added `types/`, `constant/`, `oauth/` and the real router
  function list.
- `database-guidelines.md` — SQLite driver is `github.com/glebarez/sqlite`, not
  `gorm.io/driver/sqlite`. `ALTER COLUMN` is SQLite-only; MySQL/PostgreSQL paths
  do use it behind a dialect guard. Documented the real migration entrypoint and
  `model/migration_dialector.go`.
- `error-handling.md` — removed non-existent `*Generic` quota helpers; replaced
  the invented `newAPIError` with the real `types.NewAPIError`; corrected
  `common/validate.go`, which holds no billing bounds.
- `logging-guidelines.md` — `LogInfo` and `LogError` take **no** varargs (only
  `LogWarn`/`LogDebug` do). Corrected the rotation mechanism (hand-rolled, no
  lumberjack).
- `quality-guidelines.md` — reframed `interface{}` and direct `encoding/json` as
  aspirational with real violation counts instead of absolute bans.

New files added beyond the original template:

- `custom-development.md` — the `AGENTS.md` custom-development rules and the
  `custom/` · `migrations/` · `scripts/` boundary. A later correction pass
  updated this file after those three directories were created; the spec now
  documents them as the fork boundary and records that they must not be left
  empty, because git does not track empty directories.
- `security-guidelines.md` — the OWASP mandate plus a reality map, with four
  concrete control gaps recorded (no CSRF-token middleware, no per-account
  lockout, tokens stored unhashed, partial error localization).
- `i18n-guidelines.md` — backend has 3 YAML locales (not "en, zh"); frontend has
  7 JSON locales.

Manifests: `implement.jsonl` (10 entries) and `check.jsonl` (12 entries) created
and validated with `task.py validate`.

Verification: `grep -rn "To be filled\|TODO: fill\|placeholder" .trellis/spec`
returns nothing.

---

## Spec files to populate


### Backend guidelines

| File | What to document |
|------|------------------|
| `.trellis/spec/backend/directory-structure.md` | Where different file types go (routes, services, utils) |
| `.trellis/spec/backend/database-guidelines.md` | ORM, migrations, query patterns, naming conventions |
| `.trellis/spec/backend/error-handling.md` | How errors are caught, logged, and returned |
| `.trellis/spec/backend/logging-guidelines.md` | Log levels, format, what to log |
| `.trellis/spec/backend/quality-guidelines.md` | Code review standards, testing requirements |
| `.trellis/spec/backend/custom-development.md` | Upstream preservation and the fork's actual boundary state |
| `.trellis/spec/backend/security-guidelines.md` | OWASP auth mandate, auth reality map, known control gaps |
| `.trellis/spec/backend/i18n-guidelines.md` | Backend YAML locales and frontend i18next conventions |


### Thinking guides (already populated)

`.trellis/spec/guides/` contains general thinking guides pre-filled with
best practices. Customize only if something clearly doesn't fit this project.

---

## How to fill the spec

### Step 1: Import from existing convention files first (preferred)

Search the repo for existing convention docs. If any exist, read them and
extract the relevant rules into the matching `.trellis/spec/` files —
usually much faster than documenting from scratch.

| File / Directory | Tool |
|------|------|
| `CLAUDE.md` / `CLAUDE.local.md` | Claude Code |
| `AGENTS.md` | Codex / Claude Code / agent-compatible tools |
| `.cursorrules` | Cursor |
| `.cursor/rules/*.mdc` | Cursor (rules directory) |
| `.windsurfrules` | Windsurf |
| `.clinerules` | Cline |
| `.roomodes` | Roo Code |
| `.github/copilot-instructions.md` | GitHub Copilot |
| `.vscode/settings.json` → `github.copilot.chat.codeGeneration.instructions` | VS Code Copilot |
| `CONVENTIONS.md` / `.aider.conf.yml` | aider |
| `CONTRIBUTING.md` | General project conventions |
| `.editorconfig` | Editor formatting rules |

### Step 2: Analyze the codebase for anything not covered by existing docs

Scan real code to discover patterns. Before writing each spec file:
- Find 2-3 real examples of each pattern in the codebase.
- Reference real file paths (not hypothetical ones).
- Document anti-patterns the team clearly avoids.

### Step 3: Document reality, not ideals

**Critical**: write what the code *actually does*, not what it should do.
Sub-agents match the spec, so aspirational patterns that don't exist in the
codebase will cause sub-agents to write code that looks out of place.

If the team has known tech debt, document the current state — improvement
is a separate conversation, not a bootstrap concern.

---

## Quick explainer of the runtime (share when they ask "why do we need spec at all")

- Every AI coding task spawns two sub-agents: `trellis-implement` (writes
  code) and `trellis-check` (verifies quality).
- Each task has `implement.jsonl` / `check.jsonl` manifests listing which
  spec files to load.
- The platform hook auto-injects those spec files + the task's `prd.md`
  into every sub-agent prompt, so the sub-agent codes/reviews per team
  conventions without anyone pasting them manually.
- Source of truth: `.trellis/spec/`. That's why filling it well now pays
  off forever.

---

## Completion

When the developer confirms the checklist items above are done with real
examples (not placeholders), guide them to run:

```bash
python ./.trellis/scripts/task.py finish
python ./.trellis/scripts/task.py archive 00-bootstrap-guidelines
```

After archive, every new developer who joins this project will get a
`00-join-<slug>` onboarding task instead of this bootstrap task.

---

## Suggested opening line

"Welcome to Trellis! Your init just set me up to help you fill the project
spec — a one-time setup so every future AI session follows the team's
conventions instead of writing generic code. Before we start, do you have
any existing convention docs (CLAUDE.md, .cursorrules, CONTRIBUTING.md,
etc.) I can pull from, or should I scan the codebase from scratch?"
