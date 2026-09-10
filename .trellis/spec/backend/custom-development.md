# Custom Development & Upstream Preservation

> How to add fork-specific features without breaking future upstream merges.

---

## Repository boundary

This is a fork of the upstream NewAPI project carrying custom business features.
Three directories form the fork's boundary (`AGENTS.md` "NewAPI Custom
Development Rules"):

| Directory | Purpose |
|---|---|
| `custom/` | all custom business logic |
| `migrations/` | custom database schema changes and data migrations |
| `scripts/` | deployment, upgrade, backup, rollback, and maintenance scripts |

**Put new fork-specific work in these directories.** Upstream NewAPI directories
remain upstream-owned; touch them only where integration requires it.

> **Operational note:** git does not track empty directories, so each of these
> three directories carries a `.gitkeep` marker. **Do not delete the `.gitkeep`
> files** unless the directory gains real content, and keep the marker when a
> directory is temporarily emptied — removing it silently drops the directory
> from fresh clones and CI.

---

## Upstream relationship

| Remote | URL |
|---|---|
| `origin` | `git@github.com:Sdcode12/new-api.git` — this fork |
| `upstream` | `https://github.com/Calcium-Ion/new-api.git` — the source project |

The fork tracks upstream through a plain git remote. There is no sync script,
patch series, or vendored overlay — merges are ordinary git merges, which is
exactly why diff discipline matters.

---

## Upstream preservation principles

Active requirements from `AGENTS.md`:

- **Prefer implementing new business logic entirely under `custom/`.** Determine
  whether a requirement can be met without touching an upstream file.
- **When an upstream file must change, make the smallest possible integration
  change** — the smallest required section, not a tidy-up of surrounding code.
- **Do not copy upstream core implementations** into `custom/` for independent
  maintenance. Reuse existing upstream services, models, middleware,
  authorization, configuration, and utilities.
- **Do not refactor upstream code** for the purpose of adding a custom feature.
- **Do not replace or remove existing upstream behavior** unless explicitly
  required.
- **Design for mergeability**: a future `upstream` merge should produce minimal
  conflict.

### Existing code that predates the boundary

Several fork-specific files currently live directly in upstream directories
rather than in `custom/`:

- `controller/channel_upstream_update.go`
- `model/vendor_meta.go`
- `service/codex_channel_models.go`, `service/codex_credential_refresh.go`,
  `service/codex_credential_refresh_task.go`, `service/codex_models.go`,
  `service/codex_oauth.go`, `service/codex_wham_usage.go`

Treat these as legacy placement. **New** custom features belong in `custom/`.
Do not relocate the existing files as a side effect of an unrelated change — a
move is a large upstream diff and belongs in its own dedicated change.

### Upgradeability review questions

When reviewing a fork change, answer explicitly:

1. How much upstream code was modified?
2. Can any upstream modification be eliminated?
3. Was any upstream implementation unnecessarily duplicated?
4. Will this stay easy to merge after future upstream updates?

---

## Recommended integration shape

For every custom feature, separate it into:

1. **Business logic** under `custom/`.
2. **Schema changes** recorded under `migrations/`.
3. **Minimal upstream integration points** — route registration, navigation,
   permission wiring. Keep these to the smallest diff that works.
4. **Tests for observable behavior** — see the testing rules in
   [quality-guidelines.md](./quality-guidelines.md).

### Schema isolation

- New custom entities should use **new tables** whenever practical.
- Do not modify existing upstream schema without a corresponding migration.
- **Never edit an already-applied migration** — add a new one.
- All custom schema changes must preserve SQLite / MySQL / PostgreSQL
  compatibility. See [database-guidelines.md](./database-guidelines.md).

### Frontend isolation

- Reuse existing frontend components before creating new UI primitives — this is
  mandatory per `AGENTS.md`. Read `web/AGENTS.md` and search
  `web/src/components/` first.
- Modify upstream frontend files only for necessary routing, navigation,
  permissions, registration, or integration points.
- A new implementation of common UI behavior requires a concrete capability gap
  documented in the change summary; different text, size, or color is not a
  justification.

---

## Common mistakes

- Adding custom logic directly to an upstream file when `custom/` would do.
- Touching an upstream file "while you're in there" (gofmt, renames, import
  tidying). Every incidental change becomes a merge conflict later.
- Re-implementing an upstream helper instead of reusing it.
- Editing a migration that has already been applied.
- Assuming the three-database requirement does not apply to custom tables — it
  does.
- Deleting the `.gitkeep` markers, which drops the fork boundary from fresh
  clones and CI without any warning.
