# Internationalization

> Backend and frontend translation conventions.

---

## Backend

- **Library**: `nicksnyder/go-i18n/v2` — see `i18n/i18n.go`.
- **Locale files**: `i18n/locales/*.yaml` (**YAML**, not JSON). Three locales
  exist: `zh-CN`, `zh-TW`, `en`. Note that `AGENTS.md` summarizes this as
  "en, zh" — the real set is three.
- **Message keys** are Go identifiers of the form `i18n.Msg*`, declared in
  `i18n/keys.go`.
- **Entry points**:
  - `i18n.T(c *gin.Context, key string, args...)` — translate using the
    request's language.
  - `i18n.Translate(lang, key)` — translate for an explicit language.
  - `common.TranslateMessage(c, key, args...)` — the wrapper used by request
    code; prefer this in controllers and middleware.

Real call site:

```go
// middleware/auth.go
common.TranslateMessage(c, i18n.MsgAuthUserBanned)
```

To add a backend string: add the `Msg*` constant in `i18n/keys.go`, then add the
key to **all three** locale YAML files.

### Known gap: localization is partial

Many `service/` and `model/` functions still return hardcoded strings rather than
i18n keys (for example `model/token.go` and `common/verification.go`). New
user-facing messages should go through i18n; do not assume existing ones do. Do
not mass-convert legacy strings as a side effect of an unrelated change.

---

## Frontend

Full detail lives in `web/AGENTS.md`; this is the working summary.

- **Libraries**: `i18next` + `react-i18next` + `i18next-browser-languagedetector`
  (`web/src/i18n/config.ts`).
- **Locale files**: `web/src/i18n/locales/{en,zh,fr,ru,ja,vi,zh-TW}.json` — seven
  locales. BCP-47 mapping is in `web/src/i18n/languages.ts`.
- **Keys are English source strings.** Files are flat JSON, so the key *is* the
  English text:

  ```json
  { "Save changes": "保存更改" }
  ```

- **Usage**:

  ```tsx
  const { t } = useTranslation();
  return <Button>{t('Save changes')}</Button>;
  ```

- **Sync CLI** (run from `web/`): `bun run i18n:sync`.
- **All user-facing text must be localized.** Writing UI text without i18n is a
  forbidden pattern.

### Component reuse is coupled to this

`AGENTS.md` makes component reuse mandatory before any new UI work: read
`web/AGENTS.md`, search `web/src/components/`, and read matching implementations
first. Importing a primitive like `Button` does not satisfy the rule when a
shared business component such as `CopyButton` or `ConfirmDialog` already covers
the case. A new implementation of common UI behavior requires a documented
concrete capability gap — different text, dimensions, or colors is not one.

---

## Common mistakes

- Editing only one of the three backend locale YAML files.
- Adding a backend message as a raw string instead of an `i18n.Msg*` key.
- Assuming the backend supports the same locale set as the frontend — it does
  not (3 vs 7).
- Reusing an English key string but changing its wording, which silently orphans
  the existing translations in every locale file.
- Adding new UI markup from scratch instead of reusing shared components.
- Adding unrelated files under `docs/` — that is separately prohibited.
