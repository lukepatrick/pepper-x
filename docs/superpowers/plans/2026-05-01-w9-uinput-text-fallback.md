# W9 — uinput-text Fallback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make pepper-x's text-insertion path actually use its designed-but-unreachable uinput-text fallback by (a) opening the `ensure_runtime_supported_backend` gate to accept `UINPUT_TEXT_BACKEND_NAME` and (b) wrapping AT-SPI infrastructure / target-selection failures inside `focused_friendly_target` as `SelectedBackendFailure { backend_name = UINPUT_TEXT_BACKEND_NAME, .. }` so the existing wrapper at `app/src/transcription.rs:474-494` routes to the helper as designed.

**Architecture:** Two complementary changes in one Rust file. (1) Open the gate (4-line change in a `matches!` arm). (2) Introduce a small pure helper `wrap_atspi_failure_as_uinput_fallback` and apply it via `.map_err` at three failure sites in `focused_friendly_target` (`find_focused_accessible`, `inspect_focused_target_from_accessible`, `select_friendly_insert_backend`). The wrapper at `transcription.rs` is unchanged in production code; one new mock-based test verifies the synthetic-error → wrapper → helper routing end-to-end.

**Tech Stack:**
- Rust workspace, rustup stable (1.95.0 as of 2026-05-01)
- Crate `pepperx-platform-gnome` (the file `atspi.rs`)
- Crate `pepper-x-app` (the file `app/src/transcription.rs` — test only, no production change)
- `pepperx-uinput-helper` binary already installed at `/usr/libexec/pepper-x/pepperx-uinput-helper` from W1
- Manual smoke discriminators: dictating into Kate / Konsole on KDE, `tee /tmp/w9-smoke.log` for stderr capture, `grep` / `pgrep` for liveness checks

**Source spec:** `docs/superpowers/specs/2026-05-01-w9-uinput-text-fallback-design.md`

---

## File Structure

This plan modifies/creates these files. All in-repo on the W9 branch.

**In-repo (committed on `w9-uinput-text-fallback`):**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` — open gate at `:858-877`; add new pure helper `wrap_atspi_failure_as_uinput_fallback` near `:968`; apply the helper at three sites in `focused_friendly_target` (`:971`, `:972`, `:981`); add three new tests in the `#[cfg(test)] mod accessible_insert_runtime_helpers` block (around `:1937`+).
- Modify: `app/src/transcription.rs` — add one new test mirroring `uinput_insert_routes_selected_uinput_backend_to_helper` (`:2692-2737`), no production-code changes.
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W9, populate Spec/Plan columns, append two new "Refactor / enhancement ideas (deferred)" entries (startup-time AT-SPI viability check; GNOME cross-platform regression VM), status-log entries.

**Out-of-repo (system, written via sudo):**
- Replace: `/usr/local/bin/pepper-x` via `sudo install`. Helper binaries unchanged from W1.

**Worktree-only / ephemeral:**
- `/tmp/w9-smoke.log` — pepper-x stderr capture during manual smoke (not committed).

---

## Task 1: Create W9 worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback/`
- (git) Create branch: `w9-uinput-text-fallback` from `main`

- [ ] **Step 1: Verify clean main**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
git worktree list
```

Expected: clean working tree, on `main`, only the main checkout in `worktree list` (the W6 branch may exist as a non-worktree branch — that's fine).

- [ ] **Step 2: Create the worktree**

```sh
git worktree add -b w9-uinput-text-fallback ../pepper-x.w9-uinput-text-fallback main
```

Expected: `Preparing worktree (new branch 'w9-uinput-text-fallback')` then `HEAD is now at <sha>`.

- [ ] **Step 3: Verify**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
git worktree list
git branch --show-current
```

Expected: two worktrees listed; current branch `w9-uinput-text-fallback`.

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback/`.**

---

## Task 2: Update roadmap — W9 to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W9 row in workstream table, status log.

- [ ] **Step 1: Update top status block**

Replace:

```
current_workstream:   W9 — Fix uinput-text fallback path (architectural)
phase:                1  (promoted from Phase 2 on 2026-05-01 — W6 blocked on missing Qt AT-SPI bridge)
state:                pending  (W9 awaits brainstorming)
branch:               (none yet — created when W9 spec is written)
worktree:             (none yet)
last_updated:         2026-05-01
```

With:

```
current_workstream:   W9 — Fix uinput-text fallback path (architectural)
phase:                1
state:                in-progress
branch:               w9-uinput-text-fallback
worktree:             ../pepper-x.w9-uinput-text-fallback
last_updated:         2026-05-01
```

- [ ] **Step 2: Update W9 row in the workstream table**

Find W9's row. Change the `State` column from `pending` to `in-progress`; populate `Branch` (`w9-uinput-text-fallback`), `Spec` (`2026-05-01-w9-uinput-text-fallback-design.md`), `Plan` (`2026-05-01-w9-uinput-text-fallback.md`).

- [ ] **Step 3: Append status-log entry**

At the bottom under "## Status log":
```
- `2026-05-01` — W9 plan written and execution begun. State: `pending` → `in-progress`. Branch: `w9-uinput-text-fallback`. Five must-fix corrections applied to W9 spec from review pass (FriendlyInsertSelection field count fixed, change 2 expanded from 1 to 3 wrap sites, smoke gate discriminator priority reordered, phantom stale-test deliverable removed, all 4 backends acceptance test added).
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W9 in-progress"
```

Expected: one file changed.

---

## Task 3: Pre-flight — workspace-wide audit of `UINPUT_TEXT_BACKEND_NAME` usage

The architect-review flagged: changing the gate's behavior could silently flip the truth value of any other test that asserts uinput-text is rejected. Before editing, enumerate every test-code reference to `UINPUT_TEXT_BACKEND_NAME` so we know what to expect.

- [ ] **Step 1: Enumerate references**

```sh
grep -rn UINPUT_TEXT_BACKEND_NAME crates/ app/ 2>&1 | grep -v target/
```

Expected: a list of references — uses in production code (the constant definition, the dispatch at `:703`, the existing `_BACKEND_NAME` selector arm, etc.) plus any in test code. Read every test-code line in the result.

- [ ] **Step 2: Confirm no test asserts uinput-text is rejected**

For each `#[cfg(test)]` reference found in Step 1, confirm none of them is asserting the gate's "is not implemented yet" rejection. The expected current state (per fact-check during W9 brainstorm): existing wrapper tests at `app/src/transcription.rs:2692-2737` and `:2739-2772` mock `platform_insert` with hand-built `SelectedBackendFailure` errors and don't go through the gate. No other test should be asserting rejection.

If you find a test asserting rejection — STOP. Surface the test to the user. Do not silently flip its assertion; the design assumes no such test exists. (If one exists, the spec was wrong about Chesterton's-fence territory and we need to discuss.)

- [ ] **Step 3: Note the count for the commit message later**

Mental note: how many tests touch `UINPUT_TEXT_BACKEND_NAME`. Useful when reviewing the diff later — if a count changes unexpectedly, an edit went sideways.

---

## Task 4: TDD red phase — write failing tests in atspi.rs

**Files:**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` — three new tests in the `mod accessible_insert_runtime_helpers` block (starts at `:1937`).

The new tests will fail when run against unmodified production code — that's the red phase.

- [ ] **Step 1: Locate insertion point**

```sh
grep -n "fn clipboard_insert_accepts_selected_backend\|fn clipboard_insert_restores_previous_clipboard_text_after_paste" crates/pepperx-platform-gnome/src/atspi.rs
```

Expected: `clipboard_insert_accepts_selected_backend` is a test that already passes (verifies clipboard-paste backend is accepted by the gate). New tests go immediately AFTER it (and before `clipboard_insert_restores_previous_clipboard_text_after_paste`).

- [ ] **Step 2: Add the three new tests**

After the closing brace of `clipboard_insert_accepts_selected_backend` (around `:2069`), insert these three tests:

```rust
    #[test]
    fn ensure_runtime_supported_backend_accepts_all_four_backend_names() {
        // Regression: when adding the fourth backend (uinput-text) to the
        // gate's accepted set, the existing three must still be accepted.
        for backend_name in [
            FRIENDLY_INSERT_BACKEND_NAME,
            STRING_INJECTION_BACKEND_NAME,
            CLIPBOARD_PASTE_BACKEND_NAME,
            UINPUT_TEXT_BACKEND_NAME,
        ] {
            ensure_runtime_supported_backend(
                &FriendlyInsertSelection {
                    backend_name,
                    target_application_id: "test-app".into(),
                    target_class: "text-editor",
                    attempted_backends: vec![backend_name],
                },
                "Test App",
            )
            .unwrap_or_else(|err| panic!("backend {backend_name} should be supported: {err:?}"));
        }
    }

    #[test]
    fn ensure_runtime_supported_backend_rejects_unknown_backend() {
        let result = ensure_runtime_supported_backend(
            &FriendlyInsertSelection {
                backend_name: "bogus-not-a-real-backend",
                target_application_id: "test-app".into(),
                target_class: "text-editor",
                attempted_backends: vec!["bogus-not-a-real-backend"],
            },
            "Test App",
        );
        match result {
            Err(FriendlyInsertRunError::SelectedBackendFailure { reason, .. }) => {
                let reason_text = format!("{reason}");
                assert!(
                    reason_text.contains("not implemented yet"),
                    "expected 'not implemented yet' in reason, got: {reason_text}"
                );
            }
            other => panic!("expected SelectedBackendFailure, got {other:?}"),
        }
    }

    #[test]
    fn wrap_atspi_failure_as_uinput_fallback_produces_uinput_selected_backend_failure() {
        let original = FriendlyInsertRunError::Access("simulated AT-SPI failure".into());
        let wrapped = wrap_atspi_failure_as_uinput_fallback(original);
        match wrapped {
            FriendlyInsertRunError::SelectedBackendFailure {
                selection,
                target_application_name,
                reason,
            } => {
                assert_eq!(selection.backend_name, UINPUT_TEXT_BACKEND_NAME);
                assert!(
                    !selection.target_class.is_empty(),
                    "synthetic target_class must be non-empty (consumed by transcription.rs:493)"
                );
                assert_eq!(selection.target_application_id, "");
                assert_eq!(selection.attempted_backends, Vec::<&'static str>::new());
                assert_eq!(target_application_name, "");
                let reason_text = format!("{reason}");
                assert!(
                    reason_text.contains("simulated AT-SPI failure"),
                    "original error should be preserved as reason: got {reason_text}"
                );
            }
            other => panic!("expected SelectedBackendFailure, got {other:?}"),
        }
    }
```

The third test references `wrap_atspi_failure_as_uinput_fallback` which doesn't exist yet — that's the function we'll add in Task 6.

- [ ] **Step 3: Run the new tests; expect failures**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
source $HOME/.cargo/env
cargo test -p pepperx-platform-gnome -- --include-ignored ensure_runtime_supported_backend_accepts_all_four_backend_names ensure_runtime_supported_backend_rejects_unknown_backend wrap_atspi_failure_as_uinput_fallback 2>&1 | tail -30
```

Expected:
- `ensure_runtime_supported_backend_accepts_all_four_backend_names`: **FAIL** with panic about backend "uinput-text" rejected (the existing three pass; the fourth fails because the gate currently rejects uinput-text).
- `ensure_runtime_supported_backend_rejects_unknown_backend`: **PASS** (current gate already rejects unknown backends; this test is regression protection).
- `wrap_atspi_failure_as_uinput_fallback_*`: **does not compile** — function not defined. Cargo fails earlier.

If `cargo test` won't compile because `wrap_atspi_failure_as_uinput_fallback` is undefined, that's expected — that's the red signal. Don't proceed past this step until you've confirmed at least one TEST-level failure (the gate accepts-all-four test). The compile-failure is the strongest possible red signal for the wrap test.

---

## Task 5: TDD red phase — write failing test in transcription.rs

**Files:**
- Modify: `app/src/transcription.rs` — one new test in its `#[cfg(test)]` block, mirroring `uinput_insert_routes_selected_uinput_backend_to_helper` at `:2692-2737`.

- [ ] **Step 1: Locate insertion point**

```sh
grep -n "fn uinput_insert_routes_selected_uinput_backend_to_helper\|fn uinput_insert_skips_helper_when_platform_backend_is_not_uinput" app/src/transcription.rs
```

Expected: two existing wrapper tests. New test goes immediately after `uinput_insert_skips_helper_when_platform_backend_is_not_uinput` (around `:2772`).

- [ ] **Step 2: Add the new test**

After the closing brace of `uinput_insert_skips_helper_when_platform_backend_is_not_uinput`, insert:

```rust
    #[test]
    fn uinput_insert_routes_synthetic_atspi_failure_to_helper() {
        // Mirrors the synthetic SelectedBackendFailure shape produced by
        // wrap_atspi_failure_as_uinput_fallback in atspi.rs (W9 change 2):
        // empty target_application_id, "unsupported" target_class, empty
        // attempted_backends, empty target_application_name. The wrapper
        // should still route to the helper based on backend_name alone.
        let mut helper_requests = Vec::new();

        let outcome = insert_with_uinput_fallback(
            "synthetic fallback test",
            |_| {
                Err(FriendlyInsertRunError::SelectedBackendFailure {
                    selection: pepperx_platform_gnome::atspi::FriendlyInsertSelection {
                        backend_name: UINPUT_TEXT_BACKEND_NAME,
                        target_application_id: String::new(),
                        target_class: "unsupported",
                        attempted_backends: Vec::new(),
                    },
                    target_application_name: String::new(),
                    reason: Box::new(FriendlyInsertRunError::Access(
                        "AT-SPI infrastructure failed".into(),
                    )),
                })
            },
            |request| {
                helper_requests.push(request.text.clone());
                Ok(())
            },
        )
        .expect("uinput helper should route synthetic AT-SPI failure");

        assert_eq!(helper_requests, vec!["synthetic fallback test".to_string()]);
        assert_eq!(outcome.selection.backend_name, UINPUT_TEXT_BACKEND_NAME);
        assert_eq!(outcome.target_application_name, "");
        assert!(
            !outcome.target_class.is_empty(),
            "outcome.target_class must be non-empty even for synthetic case (read by downstream consumers)"
        );
        assert_eq!(outcome.target_class, "unsupported");
    }
```

- [ ] **Step 3: Run the new test; expect PASS already**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
cargo test -p pepper-x-app uinput_insert_routes_synthetic_atspi_failure_to_helper 2>&1 | tail -10
```

Expected: **PASS**. This test exercises the EXISTING wrapper in `transcription.rs:474-494`, which already handles `SelectedBackendFailure { backend_name = UINPUT_TEXT_BACKEND_NAME }` correctly. It passes today because the wrapper is correct — what's broken is that production code never reaches the wrapper with this error shape (because the gate kills it earlier and the AT-SPI errors don't carry a uinput selection). The test locks the wrapper's behavior so that change 2 in Task 6 can rely on it.

If this test FAILS, something is wrong with the wrapper itself — STOP and investigate. The wrapper should already be correct.

---

## Task 6: TDD green phase — open the gate and add the wrap helper

**Files:**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` — add `UINPUT_TEXT_BACKEND_NAME` to gate's `matches!`; add new function `wrap_atspi_failure_as_uinput_fallback`; apply it at three sites in `focused_friendly_target`.

- [ ] **Step 1: Open the gate**

Find `ensure_runtime_supported_backend` at `:858-877`. Replace its `matches!` block:

OLD (around `:863-866`):
```rust
    if matches!(
        selection.backend_name,
        FRIENDLY_INSERT_BACKEND_NAME | STRING_INJECTION_BACKEND_NAME | CLIPBOARD_PASTE_BACKEND_NAME
    ) {
```

NEW:
```rust
    if matches!(
        selection.backend_name,
        FRIENDLY_INSERT_BACKEND_NAME
        | STRING_INJECTION_BACKEND_NAME
        | CLIPBOARD_PASTE_BACKEND_NAME
        | UINPUT_TEXT_BACKEND_NAME
    ) {
```

(Multi-line form for readability now that the list has four entries.)

Save. Don't re-run tests yet — finish the file's edits as a unit.

- [ ] **Step 2: Add the wrap helper function**

Find `focused_friendly_target` at `:968`. Insert this NEW function immediately BEFORE it (so the helper is in scope when used):

```rust
fn wrap_atspi_failure_as_uinput_fallback(
    error: FriendlyInsertRunError,
) -> FriendlyInsertRunError {
    // W9: when AT-SPI infrastructure fails (registry unreachable, snapshot
    // can't be built, no usable backend selected), translate to a structured
    // SelectedBackendFailure with backend_name = UINPUT_TEXT_BACKEND_NAME so
    // the wrapper at app/src/transcription.rs:474-494 routes to the uinput
    // helper. Without this, the wrapper bails because error.selected_backend()
    // returns None for bare AT-SPI errors, and insertion silently fails on
    // any system where AT-SPI doesn't see the focused app (e.g. KDE without
    // the Qt AT-SPI bridge plugin).
    //
    // target_class must be non-empty because transcription.rs:493 reads it
    // via .to_string() onto the outcome's target_class field.
    let synthetic_selection = FriendlyInsertSelection {
        backend_name: UINPUT_TEXT_BACKEND_NAME,
        target_application_id: String::new(),
        target_class: friendly_insert_target_class_name(
            FriendlyInsertTargetClass::Unsupported,
        ),
        attempted_backends: Vec::new(),
    };
    FriendlyInsertRunError::SelectedBackendFailure {
        selection: synthetic_selection,
        target_application_name: String::new(),
        reason: Box::new(error),
    }
}
```

- [ ] **Step 3: Apply the wrap at the three failure sites in `focused_friendly_target`**

The current function (`:968-1054`) starts:

```rust
fn focused_friendly_target(
    policy: &FriendlyInsertPolicy,
) -> Result<FocusedFriendlyTarget, FriendlyInsertRunError> {
    let focused = unsafe { find_focused_accessible()? };
    let snapshot = inspect_focused_target_from_accessible(&focused)?;
    let target = FriendlyFocusedTarget {
        application_id: snapshot.application_id.clone(),
        is_editable: snapshot.is_editable,
        supports_text: snapshot.supports_text,
        supports_editable_text: snapshot.supports_editable_text,
        supports_caret: snapshot.supports_caret,
    };
    let target_class = snapshot.target_class;
    let selection = select_friendly_insert_backend(&target, policy).map_err(|error| {
        FriendlyInsertRunError::UnsupportedTarget(
            error.with_target_application_name(snapshot.application_name.clone()),
        )
    })?;
```

Replace with:

```rust
fn focused_friendly_target(
    policy: &FriendlyInsertPolicy,
) -> Result<FocusedFriendlyTarget, FriendlyInsertRunError> {
    // W9: AT-SPI infrastructure failures are wrapped as a uinput-text
    // SelectedBackendFailure so the wrapper in transcription.rs routes to
    // the uinput helper. See wrap_atspi_failure_as_uinput_fallback above.
    let focused = unsafe { find_focused_accessible() }
        .map_err(wrap_atspi_failure_as_uinput_fallback)?;
    let snapshot = inspect_focused_target_from_accessible(&focused)
        .map_err(wrap_atspi_failure_as_uinput_fallback)?;
    let target = FriendlyFocusedTarget {
        application_id: snapshot.application_id.clone(),
        is_editable: snapshot.is_editable,
        supports_text: snapshot.supports_text,
        supports_editable_text: snapshot.supports_editable_text,
        supports_caret: snapshot.supports_caret,
    };
    let target_class = snapshot.target_class;
    let selection = select_friendly_insert_backend(&target, policy).map_err(|error| {
        // Wrap the UnsupportedTarget into a uinput-fallback signal so the
        // wrapper routes to the helper (rather than bubbling UnsupportedTarget
        // and short-circuiting on error.selected_backend() == None).
        wrap_atspi_failure_as_uinput_fallback(FriendlyInsertRunError::UnsupportedTarget(
            error.with_target_application_name(snapshot.application_name.clone()),
        ))
    })?;
```

The `target_class` local variable from `snapshot.target_class` is preserved unchanged — it's still used later in the function for the success path.

The rest of `focused_friendly_target` (lines after `let selection = ...`) stays as-is.

- [ ] **Step 4: Verify the file compiles**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
source $HOME/.cargo/env
cargo build -p pepperx-platform-gnome 2>&1 | tail -10
```

Expected: build succeeds with no errors. Warnings are fine for now (clippy comes in Task 7).

If you see "cannot find function `wrap_atspi_failure_as_uinput_fallback`" — the helper wasn't placed before its caller. Move it earlier in the file.

If you see type-mismatch errors mentioning `FriendlyInsertSelection` field counts — Step 2's helper has the wrong number of fields. Re-read the struct definition at `atspi.rs:167-173` and ensure all four are populated.

---

## Task 7: Run quality gates (TDD green-phase verification + clippy)

- [ ] **Step 1: Run only the W9 tests; expect green**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
cargo test -p pepperx-platform-gnome -- ensure_runtime_supported_backend_accepts_all_four_backend_names ensure_runtime_supported_backend_rejects_unknown_backend wrap_atspi_failure_as_uinput_fallback 2>&1 | tail -20
cargo test -p pepper-x-app -- uinput_insert_routes_synthetic_atspi_failure_to_helper 2>&1 | tail -10
```

Expected: all 4 tests PASS. (The transcription.rs test was passing before; the three atspi.rs tests now pass after Task 6's edits.)

- [ ] **Step 2: Run the full workspace test suite**

```sh
cargo test --workspace 2>&1 | tee /tmp/w9-test.log | tail -20
```

Expected: every crate ends with `test result: ok.`. Total passed count >= W1's 116 + the 4 new tests = 120 (some crates may have additional tests). Zero failures.

If anything failing now was passing on `main` before W9 — STOP. The diff broke a regression somewhere. Likely culprits: a `match` statement elsewhere assumed uinput-text is rejected (Task 3's pre-flight grep should have caught this; if the grep missed it, surface to user), or one of the three wrap-site changes broke an unrelated invariant.

- [ ] **Step 3: Run clippy on just the modified crates**

```sh
cargo clippy -p pepperx-platform-gnome -p pepper-x-app -- -D warnings 2>&1 | tail -20
```

Expected: clean — no warnings. Specifically `-p pepperx-platform-gnome` and `-p pepper-x-app` only, NOT `--workspace`. Workspace-wide clippy hits the upstream-code drift documented in W1's findings (W4c's job to fix); this gate stays scoped.

If clippy fires on the new code (e.g. unused imports if you inadvertently added one for the helper, redundant references in the `.map_err` calls), fix them inline before proceeding.

- [ ] **Step 4: cargo fmt --check is intentionally skipped**

Per W1 findings + spec done-criterion #5: upstream main fails fmt-check against rustc 1.95.0 (W4c will fix). Don't run workspace fmt-check; the W9 diff's own formatting is the implementer's responsibility (use editor-side rustfmt or `rustfmt crates/pepperx-platform-gnome/src/atspi.rs app/src/transcription.rs` standalone if you want spot-checks).

---

## Task 8: Commit code + tests

**Files committed:**
- `crates/pepperx-platform-gnome/src/atspi.rs`
- `app/src/transcription.rs`

- [ ] **Step 1: Inspect the staged diff**

```sh
git add crates/pepperx-platform-gnome/src/atspi.rs app/src/transcription.rs
git status
git diff --cached --stat
git diff --cached crates/pepperx-platform-gnome/src/atspi.rs | head -150
```

Expected: only the two files staged. `atspi.rs` diff shows: 4-arm `matches!` (was 3-arm), new function `wrap_atspi_failure_as_uinput_fallback`, `.map_err(...)` applied at three sites in `focused_friendly_target`, three new tests in the test module. `transcription.rs` shows: one new test.

- [ ] **Step 2: Commit**

```sh
git commit -m "$(cat <<'EOF'
W9: open uinput-text gate and wrap AT-SPI failures as fallback signals

Two complementary changes to make pepper-x's designed-but-unreachable
uinput-text fallback path actually fire:

1. ensure_runtime_supported_backend at atspi.rs:858-877 now accepts
   UINPUT_TEXT_BACKEND_NAME alongside the existing three constants.
   The existing branch in insert_text_into_friendly_target:703-713
   already returns SelectedBackendFailure to signal the wrapper to
   route to the helper — opening the gate makes that signal
   reachable. Chesterton's fence verified: gate added 2026-03-28
   19:53; uinput helper added 56 minutes later in ee651cf; gate was
   simply never updated.

2. New pure helper wrap_atspi_failure_as_uinput_fallback translates
   bare AT-SPI errors into SelectedBackendFailure {
   selection.backend_name = UINPUT_TEXT_BACKEND_NAME, .. } so the
   existing wrapper at transcription.rs:474-494 (which checks
   error.selected_backend()) routes them to the helper. Applied via
   .map_err at three failure sites in focused_friendly_target:
   - :971 find_focused_accessible (TuxedoOS case: AT-SPI registry
     has no Qt apps)
   - :972 inspect_focused_target_from_accessible (snapshot extraction
     fails when find_focused_accessible succeeds but downstream lookup
     can't proceed)
   - :981 select_friendly_insert_backend (wraps existing
     UnsupportedTarget into the uinput-fallback signal)

Synthetic selection populates target_class with
friendly_insert_target_class_name(FriendlyInsertTargetClass::Unsupported)
= "unsupported" so transcription.rs:493's .to_string() produces a
non-empty string for the resulting FriendlyInsertOutcome.target_class.

Tests added (TDD red-then-green):
- atspi.rs: ensure_runtime_supported_backend_accepts_all_four_backend_names
  (regression protects existing three when adding fourth)
- atspi.rs: ensure_runtime_supported_backend_rejects_unknown_backend
  (gate doesn't become permissive)
- atspi.rs: wrap_atspi_failure_as_uinput_fallback_produces_uinput_
  selected_backend_failure (helper produces correct shape, target_class
  non-empty)
- transcription.rs: uinput_insert_routes_synthetic_atspi_failure_to_helper
  (wrapper routes synthetic-error shape correctly; outcome.target_class
  non-empty downstream)

Wrapper at transcription.rs:474-494 has zero production-code changes;
the new test locks its existing correct behavior so future refactors
can't drift it without breaking the test.

Workspace cargo test: 120/120 pass (was 116). cargo clippy -p
pepperx-platform-gnome -p pepper-x-app -- -D warnings clean.
cargo fmt --check skipped per W1 findings (W4c territory).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: two files changed.

---

## Task 9: Build release binary

- [ ] **Step 1: cargo build --release**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
source $HOME/.cargo/env
cargo build --release 2>&1 | tee /tmp/w9-build.log | tail -10
```

Expected: ends with `Finished \`release\` profile [optimized] target(s) in <time>`, exits 0. Incremental build from W1's caches; should take seconds-to-minutes depending on what got invalidated by editing the platform crate.

- [ ] **Step 2: Verify all three binaries built**

```sh
ls -lh target/release/pepper-x target/release/pepperx-uinput-helper target/release/pepperx-cleanup-helper
```

Expected: three executables, freshly built (newer mtime than `/usr/local/bin/pepper-x`).

Helpers technically rebuilt because they share dependencies, but their source code didn't change — we only need to install the new `pepper-x` binary. Helpers stay at the W1-installed copies.

---

## Task 10: 🛑 User action required: install new pepper-x binary

This needs sudo. **The user runs this step.**

- [ ] **Step 1: Show the user the install command**

Print:
```
Please run, in any terminal:

  sudo install -m 755 /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback/target/release/pepper-x /usr/local/bin/

(Helpers from W1 are unchanged — do NOT reinstall pepperx-uinput-helper or pepperx-cleanup-helper.)

Reply when done.
```

- [ ] **Step 2: After user reports done, verify the install**

```sh
ls -l /usr/local/bin/pepper-x
file /usr/local/bin/pepper-x
```

Expected: file mtime matches the just-built binary (within seconds), executable, owned by root. ELF 64-bit.

If the binary was installed elsewhere or didn't update, surface to user.

---

## Task 11: 🛑 User action required: smoke test in Kate

Primary validator is text appearing in Kate. `pgrep` and stderr log are secondary; helper persistence makes `pgrep` liveness-only.

- [ ] **Step 1: Make sure no pepper-x or pepperx-uinput-helper is running yet**

User runs:
```sh
pkill -f /usr/local/bin/pepper-x 2>/dev/null
pkill -f /usr/libexec/pepper-x/pepperx-uinput-helper 2>/dev/null
sleep 1
pgrep -af pepper-x; pgrep -af pepperx-uinput-helper
```

Expected: both `pgrep` empty (or only show shell command false matches like `pgrep -af pepper-x` itself).

- [ ] **Step 2: Launch pepper-x with stderr capture**

User runs in a fresh terminal (not Claude Code's process tree — `input` group is durable post-reboot, no `sg` workaround needed):
```sh
pepper-x 2>&1 | tee /tmp/w9-smoke.log
```

GTK window opens; capture file at `/tmp/w9-smoke.log` accumulates pepper-x's stderr.

- [ ] **Step 3: Open Kate**

User runs:
```sh
kate &
```
Open or create a new document. Type a few words to confirm normal input works. Position cursor mid-line if possible.

- [ ] **Step 4: Dictate**

User holds Alt+Super, says *"hello world from kate"*, releases.

- [ ] **Step 5: Primary assertion — text appears in Kate**

User confirms: cleaned text appears at the cursor in Kate within ~2 seconds of releasing the hotkey.

If text DOES appear → ✅ primary assertion passes; proceed to Step 6.
If text DOES NOT appear → ❌ STOP. Invoke `superpowers:systematic-debugging`. See "If smoke fails" at the end of this task.

- [ ] **Step 6: Secondary assertion — log inspection**

In another terminal:
```sh
grep -E '\[Pepper X uinput\] XKB layout|\[Pepper X\] perf:' /tmp/w9-smoke.log | tail -10
```

Expected: at least one line matching `[Pepper X uinput] XKB layout` (the helper's startup log — proves helper spawned at least once) AND at least one line matching `[Pepper X] perf:` (the dictation cycle perf log).

If the perf line is there but the XKB-layout line is missing — odd; helper may have already been running. Check `pgrep` next.

- [ ] **Step 7: Liveness assertion — helper process**

```sh
pgrep -af pepperx-uinput-helper
```

Expected: at least one PID with the binary path. Confirms helper is alive after dictation; combined with the primary assertion, confirms the spawn-and-insert path worked.

- [ ] **Step 8: Decision**

All three assertions pass → ✅ Kate side of W9 is verified.
Primary fails → ❌ STOP per "If smoke fails" below.
Primary passes but secondary/liveness oddly empty → flag to user; log oddities are diagnostic but not blocking if the text actually appeared.

**If smoke fails:**

Triage by symptom:
- **Helper alive (pgrep shows PID), no text in Kate**: helper accepted connection but typing failed silently. Check `/tmp/w9-smoke.log` for any errors after the perf line; check `cat /proc/bus/input/devices | grep -A4 'Pepper X virtual keyboard'` to confirm the virtual keyboard is registered (and that udev tagged it correctly per W1's rules).
- **No helper alive after dictation**: spawn problem. Check `/tmp/w9-smoke.log` for "failed to launch Pepper X uinput helper" or socket connect/bind errors.
- **Wrong text or garbled characters in Kate**: XKB layout mismatch. Set `PEPPERX_XKB_LAYOUT` env in the launch shell.
- **Wrapper saw an error type the test didn't anticipate**: trace the actual error variant via stderr. Either refine `wrap_atspi_failure_as_uinput_fallback` to handle an additional error type, or extend the wrapper's matching in `transcription.rs:474-494` (rare; the wrapper's logic is meant to be agnostic).

Surface findings to user. Do not patch around — invoke `superpowers:systematic-debugging`.

---

## Task 12: 🛑 User action required: smoke test in Konsole

Same shape as Task 11 but in a different app, to confirm uinput injection works in a terminal context.

- [ ] **Step 1: Open Konsole (pepper-x and helper still running from Task 11)**

```sh
konsole &
```

Focus a shell prompt; type a few characters to confirm normal input works; clear the line.

- [ ] **Step 2: Dictate**

User holds Alt+Super, says *"echo hello from konsole"*, releases.

- [ ] **Step 3: Primary assertion**

User confirms: text appears at the prompt within ~2 seconds. (Punctuation may differ slightly per cleanup-model behavior — `echo hello from konsole` or `echo hello from Konsole.` both fine.)

- [ ] **Step 4: Secondary + liveness assertions**

```sh
grep -E '\[Pepper X\] perf:' /tmp/w9-smoke.log | tail -5
pgrep -af pepperx-uinput-helper
```

Expected: another perf line for the Konsole cycle; helper still alive.

- [ ] **Step 5: Decision**

Same as Task 11 Step 8 — both assertions pass → ✅ Konsole side verified.

If Kate worked but Konsole doesn't (or vice versa), that's a meaningful diagnostic — surface to user. Likely culprits: Konsole's terminal-mode keyboard handling vs Kate's text-editor handling differs. The helper's keystroke synthesis is identical in both cases; if results differ, it's the receiving app's input handling, not pepper-x.

---

## Task 13: Update roadmap — W9 done

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W9 row state, "Refactor / enhancement ideas (deferred)" section, status log.

- [ ] **Step 1: Update top status block**

Replace W9 in-progress block with:
```
current_workstream:   W2 — KDE Global Shortcut → D-Bus
phase:                1
state:                pending  (W9 done; W2 awaits brainstorming)
branch:               (none yet — created when W2 spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

Use today's date if execution spans multiple days.

- [ ] **Step 2: Mark W9 row as done**

Change W9's State column from `in-progress` to `done`.

- [ ] **Step 3: Append two new entries to "Refactor / enhancement ideas (deferred)"**

Find the deferred-ideas section. Append after the existing three entries:

```markdown
- **Startup-time AT-SPI viability check (Approach C from W9 brainstorm)** — at startup, probe the AT-SPI registry; if zero apps registered, log once (e.g. `[Pepper X] AT-SPI registry empty; using uinput-text fallback for all dictations`) and skip per-dictation AT-SPI lookups. Trigger: per-dictation AT-SPI overhead becomes measurable on TuxedoOS, OR desire for clearer one-shot "what-mode-am-I-in" startup logging.
- **GNOME cross-platform regression VM** — set up a GNOME VM (or remote test box) so changes can be smoke-tested on both KDE and GNOME. Trigger: a W9-class change feels under-tested without GNOME verification, OR a user reports a GNOME regression after changes that were only KDE-smoked.
```

- [ ] **Step 4: Append status-log entry**

At the bottom under "## Status log":
```
- `<today's date>` — W9 done. Gate opened (UINPUT_TEXT_BACKEND_NAME accepted); wrap_atspi_failure_as_uinput_fallback applied at 3 sites in focused_friendly_target. 120/120 cargo test pass; manual smoke verified in Kate AND Konsole on TuxedoOS — text appears at cursor; helper IS spawned (pgrep returns PID); pepper-x stderr shows AT-SPI fallback path triggering correctly. Two new entries added to "Refactor / enhancement ideas (deferred)" (startup-time AT-SPI viability check; GNOME cross-platform regression VM). State: `in-progress` → `done`. Next: W2 brainstorm.
```

If smoke had unexpected results (e.g. needed XKB env tweak, focus-tracking hiccup, one specific app behaved oddly), reflect honestly in the log — future re-orientation depends on it.

- [ ] **Step 5: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W9 done; advance current_workstream to W2

Append two new deferred-ideas entries:
- Startup-time AT-SPI viability check (Approach C from W9 brainstorm)
- GNOME cross-platform regression VM"
```

---

## Task 14: Merge `w9-uinput-text-fallback` to main

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: clean working tree, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w9-uinput-text-fallback -m "Merge W9: uinput-text fallback architectural fix

Two changes to atspi.rs that finish what ee651cf (the uinput helper
landing 56 minutes after the gate) started: open the gate to accept
UINPUT_TEXT_BACKEND_NAME, and wrap AT-SPI infrastructure / target-
selection failures as uinput-fallback signals so the existing wrapper
in transcription.rs routes to the helper.

Verified end-to-end on TuxedoOS 24.04 + KDE Plasma + Wayland:
dictation produces text in Kate AND Konsole even though Qt AT-SPI
bridge is missing on this system. 120/120 cargo test pass; clippy
clean for the modified crates; smoke discriminators (text appears,
stderr log shows backend selection, pgrep shows helper alive) all
satisfied.

Daily-driver bar met without depending on the missing Qt AT-SPI
bridge. W6 stays blocked on package availability; reactivate from
its preserved branch if/when the bridge becomes available.

current_workstream advances to W2 (KDE Global Shortcut)."
```

Expected: a merge commit summary listing the changed files.

- [ ] **Step 3: Do NOT push**

Per user preference (across W1 + W6): work stays local on `lukepatrick/pepper-x:main` until you decide to push. Skip `git push origin main`. Note in the conversation that there are now N commits ahead of `origin/main` (`git log --oneline origin/main..HEAD | wc -l`).

---

## Task 15: Cleanup — remove the worktree

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w9-uinput-text-fallback
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Switch back to main checkout, remove worktree, delete branch**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w9-uinput-text-fallback
git worktree list
git branch -d w9-uinput-text-fallback
git branch
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout (the `w6-atspi-kde-whitelist` branch may still exist as a non-worktree resumption point — that's fine).
- `branch -d`: `Deleted branch w9-uinput-text-fallback (was <sha>).`
- `branch`: shows `* main` (with the W6 branch alongside if preserved from W6's blocked state).

If `git branch -d` refuses with "not fully merged", don't force with `-D`. Investigate.

---

## Done

When all 15 tasks are checked, W9 is complete:

- Gate opened to accept `UINPUT_TEXT_BACKEND_NAME` alongside the three existing constants.
- `wrap_atspi_failure_as_uinput_fallback` added; applied at three failure sites in `focused_friendly_target`.
- 4 new tests (3 in `atspi.rs`, 1 in `transcription.rs`); 120/120 workspace tests pass.
- `pepper-x` binary updated in `/usr/local/bin/`; helpers unchanged.
- Manual smoke gate passed in Kate AND Konsole on TuxedoOS using primary "text appears" + secondary stderr-log + liveness `pgrep` discriminators (the AT-SPI bridge is still missing — W9 makes pepper-x work without it).
- Roadmap reflects W9=done, W2=pending+next, two new deferred-ideas entries.

**Daily-driver bar met on TuxedoOS.** Pepper-x now produces text in Kate / Konsole / any focused app via universal-uinput typing whenever AT-SPI can't reach the focused widget. W6 remains blocked but is no longer the daily-driver bar's gating dependency — caret-aware AT-SPI insertion is a quality-of-life improvement, not a viability requirement.

**Next session-start re-orientation will pick up W2.** Per ways-of-working entry-point logic: `current_workstream: W2, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

| Risk | Where addressed |
|---|---|
| Existing test asserts uinput-text rejected | Task 3 pre-flight grep enumerates all `UINPUT_TEXT_BACKEND_NAME` test references; STOP if any asserts rejection. |
| `inspect_focused_target_from_accessible` failure path unwrapped | Task 6 Step 3 wraps it with the same helper as `find_focused_accessible`. |
| `select_friendly_insert_backend` produces `UnsupportedTarget` (no selected_backend) | Task 6 Step 3 wraps it with the same helper, additionally re-wrapping the inner `UnsupportedTarget` so the wrapper-side semantics are preserved. |
| Synthetic `target_class` is empty | Task 6 Step 2 sets target_class to `friendly_insert_target_class_name(FriendlyInsertTargetClass::Unsupported)` = `"unsupported"`. Task 4 Step 2's `wrap_atspi_failure_as_uinput_fallback_*` test asserts non-empty. |
| `pgrep` is liveness-only, not proof of insert | Task 11 Steps 5-7 separate primary (text appears) from secondary (log) from liveness (pgrep). Task 11 Step 1 kills any stale helper to ensure fresh-state assertion is meaningful. |
| Helper success path with empty `target_application_name` produces "" in `FriendlyInsertOutcome` | Documented as known limitation; not addressed in W9. |
| First-spawn-on-demand latency on fresh boot | Mitigated by W1's autostart `.desktop`; helper is pre-warmed within the first dictation cycle of any session. |
| GNOME regression untested | Documented in spec; deferred-ideas entry for cross-platform VM (added in Task 13 Step 3). |
