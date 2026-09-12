# ulid-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

ULIDs for novo-lang: 128 bits of identifier — a 48-bit millisecond
timestamp then 80 random bits — whose **lexicographic order over the
text is chronological order**.  A `char(26)` primary key gets you rows
in creation order out of an ordinary index, and files named by ULID come
out of `ls` in the order they were made.

- `ulid` — the value: two `Int`s, and every accessor a shift and a mask.
- `ulidtext` — Crockford base32, as a function from an index to a
  character.
- `ulidseq` — monotonicity, as a value the caller threads.
- `ulidcvt` — the UUIDv7 bridge, and the six bits it costs.
- `ulidgen` — the one module that reads a clock (`[time, rand]`).
- `uliderr` — `UlidFault`, with the position in every variant.

```
novo pkg add ulid-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use ulidgen
use ulidseq
use ulidtext

fn main() [io, time, rand]
    // One sortable identifier.
    match ulidgen.new()
        Err(f) => println(f.message())
        Ok(u)  => println(ulidtext.encode(u))    // 01ARZ3NDEKTSV4RRFFQ69G5FAV

    // Four of them, in the order they were issued, from one clock read.
    match ulidgen.batch(ulidseq.empty(), 4)
        Err(f)      => println(f.message())
        Ok((_, us)) =>
            for u in us
                println(ulidtext.encode(u))
```

## The load-bearing interface: the text form is a function from an index to a character

```novo ignore
pub @tier(embedded)
fn char_at(u: Ulid, index: Int) -> Int []

pub @tier(embedded)
fn byte_at(u: Ulid, index: Int) -> Int []
```

Every other ULID library's encoder is `fn encode(u) -> String`: it
allocates twenty-six bytes, fills them, and hands them over.  That
function is here too — `ulidtext.encode` — and it is a convenience.  It
is not the primitive, because it cannot be:

- **the embedded runtime has no allocator**, so a `Bytes` or a `Str` is
  not something firmware can be handed;
- a caller writing into a fixed-width record does not want a second
  buffer to copy out of;
- and a caller emitting onto a UART wants one byte at a time.

So the primitive is `char_at(u, i)`: a shift, a mask and a table lookup.
Firmware writes a ULID with a bounded `for` over `0..26` and the heap is
untouched.  `ulid.byte_at` is the same inversion for the binary form,
and `encode` and `to_bytes` are those two loops written once for a host
that has an allocator.

`tests/embedded_probe.nv` builds for `--target=nrf52-qemu` and proves
it: the layout, the unsigned comparison, Crockford's ambiguity rules and
the monotonic carry all link on a Cortex-M4, with `ulidgen` left out by
the manifest's `host_modules`.  What a device can therefore do is build
an identifier from a real-time clock's millisecond and a hardware
entropy source's bits, hold it, compare it, step it, and write it out
one character at a time.  What it cannot do is parse one out of a
string; that stays the host's.

### The value is two `Int`s, and it is boxed on purpose

```novo ignore
pub struct Ulid
    hi: Int    // 48-bit timestamp, then the top 16 random bits
    lo: Int    // the low 64 random bits
```

`Ulid` is **not** `@value`.  SPEC § 14.5 keeps an unboxed struct out of a
`Result` payload, an optional, a tuple, an enum payload and a boxed
field — and a ULID occupies the first three: every parse returns
`Result<Ulid, UlidFault>`, `parse_or_none` returns `?Ulid`, and every
monotonic step returns `(UlidSeq, Ulid)`.  color-nv made the same call
for the same reason.  A boxed two-`Int` struct in all three positions
was measured on a Cortex-M4 before the claim was made, and it links.

## Why the layer is `core` and not the `host` the plan pencilled in

The plan's row said `host`, because minting a ULID reads the clock and
the entropy source.  Measured, that is **one module**: `ulidgen`,
`[time, rand]`, seven functions.  Everything the package actually *is* —
the 128-bit layout, Crockford base32, the comparison, the monotonic
increment, the UUID bridge — is integer arithmetic over values the
caller already holds.

So the manifest declares the narrow layer and names the wide module:

```toml
layer        = "core"
host_modules = ["ulidgen"]
```

Three things follow.  A consumer that only *parses* identifiers arriving
over a wire — which is most services — is charged nothing.  The embedded
probe is built without `ulidgen`, so the device claim is about the part
that can honestly make it.  And `ulidseq` can exist at all: a hidden
global sequence would need `[mutate]` on every row and the whole package
would be `host`.

**The narrowing costs one dependency**, and it is the right one to lose:
uuid-nv is `host`, a `core` package may not depend on it, so `ulidcvt`
bridges through sixteen bytes instead — which is the better shape
anyway, because the sixteen bytes *are* the UUID.

## The three rules implementations skip

**The first character cannot exceed `7`.**  Twenty-six base-32 digits
carry 130 bits and a ULID is 128, so the top two bits of the first digit
must be zero.  `8ZZZZZZZZZZZZZZZZZZZZZZZZZ` parses digit by digit and is
not a ULID; most implementations truncate it silently, which makes it
and `0ZZZ…` decode to the same value — two distinct primary keys that
compare equal.  `parse` answers `UlidTextOverflow`.

**`U` is not an alias.**  Crockford's alphabet leaves out `I`, `L`, `O`
and `U`: the first three because they are confusable with `1`, `1` and
`0`, and `U` so a random identifier cannot spell an obscenity.  So
decoding maps `i`/`I`/`l`/`L` to 1 and `o`/`O` to 0 — a ULID gets read
off a screen and typed back in — and `U` is `UlidBadDigit` naming its
index, because the whole point of excluding it is that it is not
supposed to appear.

**Encoding has exactly one spelling.**  Upper case, never an alias,
always 26 characters.  `canonical` is what a unique index needs, because
two spellings of the same identifier are the same identifier and
`char(26)` does not know that.

## Monotonicity is a value, and a backwards clock is a refusal

Two ULIDs minted in the same millisecond are ordered by their *entropy*,
which is to say randomly — so a batch insert of a thousand rows in four
milliseconds comes out shuffled in blocks of two hundred and fifty.  The
specification's answer is to increment inside a millisecond, and
`ulidseq` is that, as a value:

```novo ignore
pub fn next(s: UlidSeq, unix_ms: Int, random_hi16: Int,
            random_lo64: Int) -> Result<(UlidSeq, Ulid), UlidFault> []
```

The clock is an **argument**, so the whole of monotonicity — the
increment, the carry between the two halves, the overflow, the backwards
clock — is tested by calling it with numbers, and `ulidgen` is the only
thing that needs a clock at all.  Two independent sequences are two
values, so a service sharding by tenant keeps one each and they do not
contend.

It costs one thing, and the cost is why it is opt-in rather than what
`new` does: an attacker holding one identifier can guess the next.

**A backwards clock answers `UlidClockWentBackwards` naming both
milliseconds.**  NTP steps and virtual machines resume.  Both
corrections are worse than the refusal — reusing the last timestamp
makes the identifier lie about when it was made, and accepting the
earlier one breaks the sort that is the whole point — so the refusal is
the default and `next_relaxed` is the other choice, spelled at the call
site.

## The UUID bridge, and the six bits it costs

A ULID and a UUID are the same sixteen bytes, and that is the trap.
`ulid.to_bytes(u)` is a value `uuid.from_bytes` accepts without
complaint; what comes back is **not** a conforming UUID of any version,
because RFC 9562 spends four bits on a version nibble and two on a
variant and in a ULID those six bits are entropy.  Nothing warns about
it: the value round-trips, the string looks like a UUID, and a service
that validates versions rejects five sevenths of them.

So there are two conversions and they are not the same function:

| | lossless | timestamp readable by a UUID reader |
| --- | --- | --- |
| `to_uuid_bytes` | yes, both ways | no — the version nibble is noise |
| `to_uuidv7_bytes` | **no** — six bits overwritten | yes |

`is_lossless(u)` answers beforehand whether a particular ULID's six bits
already hold 7 and `10` — one in sixty-four does — so a migration can
count rather than assume.  UUIDv7 is the only version offered because it
is the only one with the same shape: 48 bits of Unix millisecond at the
top, then randomness, so a ULID and a UUIDv7 minted in the same
millisecond sort together.

## The layer, and why

`core`, with `host_modules = ["ulidgen"]`.  Every row outside that one
module is `[]`; `ulidgen`'s rows are `[time]`, `[rand]` and
`[time, rand]` and nothing else.  No dependencies: uuid-nv for the
reason above, and rand-nv because a package whose whole value is that it
costs nothing should not make an embedded consumer download a generator
it cannot link — entropy arrives as two `Int` arguments.

**A ULID is not a capability.**  It carries its creation time in plain
sight, so anybody holding one knows when the record was made, and a
monotonic one tells them what the next one is.  An unguessable token is
crypto-nv's business.

## What it does not do

- **No `Ord`/`Eq` trait impls** — `cmp` and `eq` are functions.  The
  comparison is *unsigned* over 128 bits stored in two signed `Int`s,
  and an operator that silently did the signed thing would put the
  newest records first with nothing else looking wrong.
- **No time formatting.**  `timestamp_ms` answers a number; turning it
  into a date is calendar-nv's and chrono-nv's job.
- **No UUID type.**  `ulidcvt` speaks bytes and text; uuid-nv owns
  `Uuid`.
- **No ULID-with-a-different-entropy-size**, no 48-bit variants, no
  base57.  One format.

## The reference implementation

The [ULID specification](https://github.com/ulid/spec) for the layout,
the alphabet, the monotonic rule and the vector
`01ARZ3NDEKTSV4RRFFQ69G5FAV` that `tests/ulidtext_tests.nv` checks
against.  Rust's [ulid](https://docs.rs/ulid) crate for the API shape.
[Crockford base32](https://www.crockford.com/base32.html) for the
alphabet and the three decoding rules, which the ULID specification
references and does not restate.  RFC 9562 § 5.7 for UUIDv7.

## Status

Every body is `todo()`.  The interface is **69 `pub` items across six
modules** — 2 structs, 1 enum and 66 functions, plus one `impl Error`:

| module | surface | rows |
| --- | --- | --- |
| `ulid` | 1 struct, 24 functions | `[]` |
| `ulidtext` | 14 functions | `[]` |
| `ulidseq` | 1 struct, 9 functions | `[]` |
| `ulidcvt` | 9 functions | `[]` |
| `ulidgen` | 7 functions | `[time]`, `[rand]`, `[time, rand]` |
| `uliderr` | 1 enum, 3 functions, `impl Error for UlidFault` | `[]` |

Sixty-two of the sixty-nine are `[]`; the seven that are not are one
module, and the manifest names it.
