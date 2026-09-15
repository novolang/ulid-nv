# Changelog

All notable changes to ulid-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ulid` — `Ulid` as two `Int` halves; `nil`, `max`, `of_halves`,
  `of_parts`, `of_parts_checked`; `hi`, `lo`, `timestamp_ms`,
  `random_hi16`, `random_lo64`, `with_timestamp`; `byte_at`,
  `to_bytes`, `of_bytes`; `cmp`, `eq`, `is_nil`; `floor_of`,
  `ceiling_of`; and the sizes as functions.
- `ulidtext` — `char_at`, the primitive; `digit_byte`, `digit_value`,
  `alphabet`, `text_len`, `timestamp_len`; `encode`, `encode_into`,
  `encode_timestamp`; `parse`, `parse_or_none`, `is_valid`,
  `canonical`, `compare_text`.
- `ulidseq` — `UlidSeq`; `empty`, `resuming`, `issued_ms`, `last`,
  `is_empty`; `next`, `next_relaxed`, `peek`, `remaining_in_ms`.
- `ulidcvt` — `to_uuid_bytes` / `of_uuid_bytes` (lossless) and
  `to_uuidv7_bytes` / `of_uuidv7_bytes` (conforming, six bits lost);
  `uuid_version`, `timestamp_is_meaningful`, `is_lossless`;
  `to_uuid_text`, `of_uuid_text`.
- `ulidgen` — `new`, `new_with`, `now_ms`, `draw_entropy`, `next_in`,
  `next_in_relaxed`, `batch`.
- `uliderr` — `UlidFault` with the position in every variant, and
  `impl Error for UlidFault`.
- `tests/embedded_probe.nv` — the device claim, built for
  `--target=nrf52-qemu`.

### Known

- **The layer is `core`, not the `host` the plan pencilled in.**
  Measured, the effects are one module: `ulidgen`, `[time, rand]`.
  `host_modules = ["ulidgen"]` declares the narrow layer and names the
  wide part, so a service that only parses identifiers is charged
  nothing and the embedded probe is built without a clock.
- **The narrowing costs the uuid-nv dependency.** uuid-nv is `host` and
  a `core` package may not depend on it, so `ulidcvt` bridges through
  sixteen bytes. That is the better shape anyway: the sixteen bytes ARE
  the UUID, and both packages already speak them.
- **The text form is a function from an index to a character.**
  `char_at` and `byte_at` are the primitives and `encode` / `to_bytes`
  are those loops written once. Firmware writes a ULID with no heap.
- **`Ulid` is boxed and not `@value`** — SPEC § 14.5 keeps an unboxed
  struct out of a `Result`, an optional and a tuple, and a ULID
  occupies all three. A boxed two-`Int` struct in all three positions
  was built for a Cortex-M4 before the device claim was made.
- **The comparison is unsigned over 128 bits in two signed `Int`s.**
  An operator that did the signed thing puts the newest records first
  and nothing else looks wrong, which is why `cmp` is a function and
  there is no `Ord` impl.
- **Three rules other implementations skip** are in the interface: the
  first character cannot exceed `7` (or two distinct keys decode
  equal), `U` is a refusal and not an alias, and encoding has exactly
  one spelling.
- **A backwards clock is a refusal**, naming both milliseconds;
  `next_relaxed` is the other choice, spelled at the call site.
- **novo-lang has no octal integer literal**, which this package does
  not need but the sibling datafile-nv does; recorded there.
