# Ways of Working — TuxedoOS/KDE Viability Work

> **READ at session start, every time.** Skill drift is silent; this doc is the corrective.

This is the prescriptive companion to `2026-04-30-tuxedoos-kde-viability-roadmap.md`. The roadmap says **what** is next; this doc says **how** to do it.

## Per-workstream rhythm — same five steps, every workstream

```
1. brainstorm    — superpowers:brainstorming        → writes spec to docs/superpowers/specs/
2. plan          — superpowers:writing-plans        → writes plan to docs/superpowers/plans/
3. isolate       — superpowers:using-git-worktrees  → branch + worktree per workstream
4. execute       — superpowers:executing-plans      → run the plan
                   ├─ if code: superpowers:test-driven-development
                   └─ if stuck: superpowers:systematic-debugging
5. verify+ship   — superpowers:verification-before-completion
                 → superpowers:requesting-code-review
                 → superpowers:finishing-a-development-branch
                 → update roadmap status table + status log
```

**Skipping any step is the failure mode.** The doc says ALWAYS, not "consider." Even one-line README changes get a spec, a plan, and a worktree.

## Situation → skill/agent reference

| When this happens | Invoke | Why |
|---|---|---|
| Session starts | Read roadmap → ways-of-working (this file) | Re-orient before any action. CLAUDE.md auto-loads but does not re-read on every prompt. |
| About to start a new workstream | `superpowers:brainstorming` | Even tiny ones get a spec. |
| Spec approved, no plan yet | `superpowers:writing-plans` | Terminal state of brainstorming is this. |
| Plan ready, work begins | `superpowers:using-git-worktrees` + `superpowers:executing-plans` | Isolation + execution discipline. |
| Writing Rust | `superpowers:test-driven-development` + `Explore` agent for context | Red-green-refactor; Explore agent for codebase searches that span multiple files. |
| Touching async Rust (zbus, tokio paths) | `systems-programming:rust-async-patterns` skill | Most pepper-x async surface is zbus + tokio — common in W2 (D-Bus method invocation) and W7 (XDG portal is async). |
| Touching FFI boundaries | `systems-programming:memory-safety-patterns` skill | pepper-x has thick FFI: `llama-cpp-sys-4`, `ort` (ONNX), `evdev`, `xkbcommon`, `pipewire`. RAII/safety patterns matter when wrapping these. |
| Build / runtime failure | `superpowers:systematic-debugging` skill + `error-debugging:debugger` agent | Hypothesis-driven, not poke-and-hope. |
| Shell script (W3-style install scripts, udev rules, autostart .desktop) | `shell-scripting:bash-defensive-patterns` skill | Defensive Bash for fragile install steps. |
| About to claim "done" | `superpowers:verification-before-completion` skill | **Run the actual commands and confirm output.** Not "tests should pass" — actually run them. |
| Code change ready for merge | `superpowers:requesting-code-review` skill + `comprehensive-review:code-reviewer` agent (general) or `systems-programming:rust-pro` agent (Rust-heavy) | Both: skill (process) + agent (independent review). |
| Plan-level second opinion needed | `comprehensive-review:architect-review` agent | For W7's two-part rewire and W8's crate split. |
| Receiving review comments | `superpowers:receiving-code-review` skill | Forces technical evaluation before knee-jerk fixes. |
| Workstream done, integrating | `superpowers:finishing-a-development-branch` skill | Decides merge vs. squash; updates roadmap. |
| Stuck three+ tool calls deep on something not in the plan | STOP. Ask user. Default: drop the side-quest. | Anti-bloat. |

## Workstream-specific notes

- **W1 (build + first-run)**: heavy on shell, udev, systemd, `.desktop` files. Use `shell-scripting:bash-defensive-patterns` when writing/updating install scripts. Use `error-debugging:debugger` agent on first build failure (clang? `ort` network? GTK floor?). Verification = manually run hold-to-record into Kate; don't claim done until typing actually appears.
- **W2 (KDE shortcut + D-Bus)**: D-Bus surface verification first — use `Explore` agent to confirm the method signature in `crates/pepperx-platform-gnome/src/service.rs` and `crates/pepperx-ipc/src/lib.rs` (verified during plan: bus name `com.obra.PepperX.Service`, object path `/com/obra/PepperX`, method `start_recording(trigger_source: &str)` with valid values `modifier-only` / `standard-shortcut` / `shell-action`). Manual `gdbus call` testing is the verification, not unit tests.
- **W4a/W4b (README + CI fixes)**: doc/infra only. W4b verification requires confirming CI still passes after the change — push the branch and watch the run before merging to `main`.
- **W5 (script skip)**: trivial. Real GNOME-presence check is at `scripts/dev-install-extension.sh:161-164`, not 90-93 as install.md claims. Verify on KDE that the script now exits 0.
- **W6 (AT-SPI whitelist)**: Rust addition at `crates/pepperx-platform-gnome/src/atspi.rs:315-339`. TDD light. Manual verification = hold-to-record into Kate and confirm caret-aware insert (text appears at the cursor position regardless of focus games), not just plain typing.
- **W7 (OCR portal fix)**: highest code risk. Two-part rewire — `screenshot.rs:95-161` portal response handling **and** `context.rs:50-68` calling `screenshot_window` from the public entry point. Brainstorm separately *before* writing the plan; consider `Plan` agent for second-opinion on approach. Apply `systems-programming:rust-async-patterns` (XDG portal is async D-Bus).
- **W8 (platform crate split, conditional)**: triggered only by gate criteria in the roadmap. When triggered, run `comprehensive-review:architect-review` agent on the proposed split *before* writing-plans. Apply `systems-programming:rust-pro` agent in addition to architect-review — workspace restructure has Rust-specific gotchas around `pub use` re-exports, feature flags, and `cfg(target_os)` boundaries.

## Anti-drift triggers

Watch for the rationalization patterns listed in the `superpowers:using-superpowers` skill — that's the canonical list ("This is just a one-liner," "I already know what to do," "Let me peek at the code first," etc.). Don't duplicate them here.

## Mid-session anti-drift rule

If three+ tool calls deep on something **not** covered by the current workstream's plan:

1. **Stop**.
2. Ask the user whether this is in-scope, becomes its own future workstream, or is a side-quest to drop.
3. Default action if unclear: **drop**. Side-quests are how plans bloat.

## Upstream PR posture

This workflow does **not** auto-file PRs upstream. Filing a PR to `obra/pepper-x` is a separate manual event triggered by the user, drawing from the roadmap's "Potential upstream contributions" table. The workstream loop ends at "merged to fork's `main`."

## Status table maintenance

After every state transition (spec → planned, planned → in-progress, etc.), update the matching row in the roadmap's status table **and** add a one-liner to the roadmap's status log. The roadmap is read at session start; if it lies, future-me drifts.
