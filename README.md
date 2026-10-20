# fin-validate

Schema-driven input validation for [Fin](https://github.com/M1778/Fin). Validate untrusted string input (forms, query params, config values) at the boundary and get human-readable errors back. Invalid values do not abort; invalid pointers, out-of-range `error_at` indices, allocation failure, or aborting custom callbacks remain caller/runtime errors.

## Verification status

**VERIFIED 2026-10-21** — tests and example built and executed with the frozen
finc 0.4.0 (contract 3) and standard library at both `-O0` and `-O2`.
The suite includes 11 original scenarios, four boundary/ownership groups, and
10,000 failure-success result-reuse cycles. It covers int32 endpoints and exact
bounds around 2²⁴, long fractions and float overflow, empty allowlists, and
idempotent schema/result cleanup.

An additional `-O2` LeakSanitizer run reports only **832 bytes in 17 frozen
`Collection.push` backing-buffer allocations** in the full suite; no schema,
result, field, error/message, or input-entry leaks were reported. Collection
buffers remain a dependency limitation, detailed below. Debug-symbol builds
(`-g`) encountered a frozen compiler invalid-IR debug-location error; normal
and sanitizer builds without `-g` run successfully.


## Install

With [finn](https://github.com/M1778/finn) (once the registry resolves bare names):

```sh
finn add M1778/fin-validate
```

Manual (today):

```sh
git clone https://github.com/M1778/fin-validate
# then compile with -I fin-validate/src
finc myapp.fin -I fin-validate/src -o myapp
```

## Quick example

```fin
import { new_schema, add_string_field, add_email_field, add_int_range_field,
         new_input, input_set, input_clear, schema_check, error_count, error_at,
         schema_dispose, result_dispose, Schema, ValidationResult } from validate::fin;
import { Collection } from collection::std;
import { InputValue } from validate::fin;

fun main() <noret> {
    let s <Schema> = new_schema();
    add_string_field(&s, "username", true, 3, 20);
    add_email_field(&s, "email", true);
    add_int_range_field(&s, "age", false, true, 13, true, 120);

    let form <&Collection<&InputValue>> = new_input();
    input_set(form, "username", "al");              // too short
    input_set(form, "email", "alice-at-example");   // bad email
    input_set(form, "age", "200");                  // above max

    let r <ValidationResult> = schema_check(&s, form);
    for (let i <int> = 0; i < error_count(&r); i++) {
        printf("  - %s\n", error_at(&r, i).message);
    }
    result_dispose(&r);
    schema_dispose(&s);
    input_clear(form);
    delete form; // Collection buffer limitation: see Ownership below.
}
```

```output
  - field 'username' is too short
  - field 'email' is not a valid email address
  - field 'age' is above the maximum
```

## API reference

Import names from `validate::fin` and call them **by name** (module-qualified
calls are refused by the Fin backend; see Fin `docs/guide/10-modules-and-imports.md`).

**Schema builders** (`s: &Schema`):

| Function | Description |
|---|---|
| `new_schema() <Schema>` | Empty schema. |
| `add_string_field(s, name, required, min_len, max_len)` | String with length bounds (`-1` = no bound). Lengths are byte counts. |
| `add_email_field(s, name, required)` | Practical email check (one `@`, dot in domain, valid chars, TLD ≥ 2). |
| `add_int_field(s, name, required, has_min, min_v, has_max, max_v)` | Legacy int32 rule with `float` bounds. Comparisons promote both operands to double, preserving the integer exactly. Bounds themselves may already be rounded; prefer the exact alternative below. |
| `add_int_range_field(s, name, required, has_min, min_v, has_max, max_v)` | Int32 rule with exact inclusive `int` bounds; no float conversion. Disabled bounds are ignored. |
| `add_float_field(s, name, required, has_min, min_v, has_max, max_v)` | Finite binary32 decimal number with `float` bounds. Optional minus and one dot; at least one digit. `.5` and `1.` are accepted; plus signs, exponents, NaN and infinities are rejected. |
| `add_bool_field(s, name, required)` | Accepts `true`/`false`/`1`/`0`. |
| `add_one_of_field(s, name, required, allowed)` | Value must equal one of the borrowed `allowed: &Collection<string>`. Empty/null configured lists reject every nonempty value. Later mutations are observed. |
| `add_custom_field(s, name, required, check, message)` | `check: fn(string) -> bool`; on `false`, `message` is reported. Lambdas cannot capture locals — write `check` as a pure function of the value. |

**Input:**

| Function | Description |
|---|---|
| `new_input() <&Collection<&InputValue>>` | Empty input set. |
| `input_set(input, key, value)` | Add/replace a borrowed key/value pair. Dotted keys (`"address.city"`) match dotted field names. |
| `input_clear(input)` | Free input-entry objects, retain collection capacity. Does not free borrowed keys/values. |

**Checking:**

| Function | Description |
|---|---|
| `schema_check(s, input) <ValidationResult>` | Return a newly owned result. `result.ok` is true when nothing failed. |
| `new_result() <ValidationResult>` | Empty reusable result. |
| `schema_check_into(s, input, r)` | Replace previous errors, reusing result-array capacity. |
| `error_count(r) <int>` / `error_at(r, i) <&ValidationError>` | Read borrowed error pointers. Each has `.field`, `.message`, and `.owns_message`. Index must be in range. |
| `result_clear(r)` / `result_dispose(r)` | Free errors and owned messages; clear retains backing capacity, dispose frees it. Both reset `ok` to true. |
| `schema_clear(s)` / `schema_dispose(s)` | Free field objects; clear retains backing capacity, dispose frees it. Borrowed names, messages and lists remain untouched. |

**Semantics:** a missing key fails only `required` fields; a present-but-empty value fails `required` fields and is skipped otherwise; unknown keys are ignored.

## Ownership and compatibility

- `Schema` and `ValidationResult` own their backing arrays and elements. Do not
  copy a live owning handle, edit its backing fields, or dispose an alias. Pass
  pointers to the named APIs. Dispose once finished; repeated clear/dispose on
  the **same** handle is safe, including `Schema{}` / `ValidationResult{}`.
  Disposed handles can be reused. `new_result()` starts with `ok = true`;
  a zero-initialized `ValidationResult{}` is safe to clear/dispose/check into.
- Errors returned by `error_at` are borrowed until the next result clear,
  check-into, or disposal. Never delete them yourself. Generated messages are
  owned (`owns_message = true`); custom messages are borrowed (`false`). Every
  error field name is borrowed. Keep names/custom messages alive as long as a
  schema or result references them. Schema disposal does not invalidate errors
  provided those caller-owned strings remain alive.
- Input keys/values, schema names, custom messages, and allowlist collections
  and strings are borrowed. The library never frees them. `input_clear` frees
  entry objects created by `input_set`, so do not insert stack objects, shared
  entry pointers, or entries whose ownership belongs elsewhere into that input.
- The frozen stdlib `Collection` has no public buffer-release method or
  destructor. `delete collection` frees its header, **not its backing buffer**.
  Schema/results now use owned arrays to avoid that limitation. Input and
  allowlist APIs retain Collection compatibility: reuse those collections in
  long-lived services (`input_clear`, `allowed.clear`) instead of recreating
  them per request. Full input/allowlist buffer reclamation requires a stdlib
  release API or a future container API change; this library does neither.
- **Struct layout/source compatibility change:** `Schema.fields` and
  `ValidationResult.errors` are now owned arrays with separate logical counts,
  not `Collection` pointers. `ValidationError` adds `owns_message`. Existing
  builder, `schema_check`, `error_count`, and `error_at` signatures remain;
  callers that accessed backing collections must migrate to the named APIs and
  recompile. No implicit destructors are added.

For repeated validation, construct a schema, input, and `new_result()` once;
call `schema_check_into(&schema, input, &result)` for each request. Consume
errors before the next call. Use `input_clear(input)` to remove old request
keys. At shutdown call `result_dispose` and `schema_dispose`.

## Dependencies

- **Fin standard library only**: `collection::std` (`Collection`), `strings::std` (`to_chars`, `concat`, `free_str`, `len`, `is_empty`), plus a local `strcmp` binding.
- No Fin package dependencies (`finn.toml` `[packages]` is empty).

## Design notes and limits

- Integer parsing checks each digit against the signed int32 limit before
  accumulation. Both endpoints and arbitrary leading zeroes are supported;
  overflow and malformed strings return errors without calling aborting
  `parse_int`. Syntax is an optional minus followed by decimal digits.
- The legacy `add_int_field` promotes the parsed int32 and float bounds to
  double for comparison, so maximum `16777216.0` rejects `"16777217"` without
  rounding the input. Its `float` parameters can still round the bounds before
  validation: `16777217.0` becomes `16777216.0`, and `2147483647.0` becomes
  `2147483648.0`. Use `add_int_range_field` for exact bounds over the entire
  int32 domain. Bounds are inclusive; disabled bounds are ignored. Supply
  ordered, finite bounds for float/legacy rules; schema configuration is
  trusted, not separately validated.
- Float parsing accumulates fractional place values with double intermediates,
  then checks the binary32 result is finite. Long trailing fractions cannot
  produce an accepted NaN or infinity. Values round to binary32; tiny values
  may underflow to zero. This is not an arbitrary-precision decimal parser or
  a correctly-rounded conversion guarantee for every decimal boundary.
- String lengths are byte counts, not grapheme clusters. Fin strings are
  NUL-terminated: embedded NUL suffixes are not visible to this API. Inputs must
  be valid live string pointers; callers must enforce request-size budgets.
- Email validation remains a practical subset, not an RFC-complete parser.
  Lookup/input insertion remain linear scans; there is no hash index or strict
  unknown-field policy.

## Testing

From the repository root, using the frozen compiler and standard library:

```sh
FINC=/home/M1778/fin-projects/.toolchain/bin/finc
STD=/home/M1778/fin-projects/.toolchain/lib/std
for OPT in -O0 -O2; do
  "$FINC" tests/test_validate.fin --fin-libs "$STD" -I src "$OPT" -o test_validate
  ./test_validate
  "$FINC" examples/basic.fin --fin-libs "$STD" -I src "$OPT" -o basic
  ./basic
done
```

Covers the original valid/missing/type/range/custom/dotted-field cases; exact
positive/negative bounds around 2²⁴; int32 endpoints, overflow, leading zeroes,
and malformed syntax; finite maximum floats, long fractional strings,
underflow policy, non-finite syntax and overflow; empty and mutated allowlists;
10,000 failing/successful reuse cycles, array growth, borrowed/owned messages,
zero-initialized handles, repeated disposal, and reuse after disposal.

## Roadmap

- `v0.2.0`: regex rules (once Builder #1's `fin-regex` lands — no duplication), `not_one_of`, conditional (`required_if`) rules, float exponents, error message localization hooks.
- License review: Fin itself is GPL-3.0 and `finn` is GPL-3.0-only; this repo ships MIT pending Eleanor's decision on ecosystem licensing.

## License

MIT — see [LICENSE](LICENSE).
