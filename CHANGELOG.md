# Changelog

All notable changes to this project. Format follows
[Keep a Changelog](https://keepachangelog.com/), versioning follows
[Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed

- `add_email_field` now enforces the shape limits a mail transport applies
  instead of only requiring a dot and two trailing characters. It rejects
  empty domain labels (`a@b..com`, `a@.b.com`), labels that start or end with a
  hyphen (`a@-b.com`, `a@b.co-`), a trailing dot, doubled or edge dots in the
  local part, a numeric top-level domain (`a@b.c0`), a local part longer than 64
  bytes, a domain label longer than 63 bytes, and a total longer than 254 bytes.
  Previously all of those were accepted. This is stricter, so callers that
  relied on the old acceptance may see new errors.
- Every generated error message now names the failing rule and its numbers:
  `field 'age' is above the maximum: 200, maximum 120` rather than
  `field 'age' is above the maximum`. Messages report computed numbers only and
  never echo the submitted value, so a CR/LF in input cannot forge a log line.
- `str_eq` uses the standard library's `strings::std` `equals`; the private
  `strcmp` C binding is gone, removing a symbol the host program could collide
  with.
- README rewritten to state what the library does and does not guarantee, with
  the email check named as a shape check rather than a security control, byte
  rather than character lengths, and the untested platforms named. The leak
  caveat is corrected: `Collection.free_collection` releases the buffer, so
  `input_clear` followed by `free_collection` does reclaim it.

### Added

- `tests/test_boundaries.fin`: for each validator, the shortest accepted input,
  the shortest rejected input, both endpoints, empty input, one byte over the
  limit, and the exact message text. Each of the four new email guards has an
  assertion that fails when that guard alone is removed.
- `tests/test_reuse.fin`: the consumer reuse path (`schema_check_into` over one
  schema and result) end to end, including error replacement and required-field
  counts.
- CI on Linux x86_64/arm64, macOS x86_64/arm64 and Windows x86_64 that compiles
  **and executes** every test program and asserts the example's output; plus a
  lane that runs the real `finn test` and the `finn do` scripts.
- `FINC_VERSION` and the shared ecosystem installer
  (`.github/scripts/install_finc.py`): fetch the pinned release archive and
  verify its published SHA-256. CI additionally requires `contract 3`, since the
  installer only checks that a contract integer is reported.
- `scripts/fin_header.py` and the project header in every source file.

### Fixed

- `exports.fin` was not valid Fin and was silently a syntax error. It is now a
  documented placeholder: Fin has no `export` keyword, and the file exists only
  because `finn`'s layout check looks for it.
- `finn.toml` scripts are consistent with the actual test files and are run by CI.

## [0.1.1] — 2026-10-20

### Fixed

- finc 0.4.0 (contract 3) incompatibilities, found on first execution: `in` is
  now a keyword; exported structs need `#[export]` and `pub:` fields; the
  `FieldType` enum was retired for an `int` code; struct collections converted to
  pointer elements; function-reference struct field defaults do not lower.
- `input_set` appended duplicate keys so `lookup` found the stale first value; it
  now replaces the existing entry.

## [0.1.0] — 2026-10-19

### Added

- Builder API: `new_schema` plus `add_string_field`, `add_email_field`,
  `add_int_field`, `add_float_field`, `add_bool_field`, `add_one_of_field`,
  `add_custom_field`.
- Input API: `new_input`, `input_set` with dotted keys for nested data.
- `schema_check` returning `ValidationResult{ok, errors}`; errors are values,
  never `blame`, and each carries `.field` and a human-readable `.message`.
- Hand-rolled int/float/email parsing that cannot abort.
- Test suite, runnable example, GitHub Actions CI, `finn.toml` manifest.
