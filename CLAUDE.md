# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What cship is

`cship` is a CLI that renders the Claude Code statusline. Claude Code pipes session JSON to cship's stdin on every render; cship resolves a TOML config, renders the configured rows, and prints them to stdout. There is a hard **≤10ms render budget** — keep the hot path (parse → render → print) free of blocking work.

## Commands

```sh
# Full verification suite — run before every PR (CI enforces all four)
cargo fmt --check          # CI fails on unformatted code; run `cargo fmt` to fix
cargo clippy -- -D warnings # all clippy warnings are errors
cargo test
cargo build --release

# Run a single test by name (substring match across unit + integration tests)
cargo test test_parse_line_styled_span_with_content

# Run only the integration tests in tests/cli.rs
cargo test --test cli

# Run cship by hand — it reads session JSON from stdin
cat tests/fixtures/sample_input_full.json | cargo run

# Inspect what cship sees from Claude Code's context JSON (debugging aid)
cargo run -- explain
```

## Architecture

### The rendering pipeline (the part that needs multiple files to understand)

Data flows in one direction; each stage has a single, enforced owner:

1. **`main.rs`** — entry point. Parses CLI args (`clap`), reads stdin via `context::from_stdin()`, discovers config via `config::discover_and_load()`, calls `renderer::render()`, and `println!`s the result. `main.rs` is the **sole stdout writer in the rendering pipeline**.
2. **`context.rs`** — the **only** place stdin is read and the **only** `serde_json` deserialization. Produces a `Context` struct. Every field is `Option<T>` because Claude Code omits fields depending on session state/version.
3. **`config.rs`** — `CshipConfig` plus the discovery chain (see below). cship config lives under the `[cship]` section of a `starship.toml`, or in a dedicated `cship.toml`.
4. **`renderer.rs`** — splits config into rows (the `lines` array, or `format` split on `$line_break`), then tokenizes each row into a `Token` enum: `Native` (`$cship.*`), `Passthrough` (any other `$word`, e.g. `$git_branch`), `StarshipPrompt` (`$starship_prompt`), `Literal` (bare text, spacing is encoded here — tokens are joined with no separator), and `StyledSpan` (`[content](style)`). Native tokens dispatch to `modules::render_module()`; passthrough tokens to `passthrough::*`.
5. **`modules/mod.rs`** — `render_module()` is a static-dispatch `match` mapping every `cship.*` token name to a module function. `ALL_NATIVE_MODULES` (the const above it) **must stay in sync** with the match arms — `cship explain` enumerates it.
6. **`modules/{name}.rs`** — each module is `pub fn render(ctx: &Context, cfg: &CshipConfig) -> Option<String>` (the non-negotiable interface). Sub-fields like `cship.cost.total_cost_usd` are additional functions in the same module file.
7. **`ansi.rs`** — all style/color logic. `apply_style` parses Starship-style strings (`"bold green"`, `"fg:#7aa2f7 bg:red"`); `apply_style_with_threshold` switches base → warn → critical style by numeric value.
8. **`format.rs`** — per-module Starship-style `format` string parser (`$value`, `$symbol`, `[..](style)` spans, `(..)` conditional groups). `render_styled_value` centralizes sub-field → parent style/threshold fallback.

### Architectural boundaries (single-owner rules — do not cross them)

- **stdin**: read only in `context.rs`.
- **stdout** (rendering pipeline): written only in `main.rs`. Modules return `Option<String>`; they never print.
- **HTTP / network**: only in `usage_limits.rs`. No other file makes external calls.
- **OAuth token**: fetched from the OS credential store in `platform.rs`, held in memory only for the API call, never written to disk/cache/log.
- **Diagnostics**: everything goes through `tracing::*` macros to stderr (configured in `main.rs`). No `eprintln!` anywhere.

### Config discovery chain (`config.rs`)

In priority order: `--config` override → walk up from the workspace dir checking `cship.toml` then `starship.toml` at each level → global `~/.config/cship.toml` then `~/.config/starship.toml` → built-in default. A dedicated `cship.toml` is accepted both with a `[cship]` header (canonical) and without one (legacy wrapper-free). Parse errors propagate and exit non-zero; a missing config is not an error.

### usage_limits — the most complex subsystem

`src/modules/usage_limits.rs` (render/format) + `src/usage_limits.rs` (HTTP/parse) + `src/cache.rs` (disk cache). It blends three data sources, freshest first:

1. **stdin `rate_limits`** — Claude Code sends 5h/7d percentages every render for Pro/Max subscribers (no network needed).
2. **OAuth API cache** — adds per-model (opus/sonnet/cowork/oauth_apps) and extra-usage breakdowns. Cached on disk (60s TTL + early invalidation on window reset; a negative marker suppresses retries after failures).
3. **Live OAuth fetch** — only on cache miss: `std::thread::spawn` with a timeout so a slow/hung API call can't blow the render budget; the result is written to cache for the next render.

The disk cache lives next to the session transcript: `{dirname(transcript_path)}/cship/...`.

## Non-Negotiable Code Patterns

- Module interface (never deviate): `pub fn render(ctx: &Context, cfg: &CshipConfig) -> Option<String>`
- Disabled flag → silent `None` (no warn); absent data → explicit `match` + `tracing::warn!` + `None`
  - Exception: `context_bar` intentionally renders a 0% empty bar (styled via `empty_style`) when `context_window` is absent, rather than returning `None`. This is a deliberate UX choice — showing an empty bar is more informative than showing nothing. It uses `tracing::debug!` (not `warn!`) because absence is the normal state at session start.
- Never use `?` operator on paths that require a warning — use explicit `match`
- stdout owned by `main.rs` only; all module diagnostics via `tracing::*` macros; no `eprintln!` anywhere
- Exception: CLI-action subcommands (e.g. `uninstall`, `explain`) may use `println!` directly — the stdout rule applies to the rendering pipeline only
- All config structs: `#[derive(Debug, Deserialize, Default)]`, all fields `pub Option<T>`
- Never add `deny_unknown_fields` to any struct — omitted intentionally on both `Context` and config structs so future versions can add fields without breaking deserialization

## Adding a native module

- Create `src/modules/{name}.rs` and update `src/modules/mod.rs` (add a `pub mod`, a `render_module()` match arm, and an `ALL_NATIVE_MODULES` entry). If the module needs config fields, also add a struct to `src/config.rs`. That's the maximum file set: 2 files (3 with config).
- Config structs → `src/config.rs` only; ANSI logic → `src/ansi.rs` only; threshold styling → `ansi::apply_style_with_threshold`.
- Sub-field configs use `SubfieldConfig` and inherit from their parent via the `HasThresholdStyle` trait; use `invert_threshold` for decreasing-health indicators (low = bad, e.g. `remaining_percentage`).

## Testing conventions

- Most tests are `#[cfg(test)]` modules colocated with the code; CLI/integration tests live in `tests/cli.rs` (driven with `assert_cmd`). JSON/TOML fixtures live in `tests/fixtures/`.
- Build `Context` and `CshipConfig` inline with `..Default::default()` to test a module in isolation — don't add new fixture files for unit tests.
- HOME-dependent discovery paths are deliberately covered by integration tests, not unit tests (mutating `HOME` is unsafe under parallel test execution in Rust 2024) — follow the existing comments in `config.rs` rather than reintroducing env-mutating unit tests.

## Documentation

When changing modules or config fields, update `docs/` per [CONTRIBUTING.md](CONTRIBUTING.md): config reference lives in `docs/configuration.md`, UX notes in `docs/faq.md`, examples in `docs/showcase.md`. `CONTRIBUTING.md` (repo root) and `docs/contributing.md` are kept in sync manually — edit both.

## Before Submitting a PR

- Run the full verification suite above and follow all guidelines in [CONTRIBUTING.md](CONTRIBUTING.md).

## Environment Quirks

- WSL2 ENOENT race: after any `cargo init` or file write, verify content with Read before proceeding.
