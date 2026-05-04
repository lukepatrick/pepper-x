# W4c — Pin Rust Toolchain + Address Drift Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pin `rustc 1.95.0` via `rust-toolchain.toml`, suppress ~117 architectural clippy lints via workspace `[lints]` config, run `cargo fmt` to fix the upstream drift that broke W4b's CI, comment out 49 dead-code items with `// W4c-deadcode:` prefix tag, fix ~5 unused vars + ~3 unused imports + 1 `duplicate_mod` + ~20 misc style lints, bump `actions/checkout@v4 → @v5`. Result: all four CI gates (fmt, clippy, test, release-build) green; W4b's "done with caveats" status retroactively becomes "done".

**Architecture:** Single-branch workstream touching ~24 fmt-affected files + 11 Cargo.toml edits + ~50 lint-fix sites + 1 `rust-toolchain.toml` create + 1 `ci.yml` edit. Verification is a four-gate local pre-push check followed by a real GitHub Actions run after merge.

**Tech Stack:**
- `rustup` (project-toolchain auto-switching via `rust-toolchain.toml`)
- Cargo workspace `[lints]` table (Cargo 1.74+)
- `cargo fmt`, `cargo clippy`, `cargo test`, `cargo build --release`
- GitHub Actions (`actions/checkout@v5`, `dtolnay/rust-toolchain@stable`)

**Source spec:** `docs/superpowers/specs/2026-05-03-w4c-rust-toolchain-pin-design.md`

---

## File Structure

This plan modifies/creates these files. All in-repo on the W4c branch.

**In-repo (committed on `w4c-rust-toolchain-pin`):**
- Create: `rust-toolchain.toml` at repo root.
- Modify: `Cargo.toml` (workspace root) — add `[workspace.lints.clippy]` table.
- Modify: `app/Cargo.toml` and `crates/*/Cargo.toml` (11 member crates total) — add `[lints]` block with `workspace = true`.
- Modify: ~24 upstream `.rs` files via `cargo fmt` — exact list per W4b's CI run output (`app/src/transcript_log.rs`, `app/src/app.rs`, `app/src/cli.rs`, `app/src/history_view.rs`, `app/src/main.rs`, `app/src/overlay.rs`, `app/src/session_runtime.rs`, `app/src/settings_view.rs`, `app/src/transcription.rs`, `app/src/window.rs`, `crates/pepperx-asr/src/{lib,speaker_filter,transcriber}.rs`, `crates/pepperx-audio/src/{level_monitor,recording}.rs`, `crates/pepperx-cleanup/src/{cleanup,lib}.rs`, `crates/pepperx-cleanup-helper/src/main.rs`, `crates/pepperx-models/src/download.rs`, `crates/pepperx-platform-gnome/src/{atspi,evdev_modifier_capture,screenshot}.rs`, `crates/pepperx-uinput-helper/src/main.rs`).
- Modify: ~50 lint-fix sites scattered across the workspace — comment-outs (49 dead-code), renames (~5 unused vars), import comment-outs (~3), 1 `#[allow(clippy::duplicate_mod)]`, ~20 misc style fixes.
- Modify: `.github/workflows/ci.yml` — `actions/checkout@v4` → `@v5`.
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W4c, W4b's row updated `done (caveats)` → `done`, new W12 row at Phase 4, Phase 4 description update, 1 new deferred-ideas entry, status-log entries.

**No out-of-repo changes.** rustup auto-installs the pinned toolchain on first cargo invocation in the worktree (the user already has 1.95.0 per W9 verification, so no install needed).

---

## Task 1: Create W4c worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w4c-rust-toolchain-pin/`
- (git) Create branch: `w4c-rust-toolchain-pin` from `main`

- [ ] **Step 1: Verify clean main**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
git worktree list
```

Expected: clean working tree, on `main`, only the main checkout in worktree list (W6 + backup branches may exist; fine).

- [ ] **Step 2: Create the worktree**

```sh
git worktree add -b w4c-rust-toolchain-pin ../pepper-x.w4c-rust-toolchain-pin main
```

Expected: `Preparing worktree (new branch 'w4c-rust-toolchain-pin')` then `HEAD is now at <sha>`.

- [ ] **Step 3: Verify**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w4c-rust-toolchain-pin
git worktree list
git branch --show-current
```

Expected: two worktrees listed; current branch `w4c-rust-toolchain-pin`.

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w4c-rust-toolchain-pin/`.**

---

## Task 2: Update roadmap — W4c to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W4c row, status log.

- [ ] **Step 1: Update top status block**

Replace:

```
current_workstream:   W4c — Pin Rust toolchain version
phase:                2
state:                pending  (W4b done with caveats; W4c awaits brainstorming and naturally unblocks W4b's CI)
branch:               (none yet — created when W4c spec is written)
worktree:             (none yet)
last_updated:         2026-05-03
```

With:

```
current_workstream:   W4c — Pin Rust toolchain version
phase:                2
state:                in-progress
branch:               w4c-rust-toolchain-pin
worktree:             ../pepper-x.w4c-rust-toolchain-pin
last_updated:         2026-05-03
```

- [ ] **Step 2: Update W4c row in workstream table**

Find W4c's row. Change `State` from `pending` to `in-progress`; populate `Branch` (`w4c-rust-toolchain-pin`), `Spec` (`2026-05-03-w4c-rust-toolchain-pin-design.md`), `Plan` (`2026-05-03-w4c-rust-toolchain-pin.md`).

- [ ] **Step 3: Append status-log entry**

At the bottom under "## Status log":

```
- `2026-05-03` — W4c plan written and execution begun. State: `pending` → `in-progress`. Branch: `w4c-rust-toolchain-pin`. Reviewer pass surfaced six must-fix corrections (fmt-before-clippy sequencing; lint counts updated to fact-check truth (49 dead-code, ~20 misc, 24 fmt-affected files, 205 total); duplicate_mod is structural — `#[allow]` pre-decided; done criteria #10 ambiguity resolved; W4c branch push doesn't trigger CI; no existing [lints] blocks). Three new risk rows added (dtolnay@stable toml-precedence is folklore; apt-cargo contributor edge case; rustfmt 1.95 imports tightening).
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W4c in-progress"
```

---

## Task 3: Add `rust-toolchain.toml` and verify rustup auto-switches

**Files:**
- Create: `rust-toolchain.toml` at repo root.

- [ ] **Step 1: Create the file**

```sh
cat > rust-toolchain.toml <<'EOF'
[toolchain]
channel = "1.95.0"
components = ["rustfmt", "clippy"]
profile = "minimal"
EOF
cat rust-toolchain.toml
```

Expected: file content matches exactly the four lines above.

- [ ] **Step 2: Verify rustup auto-switches**

```sh
source $HOME/.cargo/env
rustc --version
cargo --version
```

Expected: both report `1.95.0`. If the user already had `1.95.0` installed (the case per W9 verification), this is instant. If not, rustup will download it on the first `cargo` invocation — that takes ~30 seconds and is one-time.

If `rustc --version` reports something other than 1.95.0, rustup is not honoring the toml. Possible cause: rustup binary too old (`rustup self update` should fix). Surface to user.

- [ ] **Step 3: Commit**

```sh
git add rust-toolchain.toml
git commit -m "W4c: add rust-toolchain.toml pinning rustc 1.95.0"
```

---

## Task 4: Add workspace `[lints]` config + per-crate `[lints]` blocks

**Files:**
- Modify: `Cargo.toml` (workspace root) — add `[workspace.lints.clippy]` table.
- Modify: `app/Cargo.toml`, `crates/pepperx-asr/Cargo.toml`, `crates/pepperx-audio/Cargo.toml`, `crates/pepperx-cleanup/Cargo.toml`, `crates/pepperx-cleanup-helper/Cargo.toml`, `crates/pepperx-corrections/Cargo.toml`, `crates/pepperx-ipc/Cargo.toml`, `crates/pepperx-models/Cargo.toml`, `crates/pepperx-platform-gnome/Cargo.toml`, `crates/pepperx-session/Cargo.toml`, `crates/pepperx-uinput-helper/Cargo.toml` — add `[lints]` block with `workspace = true`.

- [ ] **Step 1: Read root Cargo.toml**

```sh
cat Cargo.toml
```

Expected: a `[workspace]` declaration with `members = [...]` listing 11 crates. No existing `[workspace.lints]` table (per fact-check).

- [ ] **Step 2: Add `[workspace.lints.clippy]` table to root Cargo.toml**

Append to `Cargo.toml` (after the `[workspace]` section):

```toml

[workspace.lints.clippy]
result_large_err = "allow"
type_complexity = "allow"
too_many_arguments = "allow"
```

- [ ] **Step 3: Add `[lints]` block to each member crate**

For each of the 11 member crate Cargo.toml files, append:

```toml

[lints]
workspace = true
```

Loop:

```sh
for crate_toml in app/Cargo.toml crates/pepperx-asr/Cargo.toml crates/pepperx-audio/Cargo.toml crates/pepperx-cleanup/Cargo.toml crates/pepperx-cleanup-helper/Cargo.toml crates/pepperx-corrections/Cargo.toml crates/pepperx-ipc/Cargo.toml crates/pepperx-models/Cargo.toml crates/pepperx-platform-gnome/Cargo.toml crates/pepperx-session/Cargo.toml crates/pepperx-uinput-helper/Cargo.toml; do
  printf '\n[lints]\nworkspace = true\n' >> "$crate_toml"
done
```

Verify all 11 got the block:

```sh
grep -l '^\[lints\]$' app/Cargo.toml crates/*/Cargo.toml | wc -l
```

Expected: `11`.

- [ ] **Step 4: Run `cargo metadata` to verify workspace config parses**

```sh
cargo metadata --format-version 1 > /dev/null && echo "workspace config OK"
```

Expected: `workspace config OK`. If cargo errors out, the `[lints]` syntax is malformed somewhere; inspect output.

- [ ] **Step 5: Confirm the architectural lints are silenced**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings 2>&1 | grep -cE "result_large_err|type_complexity|too_many_arguments"
```

Expected: `0` (the workspace allow-list silences these). If non-zero, the workspace `[lints]` propagation isn't working — investigate the Cargo.toml entries.

- [ ] **Step 6: Commit**

```sh
git add Cargo.toml app/Cargo.toml crates/*/Cargo.toml
git commit -m "W4c: add workspace [lints] config to silence architectural clippy lints"
```

---

## Task 5: Run `cargo fmt` (BEFORE clippy enumeration per architect-review)

**Files:**
- Modify: ~24 upstream `.rs` files (per W4b's failed CI run output).

- [ ] **Step 1: Run cargo fmt across the workspace**

```sh
cargo fmt --all
```

Expected: no output (silent success). The command modifies ~24 files in place per fact-check.

- [ ] **Step 2: Verify fmt produced a diff**

```sh
git status
git diff --stat
```

Expected: ~24 modified `.rs` files in `app/src/` and `crates/*/src/`. Diff is mostly line-wrapping changes and some import reorganization (rustfmt 1.95 tightened `imports_granularity` and `group_imports` defaults).

- [ ] **Step 3: Sanity-check the diff isn't catastrophic**

```sh
git diff --stat | tail -5
```

Expected: total insertions + deletions in the low thousands at most. If the diff shows tens of thousands of changes, something is wrong (e.g., rustfmt config conflict).

- [ ] **Step 4: Commit**

```sh
git add -u
git commit -m "W4c: run cargo fmt to fix rustfmt 1.95.0 drift on upstream code"
```

---

## Task 6: Re-enumerate clippy lints

After workspace `[lints]` silenced ~117 architectural lints and `cargo fmt` resolved ~24 files of fmt drift, the remaining clippy errors should be ~88 (49 dead-code + 5 unused-vars + 3 unused-imports + 1 duplicate_mod + ~20 misc + ~10 unaccounted-for).

- [ ] **Step 1: Run clippy and capture full output**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings > /tmp/w4c-clippy-after-fmt.log 2>&1
echo "exit=$?"
```

Expected: `exit=101` (clippy exits non-zero when lints fire). Log captured for inspection.

- [ ] **Step 2: Categorize remaining lints**

```sh
grep -E "^error: " /tmp/w4c-clippy-after-fmt.log | grep -v "could not compile" | sed -E 's/`[^`]*`/X/g' | sort | uniq -c | sort -rn | head -25
```

Expected (approximate): ~49 "function/constant/variant/method/enum X is never used", ~5 "unused variable: X", ~3 "unused import" / "unused imports", 1 "file is loaded as a module multiple times", ~20 misc style lints (`needless_return`, `manual_is_multiple_of`, `io_other_error`, etc.).

- [ ] **Step 3: Note the per-crate distribution**

```sh
grep -E "^error: " /tmp/w4c-clippy-after-fmt.log | grep -v "could not compile" | grep -oE "(crates/[^/]+|app)/" | sort | uniq -c | sort -rn
```

Expected: high counts for `app/` and `crates/pepperx-platform-gnome/`, smaller counts for other crates.

(No commit yet — this task is data gathering for the next code-fixing tasks.)

---

## Task 7: Comment out dead code with `// W4c-deadcode:` prefix tag

**Files:**
- Modify: 49 dead-code sites across the workspace.

The `// W4c-deadcode:` prefix tag enables future grep-discoverability per the deferred-ideas entry "cleanup of commented-out dead code".

- [ ] **Step 1: List the dead-code sites**

```sh
grep -E "^error: " /tmp/w4c-clippy-after-fmt.log | grep -E "is never (used|constructed|read)" > /tmp/w4c-deadcode-list.log
wc -l /tmp/w4c-deadcode-list.log
```

Expected: `49` lines.

- [ ] **Step 2: For each dead-code item, add `// W4c-deadcode:` marker line and `//`-prefix the body**

The pattern for commenting out an item:

```rust
// Before:
const FOO: Duration = Duration::from_secs(30);

// After:
// W4c-deadcode: never used; clippy 1.95.0 dead-code lint
// const FOO: Duration = Duration::from_secs(30);
```

For multi-line items (functions, enums):

```rust
// Before:
fn wait_with_timeout(handle: ChildHandle) -> Result<()> {
    handle.wait_with_timeout(CLEANUP_SUBPROCESS_TIMEOUT)?;
    Ok(())
}

// After:
// W4c-deadcode: never used; clippy 1.95.0 dead-code lint
// fn wait_with_timeout(handle: ChildHandle) -> Result<()> {
//     handle.wait_with_timeout(CLEANUP_SUBPROCESS_TIMEOUT)?;
//     Ok(())
// }
```

Address each item from `/tmp/w4c-deadcode-list.log`. The file:line in each clippy line tells you exactly where to edit.

This is repetitive and tedious. The pattern is identical for every item. If a subagent is dispatched for this task, the subagent can iterate the list of 49 lines and apply the comment-out pattern.

- [ ] **Step 3: Verify all 49 dead-code lints are silenced**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings 2>&1 | grep -cE "is never (used|constructed|read)"
```

Expected: `0`. If non-zero, list the remaining items and address them (likely missed during Step 2's iteration).

- [ ] **Step 4: Verify the prefix tag is grep-discoverable**

```sh
grep -rn "W4c-deadcode:" app/ crates/ | wc -l
```

Expected: `49` (one tag per dead-code block).

- [ ] **Step 5: Commit**

```sh
git add -u
git commit -m "W4c: comment out 49 dead-code items with W4c-deadcode: prefix tag"
```

---

## Task 8: Rename unused vars + comment out unused imports + add `#[allow(clippy::duplicate_mod)]`

**Files:**
- Modify: ~5 unused-variable sites.
- Modify: 3 unused-import sites in `crates/pepperx-platform-gnome/src/context.rs:7,14`.
- Modify: `crates/pepperx-platform-gnome/src/lib.rs` (or `atspi.rs`) for the `#[allow(clippy::duplicate_mod)]`.

- [ ] **Step 1: Rename unused variables**

Get the list:

```sh
grep -E "^error: unused variable" /tmp/w4c-clippy-after-fmt.log
```

For each, navigate to the file:line and prefix the variable name with `_` (e.g., `state` → `_state`). Per fact-check, the most prominent is at `crates/pepperx-uinput-helper/src/main.rs:109`.

If a rename collides with an existing identifier, double-underscore (`__state`) is acceptable but unidiomatic. Surface to user if unsure.

- [ ] **Step 2: Comment out unused imports**

Get the list:

```sh
grep -E "^error: unused import" /tmp/w4c-clippy-after-fmt.log
```

Per fact-check, the imports are at `crates/pepperx-platform-gnome/src/context.rs:7,14`. Comment-out style:

```rust
// Before:
use zbus::blocking::Connection;
use crate::screenshot::{
    introspect_interface_xml, screenshot_window, validate_interface_xml, ScreenshotContractError,
};

// After:
// W4c-deadcode: unused import; clippy 1.95.0 unused-imports lint
// use zbus::blocking::Connection;
use crate::screenshot::{
    // W4c-deadcode: introspect_interface_xml, screenshot_window unused
    validate_interface_xml, ScreenshotContractError,
};
```

(For the second case, the line has multiple imports — only some are unused. Comment out only the unused ones in-line, OR comment out the whole `use` line and re-write with only the used items if simpler.)

- [ ] **Step 3: Add `#[allow(clippy::duplicate_mod)]`**

Per fact-check, `crates/pepperx-platform-gnome/src/lib.rs:2` has `pub mod context;` AND `crates/pepperx-platform-gnome/src/atspi.rs:9-10` has `#[path = "context.rs"] pub(crate) mod context;`. Add the allow attribute at the atspi.rs site (the deliberate-duplication site):

```rust
// Before (atspi.rs:9-10):
#[path = "context.rs"]
pub(crate) mod context;

// After:
// W4c: duplicate_mod is structurally deliberate — context.rs is re-mounted
// here as a child of atspi for `crate::atspi::context::*` paths. Re-routing
// through `crate::context` would be non-trivial and out of W4c scope.
#[allow(clippy::duplicate_mod)]
#[path = "context.rs"]
pub(crate) mod context;
```

- [ ] **Step 4: Verify all three lint families silenced**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings 2>&1 | grep -cE "unused variable|unused import|file is loaded as a module multiple times"
```

Expected: `0`.

- [ ] **Step 5: Commit**

```sh
git add -u
git commit -m "W4c: rename unused vars, comment out unused imports, allow structural duplicate_mod"
```

---

## Task 9: Fix misc style lints

**Files:**
- Modify: ~20 misc-lint sites across the workspace.

Per fact-check, the distribution: `needless_return` ×4, `manual_is_multiple_of` ×3, `io_other_error` ×2, plus singletons of `useless_conversion`, `single_char_add_str`, `question_mark`, `needless_range_loop`, `needless_borrows_for_generic_args`, `needless_borrow`, `manual_range_contains`, `manual_ok_err`, `manual_clamp`, `explicit_counter_loop`, `collapsible_if`.

- [ ] **Step 1: List the misc lints**

```sh
grep -E "^error: " /tmp/w4c-clippy-after-fmt.log | grep -v -E "is never (used|constructed|read)|unused variable|unused import|file is loaded as a module" > /tmp/w4c-misc-list.log
wc -l /tmp/w4c-misc-list.log
```

Expected: `~20` lines.

- [ ] **Step 2: Apply fixes one-by-one**

For each entry, navigate to the file:line and apply the fix per clippy's suggestion. Most are 1-3 line rewrites:

- **`needless_return`**: drop the trailing `return` keyword (e.g., `return foo;` → `foo`).
- **`manual_is_multiple_of`**: replace `x % n == 0` patterns with `x.is_multiple_of(n)`.
- **`io_other_error`**: replace `io::Error::new(io::ErrorKind::Other, msg)` with `io::Error::other(msg)`.
- **`useless_conversion`**: drop `.into()` when the source and target types are the same.
- **`single_char_add_str`**: replace `s.push_str("x")` with `s.push('x')`.
- **`question_mark`**: replace `match foo { Ok(x) => x, Err(e) => return Err(e) }` with `foo?`.
- **`needless_range_loop`**: replace `for i in 0..vec.len() { ... vec[i] ... }` with `for x in &vec { ... x ... }`.
- **`needless_borrows_for_generic_args`** / **`needless_borrow`**: drop the `&` from arg position when not required.
- **`manual_range_contains`**: replace `x >= a && x <= b` with `(a..=b).contains(&x)`.
- **`manual_ok_err`**: replace `match foo { Ok(x) => Some(x), Err(_) => None }` with `foo.ok()`.
- **`manual_clamp`**: replace `min(max(x, lo), hi)` with `x.clamp(lo, hi)`.
- **`explicit_counter_loop`**: use `.enumerate()`.
- **`collapsible_if`**: merge `if a { if b {} }` into `if a && b {}`.

If any fix feels like a real refactor (>5 lines or touches API), fall back to `#[allow(<lint>)]` with a `// TODO: W12` comment pointing at the future architectural-refactor workstream.

- [ ] **Step 3: Verify all misc lints silenced**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings 2>&1 | grep -cE "^error: "
```

Expected: ≤2 (just the "could not compile" summary lines, not real errors). If real errors remain, address them.

- [ ] **Step 4: Re-run cargo fmt (cheap; usually no-op since edits since Task 5 are commented-out blocks and small renames)**

```sh
cargo fmt --all
git diff --stat
```

Expected: empty diff (no changes). If non-empty, that means the lint fixes introduced fmt drift; commit those changes too.

- [ ] **Step 5: Commit**

```sh
git add -u
git commit -m "W4c: fix ~20 misc style lints (needless_return, manual_is_multiple_of, io_other_error, etc.)"
```

---

## Task 10: Local verification — all four CI gates

**Files:** none modified; pure verification.

- [ ] **Step 1: cargo fmt --check**

```sh
cargo fmt --check && echo "FMT OK"
```

Expected: `FMT OK` and exit 0. If diff-output appears, run `cargo fmt --all` again and re-commit (Step 4 of Task 9 should have caught this, but defense-in-depth).

- [ ] **Step 2: cargo clippy --workspace --no-deps --locked -- -D warnings**

```sh
cargo clippy --workspace --no-deps --locked -- -D warnings 2>&1 | tail -5
echo "exit=$?"
```

Expected: `exit=0` and `Finished` line. If any errors, list and address.

- [ ] **Step 3: cargo test --locked --workspace**

```sh
cargo test --locked --workspace 2>&1 | tail -15
echo "exit=$?"
```

Expected: `exit=0` and all crates report `test result: ok`. Per W9's verification, expect ~378 tests passing.

- [ ] **Step 4: cargo build --locked --release**

```sh
cargo build --locked --release 2>&1 | tail -5
echo "exit=$?"
```

Expected: `exit=0` and `Finished release` line. This may take 5-30 minutes on a cold cache (release build of llama.cpp + GTK + bindgen).

If any of the four gates fail, **STOP** — fix locally and re-run all four gates from scratch. Don't proceed to Task 11 with a red local gate.

---

## Task 11: Bump `actions/checkout@v5` in `ci.yml`

**Files:**
- Modify: `.github/workflows/ci.yml` — one-character bump.

- [ ] **Step 1: Apply the bump**

```sh
sed -i 's|actions/checkout@v4|actions/checkout@v5|' .github/workflows/ci.yml
grep "actions/checkout" .github/workflows/ci.yml
```

Expected: `      - uses: actions/checkout@v5`.

- [ ] **Step 2: Validate the YAML still parses**

```sh
yq '.' .github/workflows/ci.yml > /dev/null && echo "yq OK"
```

Expected: `yq OK`.

- [ ] **Step 3: Commit**

```sh
git add .github/workflows/ci.yml
git commit -m "W4c: bump actions/checkout@v4 → @v5 (Node.js 20 deprecation)"
```

---

## Task 12: Add W12 + Phase 4 description update + 1 deferred-ideas entry + W4b row update

Per the established W2/W4b pattern: this goes in a separate commit so reviewers can revert independently of the main W4c deliverables.

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Add W12 row to the workstream table**

Find the existing W11 row (Phase 4). Insert a new row immediately after W11:

```
| W12 | 4 | Architectural clippy refactor (box large errors, simplify complex types) | `pending` | — | — | — | maybe | **New 2026-05-03 from W4c brainstorm.** Comprises B deferred from W4c Q3: refactor `FriendlyInsertRunError` to `Box` variants triggering `result_large_err` (~103 sites), simplify `type_complexity` sites via type aliases (~10 sites), split `too_many_arguments` functions (~4 sites). Trigger: real friction from the current crate-level allow-list (e.g. an arch lint we want enforced everywhere except specific cases), OR a deliberate architectural cleanup phase. |
```

- [ ] **Step 2: Update Phase 4 description**

Find the existing Phase 4 paragraph in "## Phase descriptions". Update it to cover W12:

OLD:
```
**Phase 4 — Future polish.** Off-critical-path improvements: KDE-native UX (W10) and CI performance + matrix expansion (W11). Both review after Phase 2 work has settled. Either may decompose into sub-workstreams when brainstormed.
```

NEW:
```
**Phase 4 — Future polish.** Off-critical-path improvements: KDE-native UX (W10), CI performance + matrix expansion (W11), and architectural clippy refactor (W12). All review after Phase 2 work has settled. Each may decompose into sub-workstreams when brainstormed.
```

- [ ] **Step 3: Append one new deferred-ideas entry**

Find the "## Refactor / enhancement ideas (deferred)" section. After the last existing entry and before the closing line ("Future workstreams that surface deferred ideas append entries here..."), add:

```markdown
- **Cleanup of commented-out dead code (W4c-deadcode: blocks)** — periodically `grep -rn "W4c-deadcode:" app/ crates/` to find blocks added during W4c and decide which are truly archive-only vs. should be uncommented or deleted outright. Trigger: a comment-out block has been there 6+ months with no uncommenter, OR a refactor near commented-out code surfaces churn.
```

- [ ] **Step 4: Update W4b's row from `done (caveats)` → `done`**

Find W4b's row in the workstream table. Change the State column from `done (caveats)` to `done`. Append to the Notes column: "**Caveat resolved 2026-05-03 by W4c** (rust-toolchain.toml pinning + cargo fmt + workspace [lints] config). CI green on the next push to origin/main after W4c lands."

- [ ] **Step 5: Append a status-log entry**

```
- `2026-05-03` — Created W12 (Architectural clippy refactor) at Phase 4 alongside W10/W11; updated Phase 4 description; appended one new deferred-ideas entry (cleanup of W4c-deadcode: blocks). Updated W4b's row from `done (caveats)` → `done` (caveat retroactively resolved by W4c). Separate commit per architect-review's "scope creep" pattern so reviewers can revert independently of W4c's main code changes.
```

- [ ] **Step 6: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "$(cat <<'EOF'
Add W12 + 1 deferred-ideas entry; resolve W4b caveat retroactively

W12 captures the B option deferred from W4c brainstorm Q3:
architectural clippy refactor (box FriendlyInsertRunError to silence
~103 result_large_err sites, simplify ~10 type_complexity sites via
type aliases, split ~4 too_many_arguments functions). Phase 4 —
Future polish, alongside W10 and W11. Trigger: real friction from
the current crate-level allow-list, OR a deliberate architectural
cleanup phase.

One new deferred-ideas entry: periodic cleanup of W4c-deadcode:
commented-out blocks (grep-discoverable via the prefix tag W4c
introduced).

W4b's row updated from "done (caveats)" → "done": the caveat
(rustfmt drift breaking CI) is retroactively resolved by W4c
landing.

Separate commit per the W2 pattern so reviewers can revert this
independently of W4c's main code changes.
EOF
)"
```

---

## Task 13: Merge `w4c-rust-toolchain-pin` to main

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: clean working tree, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w4c-rust-toolchain-pin -m "$(cat <<'EOF'
Merge W4c: pin Rust toolchain + address drift

Pin rustc 1.95.0 via rust-toolchain.toml; suppress ~117 architectural
clippy lints (result_large_err ×103, type_complexity ×10,
too_many_arguments ×4) via workspace [lints] config; run cargo fmt
to fix the rustfmt drift that broke W4b's CI; comment out 49
dead-code items with W4c-deadcode: prefix tag for future grep-
discoverability; rename ~5 unused vars; comment out ~3 unused
imports; add #[allow(clippy::duplicate_mod)] for the deliberate
re-mount of context.rs in atspi.rs; fix ~20 misc style lints; bump
actions/checkout@v4 → @v5.

W12 (architectural clippy refactor) created at Phase 4 in a
separate commit; one new deferred-ideas entry (cleanup of W4c-
deadcode: blocks). W4b's row updated from "done (caveats)" → "done"
since W4c retroactively resolves the caveat.

CI green verification deferred to the next push to origin/main.

current_workstream advances to W5 (dev-install-extension.sh
graceful skip) on the wrap-up commit after CI is verified green.
EOF
)"
```

Expected: merge commit with the changed files.

- [ ] **Step 3: Do NOT push yet**

Per established pattern: work stays local until the user decides to push (with optional squash workflow). The push triggers CI which is the W4c verification step.

---

## Task 14: Cleanup — remove the worktree

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w4c-rust-toolchain-pin
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Switch back to main checkout, remove worktree**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w4c-rust-toolchain-pin
git worktree list
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout (W6 + backup branches still listed if present).

- [ ] **Step 3: Branch stays for now**

Don't delete `w4c-rust-toolchain-pin` yet — keep it as a recovery point until the user has pushed and CI is verified green.

---

## Task 15: 🛑 User action required — push and verify CI run

This is the W4c smoke gate.

- [ ] **Step 1: User pushes to origin**

User runs whichever matches their squash workflow:

Option A (preserve all commits as-is):
```sh
git push origin main
```

Option B (squash W4c's commits into a single workstream-shaped commit before pushing — matches W1/W6/W9/W2/W4b pattern):
```sh
git branch backup-pre-w4c-squash HEAD
git reset --soft origin/main
git commit -m "W4c: <workstream-shaped condensed message>"
git push origin main
```

The autonomous portion of W4c makes no assumption about which the user picks.

- [ ] **Step 2: User watches the GitHub Actions run**

After push, the user goes to **github.com/lukepatrick/pepper-x → Actions → CI** (the most recent run on `main`).

Expected outcome:
- All 7 steps green: checkout, install dependencies, Set up Rust, Check formatting, Lint with clippy, Run tests, Build release.
- Total wall-clock inside `timeout-minutes: 30`.
- **Verify dtolnay action toml-precedence**: in the "Set up Rust" step's logs, confirm rustup installed and used `rustc 1.95.0` (per the rust-toolchain.toml). If the logs show a different version (e.g. 1.95.1 or 1.96.0), the `dtolnay/rust-toolchain@stable` action did NOT defer to the toml — apply the mitigation in Step 3 below.

- [ ] **Step 3: User reports outcome**

User reports back with one of:
- **"CI green"** → proceed to Task 16.
- **"CI red — `<step>` failed: `<error>`"** → triage. Likely paths:
  - **`Check formatting` red**: rustup may have installed a different patch version than 1.95.0 (per the dtolnay@stable folklore risk). Fix-forward: pin the action explicitly via `dtolnay/rust-toolchain@1.95.0` instead of `@stable`. Make the change locally, commit, push.
  - **`Lint with clippy` red**: a clippy lint slipped through local verification (or 1.95.x patch differences). Fix locally and re-push.
  - **`Run tests` red**: a test that passed locally fails in CI — surface to user; investigate.
  - **`Build release` red**: a release-build issue not surfaced by local verification (unlikely since Task 10 Step 4 covers it).
  - **Timeout exceeded**: bump `timeout-minutes` or promote W11 caching ahead of schedule.

If the failure mode is the dtolnay@stable folklore (Step 2's verification of `rustc 1.95.0` in logs surfaces a different version), the fix-forward is straightforward.

---

## Task 16: After CI green — update roadmap to W4c done; delete merged branch

Only fires after the user reports CI green.

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Update top status block**

Replace W4c's in-progress block with:

```
current_workstream:   W5 — dev-install-extension.sh graceful skip on non-GNOME
phase:                2
state:                pending  (W4c done; W5 awaits brainstorming)
branch:               (none yet — created when W5 spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

- [ ] **Step 2: Mark W4c row as done**

Change W4c's State column from `in-progress` to `done`.

- [ ] **Step 3: Append status-log entry**

```
- `<today's date>` — W4c done. rust-toolchain.toml pins rustc 1.95.0; workspace [lints] silences ~117 architectural lints; cargo fmt fixed 24 upstream files; 49 dead-code items commented out with W4c-deadcode: prefix tag; ~5 unused vars renamed; ~3 unused imports commented out; #[allow(clippy::duplicate_mod)] on atspi.rs's re-mount of context.rs; ~20 misc style lints fixed; actions/checkout@v5 bumped. **CI green on the post-merge push** — all four gates (fmt, clippy, test, release-build) complete. dtolnay/rust-toolchain@stable correctly deferred to rust-toolchain.toml (rustc 1.95.0 used in CI). W4b's caveat retroactively resolved (its row updated to `done` in Task 12). State: `in-progress` → `done`. **current_workstream advances to W5**.
```

If CI was red and we fix-forwarded, reflect that honestly in the log entry (e.g., "CI red on first push due to dtolnay@stable not deferring to toml; fixed by pinning to @1.95.0; CI green on second push").

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W4c done; advance current_workstream to W5"
```

- [ ] **Step 5: Delete the merged branch**

```sh
git branch -d w4c-rust-toolchain-pin
git branch
```

If `git branch -d` refuses with "not fully merged" (likely after the user squashed in Task 15 Option B), force-delete with `-D` after verifying content equivalence:

```sh
git diff w4c-rust-toolchain-pin <squashed-commit-sha> --stat
# Should be empty diff. Then:
git branch -D w4c-rust-toolchain-pin
```

- [ ] **Step 6: No autonomous push**

The done-state commit is now ahead of origin/main (or part of the user's squashed commit if they squashed Task 15 Step 1 with this Task 16's commits). User pushes when ready.

---

## Done

When all 16 tasks are checked, W4c is complete:

- `rust-toolchain.toml` pins `rustc 1.95.0` for both local devs and CI.
- Workspace `[lints]` config silences ~117 architectural clippy lints; per-crate `[lints]` blocks added to all 11 members.
- `cargo fmt` applied across 24 upstream files; rustfmt 1.95.0 drift resolved.
- 49 dead-code items commented out with `// W4c-deadcode:` prefix tag for future grep-discoverability.
- ~5 unused vars renamed; ~3 unused imports commented out; `#[allow(clippy::duplicate_mod)]` on atspi.rs's deliberate re-mount; ~20 misc style lints fixed.
- `actions/checkout@v4 → @v5` resolves the Node.js 20 deprecation warning.
- All four CI gates (fmt, clippy, test, release-build) green on the real GitHub Actions run.
- W4b's row updated from `done (caveats)` → `done` (caveat retroactively resolved).
- W12 (architectural clippy refactor) created at Phase 4; 1 new deferred-ideas entry (cleanup of W4c-deadcode: blocks).
- Roadmap reflects W4c=done, W5=pending+next.

**CI is green for the first time on the fork.** Future contributors get deterministic clippy/fmt behavior; future rustc rolls require a deliberate `rust-toolchain.toml` bump rather than silent breakage.

**Next session-start re-orientation will pick up W5.** Per ways-of-working entry-point logic: `current_workstream: W5, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

| Risk | Where addressed |
|---|---|
| Pinning rustc 1.95.0 forces local devs to use that version | Task 3 Step 2 verifies; rustup auto-installs if not present. |
| `dtolnay/rust-toolchain@stable` may not defer to rust-toolchain.toml | Task 15 Step 2 verification + Step 3 fix-forward to `@1.95.0`. |
| Workspace `[lints]` mechanics gotchas | Task 4 Step 5 verifies architectural lints actually silenced. |
| Comment-out style preserves dead code that may rot | `// W4c-deadcode:` prefix tag in Task 7 enables future cleanup workstream (deferred-ideas entry created in Task 12). |
| `cargo fmt` modifies upstream files; merge conflicts on upstream sync | User-side concern; documented in spec, not addressed by plan. |
| `duplicate_mod` is structurally deliberate | Task 8 Step 3 applies `#[allow]` per fact-check finding. |
| The 4 manual-impl style lints might require code restructuring | Task 9 Step 2 fall-back to `#[allow]` with TODO pointing at W12. |
| `cargo build --release` was never verified green | Task 10 Step 4 makes it a hard pre-push gate. |
| Runner rustup resolves a patch release with different fmt rules | Task 15 Step 3 covers fix-forward. |
| Fresh contributor with apt-cargo silently uses 1.75 | Plan does not address; documented in spec as known limitation. |
| rustfmt 1.95.0 imports tightening makes diff wider than line-wrapping | Task 5 Step 3 sanity-check on diff size. |
