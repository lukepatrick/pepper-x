# W4c — Pin Rust Toolchain + Address Drift

Workstream W4c of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). Pins `rustc` via `rust-toolchain.toml`, runs `cargo fmt` to fix the rustfmt drift that broke W4b's CI, suppresses architectural clippy lints via workspace `[lints]` config, comments out dead code, fixes a handful of misc lints, and bumps `actions/checkout@v4 → v5`.

## Goal

Make pepper-x's CI deterministic across rustup updates so future runs don't break randomly when the rustc/rustfmt/clippy version that `dtolnay/rust-toolchain@stable` resolves to rolls forward. Concrete outcomes: W4b's red CI goes green; the four CI gates (fmt, clippy, test, release-build) all pass; future rustc rolls require a deliberate `rust-toolchain.toml` bump rather than silent breakage.

## Background — why this is Phase 2 (and why now)

W4b shipped "done with caveats" because `cargo fmt --check` failed on upstream-authored `app/src/*.rs` files. Investigation: the failure was rustfmt 1.95.0 changing line-wrapping rules; upstream's code was last fmt-clean against rustc 1.94.x (their commit `66c927b` predates the 1.95 release by ~1 week). The CI was passing before because runs hadn't happened since the rustfmt roll.

Per W4c brainstorm Q1: pin to **`1.95.0`** (current latest stable) and run `cargo fmt` to update upstream code, rather than pin to `1.94.0` (which would preserve upstream code as-is). This trades a one-time upstream-code modification for keeping the toolchain fresh.

Per Q2 + Q3: address clippy drift in W4c too, comment-out dead code (per user preference vs `#[allow(dead_code)]`), and use workspace `[lints]` config (Q3 option C) to silence the ~130 architectural lints (`result_large_err`, `type_complexity`, `too_many_arguments`) that can't be commented out because the affected code is actively used.

After W4c lands and CI is green, **W4b's "done (caveats)" status retroactively becomes "done"** — the caveat was the rustfmt drift that W4c fixes.

## Done criteria

1. **`rust-toolchain.toml` exists at repo root** with `channel = "1.95.0"`, components `["rustfmt", "clippy"]`, profile `"minimal"`. Picked up automatically by rustup (local devs auto-switch on `cd`) and by `dtolnay/rust-toolchain@stable` action in CI (defers to the toml when present).
2. **Workspace `[lints]` configured** in root `Cargo.toml` with `result_large_err = "allow"`, `type_complexity = "allow"`, `too_many_arguments = "allow"` under `[workspace.lints.clippy]`. Each member crate has a fresh `[lints]` block with `workspace = true` (fact-check confirmed: no member crate currently has any `[lints]` block, so all 11 additions are add-fresh, not extend).
3. **`cargo fmt` applied** across the workspace. Upstream files modified in idiomatic line-wrapping. After this, `cargo fmt --check` passes.
4. **Dead code commented out** (per user preference, NOT `#[allow(dead_code)]`). Total **49 items** across the workspace per the fact-check. Distribution:
   - `pepperx-platform-gnome/src/atspi.rs`: `V_KEYSYM`, `CLIPBOARD_RESTORE_DELAY`, `ClipboardPaste` enum variant, plus the dead functions/methods enumerated at execution time
   - `pepperx-cleanup/src/cleanup.rs`: `CLEANUP_SUBPROCESS_TIMEOUT` constant, `wait_with_timeout` function (`:506`)
   - `pepper-x-app`: dead functions, dead associated functions, dead methods, dead enums (~16 functions + ~5 assoc + ~2 methods + ~2 enums per cargo clippy)
   - Smaller counts in `pepperx-asr` (3), `pepperx-audio` (4), `pepperx-models` (1), `pepperx-cleanup-helper` (3)
   - **Comment style**: each line `//`-prefixed AND a leading marker `// W4c-deadcode:` on the first line of each commented block. The marker enables future grep-discoverability (the deferred-ideas entry for "cleanup of commented-out code" can `grep -rn "W4c-deadcode:"` to find blocks for review).
5. **Unused variables renamed** to `_var` form (idiomatic Rust):
   - `pepperx-uinput-helper/src/main.rs:109`: `state` → `_state`
   - `pepper-x-app`: ~4 unused variables
6. **Unused imports commented out** (per user preference, same as dead code):
   - `pepperx-platform-gnome/src/context.rs:7,14`: `use zbus::blocking::Connection;` and the line containing `introspect_interface_xml, screenshot_window`
7. **`duplicate_mod` lint resolved** with `#[allow(clippy::duplicate_mod)]` (decided pre-execution per fact-check): the duplication is **structurally deliberate** — `crates/pepperx-platform-gnome/src/lib.rs:2` has `pub mod context;` AND `crates/pepperx-platform-gnome/src/atspi.rs:9-10` has `#[path = "context.rs"] pub(crate) mod context;`. The atspi.rs declaration uses `#[path]` to re-mount the same file as a child module of `atspi`, presumably for `crate::atspi::context::*` paths. Re-routing through `crate::context` would be non-trivial and out of W4c scope. Add the `#[allow]` at one of the declaration sites with a comment pointing at this finding.
8. **Misc style lints addressed individually**: per fact-check, **~20 items total** (not 5-7 as initially estimated). Distribution: `needless_return` ×4, `manual_is_multiple_of` ×3, `io_other_error` ×2, plus singletons of `useless_conversion`, `single_char_add_str`, `question_mark`, `needless_range_loop`, `needless_borrows_for_generic_args`, `needless_borrow`, `manual_range_contains`, `manual_ok_err`, `manual_clamp`, `explicit_counter_loop`, `collapsible_if`. Each a tiny rewrite (1-3 lines). Fall back to `#[allow]` with TODO comment pointing at W12 if any feels like a real refactor.
9. **`.github/workflows/ci.yml` updated**: `actions/checkout@v4` → `@v5` (one-character bump). Fixes the Node.js 20 deprecation warning surfaced in W4b's CI run.
10. **CI green on the next push to origin/main after merge.** Note: the W4c branch push won't trigger CI — `.github/workflows/ci.yml` is configured for `push: branches: [main]` + `pull_request: branches: [main]`. We don't open PRs in this fork's workflow, so the CI run that gates W4c-done is the one triggered by the merged squashed commit reaching `origin/main`. **All four gates (fmt, clippy, test, release-build) must complete green in <30 minutes per the W4b-set timeout — including release-build, which is now part of the local-verification gate (criterion #10) AND the CI verification.** This resolves the ambiguity from the original spec where release-build was listed in both local-verify and "fix-forward" risk: it's a hard pre-push gate and a hard post-push gate. If release-build fails locally, fix before push; if it fails in CI despite local-green, fix-forward via a follow-up commit.
11. **W4b's roadmap row updated**: `done (caveats)` → `done`. One-line note in W4b's row referencing W4c as the resolution.
12. **Roadmap updates**:
    - W4c row state advances `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populated.
    - `current_workstream:` advances to W5 on completion.
    - W12 row added (Phase 4 — Architectural clippy refactor) in a separate commit so reviewers can revert independently.
    - One new entry appended to "Refactor / enhancement ideas (deferred)" — cleanup of commented-out dead code — in the same separate commit.
    - Phase 4 description updated to cover W10 + W11 + W12.

## Approach

### Change 1 — `rust-toolchain.toml`

New file at `/rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.95.0"
components = ["rustfmt", "clippy"]
profile = "minimal"
```

`profile = "minimal"` skips installing docs/source by default (faster `rustup toolchain install` on fresh local checkouts).

### Change 2 — Workspace `[lints]` config

Add to root `Cargo.toml`:

```toml
[workspace.lints.clippy]
result_large_err = "allow"
type_complexity = "allow"
too_many_arguments = "allow"
```

Plus add `[lints]` with `workspace = true` to each member crate (`app/Cargo.toml`, `crates/*/Cargo.toml`):

```toml
[lints]
workspace = true
```

If a member crate already has a `[lints]` block, extend it rather than overwrite. **Silences ~130 errors with one config table.**

### Change 3-8 — Code-level fixes

Per the per-crate clippy enumeration:
- ~30-40 dead-code items: comment-out (lines `//`-prefixed)
- ~4-8 unused variables: rename to `_var`
- ~3 unused imports: comment-out
- 1 duplicate_mod: investigate; fix or `#[allow]`
- ~5-7 misc style lints: individual rewrites

The exact item list is enumerated by running `cargo clippy --workspace --no-deps --locked -- -D warnings` after Change 2 lands (the workspace `[lints]` allows reduce the noise). Execution-time enumeration; not pre-locked in the spec because the lint set may shift slightly when other items in the same crate are addressed.

### Change 9 — `cargo fmt`

After Changes 1-8 are in place, run `cargo fmt` once. Modifies upstream `app/src/transcript_log.rs:733`, `app/src/app.rs:18/51/106/192`, `app/src/cli.rs:374`, and a few others surfaced by W4b's failed CI run. The user's reading of the run page is the canonical list.

### Change 10 — `actions/checkout@v5`

In `.github/workflows/ci.yml`, change `- uses: actions/checkout@v4` to `- uses: actions/checkout@v5`. v5 is a drop-in replacement for our usage; runs on Node.js 20+ (will run on 24 by default later, opt-in available now).

### Sequencing notes (corrected per architect-review)

The fixes don't strictly need to happen in order, but a correct sequence is:
1. Add `rust-toolchain.toml` first (locks the toolchain so subsequent `cargo *` runs are deterministic).
2. Verify rustup auto-switches to 1.95.0 (`rustc --version` shows 1.95.0).
3. Add workspace `[lints]` config + per-crate `[lints]` blocks (silences ~117 architectural lints up front: 103 `result_large_err` + 10 `type_complexity` + 4 `too_many_arguments`).
4. **Run `cargo fmt` BEFORE clippy enumeration.** Reason (architect-review): rustfmt rewrites can resurface or silence lints (e.g. `needless_borrow` triggered by line-wrap; `single_char_add_str` resolved by reformatting). Doing fmt first means clippy enumeration sees the FINAL formatted state. Modifies ~24 upstream files.
5. Re-run `cargo clippy --workspace --no-deps --locked -- -D warnings` to enumerate the remaining ~88 lints to address (49 dead-code + 5 unused-variables + 3 unused-imports + 1 duplicate_mod + ~20 misc + ~10 unaccounted-for).
6. Address: comment-out dead code (use prefix tag `// W4c-deadcode:` for grep-discoverability), rename unused vars to `_var`, comment-out unused imports, **`#[allow(clippy::duplicate_mod)]` on `crates/pepperx-platform-gnome/src/lib.rs`** (the duplication is deliberate — `atspi.rs` uses `#[path = "context.rs"]` to re-mount the same module; fact-checked), fix ~20 misc style lints individually (or `#[allow]` with TODO pointing at W12 if any feels like a real refactor).
7. Re-run `cargo fmt` (cheap; usually a no-op since the only edits since step 4 are commented-out blocks and small renames).
8. Run all four CI gates locally — `cargo fmt --check`, `cargo clippy --workspace --no-deps --locked -- -D warnings`, `cargo test --locked --workspace`, `cargo build --locked --release`. All four must exit 0 before push.
9. Bump `actions/checkout@v5` in `.github/workflows/ci.yml`.
10. **Verify `dtolnay/rust-toolchain@stable` actually defers to `rust-toolchain.toml`.** The action's README does not document toml-precedence behavior (folklore per fact-check); empirical verification at execution. Mitigation if it doesn't: pin the action to `dtolnay/rust-toolchain@1.95.0` instead of `@stable` as belt-and-braces.
11. Commit, merge, push, watch CI.

## Testing strategy

### Local verification (mandatory before push)

After all changes are in place, run the four CI gates locally to verify they pass:

```sh
cargo fmt --check
cargo clippy --workspace --no-deps --locked -- -D warnings
cargo test --locked --workspace
cargo build --locked --release
```

All four must exit 0. If any fails, that's a sign W4c isn't done yet — fix and re-run.

### CI verification (post-push)

After the push, watch the GitHub Actions run on `github.com/lukepatrick/pepper-x → Actions → CI`. Expected: same four gates green; total wall-clock <30 minutes per the W4b-set timeout.

If CI is red despite local-green, possible causes:
- Runner-image rustup picks up 1.95.1 (or some patch release) that has different fmt rules than my local 1.95.0. Mitigation: bump pin to the patch version that's failing.
- Caching artifact mismatch (unlikely — no caching configured per W11 deferral).
- A genuine packaging issue not surfaced locally (e.g., release-build needs a sys-lib not in the apt list). Surface to user; fix-forward.

### What this can't easily test

- `actions/checkout@v5` runtime behavior — but it's a drop-in replacement for our usage; CI's checkout step will surface any incompatibility immediately.
- `cargo fmt --check` exact output across rustfmt 1.95.0 patch versions — see CI verification mitigation above.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Pinning rustc 1.95.0 forces local devs to use that version | Low | rustup auto-installs/switches per-project. Other projects unaffected. User already runs 1.95.0 locally per W9 verification. |
| New rust release (1.96+) introduces fmt/lint changes requiring a deliberate pin bump | Low | `rust-toolchain.toml` makes the bump a deliberate commit. Drift no longer surprises CI. |
| Workspace `[lints]` requires `lints.workspace = true` in 11 member crates | Low | Mechanical edit. Extend existing `[lints]` blocks if present. |
| Comment-out style preserves dead code that may rot over time | Medium | User's explicit preference; deferred-ideas entry created for future cleanup-of-commented-out-code. |
| `cargo fmt` modifies upstream files; merge conflicts if upstream `obra/pepper-x` updates the same files | Low | Upstream's last commit was 2026-04-07; minimal recent activity. User handles merge conflicts manually if/when syncing from upstream. |
| Change 8 (`duplicate_mod` investigation) might reveal duplication is structurally needed | Low | Fall back to `#[allow(clippy::duplicate_mod)]` with explanatory comment. |
| The 4 manual-impl style lints + others might require code restructuring beyond `#[allow]` | Low | Each is small (1-3 lines). Fall back to `#[allow]` with TODO comment pointing at W12 if any feels like a real refactor. |
| `cargo build --release` (W4b-added step) was never verified green; might fail post-W4c too | Medium | After W4c, CI gates run in order: fmt → clippy → test → release-build. If release-build fails, surface as separate fix-forward; W4c's primary goal (fmt + clippy green) is still met. |
| Runner rustup resolves a patch release (e.g. 1.95.1) with different fmt rules than my local 1.95.0 | Very Low | Per fact-check: `channel = "1.95.0"` in `rust-toolchain.toml` is **exact-pinned** by rustup (MAJOR.MINOR.PATCH treated as a literal version, not a range). So a runner won't silently jump to 1.95.1. The actual risk is downstream — see next row. |
| `dtolnay/rust-toolchain@stable` doesn't actually defer to `rust-toolchain.toml` | Medium | Architect-review + fact-check both flag this is **folklore** — the action's README does not document toml-precedence behavior. **Mitigation**: empirical verification at execution time (Step 10 of sequencing). If the action overrides the toml's pinned patch, fall back to `dtolnay/rust-toolchain@1.95.0` as belt-and-braces. |
| Fresh contributor with apt-cargo (not rustup) silently uses Rust 1.75 and ignores the toml | Low | Toml is a rustup convention; apt-cargo doesn't read it. For a one-contributor personal fork this is negligible; CLAUDE.md and W4a's README already direct contributors to install via rustup. Documented for awareness. |
| rustfmt 1.95.0 tightened `imports_granularity` and `group_imports` defaults | Low | The fmt diff may be wider than just line-wrapping — could rewrite imports broadly. Acceptable; the diff is reviewable in the W4c commit. No `rustfmt.toml` exists in the repo (fact-check confirmed), so we're using rustfmt defaults; the bump is intentional. |
| `profile = "minimal"` only honored on first install for a given channel | Cosmetic | If the contributor already has 1.95.0 installed via `default` profile, `minimal` is a no-op. Doesn't affect correctness; just installation size. |

## Out of scope

- **Genuine architectural refactor** (boxing `FriendlyInsertRunError`, simplifying complex types via type aliases, splitting wide-arg functions) — captured as **W12** for future. Trigger: real friction from the current crate-level allow-list, OR a deliberate architectural cleanup phase.
- **Deletion of commented-out code** — preserved per user preference. Future cleanup candidate (deferred-ideas entry created in this workstream).
- **Bumping rustc beyond 1.95.0** — out of scope; deliberate-bump-when-needed posture.
- **Adding more clippy categories to the allow-list** — only the 3 architectural categories are bulky enough to need workspace-level allow. Other lints stay enforced.
- **Cleaning up upstream code beyond what clippy lints require** — out of scope.
- **Upstream PR filing** — local-only per established pattern.

## Auxiliary deliverables (folded into W4c's commits)

Per established W2/W4b pattern: W12 creation goes in a separate commit on the W4c branch so reviewers can revert independently.

1. **Roadmap state transitions** — W4c row `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populate. `current_workstream:` advances to W5.
2. **W4b's row updated** from `done (caveats)` → `done` with a one-line note pointing at W4c as the resolution.
3. **Status-log entries** — one per state transition; final entry captures CI green + W4b caveat-resolved.
4. **New workstream W12** added to the workstream table at Phase 4 (alongside W10, W11), in a separate commit:
   - `W12 | 4 | Architectural clippy refactor (box large errors, simplify complex types) | pending | — | — | — | maybe | New 2026-05-03 from W4c brainstorm. Comprises B deferred from W4c Q3: refactor FriendlyInsertRunError to Box variants triggering result_large_err (~120 sites), simplify type_complexity sites via type aliases (~10 sites), split too_many_arguments functions (~2 sites). Trigger: real friction from the current crate-level allow-list (e.g. an arch lint we want enforced everywhere except specific cases), OR a deliberate architectural cleanup phase.`
5. **One new entry appended to "Refactor / enhancement ideas (deferred)"** — cleanup of commented-out dead code (periodically review `// fn …` / `// const …` blocks added during W4c and decide which are truly archive-only vs. should be uncommented or deleted outright). Trigger: a comment-out block has been there 6+ months with no uncommenter, OR a refactor near commented-out code surfaces churn.
6. **Phase 4 description updated** to cover W10 + W11 + W12.

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. Plan will translate the sequenced changes into ordered, individually-verifiable tasks. The local-verification step (running the four CI gates locally) is the autonomous gate; the CI run on origin is the user-action checkpoint after push.
