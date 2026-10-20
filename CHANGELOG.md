# Changelog — fin-validate

## [0.1.1] — 2026-10-20
### Fixed
- finc 0.4.0 (contract 3) incompatibilities, found on first execution:
  `in` is now a keyword (test var renamed to `inp`); `Schema`,
  `ValidationError`, `ValidationResult`, `InputValue` need `#[export]` +
  `pub:` field labels to be importable; `FieldType` enum retired —
  `ftype` is now an `int` code (backend limit 1: enum-typed struct fields
  do not lower); struct collections converted to pointer elements
  (`Collection<&Field>` etc., backend limit 3); function-reference struct
  field defaults do not lower — `custom` defaults to `null` (never called:
  `has_custom` gates it).
- REAL bug: `input_set` appended duplicate keys so `lookup` found the stale
  first value; it now replaces the existing entry.
- All tests pass at runtime.

## [0.1.0] — 2026-10-19

All notable changes to this library. Format follows [Keep a Changelog](https://keepachangelog.com/),
versioning follows [SemVer](https://semver.org/).

## [0.1.0] — 2026-10-19

### Added
- Builder API: `new_schema` + `add_string_field` / `add_email_field` /
  `add_int_field` / `add_float_field` / `add_bool_field` / `add_one_of_field` /
  `add_custom_field` (custom validators as `fn(string) -> bool`).
- Input API: `new_input` / `input_set` with dotted keys for nested data.
- `schema_check` returning `ValidationResult{ok, errors}` — errors are values,
  never `blame`; each `ValidationError` carries `.field` and a human-readable `.message`.
- Hand-rolled int/float/email parsing that cannot abort (integer overflow is a
  validation error, not a crash).
- Test suite (11 scenarios), runnable example, GitHub Actions CI, real
  `finn.toml` manifest, `exports.fin` in finn's re-export syntax.

### Notes
- UNVERIFIED: no `finc` binary was available; nothing type-checked or executed yet.
- Regex rules deferred to 0.2.0 (waiting on Builder #1's fin-regex; no duplication).
