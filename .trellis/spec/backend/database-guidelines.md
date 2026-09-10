# Database Guidelines

> ORM usage, cross-dialect rules, and the migration mechanism.

---

## Overview

**All database code MUST work on SQLite, MySQL >= 5.7.8, and PostgreSQL >= 9.6.**
This is a hard requirement (`AGENTS.md`). A change is only "database compatible"
after it has been exercised against real instances of all three engines.

- **ORM**: GORM (`gorm.io/gorm v1.25.12` — the v2 API line).
- **Drivers**: sqlite `github.com/glebarez/sqlite v1.11.0` (**not**
  `gorm.io/driver/sqlite`), mysql `gorm.io/driver/mysql v1.5.7`,
  postgres `gorm.io/driver/postgres v1.5.9`,
  clickhouse `gorm.io/driver/clickhouse v0.6.0`.
- **ClickHouse is log-database only.** `model/main.go` `chooseDB()` rejects a
  ClickHouse DSN for the main database; it is accepted only when `isLog` is true.
- **Log database** is configured separately from the main database.

---

## Initialization map

| Concern | Location |
|---|---|
| Main DB init | `model/main.go` → `InitDB()` |
| Log DB init | `model/main.go` → `InitLogDB()` |
| DSN selection | `model/main.go` → `chooseDB(envName, isLog)` |
| Env vars | `SQL_DSN` (main), `LOG_SQL_DSN` (log) |
| Dialect selection | DSN prefix: `postgres://`, `clickhouse://`, `local`; otherwise MySQL |
| Connection pool | `SetMaxIdleConns` / `SetMaxOpenConns` / `SetConnMaxLifetime` in `model/main.go` |
| Migration entrypoint | `model/main.go` → `InitDB()` → `migrateDB()` → `DB.AutoMigrate(...)` |
| MySQL charset precheck | `model/main.go` → `checkMySQLChineseSupport()` |

MySQL DSNs are auto-augmented with `parseTime=true` by `chooseDB()` — do not
hand-write `parseTime` at call sites.

Runtime branching between the two databases uses
`common.UsingMainDatabase(...)` / `common.UsingLogDatabase(...)`
(`common/database.go`), set by `SetMainDatabaseType` / `SetLogDatabaseType`.

---

## Reserved words and dialect quoting

`group` and `key` are reserved. Never interpolate them as bare identifiers.
`model/main.go` defines the quoting helpers, filled in by `initCol()`:

- `commonGroupCol`, `commonKeyCol` — main database
- `logGroupCol`, `logKeyCol` — log database

PostgreSQL gets `"group"` / `"key"`; MySQL and SQLite get `` `group` `` /
`` `key` ``. Use these in every query that references those columns
(`model/ability.go`, `model/channel.go`, `model/log.go`,
`model/model_pricing_config.go` are the reference usages).

`model/main.go` also defines `commonTrueVal` / `commonFalseVal` for dialect
boolean literals. **These are currently unreferenced anywhere in the repo** — do
not introduce new uses without first confirming they are still intended.

---

## Query patterns

Prefer GORM query methods (`Create`, `Find`, `Where`, `Updates`, …) over raw SQL.
Raw SQL does appear and is acceptable for `information_schema` probes, ClickHouse
DDL, and `PRAGMA table_info` — not for ordinary CRUD.

### Row locking

Use the helper, never the legacy GORM v1 form:

```go
// model/locking.go
func lockForUpdate(tx *gorm.DB) *gorm.DB {
	return tx.Clauses(clause.Locking{Strength: "UPDATE"})
}
```

- Emits `FOR UPDATE` for MySQL/PostgreSQL and is a no-op on SQLite.
- Do **not** duplicate `clause.Locking` at call sites.
- The GORM v1 pattern `tx.Set("gorm:query_option", "FOR UPDATE")` is **silently
  ignored** by GORM v2 and acquires no lock. It appears nowhere in real code
  except a comment in `model/locking.go` — keep it that way.
- Dialect-specific locks with different semantics (e.g. MySQL next-key/gap locks)
  need explicit database-type branches with valid fallbacks for every engine.

### Transactions

There is no custom transaction wrapper. Use GORM's built-in form:

```go
model.DB.Transaction(func(tx *gorm.DB) error { ... })
```

Reference usages: `model/account_security.go:26`, `model/auth_flow.go:176`,
`model/checkin.go:96`.

---

## Migrations

There are **two** migration paths, and they are not interchangeable:

| Path | Location | Scope |
|---|---|---|
| Upstream schema | Go code in `model/` (`InitDB` → `migrateDB` → `AutoMigrate`) | The project's own tables |
| Fork-custom schema | `migrations/` | Custom tables and data changes only |

Upstream migrations are Go code executed at startup, not files in a migrations
directory. **Do not add upstream-table changes to `migrations/`, and do not put
custom-table changes into `model/migrateDB()`** — that would edit an
upstream-owned file and create a merge conflict on the next upstream sync.

### Upstream path (`model/`)

1. **`migrateDB()` in `model/main.go` runs hand-written migrations first, then
   `DB.AutoMigrate(...)`.** Add new schema changes in that sequence.
2. **SQLite requires `ALTER TABLE ... ADD COLUMN`; it does not support
   `ALTER COLUMN`.** See `ensureSubscriptionPlanTableSQLite()` in `model/main.go`.
3. **MySQL/PostgreSQL paths DO use `ALTER`/`MODIFY COLUMN`** — but only after an
   early `if <dialect is SQLite> { return }` guard. Reference:
   `migrateTokenModelLimitsToText()` and `migrateSubscriptionPlanPriceAmount()`
   in `model/main.go`.
4. **`model/migration_dialector.go` defines `mysqlMigrationDialector` and
   `postgresMigrationDialector`**, which override `MigrateColumn` to stop GORM
   from issuing spurious `ALTER`s driven by tag drift.

### The established migration idiom (read this before designing anything)

Upstream migrations are **Go functions**, not SQL files, and there is **no
version-ledger table** anywhere in the repo. Each migration is called
unconditionally on every startup and decides for itself whether it still has
work to do:

```go
// model/main.go
migrateTokenKeyUniqueness(DB)
migratePrefillGroupUniqueness(DB)
migrateSubscriptionPlanPriceAmount()
migrateTokenModelLimitsToText()
migrateOptionPrimaryKey(DB)
DB.AutoMigrate(...)
```

Idempotency is **self-checking**, by schema inspection:

- PostgreSQL / MySQL: query `information_schema.columns`, return `nil` early if
  the column is already the target type (see `migrateTokenModelLimitsToText`).
- SQLite: `DB.Migrator().HasTable(...)` and `PRAGMA table_info`, e.g.
  `ensureSubscriptionPlanTableSQLite()`.
- Dialect branches are explicit via
  `common.UsingMainDatabase(common.DatabaseTypePostgreSQL / DatabaseTypeMySQL /
  DatabaseTypeSQLite)`; unsupported dialects return `nil`.

Naming follows `<verb><Thing>`: `migrateTokenModelLimitsToText`,
`migrateOptionPrimaryKey`, `ensureSubscriptionPlanTableSQLite`.

**This idiom is why `migrations/` has no format of its own to invent.** Custom
work should extend it, not parallel it.

### Design SQL files only if you are willing to build a runner

The three-database requirement makes SQL migration files expensive: each
migration needs either three dialect variants or dialect branches inside the SQL,
plus a runner, an ordering system, and a version ledger. Go functions get all of
that from GORM's `Migrator` and `common.UsingMainDatabase` for free. Do not
introduce SQL files without a concrete reason the Go idiom cannot serve.

### Fork-custom path (`migrations/`)

`migrations/` currently holds only a `.gitkeep`. **The per-migration content is
deliberately undecided** and should be designed when the first real custom
migration arrives. That is safe, because the *mechanism* is already settled:

**Locked now (cheap, and expensive to get wrong later):**

1. **Location** — custom migrations are Go functions under `custom/`, not SQL
   files and not added to the body of `model/migrateDB()`.
2. **Invocation** — `model/main.go` gains exactly **one** line calling a single
   exported custom entry point. This is the minimal upstream integration point;
   keep it to one line.
3. **Idempotency** — reuse the self-checking idiom above. Do not add a ledger
   preemptively.
4. **Dialect handling** — branch with `common.UsingMainDatabase`, and return
   `nil` for dialects the migration does not need to touch.

**Deferred until the first real migration:**

- What each migration does.
- Shared helpers for backfill / rename, if a pattern repeats.
- Any naming or numbering convention beyond the existing `<verb><Thing>` style.

**Trigger to revisit — and the only thing that forces a design change:** the
first **non-additive** migration. Self-checking works when the migration is
purely additive (a new table, a new column, a widened column) because the current
schema proves whether it ran. It **cannot** work for:

- data backfills — the schema looks identical before and after,
- column renames — inspecting the schema cannot distinguish "not yet renamed"
  from "already renamed",
- column drops.

The first of those requires a real version ledger (a `schema_migrations` table
recording applied IDs) and an ordered runner. Decide it then, with the concrete
migration in hand, and record the decision here.

Also binding regardless of format:

- Custom schema changes go in `migrations/`, **never** into `model/migrateDB()`.
- New custom entities get **new tables**; avoid altering upstream tables.
- **Never edit a migration that has already been applied** — add a new one.
- Every custom migration must run on SQLite, MySQL, and PostgreSQL.
- Custom migrations must run **after** the main DB connection is initialized.

Until a real migration exists, treat the gap as intentional, not as an oversight
to fill in.

### Applies to both paths

5. **Three-database verification is mandatory** for anything that can affect DB
   behavior: ORM/driver deps, DSN handling, model + GORM tags, migrations,
   constraints/indexes, `Scanner`/`Valuer`, raw SQL, transactions, row locking.
   Mocks, a successful build, or testing one dialect are **not** substitutes.
6. **Migrate twice to prove idempotency** — once on a fresh database and once by
   upgrading a database created by the latest released version. Startup must be
   run at least twice.
7. **Treat GORM core + dialect drivers as a compatible version set.** Never
   upgrade only the core and assume drivers still work — re-run the full matrix.
8. **Never edit a migration that has already shipped.** Add a new one.

---

## Naming and tags

- Tables use GORM's default pluralized naming (`AutoMigrate(&Channel{})` →
  `channels`).
- `gorm:"column:..."` overrides are used where the Go name differs. Examples:
  `model/channel.go`, `model/log.go`, `model/audit_log.go`.
- **Avoid `gorm:"default:true"` / `default:false` as a business-rule default.**
  MySQL and PostgreSQL normalize boolean defaults differently, and GORM
  `AutoMigrate` will re-issue `ALTER TABLE` on every restart. Set defaults in
  request/model normalization, hooks, or service logic instead.
  Two violations remain in the tree — `model/subscription.go` and
  `model/custom_oauth_provider.go`. Do not add more.

### Primary keys

Let GORM generate primary keys; do not hand-write `AUTO_INCREMENT` or `SERIAL`.
Note `model/task.go` still carries `gorm:"primary_key;AUTO_INCREMENT"` — a known
legacy exception. Do not copy it into new models.

---

## JSON

Business code should marshal through the wrappers in `common/json.go`:
`common.Marshal`, `common.Unmarshal`, `common.UnmarshalJsonStr`,
`common.DecodeJson`, `common.GetJsonType`. Inside `relaykit/`, use the equivalent
`relaykit/relayconvert/kitutil/json.go`.

`json.RawMessage` and `json.Number` are fine as **types**; the rule concerns
direct calls to `encoding/json`.

**Reality check**: this rule is aspirational, not enforced. 27 non-test files
still call `encoding/json` directly (`controller/channel.go`, `common/str.go`,
`common/utils.go`, `service/midjourney.go`, `relay/channel/*`, `pkg/ionet/*`, …).
For new code, use the wrappers. Do not mass-migrate legacy call sites.

---

## Common mistakes

- Using `gorm.io/driver/sqlite` — this project uses `github.com/glebarez/sqlite`.
- GORM v1 `tx.Set("gorm:query_option", "FOR UPDATE")` — no lock is acquired.
- Interpolating `group` / `key` without `commonGroupCol` / `commonKeyCol`.
- SQLite migrations using `ALTER COLUMN`.
- Assuming a ClickHouse DSN is valid for the main database.
- Assuming dialect compatibility after upgrading only `gorm.io/gorm`.
- Adding `gorm:"default:true"` and causing `ALTER TABLE` churn on every boot.
