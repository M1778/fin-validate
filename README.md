# fin-validate

Schema-driven input validation for [Fin](https://github.com/M1778/Fin). Validate untrusted string input (forms, query params, config values) at the boundary and get human-readable errors back — never aborts.

## Verification status

**VERIFIED 2026-10-20** — built and executed with finc 0.4.0 (contract 3).
`tests/test_validate.fin` passes: required/email/int/float/bool rules,
length and numeric ranges, allowed lists, custom validators, nested dotted
names, error messages. (This run fixed several finc-0.4.0 incompatibilities:
`in` is now a keyword, structs need `#[export]`+`pub:`, enum-typed struct
fields do not lower, struct collections need pointer elements, and
`input_set` now replaces duplicate keys — see CHANGELOG.)


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
import { new_schema, add_string_field, add_email_field, add_int_field,
         new_input, input_set, schema_check, error_count, error_at,
         Schema, ValidationResult } from validate::fin;
import { Collection } from collection::std;
import { InputValue } from validate::fin;

fun main() <noret> {
    let s <Schema> = new_schema();
    add_string_field(&s, "username", true, 3, 20);
    add_email_field(&s, "email", true);
    add_int_field(&s, "age", false, true, 13.0, true, 120.0);

    let form <&Collection<InputValue>> = new_input();
    input_set(form, "username", "al");              // too short
    input_set(form, "email", "alice-at-example");   // bad email
    input_set(form, "age", "200");                  // above max

    let r <ValidationResult> = schema_check(&s, form);
    for (let i <int> = 0; i < error_count(&r); i++) {
        printf("  - %s\n", error_at(&r, i).message);
    }
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
| `add_int_field(s, name, required, has_min, min_v, has_max, max_v)` | 32-bit decimal integer with optional range. Overflow is a validation error, never an abort. |
| `add_float_field(s, name, required, has_min, min_v, has_max, max_v)` | Decimal number (`[-]digits[.digits]`; no exponents in v0.1.0). |
| `add_bool_field(s, name, required)` | Accepts `true`/`false`/`1`/`0`. |
| `add_one_of_field(s, name, required, allowed)` | Value must equal one of `allowed: &Collection<string>`. |
| `add_custom_field(s, name, required, check, message)` | `check: fn(string) -> bool`; on `false`, `message` is reported. Lambdas cannot capture locals — write `check` as a pure function of the value. |

**Input:**

| Function | Description |
|---|---|
| `new_input() <&Collection<InputValue>>` | Empty input set. |
| `input_set(input, key, value)` | Add a key/value pair. Dotted keys (`"address.city"`) match dotted field names for nested data. |

**Checking:**

| Function | Description |
|---|---|
| `schema_check(s, input) <ValidationResult>` | Validate. `result.ok` is true when nothing failed. |
| `error_count(r) <int>` / `error_at(r, i) <ValidationError>` | Read the errors. Each has `.field` and `.message`. |

**Semantics:** a missing key fails only `required` fields; a present-but-empty value fails `required` fields and is skipped otherwise; unknown keys are ignored.

## Dependencies

- **Fin standard library only**: `collection::std` (`Collection`), `strings::std` (`to_chars`/`from_chars`/`concat`/`parse_int`), plus a local `strcmp` binding.
- No Fin package dependencies (`finn.toml` `[packages]` is empty).

## Design notes

- `parse_int` **aborts** on bad input, so integers are pre-validated by hand (digits + 10-digit lexicographic bound check) and `parse_int` is only called on provably-safe input. `"2147483648"` is a validation error, not a crash.
- Integer range bounds are `float`: bounds beyond 2²⁴ compare approximately (float precision). Real-world bounds (ages, ports, counts) are exact.
- String lengths are byte counts, not grapheme clusters.

## Testing

```sh
finc tests/test_validate.fin -I src -o test_validate && ./test_validate
```

Covers: valid input, missing required, bad email, int range, non-numeric int, int overflow (no abort), disallowed value, string length, custom validator, bad float/bool, dotted nested fields.

## Roadmap

- `v0.2.0`: regex rules (once Builder #1's `fin-regex` lands — no duplication), `not_one_of`, conditional (`required_if`) rules, float exponents, error message localization hooks.
- License review: Fin itself is GPL-3.0 and `finn` is GPL-3.0-only; this repo ships MIT pending Eleanor's decision on ecosystem licensing.

## License

MIT — see [LICENSE](LICENSE).
