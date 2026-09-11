# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.1 — 2026-09-08

- **Declares its layer**: `layer = "core"` in the manifest — the public API requires no effects, and `novo pkg publish` now checks the code against that budget.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- `bits.*` calls are the operators they lower to (`&`, `|`, `^`, `<<`, `>>>`, `~`), rewritten by `novo rewrite --bits-to-operators` where the checker proves the operands `Int`; every test vector byte-identical.  Sources reformatted to the canonical form.

## 0.1.0 — 2026-09-07

First release.

- `digits` — integers into a buffer the caller owns and back out of one,
  allocating nothing. `int_into` / `uint_into` with `int_len` /
  `uint_len` beside them, in base 2, 8, 10 and 16 with a lower- and an
  upper-case hexadecimal, a minimum field width and a choice of space or
  zero padding. `digit_count` and `digit_byte` carry `@tier(embedded)`,
  so firmware formats a number straight into a UART with no buffer at
  all.
- `digits` parsing — `parse_int`, `parse_uint` and `parse_int_bytes`
  answer with a value or with one of four reasons and the offset it was
  found at: `Empty`, `BadDigit`, `Overflow`, `Trailing`. `IntScan` is the
  same parse held in six scalars on the stack, one byte at a time, for a
  reader with nowhere to put a buffer.
- `digits` constants — `MAX_UINT_DEC` through `MAX_INT_ANY`, the exact
  width of the widest value in each form, so a caller sizes a buffer
  without guessing.
- `real` — floats, allocating nothing. `shortest_into` writes the
  shortest decimal that reads back as the value; `fixed_into` writes a
  fixed number of decimal places, rounded to nearest with ties to even;
  `parse_float` and `parse_float_bytes` read either back. `significand`
  and `binary_exponent` recover the exact `m * 2^e` a double holds.
- `numstr` — the same calls returning a `Str`, and the only module in the
  package that allocates.
- **This is not ryu.** The shortest form is found by searching precisions
  and verifying each candidate by parsing it back, with the scaling in
  double-double arithmetic. Measured against CPython's `repr` over a
  497-value corpus, 496 agree; the exception is `1.7e-308`, which is
  written correctly but not shortest. See the README.
