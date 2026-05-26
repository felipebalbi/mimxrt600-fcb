# AGENTS.md

Canonical guide for AI coding assistants (and humans) working on this
repository. Keep this file in sync with reality: if you change build
commands, CI, or conventions, update this document in the same PR.

## What this crate is

`mimxrt600-fcb` provides Rust types and enumerations for constructing a
valid **Flash Configuration Block (FCB)** for NXP MIMXRT6xx
microcontrollers (in particular the MIMXRT685S family). The FCB is a
fixed-layout, packed binary structure that the boot ROM reads from
external flash in order to initialize the FlexSPI controller before
fetching application code.

Key facts:

- Library crate, `no_std` (see `#![no_std]` at the top of `src/lib.rs`).
- Public API is a builder-style API on `FlexSPIFlashConfigurationBlock`
  using `const fn` methods so an FCB can be constructed at compile time
  and placed in a dedicated linker section by the consumer.
- Single dependency: [`bitfield`](https://crates.io/crates/bitfield)
  (`Cargo.toml`).
- License: MIT (`LICENSE`).
- Published on crates.io as `mimxrt600-fcb` (`Cargo.toml`).
- MSRV: **Rust 1.75** (`Cargo.toml` `rust-version = "1.75"`, also
  enforced by `.github/workflows/check.yml` `msrv` job).
- Edition: 2021.

This crate does **not** contain a HAL or a PAC — it only exposes types
that mirror the on-flash layout the boot ROM expects. There is no SVD,
IDL, or code generator: every type in `src/lib.rs` is hand-written and
must stay byte-compatible with the layout documented in the MIMXRT6xx
reference manual.

## Repository layout

```
.
├── AGENTS.md                       # this file (canonical agent guide)
├── Cargo.toml                      # package metadata, MSRV, deps
├── Cargo.lock                      # committed (binary-style crate)
├── LICENSE                         # MIT
├── README.md                       # short user-facing description
├── CONTRIBUTING.md                 # ODP-wide contribution licensing
├── CODE_OF_CONDUCT.md
├── CODEOWNERS
├── SECURITY.md
├── deny.toml                       # cargo-deny configuration
├── rustfmt.toml                    # rustfmt config (nightly-only opts)
├── src/
│   └── lib.rs                      # entire crate; ~620 lines
└── .github/
    ├── copilot-instructions.md     # commit-message & AI-attribution rules
    └── workflows/
        ├── check.yml               # fmt, clippy, doc, hack, deny, semver, msrv
        └── nostd.yml               # `cargo check` on a bare-metal target
```

There is no `tests/` directory, no `examples/` directory, no `benches/`,
and no `build.rs`. Everything the crate ships lives in `src/lib.rs`.

## Building and testing

The commands below are taken verbatim from `.github/workflows/check.yml`
and `.github/workflows/nostd.yml`. They are the authoritative set of
checks that CI runs on every pull request and on pushes to `main`. All
commands have been verified locally on Rust stable.

### Stable host build / check

```sh
cargo check
```

### Formatting

```sh
cargo fmt --check
```

Note: `rustfmt.toml` enables `group_imports = "StdExternalCrate"` and
`imports_granularity = "Module"`, which are **nightly-only** options.
On stable, `cargo fmt --check` will emit warnings of the form
`can't set ... unstable features are only available in nightly channel`
but still exits 0. CI uses stable rustfmt; running `cargo +nightly fmt`
locally is the only way to actually apply those import rules.

### Lints (clippy)

```sh
cargo clippy
```

CI runs clippy on both `stable` and `beta` toolchains. There is no
`-D warnings` flag in CI, but treat new clippy warnings as something you
must justify in the PR description. There is one pre-existing
`clippy::identity_op` warning in `flexspi_lut_seq` (`src/lib.rs` ~615);
do not let new code introduce additional warnings.

### Docs

```sh
cargo doc --no-deps --all-features
```

CI runs this on **nightly** with `RUSTDOCFLAGS=--cfg docsrs` to enable
`doc_cfg`. It still builds cleanly on stable.

### Feature powerset

```sh
cargo hack --feature-powerset check
```

Requires `cargo install cargo-hack` (CI uses `taiki-e/install-action`).
The crate currently declares no features in `Cargo.toml`, so this is
effectively a single `cargo check`, but the job exists to catch the
moment any feature is added.

### Dependency policy

```sh
cargo deny check --all-features
```

Requires `cargo install cargo-deny`. Configuration is in `deny.toml`.

### Semver checks

```sh
cargo semver-checks
```

Requires `cargo install cargo-semver-checks`. CI runs this on every PR
to catch accidental breaking changes to the public API. Because the
crate's surface is essentially the on-flash binary layout, **any change
to a public type, field order, or enum discriminant is almost certainly
a breaking change** and must be released with a major version bump.

### MSRV verification

```sh
cargo +1.75 check
```

The MSRV is pinned to 1.75 because:
- We rely on namespaced features (1.60).
- A transitive dependency requires Rust 1.71 (`fixed`).
- We also depend on `embedded-hal-async`, which requires 1.75.

Bumping MSRV must be a deliberate decision and must also bump the
`matrix.msrv` value in `.github/workflows/check.yml`.

### `no_std` target check

```sh
rustup target add aarch64-unknown-none-softfloat
cargo check --target aarch64-unknown-none-softfloat --no-default-features
```

This is what `nostd.yml` runs. The crate has no default features today,
so `--no-default-features` is a no-op, but keep the flag for parity with
CI in case features are added later. The real MIMXRT685S target is
`thumbv8m.main-none-eabihf`; the CI target is intentionally just *any*
bare-metal target to prove `no_std` portability.

### Tests

There are no unit or integration tests in this repository
(`grep -R "#\[test\]" src/` returns nothing, and there is no `tests/`
directory). `cargo test` runs successfully but executes 0 tests. If you
add tests, they must be gated so they do not break the `no_std` build —
either put them in `#[cfg(test)] mod tests` (which gets `std` because
test harnesses always link `std`) or in `tests/` as integration tests.

## Code conventions

### Source-file conventions

- `#![no_std]` at the top of `src/lib.rs` — do not remove without a very
  good reason; this crate is consumed from bare-metal firmware.
- `max_width = 120` (`rustfmt.toml`).
- Imports are grouped `std`/external/crate with `Module` granularity
  *on nightly only*; do not manually reformat imports against this rule.
- LF line endings everywhere (`git ls-files --eol` shows `i/lf w/lf` for
  every tracked file). There is no `.gitattributes` and no
  `.editorconfig`; agents must configure their editor for LF and run
  `git config core.autocrlf false` in the local clone.
- ASCII only in source files; no BOM.

### API-shape conventions

- All public constructors on `FlexSPIFlashConfigurationBlock` and its
  sub-structs are `pub const fn`. New methods should follow the same
  pattern so consumers can place an FCB in a `static` placed in a custom
  linker section.
- Builder methods take `self` by value and return `Self`, e.g.

  ```rust
  pub const fn block_size(self, _block_size: u32) -> Self {
      Self { _block_size, ..self }
  }
  ```

  Match this exact shape when adding new fields, including the leading
  underscore on the parameter name (it mirrors the private field name).
- `FlexSPIFlashConfigurationBlock` is `#[repr(C)]` **and** `#[repr(packed)]`
  because the boot ROM reads it as a packed C struct. Do not change
  field order, add fields, remove fields, or change types without
  cross-referencing the NXP MIMXRT6xx reference manual chapter on FlexSPI
  NOR boot. Every change here is an ABI break for downstream firmware.
- Enum discriminants are explicit (`= 0`, `= 1`, ...) and the enums are
  `#[repr(u8)]` (or whatever width the on-flash field requires).
  Discriminants are not free to change — they are wire values.
- Reserved fields are named `_reservedN` and initialized to the exact
  byte pattern documented by NXP (mostly zeros, with a couple of
  documented non-zero defaults such as `_reserved1: [2; 1]`).
  Do not "normalize" these to zero.

### Bitfields

- Use the `bitfield!` macro from the `bitfield` crate for register-style
  bitfields (see `ControllerMiscOption` and `SFlashPadType` in
  `src/lib.rs`). Bit positions are part of the ABI; do not reorder.

## Driver / PAC / generated-code specifics

This crate is not generated. There is no SVD, no `build.rs`, no
generator script. Every line in `src/lib.rs` is hand-maintained against
the NXP reference manual. When updating fields:

1. Cite the exact section of the NXP MIMXRT685S (or relevant MIMXRT6xx)
   reference manual in the PR description.
2. Preserve byte offsets — if you add a field, you must remove the
   corresponding number of bytes from a `_reservedN` array, not append.
3. Re-derive any default values from the reference manual; do not copy
   them from downstream firmware.
4. Bump the major version (`Cargo.toml`) and let `cargo semver-checks`
   confirm the break.

## Commit & PR conventions

These are derived from `.github/copilot-instructions.md` and verified
against the actual commit history (`git log --pretty=%s upstream/main`,
last 30 commits).

- **Subject line**: capitalized, ≤ 50 characters, imperative mood
  ("Fix bug", not "Fixed bug"). The history contains a mix of merge
  commits and short imperative subjects such as `Transfer ownership`,
  `Fix FCB regression`, `Add badges`, `Allow users to define custom FCBs`,
  `ci: enable SEMVER checks`. A short `type:` prefix (e.g. `ci:`,
  `docs:`) is accepted but not required.
- **Blank line** between subject and body.
- **Body** wrapped at 72 characters; explain *what* and *why*, not *how*.
- **AI attribution trailer** is mandatory for any commit that includes
  AI-generated or AI-assisted work:

  ```
  Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
  ```

  Examples: `Assisted-by: GitHub Copilot:claude-opus-4.7`,
  `Assisted-by: GitHub Copilot:gpt-5`. Basic dev tools (git, cargo,
  editors) are **not** listed. Agents must verify their own current
  identity before writing this trailer — never hard-code a model name
  carried over from a previous session.
- **`Signed-off-by` trailers are forbidden for AI agents.** Only humans
  can certify the Developer Certificate of Origin. If you are an AI
  agent, do not add `Signed-off-by`.
- PRs target `main`. CI must be green before merge: `fmt`, `clippy`
  (stable and beta), `doc`, `hack`, `deny`, `semver`, `msrv`, `no-std`.

## What not to do

- Do **not** change field order, field types, or enum discriminants in
  `src/lib.rs` without a corresponding reference-manual citation and a
  major version bump. These are wire-format values consumed by the boot
  ROM.
- Do **not** remove `#[repr(C)]` or `#[repr(packed)]` from
  `FlexSPIFlashConfigurationBlock` or any sub-struct that participates
  in the on-flash layout.
- Do **not** introduce `std`, `alloc`, or any dependency that pulls in
  `std` transitively. Run the `nostd.yml` check locally before pushing.
- Do **not** add `Signed-off-by` from an AI agent (see above).
- Do **not** force-push to `main`, and do **not** rewrite the history of
  any branch that already has an open PR.
- Do **not** edit `Cargo.lock` by hand. Let cargo update it; commit the
  result.
- Do **not** add a `.gitattributes` that enables CRLF — every file in
  this repo is LF and must stay LF.
- Do **not** enable nightly-only rustfmt options beyond those already in
  `rustfmt.toml`; CI runs stable `cargo fmt --check`.
- Do **not** add features without also verifying
  `cargo hack --feature-powerset check` passes — the existence of the
  `hack` CI job assumes feature additivity.

## How to find more context

- **NXP reference manual** for MIMXRT685S, chapter on FlexSPI NOR boot
  and the Flash Configuration Block: authoritative source for every
  field, default, and bit position. Cite it in PRs.
- `.github/workflows/check.yml` and `.github/workflows/nostd.yml`:
  authoritative source for build and lint commands.
- `.github/copilot-instructions.md`: commit-message and AI-attribution
  rules. Folded into this file (see below) and kept as a pointer.
- `CONTRIBUTING.md`: ODP-wide contribution licensing (MIT, you must own
  the rights to what you submit).
- `deny.toml`: license / advisory / source policy enforced by
  `cargo deny`.
- `README.md`: user-facing one-paragraph description and CI badges.

## Incorporated from `.github/copilot-instructions.md`

The following is folded in verbatim from `.github/copilot-instructions.md`
as of the time this file was written. `AGENTS.md` is a strict superset
of `copilot-instructions.md`; if the two ever diverge, `AGENTS.md` wins
and `copilot-instructions.md` must be updated to match.

---

### Commit Messages

- Subject line: capitalized, 50 characters or less, imperative mood
  (e.g., "Fix bug" not "Fixed bug")
- Separate subject from body with a blank line
- Wrap body text at 72 characters
- Use the body to explain *what* and *why*, not *how*

### AI Attribution

Every commit that includes AI-generated or AI-assisted work **must**
contain an `Assisted-by` trailer in the commit message:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

Where:

- `AGENT_NAME` is the name of the AI tool or framework
  (e.g., `GitHub Copilot`)
- `MODEL_VERSION` is the specific model version used
  (e.g., `claude-opus-4.6`)
- `[TOOL1] [TOOL2]` are optional specialized analysis tools used
  (e.g., `coccinelle`, `sparse`, `smatch`, `clang-tidy`)

Basic development tools (git, cargo, editors) should not be listed.

AI agents **must** verify their own identity (agent name and model
version) before composing the `Assisted-by` trailer — do not assume or
hard-code a model name from a previous session.

AI agents **MUST NOT** add `Signed-off-by` tags. Only humans can certify
the Developer Certificate of Origin.
