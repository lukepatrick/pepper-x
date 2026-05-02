# W9 — Fix uinput-text Fallback Path (Architectural)

Workstream W9 of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). Promoted from Phase 2 to Phase 1 on 2026-05-01 when W6 blocked on the missing Qt AT-SPI bridge plugin in TuxedoOS — W9 is now the daily-driver bar unblocker.

## Goal

Make pepper-x's text-insertion path actually use its designed-but-unreachable uinput-text fallback. Two changes in `crates/pepperx-platform-gnome/src/atspi.rs`:

1. **Open the gate.** Add `UINPUT_TEXT_BACKEND_NAME` to the accepted set in `ensure_runtime_supported_backend` (`:858-877`). The existing branch in `insert_text_into_friendly_target` (`:703-713`) deliberately returns `SelectedBackendFailure` to signal the wrapper to route to the helper — but the gate currently rejects uinput-text outright with "is not implemented yet" before that branch is ever reached.

2. **Wrap AT-SPI infrastructure / selection failures as a uinput-text fallback signal.** Inside `focused_friendly_target` (`atspi.rs:968-1054`), translate three early-return error sites into `SelectedBackendFailure { backend_name: UINPUT_TEXT_BACKEND_NAME, .. }` so the existing wrapper at `app/src/transcription.rs:474-494` routes to the helper as designed. The three sites:
   - `:971` `find_focused_accessible` failure (TuxedoOS case: AT-SPI registry has no Qt apps).
   - `:972` `inspect_focused_target_from_accessible` failure (currently bare `?` — fires when `find_focused_accessible` succeeds but snapshot extraction fails, e.g. AT-SPI returns a desktop root with no app context).
   - `:981` `select_friendly_insert_backend` failure (currently produces `UnsupportedTarget`, not a `SelectedBackendFailure` — wrapper bails because `error.selected_backend()` returns `None`).
   The wrapping logic is extracted into a small pure helper function so it's unit-testable without touching the unsafe FFI path. The other `find_focused_accessible` caller in the file at `:1058` is dead code (`#[allow(dead_code)] pub(crate) fn inspect_focused_target`) and is NOT in the production insertion path — no wrapping needed there.

After W9: pepper-x's universal-uinput fallback works as the upstream architecture always intended. Daily-driver bar met on TuxedoOS without depending on the Qt AT-SPI bridge.

## Background — why this is Phase 1

W1 surfaced an upstream gap: pepper-x is *designed* to fall back to a uinput helper when AT-SPI fails, but `ensure_runtime_supported_backend` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet". Result: insertion silently fails after a successful transcription on every focused-app/desktop combination where AT-SPI doesn't reach.

W6 was originally Phase 1 — extend the AT-SPI app whitelist so KDE apps take the AT-SPI path that *does* work. **W6 blocked at pre-flight**: TuxedoOS's `libqt6gui6:amd64` 6.9.2 ships without the Qt AT-SPI accessibility bridge plugin (`/usr/lib/x86_64-linux-gnu/qt6/plugins/` has no `accessible/` subdirectory; `apt-cache search atspi` finds no Qt-side bridge). Kate/Konsole never register with AT-SPI, so no whitelist patch can route them to a path that doesn't see them.

W9 bypasses AT-SPI entirely for non-AT-SPI-visible apps. It is the durable fix that makes pepper-x work on any system regardless of AT-SPI bridge availability. With W6 blocked indefinitely on external package availability, W9 is now the unblocker for daily-driver use.

### Chesterton's fence — gate is a clear oversight, not a defensive measure

Git archeology (verified during W9 brainstorm Explore):

- `8f190ba` (2026-03-28 19:53) — gate introduced, accepting only `FRIENDLY_INSERT_BACKEND_NAME` ("atspi-editable-text").
- `4b57791` (2026-03-28 20:08) — gate expanded to accept `STRING_INJECTION_BACKEND_NAME`.
- `c0fbef3` (2026-03-28 20:32) — gate expanded to accept `CLIPBOARD_PASTE_BACKEND_NAME`.
- `ee651cf` (2026-03-28 20:49) — **uinput helper added** (full server + client + wrapper). **Gate never updated** to accept `UINPUT_TEXT_BACKEND_NAME`.

The fix opens what was simply forgotten. The wrapper logic in `app/src/transcription.rs:474-494` is fully implemented and correct; it just never receives the structured error because the gate fires first.

## Done criteria

1. **Gate opened.** `ensure_runtime_supported_backend` accepts `UINPUT_TEXT_BACKEND_NAME` alongside the existing three constants. Unknown backends still rejected.
2. **AT-SPI failure → uinput-text fallback wrapping.** Inside `focused_friendly_target` (`atspi.rs:968-1054`), three error sites are translated to `FriendlyInsertRunError::SelectedBackendFailure { selection.backend_name = UINPUT_TEXT_BACKEND_NAME, reason = original_error, .. }`: `find_focused_accessible` failure (`:971`), `inspect_focused_target_from_accessible` failure (`:972`), and `select_friendly_insert_backend` failure (`:981`, currently `UnsupportedTarget`). The wrapping logic is extracted into a small pure helper function `wrap_atspi_failure_as_uinput_fallback(error: FriendlyInsertRunError) -> FriendlyInsertRunError` so it's unit-testable in isolation.
3. **Unit tests added** in the existing `#[cfg(test)] mod accessible_insert_runtime_helpers` block:
   - **All four backends accepted by the gate** — direct call to `ensure_runtime_supported_backend` for each of the four valid `_BACKEND_NAME` constants. Asserts `Ok(())` for each. Catches accidental break of the existing three when adding the fourth.
   - **Gate rejects unknown backend** — same call with `backend_name: "bogus-backend"`. Asserts `Err(SelectedBackendFailure)`. Regression-protects against gate becoming permissive.
   - **`wrap_atspi_failure_as_uinput_fallback` produces correct structure** — pass an arbitrary `FriendlyInsertRunError`; assert the result is `SelectedBackendFailure` with `selection.backend_name == UINPUT_TEXT_BACKEND_NAME`, the original error preserved as `reason`, and a non-empty placeholder `target_class` (see Approach section).
4. **Wrapper-end-to-end test added** in `app/src/transcription.rs`'s `#[cfg(test)]` block, mirroring the existing `uinput_insert_routes_selected_uinput_backend_to_helper` (`:2692-2737`) pattern: passes a `platform_insert` mock that returns the synthetic-uinput error from change 2; asserts `helper_insert` is invoked with the right text. Catches drift if either side changes without the other.
5. **Build green.** `cargo build --release`, `cargo test --workspace`, and `cargo clippy -p pepperx-platform-gnome -p pepper-x-app -- -D warnings` exit 0. (`cargo fmt --check` and workspace-wide clippy excluded — same upstream-drift situation as W6, deferred to W4c.)
6. **Manual smoke pass in Kate.** Hold Alt+Super, dictate, release. **Cleaned text appears at the cursor.** Discriminator: `pgrep pepperx-uinput-helper` shows a PID during/after dictation (helper IS spawned now). Stderr log shows the helper-spawn line + a successful insert. **Opposite of W6's expectation** — for W9, helper presence proves the fix worked.
7. **Manual smoke pass in Konsole.** Same shape as Kate; text appears at shell prompt.
8. **Roadmap updated.** W9 row state advances `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populated. `current_workstream:` advances to W2 when W9 lands. Status log captures smoke outcomes.
9. **"Refactor / enhancement ideas (deferred)" extended** with two new entries:
   - **Startup-time AT-SPI viability check (Approach C from W9 brainstorm)** — at startup, probe the AT-SPI registry; if zero apps registered, log once and skip per-dictation AT-SPI lookups. Trigger: measurable per-dictation overhead, OR desire for clearer "what-mode-am-I-in" logging.
   - **GNOME cross-platform regression VM** — set up a GNOME environment so changes can be smoke-tested on both desktops. Trigger: a W9-class change feels under-tested without GNOME verification, OR a user reports a GNOME regression.
10. **Branch merged to fork's main.** `w9-uinput-text-fallback` worktree branch merges via `--no-ff`, then is removed. (W6's `w6-atspi-kde-whitelist` branch stays preserved as the AT-SPI-bridge-availability resumption point.)

## Approach

### Change 1 — Open the gate

Single edit in `crates/pepperx-platform-gnome/src/atspi.rs:858-877`:

```rust
fn ensure_runtime_supported_backend(
    selection: &FriendlyInsertSelection,
    target_application_name: &str,
) -> Result<(), FriendlyInsertRunError> {
    if matches!(
        selection.backend_name,
        FRIENDLY_INSERT_BACKEND_NAME
        | STRING_INJECTION_BACKEND_NAME
        | CLIPBOARD_PASTE_BACKEND_NAME
        | UINPUT_TEXT_BACKEND_NAME
    ) {
        return Ok(());
    }
    // ... rest unchanged
}
```

After this, the existing dispatch in `insert_text_into_friendly_target:703-713` (which deliberately returns `SelectedBackendFailure` for `UINPUT_TEXT_BACKEND_NAME` to signal the wrapper) becomes reachable.

### Change 2 — Wrap AT-SPI infrastructure / selection failures

Three failure sites inside `focused_friendly_target` (`atspi.rs:968-1054`) currently bubble bare errors that the wrapper at `transcription.rs:474-494` cannot route (because `error.selected_backend()` returns `None`):

- **`:971`** `find_focused_accessible` — fails on TuxedoOS because Kate/Konsole aren't AT-SPI-registered.
- **`:972`** `inspect_focused_target_from_accessible` — bare `?` propagation; fails when `find_focused_accessible` succeeds (e.g. returns desktop root) but downstream snapshot extraction (application_name / application_id) can't proceed. Per architect-review: this is a real risk because on TuxedoOS the actual fault may be at `:972`, not `:971`.
- **`:981`** `select_friendly_insert_backend` — produces `FriendlyInsertRunError::UnsupportedTarget(_)` with a target_application_name attached, but the variant doesn't carry a selected_backend, so the wrapper short-circuits.

W9 wraps all three as `SelectedBackendFailure { backend_name = UINPUT_TEXT_BACKEND_NAME, .. }` via a single small helper.

Implementation choice: **extract a pure helper function** for the wrapping logic. Three styles considered:

- **Approach A (chosen).** Wrap at the `focused_friendly_target` boundary, applied to all three failure sites. Pure function `wrap_atspi_failure_as_uinput_fallback(error: FriendlyInsertRunError) -> FriendlyInsertRunError`. Function is unit-testable in isolation; production code uses it via `.map_err(...)` or `match` on each of the three sites.
- Approach B. Modify the wrapper at `transcription.rs:474-494` to recognize "no selected_backend" as a uinput-fallback case. Smaller diff but mixes concerns (wrapper now has policy about AT-SPI internals).
- Approach C. Restructure all insertion to a higher-level strategy chooser that does a startup-time AT-SPI viability probe. Larger refactor; the deferred-ideas section captures it as a future enhancement (trigger: per-dictation overhead becomes a real complaint, OR desire for clearer "what-mode-am-I-in" startup logs).

Sketch of approach A:

```rust
// New pure helper (in atspi.rs, near focused_friendly_target):
fn wrap_atspi_failure_as_uinput_fallback(
    error: FriendlyInsertRunError,
) -> FriendlyInsertRunError {
    // FriendlyInsertSelection has 4 fields (atspi.rs:167-173):
    //   backend_name: &'static str
    //   target_application_id: String
    //   target_class: &'static str
    //   attempted_backends: Vec<&'static str>
    //
    // For the synthetic case we don't know app/class, but consumers in
    // transcription.rs:493 read selection.target_class.to_string(), so
    // target_class must be a non-empty &'static str. Reuse the existing
    // friendly_insert_target_class_name(FriendlyInsertTargetClass::Unsupported)
    // result ("unsupported") as the placeholder.
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

// In focused_friendly_target:
let focused = unsafe { find_focused_accessible() }
    .map_err(wrap_atspi_failure_as_uinput_fallback)?;
let snapshot = inspect_focused_target_from_accessible(&focused)
    .map_err(wrap_atspi_failure_as_uinput_fallback)?;
// ... selection step:
let selection = select_friendly_insert_backend(&target, policy)
    .map_err(|error| {
        // existing UnsupportedTarget wrap stays for the error semantics it needs,
        // but we ALSO route to uinput-text fallback by re-wrapping:
        wrap_atspi_failure_as_uinput_fallback(
            FriendlyInsertRunError::UnsupportedTarget(
                error.with_target_application_name(snapshot.application_name.clone()),
            )
        )
    })?;
```

Synthetic-selection field rationale:
- `attempted_backends: Vec::new()` — honest: we tried nothing AT-SPI-side because we couldn't even establish a target.
- `target_application_id: String::new()` — honest: we don't know the app (especially in the `:971` fault). The wrapper doesn't read this for routing.
- `target_class: friendly_insert_target_class_name(Unsupported)` (= `"unsupported"`) — non-empty placeholder so `transcription.rs:493`'s `selection.target_class.to_string()` produces a meaningful (if generic) value rather than `""`.
- `target_application_name: String::new()` — same as above; consumed by error formatters and downstream `FriendlyInsertOutcome.target_application_name`. Acceptable for the universal-fallback case where we genuinely don't know.

### Why no change to `app/src/transcription.rs` production code

The existing wrapper at `:474-494` already handles `SelectedBackendFailure` with `selection.backend_name == UINPUT_TEXT_BACKEND_NAME` correctly — calls `helper_insert`, returns success. Both W9 changes feed into the wrapper as it's already designed. No production-code changes needed there; only a new test.

## Testing strategy

### Unit tests (gating, run by `cargo test`)

Three new tests in `crates/pepperx-platform-gnome/src/atspi.rs`'s `#[cfg(test)]` block:

1. **All four backends accepted by gate** — direct call to `ensure_runtime_supported_backend` for each of `FRIENDLY_INSERT_BACKEND_NAME`, `STRING_INJECTION_BACKEND_NAME`, `CLIPBOARD_PASTE_BACKEND_NAME`, `UINPUT_TEXT_BACKEND_NAME`. Asserts `Ok(())` for each. Catches accidental break of the existing three when adding the fourth.
2. **Gate rejects unknown backend** — same call with `backend_name: "bogus-backend"`. Asserts `Err(SelectedBackendFailure)` with the "is not implemented yet" reason. Regression-protects against gate becoming permissive.
3. **`wrap_atspi_failure_as_uinput_fallback` produces correct structure** — pass an arbitrary `FriendlyInsertRunError` (e.g. `Access("test")`); assert the result is `SelectedBackendFailure` with `selection.backend_name == UINPUT_TEXT_BACKEND_NAME`, `selection.target_class != ""`, and the original error preserved as `reason`.

(No "stale test rewrite" — the W1 brainstorm Explore agent originally cited a test named `ensure_string_injection_backend_is_rejected` at `:2055-2077`. The fact-check agent verified this test does **not** exist; the test at that line range is `clipboard_insert_accepts_selected_backend` and is correct as-is.)

One new test in `app/src/transcription.rs`'s `#[cfg(test)]` block:

4. **Synthetic-uinput-error routes to helper** — mirrors `uinput_insert_routes_selected_uinput_backend_to_helper` at `:2692-2737`. Closure for `platform_insert` returns the *exact* `SelectedBackendFailure` shape produced by `wrap_atspi_failure_as_uinput_fallback` (synthetic selection with `target_class = "unsupported"` and empty `target_application_id` / `target_application_name`). Closure for `helper_insert` records its argument. Asserts the wrapper invoked `helper_insert` with the original text. Critically: also asserts that the resulting `FriendlyInsertOutcome.target_class` is non-empty (catches the empty-target_class bug the architect-review flagged).

### Manual smoke gate (run by user, post-build, pre-merge)

The gate's primary validator is **text actually appearing in Kate / Konsole**. The helper is persistent (`crates/pepperx-uinput-helper/src/main.rs:66-72` is `loop { listener.accept() }`), so `pgrep` returning a PID just means "binary started" — it does not prove keystroke synthesis happened. Discriminator priority for W9:

- **Primary**: cleaned text appears at the cursor in the focused app.
- **Secondary**: `/tmp/w9-smoke.log` contains the helper-spawn log line (literal substring `[Pepper X uinput] XKB layout` from `crates/pepperx-uinput-helper/src/main.rs`) AND the per-cycle dictation perf line (`[Pepper X] perf:`) without any preceding error from the wrapper or platform path.
- **Liveness only**: `pgrep pepperx-uinput-helper` returns a PID — confirms the helper is alive but does not by itself prove insert worked.

1. `cargo build --release` from the W9 worktree.
2. `sudo install -m 755 target/release/pepper-x /usr/local/bin/`. Helpers unchanged from W1; do not reinstall them.
3. **Kill any stale helper from prior runs**: `pkill -f /usr/libexec/pepper-x/pepperx-uinput-helper 2>/dev/null; sleep 0.5; pgrep -af pepperx-uinput-helper` — must be empty before launch.
4. **Kill any old pepper-x and restart with stderr capture**: `pkill -f /usr/local/bin/pepper-x 2>/dev/null; sleep 0.5; pepper-x 2>&1 | tee /tmp/w9-smoke.log` (in a fresh terminal — `input` group is durable post-reboot, no `sg` workaround needed).
5. **Kate test**:
   - Open Kate, focus a document, position cursor mid-line if possible.
   - Hold Alt+Super, dictate "hello world from kate", release.
   - **Primary assertion**: cleaned text appears at the cursor.
   - **Secondary assertion**: in another terminal, `grep -E '\[Pepper X uinput\] XKB layout|\[Pepper X\] perf:' /tmp/w9-smoke.log` shows at least one helper-spawn line and the perf line for this cycle.
   - **Liveness assertion**: `pgrep -af pepperx-uinput-helper` shows a PID.
6. **Konsole test**: same shape; dictate `echo hello from konsole` at shell prompt; expect text at prompt + same secondary/liveness assertions.
7. **GNOME regression**: not tested (no GNOME environment available). Documented as known-untested risk; future cross-platform-VM workstream candidate (deferred-ideas entry).

If smoke fails — invoke `superpowers:systematic-debugging`. Failure-mode triage:
- **Helper alive, no text**: helper received connection but typing failed silently. Check helper stderr in `/tmp/w9-smoke.log` for keystroke errors; check `cat /proc/bus/input/devices | grep -A4 'Pepper X virtual keyboard'` for the virtual device registration.
- **No helper alive after dictation**: spawn-side problem. Check pepper-x stderr for "failed to launch Pepper X uinput helper" or socket connect errors.
- **Wrong text or garbled characters**: XKB layout mismatch → set `PEPPERX_XKB_LAYOUT` env.
- **Wrapper saw unexpected error type**: trace the actual error variant via stderr; either refine the wrap function or extend the wrapper's matching.

### What's NOT tested

- AT-SPI caret-aware insertion — that's W6's territory and is blocked.
- Keystroke synthesis correctness across XKB layouts — covered by `pepperx-uinput-helper`'s own tests.
- GNOME live regression — no environment.
- Performance characteristics of uinput typing — out of W9 scope.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Opening the gate breaks something on GNOME | Low | Purely additive: GNOME apps reach `EditableText` backend (always have, always will). uinput-text was *already selected* for non-whitelisted GNOME apps per `select_friendly_insert_backend` fallback chains; W9 just lets the selection actually run. **No GNOME app's selected backend changes.** |
| `focused_friendly_target` change types incorrect text into a non-text widget | Medium-low | Helper-typing into a button or non-text-input is annoying but not destructive on Wayland (kernel-level uinput; characters might be ignored or treated as accelerators). Same risk profile as any "always type" pepper-x mode. Mitigated by user-controlled trigger (deliberate Alt+Super hold) and by the fact that on GNOME the AT-SPI path mostly works anyway. Documented as known limitation. |
| Smoke passes in Kate but fails in some specific KDE app (unusual focus model) | Medium | Surface to user; capture as either deferred-ideas entry or follow-up workstream. Not a W9 blocker since "works in Kate + Konsole" meets the daily-driver bar. |
| Helper's socket path collides if pepper-x crashes mid-insert | Low | Existing socket-cleanup code in `connect_or_spawn_uinput_helper` already handles stale-socket recovery. W1's standalone-helper test confirmed cleanup. |
| Test 3 (`wrap_find_focused_error_as_uinput_fallback`) ends up trivial | Low | If the wrapping logic is genuinely a one-liner, the test is still cheap insurance against a future refactor that drops the wrap. Trivial-function-with-test is a known-acceptable pattern. |
| GNOME regression untested | Medium-low (no immediate way to verify) | Documented as explicit risk + deferred-ideas entry for a future cross-platform VM. Daily-driver-on-TuxedoOS doesn't depend on GNOME verification. |
| Synthetic `FriendlyInsertSelection` field shape wrong | Low | Implementer reads the actual struct definition before writing the wrap function. Spec says "field names illustrative." Easy to catch in code review. Either the implementer subagent or the spec-compliance reviewer will surface the right shape. |
| `cargo fmt --check` / workspace-wide `cargo clippy -- -D warnings` fail | Low | Same approach as W6 plan: scope clippy to `-p pepperx-platform-gnome -p pepper-x-app`; skip workspace-wide fmt-check (W4c will fix that drift later). |
| Helper success path produces empty `target_application_name` in the resulting `FriendlyInsertOutcome`, which downstream consumers (archive / log / history view) render as `""` for the app name | Low–Medium | Visibly degraded but not destructive. Not a daily-driver bar blocker. Documented as a known limitation that can be improved later (e.g. a follow-up workstream that injects best-effort app identification — `xprop`, focused-window heuristics — when AT-SPI can't supply it). |
| First-spawn-on-demand latency on fresh boot (helper not yet running, dictation stalls until spawn completes) | Low | The helper spawn takes ~tens-of-ms; user perceives a slight stall on the first dictation after boot. Not a correctness issue. If perceptible, autostart via the `~/.config/autostart/pepper-x.desktop` from W1 ensures pepper-x (and thus a pre-warmed helper after first-dictation-of-session) is always already running when the user holds the hotkey. |
| Pre-existing test elsewhere in the codebase asserts `UINPUT_TEXT_BACKEND_NAME` is rejected by the gate | Low | Implementer pre-flights: workspace-wide `grep -rn UINPUT_TEXT_BACKEND_NAME crates/ app/` to enumerate test usage and confirm no other test will silently flip green/red because of the gate change. (The W6 fact-check verified there's no `ensure_string_injection_backend_is_rejected` test; same caution applies for any uinput-text-rejection test.) |

## Out of scope (explicit non-goals)

- **Approach C startup-time AT-SPI viability check.** Captured in the deferred-ideas section per Q1 decision; not implemented in W9.
- **Improving `pepperx-uinput-helper` keystroke synthesis behavior.** Helper code unchanged in W9. XKB layout / hold-timing tweaks would be separate workstreams.
- **Re-enabling W6 (caret-aware AT-SPI insertion).** W9 makes pepper-x work *adequately* on KDE without W6, but caret-aware insertion is genuinely better when AT-SPI works. W6 stays `blocked` on the Qt AT-SPI bridge availability, independent of W9.
- **Cleanup-context lookup (`crates/pepperx-platform-gnome/src/context.rs:50-67`).** The W1 log line `failed to inspect focused target for cleanup context` originates here; `capture_supporting_context()` is a no-op on every platform regardless of W9. Fixing it is W7's territory (OCR portal-response fix + rewire). W9 does NOT make cleanup-context start working.
- **`UnsupportedTarget` semantics elsewhere.** W9 wraps the `UnsupportedTarget` produced inside `focused_friendly_target` (at `:981`). If `UnsupportedTarget` is constructed elsewhere in the codebase and reaches the wrapper from a different path, W9 does NOT change that path's routing; documented as a follow-up if surfaced by smoke or future use.
- **User-visible parity with caret-aware AT-SPI insert.** Post-W9, dictation on KDE uses kernel-level `uinput` keystroke synthesis: characters land wherever current keyboard focus is, with no caret-position knowledge, no readback verification, no undo grouping, and no app-aware behavior. **This is the daily-driver bar — adequate for typing, not the full experience.** W6 (still blocked) is what enables caret-aware insert. The roadmap and `pepper-x-install.md` need to be honest about this distinction; "text appears at cursor" in W9 means "at current keyboard focus", not "at AT-SPI caret offset."
- **GNOME live regression test.** No GNOME environment available; documented as untested risk; future cross-platform-VM workstream candidate.
- **Renaming or splitting `pepperx-platform-gnome`.** That's W8 (conditional), separately gated.
- **Upstream PR filing.** W9 lands on `lukepatrick/pepper-x:main` only. Filing to `obra/pepper-x` is a separate manual decision later.

## Auxiliary deliverables (folded into W9's commits)

1. **Roadmap state transitions** — W9 row `pending` → `planned` (when plan written) → `in-progress` (when execution starts) → `done` (when merged). Spec/Plan columns populate. `current_workstream:` advances on each transition; advances to W2 when W9 lands.
2. **Status-log entries** — one line per transition; final entry captures smoke outcomes (especially: helper IS spawned, text DOES appear, end-to-end on TuxedoOS).
3. **Two new entries appended to "Refactor / enhancement ideas (deferred)"**:
   - **Startup-time AT-SPI viability check** (Approach C from W9 brainstorm). Trigger: per-dictation overhead becomes measurable, OR desire for clearer one-shot "what-mode-am-I-in" startup logging.
   - **GNOME cross-platform regression VM**. Trigger: a W9-class change feels under-tested without GNOME verification, OR a user reports a GNOME regression.
(No "stale gate test rename" auxiliary deliverable — the test cited by an earlier brainstorm Explore agent does not exist with that name. The test at `atspi.rs:2055-2069` is `clipboard_insert_accepts_selected_backend` and is correct as-is.)

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. Plan will translate the four-stage structure (worktree setup → TDD red phase + green phase + quality gates + commit → build + install → manual smoke → wrap-up) into ordered, individually-verifiable tasks. Manual smoke is the hard sync point requiring user action; everything else is autonomous-friendly.
