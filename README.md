# ulid-nv

A ULID is a 128-bit identifier made of a 48-bit millisecond timestamp
followed by 80 random bits, written as 26 characters. Sorting the text
sorts the identifiers into the order they were created. The format is
defined by the
[ULID specification](https://github.com/ulid/spec). This package
implements it for novo-lang: the value, the text form, monotonic
generation, and the bridge to UUID.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a ULID is

The first 48 bits are the number of milliseconds since 1970, the Unix
epoch. The remaining 80 bits are random. Because the timestamp is at the
top and the whole value is compared as one unsigned 128-bit number, an
identifier made later is always larger than one made earlier.

The text form is 26 characters of **Crockford base32**, an alphabet of
`0123456789ABCDEFGHJKMNPQRSTVWXYZ`. It leaves out `I`, `L` and `O`,
which are confusable with `1`, `1` and `0`, and `U`, so that a random
identifier cannot spell an obscenity. The alphabet is in value order, so
comparing two texts character by character gives the same answer as
comparing the two 128-bit values.

Two identifiers made in the same millisecond differ only in their random
bits, so their order is random. **Monotonic generation** fixes that:
within one millisecond, the next identifier is the previous one with its
80 random bits incremented by one. A sequence that does this is a value
in this package, which the caller threads from call to call.

A **UUID** is also sixteen bytes, and RFC 9562 gives four of its bits to
a version number and two to a variant. In a ULID those six bits are
random. The two formats are therefore the same size and not the same
thing.

| Quantity | Value |
| --- | --- |
| Bits in a ULID | 128 |
| Bytes in the binary form | 16 |
| Timestamp bits | 48 |
| Random bits | 80 |
| Characters in the text form | 26 |
| Characters carrying the timestamp | 10 |
| Characters in the alphabet | 32 |
| Highest millisecond a timestamp holds | 281474976710655, which is 10889-08-02 |
| Highest value of the first character | `7` |
| Identifiers one millisecond can issue monotonically | 2^80 |
| Chance a ULID is already a valid UUIDv7 | one in 64 |

## Install

```
novo pkg add ulid-nv
```

## Example

```novo
use ulid
use ulidgen
use ulidseq
use ulidtext

fn main() [io, time, rand]
    // One identifier, from the clock and the entropy source.
    match ulidgen.new()
        Err(f) => println(f.message())
        Ok(u)  =>
            // The 26-character text form, such as 01ARZ3NDEKTSV4RRFFQ69G5FAV.
            println(ulidtext.encode(u))
            // The millisecond it was made in.
            println("${ulid.timestamp_ms(u)}")

    // Four identifiers that sort in the order they were issued, even
    // when the clock does not move between them.
    match ulidgen.batch(ulidseq.empty(), 4)
        Err(f)      => println(f.message())
        Ok((_, us)) =>
            for u in us
                println(ulidtext.encode(u))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: ulid-nv.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ulid` | The value as two integers, the field accessors, the sizes and the ceiling, the binary form, the unsigned comparison, and the two range bounds for a timestamp. |
| `ulidtext` | Crockford base32: the alphabet, one character at an index, the whole text, the timestamp prefix, parsing with its three refusals, and the comparison over text. |
| `ulidseq` | Monotonic generation as a value: the next identifier in a millisecond, the strict and relaxed answers to a backwards clock, and how many identifiers a millisecond has left. |
| `ulidcvt` | The UUID bridge: the lossless conversion, the UUIDv7 conversion that overwrites six bits, and the questions a caller asks before choosing one. |
| `ulidgen` | The only module that reads a clock or an entropy source. One identifier, a batch, and the two readings on their own. |
| `uliderr` | The six refusals, each naming its position or its value, and the accessors that pull those out. |

## How to choose an entry point

**`ulidgen.new` mints one identifier.** It reads the clock and the
entropy source, which is why it is the only module with effects.

**`ulidgen.next_in` and `ulidgen.batch` mint monotonic ones.** They take
a sequence value and answer the sequence that follows. Reach for them
when a run of identifiers must sort in the order it was issued.

**`ulidseq.next` is the same step with the clock and the entropy as
arguments.** It reads nothing, so it runs on a device and it is testable
with plain numbers.

**`ulidtext.parse` and `ulidtext.encode` are the text form on a host.**

**`ulidtext.char_at` and `ulid.byte_at` are the primitives.** Each
answers one character or one byte of an identifier at an index, with no
buffer and no allocation. Firmware writes an identifier with a bounded
loop over 26 indexes. See "Running on a microcontroller".

## The rules a user needs

1. **The first character cannot exceed `7`.** Twenty-six base-32 digits
   carry 130 bits and a ULID is 128, so the top two bits of the first
   digit must be zero. `8ZZZZZZZZZZZZZZZZZZZZZZZZZ` reads digit by
   digit and is not a ULID. `ulidtext.parse` answers
   `UlidTextOverflow`; an implementation that truncates instead makes
   two different strings decode to one value.
2. **Decoding applies Crockford's ambiguity rules and encoding does
   not.** `i`, `I`, `l` and `L` read as 1, and `o` and `O` read as 0,
   because an identifier gets read off a screen and typed back in.
   Encoding produces upper case, no alias, always 26 characters.
   `ulidtext.canonical` is what a unique index needs, because two
   spellings are one identifier and a `char(26)` column does not know
   that.
3. **`U` is not an alias for anything.** It is left out of the alphabet
   on purpose, so `U` in a text is `UlidBadDigit` naming its index.
4. **The comparison is unsigned over 128 bits held in two signed
   integers.** Use `ulid.cmp` and `ulid.eq`. There are no operator
   implementations, because a signed comparison would put the newest
   records first with nothing else looking wrong.
5. **A backwards clock is refused.** `ulidseq.next` answers
   `UlidClockWentBackwards` naming both milliseconds. Network time
   corrections step and virtual machines resume, and both repairs are
   worse than the refusal: reusing the last millisecond makes the
   identifier lie about when it was made, and accepting the earlier one
   breaks the sort. `ulidseq.next_relaxed` carries on, and the name
   says so at the call site.
6. **A monotonic step inside a millisecond ignores the entropy it is
   offered.** It increments the previous identifier's 80 random bits by
   one, with a carry between the two halves. A new millisecond takes the
   entropy.
7. **Monotonic identifiers are guessable.** Someone holding one can
   compute the next. That is why monotonicity is a call a caller makes
   rather than what `ulidgen.new` does.
8. **A ULID is not a secret.** It carries its creation time in plain
   sight. An unguessable token comes from a cryptographic random source,
   which is [crypto-nv](https://novo-lang.org/packages/crypto-nv)'s and
   [rand-nv](https://novo-lang.org/packages/rand-nv)'s business.
9. **A ULID's sixteen bytes are not a conforming UUID.**
   `ulidcvt.to_uuid_bytes` is lossless and the result has no valid
   version nibble. `ulidcvt.to_uuidv7_bytes` overwrites six bits so that
   a UUID reader sees version 7 and reads the timestamp, and that
   conversion does not come back. `ulidcvt.is_lossless` answers
   beforehand whether a particular identifier's six bits already hold
   the right values, which one in sixty-four does.
10. **UUIDv7 is the only version offered.** RFC 9562 section 5.7 gives
    it the same shape: 48 bits of Unix millisecond, then randomness. A
    ULID and a UUIDv7 made in the same millisecond sort together.
11. **A timestamp above 2^48 - 1 is refused.**
    `ulid.of_parts_checked` answers `UlidTimestampOverflow` and
    `ulid.of_parts` masks. Two functions rather than one argument.
12. **A prefix range scan uses the first ten characters.**
    `ulidtext.encode_timestamp` answers them, and `ulid.floor_of` and
    `ulid.ceiling_of` are the two bounds of one millisecond.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers every module except `ulidgen`, and it
covers the functions in them that take and answer integers. The manifest
names the exception with `host_modules = ["ulidgen"]`.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds today:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe produces a Cortex-M4 executable that reads the field layout,
runs the unsigned comparison, applies Crockford's ambiguity rules and
steps the monotonic carry. It builds and it is not run: every function
it calls is a `todo()` today.

What a device can therefore do is build an identifier from a real-time
clock's millisecond and a hardware entropy source's bits, hold it,
compare it, step it, and write it out one character at a time through
`ulidtext.char_at`. What it cannot do is parse one out of a string or
allocate the 26-byte text, because those need `Str` and `Bytes`, which
the embedded runtime does not define.

## What is not included

- **A generator inside the `core` modules.** Entropy arrives as two
  integer arguments and the millisecond as one more. `ulidgen` is the
  one module that reads them for you, and a device supplies its own.
- **A dependency on [uuid-nv](https://novo-lang.org/packages/uuid-nv).**
  That package is a host package and this one is not, so the bridge is
  the sixteen bytes both already speak.
- **Operator implementations for ordering and equality.** See rule 4.
- **Date formatting.** `ulid.timestamp_ms` answers a number. Turning it
  into a date is
  [calendar-nv](https://novo-lang.org/packages/calendar-nv)'s and
  [chrono-nv](https://novo-lang.org/packages/chrono-nv)'s work.
- **Other ULID-like formats.** No alternative entropy sizes, no 48-bit
  variants, no base57. One format.
- **A hidden global sequence.** A sequence is a value, so two shards
  keep one each and they do not contend.

## Related packages

- [uuid-nv](https://novo-lang.org/packages/uuid-nv) is UUIDs, including
  version 7, which is the version with a millisecond timestamp at the
  top. Reach for it when something else in the system requires a UUID.
- [rand-nv](https://novo-lang.org/packages/rand-nv) is the random number
  generator a caller draws the 80 bits from when it is not using
  `ulidgen`.
- [base64-nv](https://novo-lang.org/packages/base64-nv) and
  [bech32-nv](https://novo-lang.org/packages/bech32-nv) are the other
  text encodings of binary values. Neither has an ordering guarantee.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) turns the
  millisecond this package answers into a civil date.

## Tests

```bash
novo test tests/ulid_tests.nv         # 17 tests: the value and the sequence
novo test tests/ulidtext_tests.nv     # 12 tests: Crockford base32
novo test tests/ulidcvt_tests.nv      #  9 tests: the UUID bridge and the generator
```

The reference data is the ULID specification's own, including the vector
`01ARZ3NDEKTSV4RRFFQ69G5FAV`, together with
[Crockford base32](https://www.crockford.com/base32.html) for the
alphabet and the decoding rules, and RFC 9562 section 5.7 for UUIDv7.
The API shape follows the `ulid` crate in Rust.

The suite asserts the field layout and the published sizes, that the
comparison is unsigned, that an earlier millisecond always sorts first,
that the text sorts the way the bits sort, that the first character
cannot exceed `7`, that decoding is forgiving and encoding is not, that
the same millisecond increments rather than redrawing, that the
increment carries across the two halves, that exhausting a millisecond
is named rather than wrapped, that a backwards clock is refused and the
relaxed form carries on, that a raw ULID is not a conforming UUID of any
version, that the version 7 conversion stamps six bits and says it did,
and that one identifier in sixty-four survives that stamp unchanged.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

Every function is declared and none is implemented. The interface is 69
public items across six modules.

| Module | Surface | Implemented |
| --- | --- | --- |
| `ulid` | 1 struct, 24 functions | no |
| `ulidtext` | 14 functions | no |
| `ulidseq` | 1 struct, 9 functions | no |
| `ulidcvt` | 9 functions | no |
| `ulidgen` | 7 functions | no |
| `uliderr` | 1 enum, 3 functions, `UlidFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
