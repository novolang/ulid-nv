# Changelog

All notable changes to ulid-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## Unreleased

### Fixed

- `ulidtext.encode_into` leaves the buffer alone for a position whose write would pass the end of the buffer, for a position of any size.

## 0.1.0 — 2026-10-11

The **implementation**: every function declared by the 0.0.2 interface
has a body. `stability = "experimental"`, the `novo` floor is
`>= 0.20.2`, and the four `novo test` suites plus the device probe's
build are green. Coverage over `src/` is 100% of 321 instrumented
lines, with no `cov: skip` marker anywhere.

### Added

- `tests/ulid_edge_tests.nv` — eleven tests for the boundaries the
  oracle suites leave: the five fault sentences the oracle never prints,
  `index_of` and `is_text_fault` over every variant, the two digit
  tables' off-by-one ends, `canonical`'s refusal arm, `compare_text`'s
  different-length and non-digit endings, the unsigned comparison of two
  halves above 2^63, `uuid_version` on a wrong-sized buffer,
  `of_uuid_text`'s two hyphen faults, and the `ulidgen.batch` refusal a
  sequence ahead of the clock produces.

### Changed

- **Ten `x == false` sites in the three oracle suites were rewritten to
  `not x`**, the 0.20.2 linter's own `redundant-bool` suggestion, which
  cannot change a predicate. Every site is pre-existing: the counts at
  `HEAD` and now agree at 5 and 5 (`ulid_tests.nv`), 4 and 4
  (`ulidcvt_tests.nv`) and 1 and 1 (`ulidtext_tests.nv`). The six
  further sites in the new edge suite are written as `not` from the
  start.
- `ulidtext.encode_into`'s doc prose now says the caller's buffer is not
  written, which is what the body can do — see `### Known`.
- `novo.toml` — `version` is `0.1.0`, `stability` is `experimental`, the
  interface-release comment and the `Interface only — every body is a
  todo()` tail are gone, and the `novo` floor is the version this work
  was verified against.
- `README.md` — the `**Status: NOT IMPLEMENTED**` block and the
  implementation-status table are deleted, and the two paragraphs that
  said the tests fail on purpose now say what they do.
- **Seven functions now wrap their heap constructions in `system`
  blocks**, because the device build refused them.  `novo build
  --target=nrf52-qemu tests/embedded_probe.nv`, which the 0.0.2
  interface shipped as the package's device claim, reported thirteen
  E4000 tier errors: `ulid.of_parts_checked`'s `Ok`/`Err`,
  `uliderr.sentence_of`'s interpolated match, `ulidseq.issued`'s struct
  literal and tuple, and the payload-carrying `Ok`/`Err` of `step`,
  `next`, `next_relaxed` and `peek`.  A device build checks a function
  that carries no `@tier` at the embedded tier; each construction is
  now wrapped, with the answer delivered by `return` — see `### Known`.
  No function's signature, tier or answer changed, and the probe builds.

### Known

- **A device build stamps `embedded` onto every function that carries no
  `@tier`, so seven functions needed `system` blocks to build for a
  target.**  The documentation says the opposite — `docs/language/tiers.md`
  has "Unannotated functions are not themselves checked at all …
  unless a project tier is set", and `docs/annotations.md` calls `app`
  "the default tier for unannotated functions" — while the build that
  gates an ELF checks them at the embedded tier and refuses a
  payload-carrying `Result`, a boxed struct or tuple literal and an
  interpolated string.  Filed in the `novo/docs/` area, slug
  `the-tiers-page-says-an-unannotated-function-is-not-checked-and-a-device-build-checks-it-at-embedded`;
  the same behaviour seen from the publish facets line is
  `novo/tooling/the-embedded-facet-names-modules-a-device-build-refuses`.
  The workaround is the shape `WORKAROUNDS.md` §2 records: the
  construction moves inside `system` and the answer is delivered by
  `return`, because a block whose own value is a `Result` is refused at
  a device boundary.  **Do not drop the `return`** when that section's
  filing closes without re-running the probe.  Sites: `ulid`'s
  `of_parts_checked`; `uliderr`'s `sentence_of`; `ulidseq`'s `issued`,
  `step`, `next`, `next_relaxed` and `peek`.
- **`ulidtext.encode_into` allocates a copy per call and cannot fill the
  buffer it is given.** The 0.0.2 interface froze
  `fn encode_into(u, out: Bytes, at: Int) -> (Bytes, Int)` and promised
  the characters go into `out`; a plain (non-`var`) parameter cannot be
  written through (E2036), and a `var` parameter would stop the same
  function's own doc example compiling, because the example binds the
  buffer with `let` (E2038) and `novo doc` compiles every example. The
  body answers `bytes.clone(out)` with the characters in it, so the
  caller threads the answer and `out` is a checked declaration rather
  than storage. Filed as
  `language/a-frozen-into-signature-cannot-fill-its-buffer`, sighted
  there beside totp-nv's identical `decode_into`; the site comment names
  the filing and the revert.
- **`ulidtext_tests.nv` asserted one wrong timestamp, twice.** The
  specification's own vector `01ARZ3NDEKTSV4RRFFQ69G5FAV` was asserted
  to carry millisecond 1469918176385; decoded with the rules the same
  file states — its alphabet, its digit values, its own round-trip case
  — the string carries 1469922850259. 1469918176385 belongs to
  `01ARYZ6S41TSV4RRFFQ69G5FAV`, the worked example in the JavaScript
  port's README, whose last sixteen characters are the same. Both
  assertions now read 1469922850259 and the two prose sentences carrying
  the same claim were corrected with them. Recorded in `ORACLE-EDITS.md`
  under the class the packet-nv correction established: a value that
  contradicts the specification the artifact names, settled by the
  artifact's own words.
- **`ulidgen.batch` reads the clock once, so a sequence whose last
  millisecond is ahead of it fails on the first step** with
  `UlidClockWentBackwards` rather than minting an identifier below the
  one it resumed from. That is the intended trade, and the one way to
  reach the batch's refusal without exhausting 2^80 values; the edge
  suite pins it.
- **`novo test` accepts several paths and runs only the last, silently.**
  Found while measuring coverage by hand and filed as
  `testing-quality/novo-test-with-two-paths-silently-runs-only-the-last`;
  the publish gate measures each suite separately and merges, so the
  package's own coverage number is not affected.

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
