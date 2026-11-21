# fin-validate

Schema-driven validation for [Fin](https://github.com/M1778/Fin). Declare the shape
of a request once, feed it untrusted string input, and get back a list of
human-readable, field-specific errors. Invalid values never abort the program.

```fin fragment
let r <ValidationResult> = schema_check(&s, form);
if (!r.ok) {
    for (let i <int> = 0; i < error_count(&r); i++) {
        printf("  - %s\n", error_at(&r, i).message);
    }
}
```

## Verification status

Verified against **finc 1.0.0 (contract 3)** from the published release archive
(`x86_64-unknown-linux-musl`), with its bundled standard library. All three test
programs and the example compile, link and run; see [Testing](#testing) for the
commands and their output.

| Platform | Status |
| --- | --- |
| Linux x86_64 (musl release archive) | Run on this machine: all three suites and the example |
| Linux arm64, macOS arm64, Windows x86_64 | CI compiles **and executes** every suite on each; not run on this machine |
| macOS x86_64 (Intel) | **Untested.** The published `x86_64-apple-darwin` archive links Homebrew `llvm@22`, which Homebrew ships for Apple Silicon only |
| Other architectures | Untested |

A platform is only claimed tested where the run happened. CI runs the four lanes
above on every pull request.

## What it does

- Validates string-keyed input against a schema of typed fields: string with
  byte-length bounds, email shape, int32 with exact or float bounds, finite
  binary32 float with bounds, boolean, allowlist membership, and a custom
  `fn(string) -> bool` predicate.
- Reports **one error per failing rule**, each naming the field and the rule that
  failed, with the numbers involved (`field 'age' is below the minimum: 9,
  minimum 13`). Errors carry `.field` and `.message`; the result also has `.ok`.
- Separates missing, empty and present values: a missing key fails only required
  fields; a present-but-empty value fails required fields and is skipped
  otherwise; unknown keys are ignored.
- Never aborts on untrusted input. Integer overflow, malformed numbers, a
  300-byte email and an empty allowlist are validation errors, not crashes.
- Reuses a schema and a result across requests with `schema_check_into`, which
  replaces previous errors instead of appending.

## What it does not do, and what it does not guarantee

- **The boolean-only case.** `schema_check` returns a `ValidationResult`, never a
  bare `bool`. Every failure mode above produces a structured error with a
  message; there is no variant that returns only true/false. If you want just the
  verdict, read `r.ok` — but the reason is always available in the errors, and
  callers should surface it rather than discard it.
- **Email is a shape check, not a security control.** `add_email_field` checks:
  at most 254 bytes total; one `@`; a local part of at most 64 bytes with allowed
  ASCII characters and no leading, trailing or doubled dot; a domain of nonempty
  dot-separated labels of at most 63 bytes that start and end alphanumerically;
  and a top-level domain of at least two letters. It does **not** prove the mailbox
  exists, that the domain resolves or accepts mail, or that the address is safe to
  interpolate into a mail header. It is not RFC 5322 complete: quoted local parts,
  address literals, and internationalised addresses are rejected. Do not use it
  as authorization, rate limiting, or injection prevention.
- **No URL or identifier validators.** Only the field types listed above exist.
- **Strings are bytes.** Length bounds count bytes, not characters or grapheme
  clusters, so a multi-byte UTF-8 character costs more than one. Fin strings are
  NUL-terminated, so an embedded NUL hides the suffix from every validator here.
- **No size cap on input values.** A caller must enforce its own request-size
  budget before building the input collection.
- **No schema validation.** Bounds are trusted configuration: `min_len` greater
  than `max_len`, or reversed float bounds, are not diagnosed. Supply ordered,
  finite bounds.
- **Not thread-safe and not reentrant.** A schema and a result are mutable owning
  handles; use one per thread. No locking is provided or attempted.
- **No regex, rule DSL, async validators, or locale-dependent rules.** None are
  planned; comparison and case rules are byte-exact and ASCII.

## Install

With [finn](https://github.com/M1778/finn), from a checkout of this repository:

```sh
finn add --yes M1778/fin-validate
```

No registry resolves these names yet, so `--yes` is required: without it `finn add` refuses
a source the register has not vouched for rather than installing it unasked. Drop the flag
once a registry entry exists.

For a local checkout, `finn add --yes /path/to/fin-validate` records a path dependency.
Either way the package lands under `.finn/packages/fin-validate`, and it is imported by the
quoted package path, because a hyphen is not an identifier in Fin and a bare
`"fin-validate"` names no module file:

```fin fragment
import { new_schema } from "fin-validate/src/validate";
```

Manual, with the pinned compiler from the release archive:

```sh
git clone https://github.com/M1778/fin-validate
finc myapp.fin -I fin-validate/src -o myapp
```

Relative imports are resolved from the importing file, which is why `-I src`
works from the repository root. The suffix is always the module file:
`validate::fin` is the namespace inside `src/validate.fin`, not a separate file.

## Complete runnable example

[`examples/basic.fin`](examples/basic.fin), whose output CI asserts verbatim:

```fin
import { new_schema, add_string_field, add_email_field, add_int_range_field, add_one_of_field, new_input, input_set, input_clear, schema_check, schema_dispose, result_dispose, error_count, error_at, Schema, ValidationResult } from validate::fin;
import { Collection } from collection::std;
import { InputValue } from validate::fin;

fun main() <noret> {
    let s <Schema> = new_schema();
    add_string_field(&s, "username", true, 3, 20);
    add_email_field(&s, "email", true);
    add_int_range_field(&s, "age", false, true, 13, true, 120);
    let roles <&Collection<string>> = new Collection::<string>{};
    roles.push("admin");
    roles.push("user");
    roles.push("guest");
    add_one_of_field(&s, "role", true, roles);

    // Simulate a sloppy form submission.
    let form <&Collection<&InputValue>> = new_input();
    input_set(form, "username", "al");
    input_set(form, "email", "alice-at-example.com");
    input_set(form, "age", "200");
    input_set(form, "role", "root");

    let r <ValidationResult> = schema_check(&s, form);
    if (r.ok) {
        printf("signup accepted
");
    } else {
        printf("signup rejected, %d problem(s):
", error_count(&r));
        for (let i <int> = 0; i < error_count(&r); i++) {
            printf("  - %s
", error_at(&r, i).message);
        }
    }
    result_dispose(&r);
    schema_dispose(&s);
    // input_clear frees the entry objects; free_collection then releases the
    // Collection's buffer. Both are needed; `delete` only frees the handle.
    input_clear(form);
    form.free_collection();
    delete form;
    roles.free_collection();
    delete roles;
}
```

Run it from the repository root:

```sh
finc examples/basic.fin -I src -o basic
./basic
```

```output
signup rejected, 4 problem(s):
  - field 'username' is too short: 2 bytes, minimum 3
  - field 'email' is not a valid email address: expected local@domain.tld, letters-only TLD of at least 2
  - field 'age' is above the maximum: 200, maximum 120
  - field 'role' is not one of the 3 allowed values
```

## Public API

Import names from `validate::fin` (via `-I src`) and call them **by name**;
module-qualified calls are refused by the Fin backend.

**Schema builders** (`s: &Schema`):

| Function | Description |
|---|---|
| `new_schema() <Schema>` | Empty schema. |
| `add_string_field(s, name, required, min_len, max_len)` | Byte length bounds; `-1` disables a bound. |
| `add_email_field(s, name, required)` | Email shape check; see the guarantee list above. |
| `add_int_field(s, name, required, has_min, min_v, has_max, max_v)` | Legacy int32 rule with `float` bounds. Bounds are rounded before validation; prefer the exact form below. |
| `add_int_range_field(s, name, required, has_min, min_v, has_max, max_v)` | Int32 rule with exact inclusive `int` bounds; no float conversion. |
| `add_float_field(s, name, required, has_min, min_v, has_max, max_v)` | Finite binary32 with `float` bounds. Optional minus, at least one digit, at most one dot. `.5` and `1.` are accepted; `+`, exponents, NaN and infinity are rejected. |
| `add_bool_field(s, name, required)` | Accepts exactly `true`, `false`, `1`, `0`. |
| `add_one_of_field(s, name, required, allowed)` | Value must equal a member of the borrowed `allowed: &Collection<string>`. A null or empty list accepts no nonempty value. Later mutations are observed. |
| `add_custom_field(s, name, required, check, message)` | `check: fn(string) -> bool`; `message` is reported verbatim on `false`. Lambdas cannot capture locals. |

**Input:**

| Function | Description |
|---|---|
| `new_input() <&Collection<&InputValue>>` | Empty input set. |
| `input_set(input, key, value)` | Add or replace a borrowed key/value pair. Replacing does not free either string. Dotted keys (`"address.city"`) match dotted field names. |
| `input_clear(input)` | Free the entry objects, retaining collection capacity. Does not free borrowed keys or values. |

**Checking:**

| Function | Description |
|---|---|
| `schema_check(s, input) <ValidationResult>` | Newly owned result. `r.ok` is true when nothing failed. |
| `new_result() <ValidationResult>` | Empty reusable result, `ok = true`. |
| `schema_check_into(s, input, r)` | Validate into an existing result, replacing previous errors and reusing array capacity. |
| `error_count(r) <int>` / `error_at(r, i) <&ValidationError>` | Borrowed error pointers, each with `.field`, `.message`, `.owns_message`. `error_at` blames on an out-of-range index. |
| `result_clear(r)` / `result_dispose(r)` | Free errors and owned messages. `clear` retains capacity, `dispose` frees it. Both reset `ok` to true. |
| `schema_clear(s)` / `schema_dispose(s)` | Free field objects. `clear` retains capacity, `dispose` frees it. Borrowed names, messages and allowlists are never freed. |

## Ownership

- `Schema` and `ValidationResult` own their backing arrays and elements. Do not
  copy a live owning handle, edit its backing fields, or dispose an alias; pass
  pointers to the named APIs. Dispose once when finished. Repeated clear/dispose
  on the **same** handle is safe, including zero-initialized `Schema{}` and
  `ValidationResult{}`, and a disposed handle can be reused.
- Errors from `error_at` are borrowed until the next clear, check-into, or
  disposal of that result. Never delete them yourself. Generated messages are
  owned (`owns_message = true`) and freed by the result; custom messages are
  borrowed (`false`) and must outlive the result. Every error's field name is
  borrowed.
- Input keys and values, schema names, custom messages, and allowlist collections
  are borrowed and never freed. `input_clear` frees entry objects created by
  `input_set`, so do not insert stack objects or shared entry pointers.
- `Collection` owns a buffer that `delete` does not release. After
  `input_clear`, call `input.free_collection()` (and `allowed.free_collection()`)
  before `delete` to release it. Reuse collections in a long-lived service
  instead, with `input_clear` between requests.

## Dependencies

Fin standard library only: `collection::std` (`Collection`), `strings::std`
(`to_chars`, `concat`, `free_str`, `len`, `is_empty`, `equals`). No Fin package
dependencies; `finn.toml` `[packages]` is empty.

## Design notes

- String comparison uses the standard library's `equals`, a `strcmp` of the
  contents, so `"user"` never matches `"user2"`. No local C binding is declared,
  which keeps the library free of symbol names that could collide with a host
  program's.
- Error messages never echo the submitted value: they report a length, a parsed
  number, a configured bound, or the rule name. A CR/LF in untrusted input
  therefore cannot forge a line in a log that prints errors.
- Integer parsing checks each digit against the int32 limit before accumulating,
  so `2147483648` and a 50-digit run are errors rather than overflow. Both
  endpoints, arbitrary leading zeroes and `-0` are supported.
- Float parsing accumulates with `double` intermediates and then checks the
  binary32 result is finite, so a long fraction cannot produce an accepted NaN or
  infinity. Values round to binary32; tiny values may underflow to zero.
- The legacy `add_int_field` promotes the int and its `float` bounds to `double`
  for comparison. The promotion is exact for the integer, but the caller's
  `float` bounds may already have rounded (`16777217.0` becomes `16777216.0`).
  Use `add_int_range_field` for exact bounds over the whole int32 domain.
- Lookup is a linear scan. There is no hash index and no strict unknown-field
  policy: unknown keys are ignored, which is deliberate, not an omission.

## Testing

From the repository root, with `finc` 1.0.0 (contract 3) on `PATH`:

```sh
finc --version
finc src/validate.fin

finc tests/test_validate.fin   -I src -o test_validate   && ./test_validate
finc tests/test_boundaries.fin -I src -o test_boundaries && ./test_boundaries
finc tests/test_reuse.fin      -I src -o test_reuse      && ./test_reuse

finc examples/basic.fin -I src -o basic && ./basic
```

Or through `finn`:

```sh
finn test            # compiles and executes every tests/*.fin
finn do check        # frontend check of src/validate.fin
finn do test         # the three suites above
finn do example      # builds and runs examples/basic.fin
```

The suites cover, per validator, the shortest accepted input, the shortest
rejected input, both endpoints, empty input, one byte over the limit, and the
exact message text. `tests/test_boundaries.fin` also holds the guard tests:
removing the total-length, local-part, domain-label or TLD check makes a named
assertion fail. `tests/test_validate.fin` adds 10,000 failure/success
result-reuse cycles, array growth, message ownership, zero-initialized handles,
and repeated disposal. `tests/test_reuse.fin` is the consumer-shaped reuse path.

CI fetches the pinned release in [`FINC_VERSION`](FINC_VERSION) through
[`.github/scripts/install_finc.py`](.github/scripts/install_finc.py), which
verifies the published SHA-256 and refuses on a mismatch, on a release with no
artifact for the host triple, or on an archive that does not contain a working
`bin/finc` and `lib/std`. The installer checks that the compiler reports *some*
contract integer; the workflow then requires `contract 3` explicitly.

## License

MIT — see [LICENSE](LICENSE).
