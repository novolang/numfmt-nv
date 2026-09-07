# numfmt-nv

Numbers written into a buffer you already own, and read back out of one.
Integers in base 2, 8, 10 and 16, with widths and zero padding. Floats as
the shortest decimal that reads back as the same value, or with a fixed
number of decimal places. Parses that say **why** they failed rather than
answering "no". The core allocates nothing, and there is a test that
reads the emitted machine code to prove it.

`str.from_int` builds a `Str`, which means it asks the allocator for
memory. A logger on a microcontroller, a protocol encoder writing into a
frame it already sized, and a panic handler that must not fail cannot
afford that call. This package is the other half.

```bash
novo pkg add numfmt-nv --git https://github.com/novolang/numfmt-nv
```

## The shape of every call

Two calls, one answer between them: ask how wide the result is, provide
the bytes, and no heap is touched.

```novo
use std.bytes
use digits

let f = digits.fmt(HexUpper).zero_padded(8)
var buf = bytes.zeros(digits.uint_len(48879, f))
buf = digits.uint_into(buf, 0, 48879, f)          // rebind the buffer
println(bytes.to_str(buf))                        // 0000BEEF
```

`digits.MAX_INT_ANY` is the size that is always enough for any integer in
any radix, so a caller can declare a fixed buffer without guessing:

```novo
var scratch = bytes.zeros(digits.MAX_INT_ANY)
scratch = digits.int_into(scratch, 0, -9223372036854775808, digits.fmt(Dec))
```

## On a device with no heap

`digits` carries `@tier(embedded)` on its digit algebra, so firmware can
format a number with no buffer at all — the number is asked how many
digits it has and then asked for them, and each one goes straight out of
the port:

```novo
@tier(embedded)
fn put_int(v: Int, r: Radix) [hw]
    for i in 0 .. digits.digit_count(v, r)
        hal.uart.write_byte(digits.digit_byte(v, r, i))
```

Reading works the same way. `IntScan` is a parse in progress held in six
scalars on the stack, so bytes arriving one interrupt at a time need
nowhere to be put:

```novo
var s = digits.scan(Dec, true)
for c in [45, 49, 50]                  // the bytes of `-12`
    s = s.push(c)
println(str.from_int(s.finish().num))  // -12
```

## Floats

```novo
use numstr

println(numstr.float_to_str(0.1))          // 0.1
println(numstr.float_to_str(1.0e23))       // 1e23
println(numstr.float_to_str(-0.0))         // -0
println(numstr.fixed_to_str(3.14159, 2))   // 3.14
```

`real.shortest_into` and `real.fixed_into` are the buffer forms, with
`real.shortest_len` and `real.fixed_len` beside them.

## What allocates, and where the line is

The line is a **module boundary**, not a naming convention:

| Module | Allocates | What it is |
| --- | --- | --- |
| `digits` | never | integers: format, parse, the digit algebra, the streaming scanner |
| `real` | never | floats: format, parse, the exact `m * 2^e` decomposition |
| `numstr` | always | the same calls, returning a `Str` |

A program with no heap imports `digits` and `real` and stops there.
Importing `numstr` is the decision to have an allocator, and it is one
line of the program, in the imports, where a reader looks for exactly
that kind of decision.

Novo has no `[alloc]` effect to draw the line with — the effect
vocabulary is exhaustive and `alloc` is not in it, because
`@tier(embedded)` refuses the allocating constructs directly and a
declaration nobody could enforce is worse than no declaration. So a
module boundary is what a build can actually see.

## The surface

**`digits` — integers.** Nothing allocates.

| Item | What it is |
| --- | --- |
| `Radix` | `Bin`, `Oct`, `Dec`, `Hex`, `HexUpper` — an enum, so there is no invalid radix |
| `IntFormat`, `fmt`, `.padded(w)`, `.zero_padded(w)` | the radix, the letter case and the field width, as a `@value` struct |
| `int_len`, `uint_len` | exactly how many bytes the writer will write |
| `int_into`, `uint_into` | write into the caller's `Bytes` at an offset; rebind the buffer |
| `digit_count`, `digit_byte` | `@tier(embedded)` — the digits themselves, with no buffer anywhere |
| `base_of`, `is_upper` | the radix table |
| `parse_int`, `parse_uint`, `parse_int_bytes` | the whole input, from a `Str` or inside a `Bytes` |
| `IntScan`, `scan`, `.push`, `.finish` | a parse in progress, one byte at a time |
| `IntParse`, `.is_ok`, `.value_or`, `.error`, `.at` | the value, or the reason and where it was found |
| `NumError` | `Empty`, `BadDigit`, `Overflow`, `Trailing` |
| `MAX_UINT_DEC` … `MAX_INT_ANY` | the widest each form can be, measured rather than guessed |

**`real` — floats.** Nothing allocates.

| Item | What it is |
| --- | --- |
| `shortest_len`, `shortest_into` | the shortest decimal that reads back as the value |
| `fixed_len`, `fixed_into` | a fixed number of decimal places, rounded to nearest with ties to even |
| `parse_float`, `parse_float_bytes` | nearest representable, with the same four reasons |
| `FloatParse`, `.is_ok`, `.value_or`, `.error`, `.at` | the outcome |
| `significand`, `binary_exponent` | the exact `m * 2^e` a double holds |
| `is_finite`, `is_negative` | including the sign of `-0.0`, which a comparison cannot see |
| `MAX_SHORTEST`, `MAX_FIXED`, `MAX_DECIMALS` | the widest each form can be |

**`numstr` — the same, as a `Str`.** Every function allocates.
`int_to_str`, `uint_to_str`, `float_to_str`, `fixed_to_str`.

## What is honestly implemented, and what is not

The package is named after `itoa` and `ryu`. The integer half is `itoa`'s
job done in full: exact in every radix, at every boundary, including
`Int.min`, whose magnitude no positive `Int` holds.

**The float half is not ryu, and saying so is worth more than the claim
would be.** Ryu decides the shortest round-tripping decimal with a proof,
using 128-bit multiplication against a table of split powers of five.
Novo's `Int` is 64 bits, the language has no 128-bit multiply and no
float-to-bits reinterpretation, and that algorithm cannot be written here
today — filed against the toolchain as
`float-bit-reinterpretation-missing`.

What is here instead:

- The **decomposition is exact**. `significand` and `binary_exponent`
  recover the exact `m * 2^e` by scaling with powers of two, which
  IEEE-754 does without error, subnormals included. Checked against
  CPython's `math.frexp` over the corpus.
- **Shortest** searches precisions from one digit to seventeen, takes the
  decimal nearest the value at each, and verifies it by parsing it back.
  The scaling runs in double-double arithmetic — a leading double and its
  exact error term, about 106 bits — because 53 bits cannot decide a
  seventeenth decimal digit.
- Measured against CPython's `repr` over a 497-value corpus:
  **496 agree**. The one that does not is `1.7e-308`, a subnormal, where
  the answer is written with seventeen digits instead of two. It is
  *correct* — it reads back as the same double, in this package and in
  CPython — but it is not shortest. That value is pinned by name in the
  test suite so a second one would be a failure rather than a quiet
  companion.
- **Parsing** is exactly correctly rounded on Clinger's fast path (a
  significand under `2^53` with a decimal exponent the scaling stays
  exact through) and runs in the same double-double arithmetic outside
  it.

## Building and testing

```bash
novo pkg build            # type-checks and effect-checks every module
novo test tests/int_vector_tests.nv
novo test tests/float_vector_tests.nv
novo test tests/int_edge_tests.nv
novo test tests/float_edge_tests.nv
novo test src/digits.nv   # the examples in the doc comments are tests too
```

The expectations in the two vector suites came out of CPython, not out
of anyone's head: `str(v)` and `format(v & 0xFFFFFFFFFFFFFFFF, "b"/"o"/"x")`
for the integers, `repr(x)` and `"%.*f"` and `math.frexp` for the floats.
The integer corpus holds every boundary a digit count can turn on — zero,
plus and minus one, `Int.min` and `Int.max` and their neighbours, every
power of ten, every other power of two, and the powers of eight and
sixteen inside the range — with pseudo-random values on a fixed seed on
top.

Two claims a `@test` cannot make are checked from outside, by
`tests/orbit/test_numfmt_nv.sh`:

- **Nothing allocates.** `tests/alloc_probe.nv` exercises every core path
  and is built at `--opt=0`, so the functions stay separate and nothing
  has inlined into `main`; every `digits_*` and `real_*` function in the
  emitted LLVM is then read for a call to `novo_alloc`. There are 100 of
  them and none of them calls it.
- **It runs on a microcontroller.** `tests/embedded_probe.nv` is firmware
  that formats a number straight into a UART with no buffer anywhere, and
  it cross-compiles and links for `--target=nrf52-qemu`. The whole
  `digits` module is copied into that build, host layer and all: a device
  build no longer emits a host-tier function it cannot reach, so the
  package needs no module split to be firmware-safe.

## Licence

Apache-2.0.
