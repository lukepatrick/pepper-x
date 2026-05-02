# W6 — AT-SPI KDE Whitelist Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `friendly_insert_target_class_from_application_id` in `crates/pepperx-platform-gnome/src/atspi.rs` to recognize 6 KDE apps (Kate, KWrite, Konsole, Kontact, KMail, Falkon) so they take pepper-x's caret-aware AT-SPI insertion path on KDE rather than falling through to the broken `UINPUT_TEXT_BACKEND_NAME` gate.

**Architecture:** A small flat-`match` extension. Six apps × two ID forms each = twelve new match-arm strings, distributed across three existing categories (`TextEditor`, `BrowserTextarea`, `Terminal`). Three existing per-category unit tests get extended; one new test pins the executable-name → application-id passthrough. The smoke gate uses `pgrep pepperx-uinput-helper` and stderr log inspection to discriminate AT-SPI insertion from plain uinput typing — without that, "text appears at cursor" is observationally indistinguishable between the two backends.

**Tech Stack:**
- Rust (workspace member `pepperx-platform-gnome`), `cargo` toolchain (rustup stable, currently 1.95.0)
- Test module `mod accessible_insert_runtime_helpers` at `crates/pepperx-platform-gnome/src/atspi.rs:1937`
- AT-SPI runtime: `at-spi2-core` package (already installed and active per W1 verification)
- Smoke discriminator tools: `pgrep`, `tee`, KDE Plasma `Kate` and `Konsole`

**Source spec:** `docs/superpowers/specs/2026-05-01-w6-atspi-kde-whitelist-design.md`

---

## File Structure

This plan modifies/creates these files. Almost everything in-repo on the W6 branch.

**In-repo (committed on the `w6-atspi-kde-whitelist` branch):**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` — extend `friendly_insert_target_class_from_application_id` at `:315-339` (12 new match-arm strings); extend three existing tests at `:2002`, `:2014`, `:2026`; add one new test for executable-name pinning; add one-line `// TODO(W9):` comment near `:858-877`.
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W6, populate W6 row's `Spec` and `Plan` columns, add new "Refactor / enhancement ideas (deferred)" section, status-log entries.

**Out-of-repo (system, written via sudo):**
- Replace: `/usr/local/bin/pepper-x` via `sudo install` (helper binaries unchanged from W1).

**Worktree-only / ephemeral:**
- `/tmp/w6-smoke.log` — pepper-x stderr capture during manual smoke (not committed).

**Pre-flight diagnostic (no file output, just verification):**
- `busctl --user tree org.a11y.atspi.Registry` — confirms AT-SPI sees Kate/Konsole on this box. Run from any terminal (not committed).

---

## Task 1: Create W6 worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist/`
- (git) Create branch: `w6-atspi-kde-whitelist` from `main`

- [ ] **Step 1: Verify clean main**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
git worktree list
```

Expected output:
- `git status`: working tree clean (or only untracked/ignored — no modified tracked files); on branch `main`.
- `git branch --show-current`: `main`
- `git worktree list`: only the main checkout (no leftover w-prefixed worktrees from prior workstreams).

- [ ] **Step 2: Create worktree**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree add -b w6-atspi-kde-whitelist ../pepper-x.w6-atspi-kde-whitelist main
```

Expected output:
```
Preparing worktree (new branch 'w6-atspi-kde-whitelist')
HEAD is now at <sha> <message of latest main commit>
```

- [ ] **Step 3: Verify worktree**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist
git worktree list
git branch --show-current
```

Expected output:
- `git worktree list`: two entries — main checkout, plus the new worktree at `../pepper-x.w6-atspi-kde-whitelist` on `w6-atspi-kde-whitelist`.
- `git branch --show-current`: `w6-atspi-kde-whitelist`

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist/` unless explicitly noted.**

---

## Task 2: Update roadmap — W6 to in-progress; populate Spec/Plan columns; add deferred-ideas section

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W6 row in workstream table, new section, status log.

- [ ] **Step 1: Update top status block**

In `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`, replace the status block:

OLD:
```
current_workstream:   W6 — AT-SPI app whitelist additions for KDE
phase:                1  (promoted from Phase 2 — see status log 2026-05-01)
state:                pending  (W6 awaits brainstorming; W1 done with caveats)
branch:               (none yet — created when W6 spec is written)
worktree:             (none yet)
last_updated:         2026-05-01
```

NEW:
```
current_workstream:   W6 — AT-SPI app whitelist additions for KDE
phase:                1
state:                in-progress
branch:               w6-atspi-kde-whitelist
worktree:             ../pepper-x.w6-atspi-kde-whitelist
last_updated:         2026-05-01
```

- [ ] **Step 2: Update W6 row in workstream table**

In the same file, find the W6 row (currently has `pending` in the State column and `—` in the Branch/Spec/Plan columns). Replace the row's leading columns to:

OLD (the State, Branch, Spec, Plan columns of W6's row):
```
| W6 | 1 | AT-SPI app whitelist additions for KDE | `pending` | — | — | — | yes |
```

NEW:
```
| W6 | 1 | AT-SPI app whitelist additions for KDE | `in-progress` | `w6-atspi-kde-whitelist` | `2026-05-01-w6-atspi-kde-whitelist-design.md` | `2026-05-01-w6-atspi-kde-whitelist.md` | yes |
```

(Preserve the existing Notes column unchanged — only edit the State, Branch, Spec, Plan cells.)

- [ ] **Step 3: Add the "Refactor / enhancement ideas (deferred)" section**

Locate the "## Non-goals (explicit)" section header. Insert this new section **immediately before** "## Non-goals (explicit)":

```markdown
## Refactor / enhancement ideas (deferred)

Considered, rejected for now, might revisit later. Each entry has a trigger condition that would promote it to a real workstream.

- **Whitelist data-table refactor** — replace the flat `match` in `crates/pepperx-platform-gnome/src/atspi.rs:315-339` with a `static` array of `(application_id, FriendlyInsertTargetClass)` pairs and look up by linear scan. Trigger: whitelist grows past ~50 entries OR an entry needs metadata beyond the category enum (e.g. per-app insertion strategy override). (Considered during W6 brainstorm; rejected as over-engineering at the current ~30-entry scale.)
- **Whitelist split into GNOME/KDE sub-functions** — `..._for_gnome_app` and `..._for_kde_app` returning `Option<FriendlyInsertTargetClass>`, composed by the public function. Trigger: ≥10 KDE-specific entries accumulate, or the flat match becomes hard to scan in a single screen. (Considered during W6 brainstorm; rejected — pure indirection for the current 6 KDE entries.)
- **Conditional helper stderr suppression** — make `fc04b8b`'s suppression toggleable via env var like `PEPPERX_HELPER_STDERR=1`. Trigger: another debugging session blocked on invisible helper logs (W1 was the first; second occurrence promotes this to a workstream — likely as W4d).

Future workstreams that surface deferred ideas append entries here rather than burying them in commit messages or PR comments.
```

- [ ] **Step 4: Append status-log entry**

At the very bottom of the file (under "## Status log"), add:
```
- `2026-05-01` — W6 plan written and execution begun. State: `pending` → `in-progress`. Branch: `w6-atspi-kde-whitelist`. New "Refactor / enhancement ideas (deferred)" section seeded with three entries from W6 brainstorm.
```

- [ ] **Step 5: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W6 in-progress; add deferred-ideas roadmap section"
```

Expected: one file changed.

---

## Task 3: Pre-flight — verify AT-SPI registry sees Kate and Konsole

This is a Phase 0 gate from the spec's "deferred structural improvements." If AT-SPI on this box doesn't see Qt apps, the W6 whitelist patch isn't the unblocker — escalate to W9 instead of writing code. **This task is run BEFORE editing any source code.**

- [ ] **Step 1: Make sure pepper-x is NOT running**

```sh
pgrep -af pepper-x; echo "---"
```

If a `pepper-x` PID is shown, kill it:
```sh
pkill -f /usr/local/bin/pepper-x
```

We want a clean AT-SPI registry (without pepper-x's own `com.obra.PepperX` introspection in the way of reading the tree).

- [ ] **Step 2: Open Kate and Konsole**

In the desktop, manually launch:
- Kate (open any document or new file)
- Konsole (any shell prompt)

Leave both windows open; place focus on Kate first.

- [ ] **Step 3: Inspect the AT-SPI registry tree**

Run:
```sh
busctl --user tree org.a11y.atspi.Registry 2>&1 | head -80
```

Expected output: a tree of D-Bus paths under `/org/a11y/atspi/...`. Look specifically for entries with names that include `Kate` or `Konsole` — they should appear as branches in the tree.

If the command itself fails with `Could not find unique name for org.a11y.atspi.Registry`, the AT-SPI bus isn't reachable from this shell — try the same command without `--user` (system bus), or set `XDG_RUNTIME_DIR` if missing. Note: AT-SPI uses a session-bus connection that's set up at login.

- [ ] **Step 4: Concrete visibility check**

Run a more direct probe (script the busctl tree output through grep):
```sh
busctl --user tree org.a11y.atspi.Registry 2>&1 | grep -iE "kate|konsole" | head -10
```

Expected output: at least one matching line each for `kate` and `konsole` (case-insensitive). The exact path format is `/org/a11y/atspi/accessible/...`.

- [ ] **Step 5: Decision branch**

| Outcome of Step 4 | Action |
|---|---|
| Both Kate and Konsole appear in the tree | ✅ Proceed to Task 4. The whitelist patch is the right unblocker. |
| One appears, one doesn't | ⚠️ STOP. Surface to user. The non-visible app may need `QT_ACCESSIBILITY=1` env or a missing `qt-at-spi` plugin. Consider whether to proceed with W6 anyway (helps the visible app) or pivot. |
| Neither appears | ❌ STOP. AT-SPI on this KDE box does not see Qt apps. The W6 whitelist patch will NOT make insertion work. Surface to user; recommend promoting **W9** ahead of W2 instead of continuing W6. Possible diagnostic: `dpkg -l qt6-at-spi 2>&1 \| grep -i ^ii \|\| dpkg -l qt6-base 2>&1 \| grep -E 'qt-at-spi'` to check whether the `qt-at-spi` plugin is installed for Qt6. |

If proceeding (✅ branch), continue to Task 4. If stopping (⚠️/❌), invoke `superpowers:systematic-debugging`.

---

## Task 4: Write failing tests (TDD red phase)

**Files:**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` — extend three existing tests (`:2002`, `:2014`, `:2026`); add one new test for executable-name pinning.

We add the new assertions BEFORE editing the production match function. They will fail (with `Unsupported` returned where we expect `TextEditor`/`Terminal`/`BrowserTextarea`) — that's the red phase.

- [ ] **Step 1: Read current test file location to confirm line numbers**

```sh
grep -n "fn accessible_insert_runtime_helpers_classify_text_editor_targets\|fn accessible_insert_runtime_helpers_classify_browser_textarea_targets\|fn accessible_insert_runtime_helpers_classify_terminal_targets\|fn accessible_insert_runtime_helpers_preserve_unknown_executable_names" crates/pepperx-platform-gnome/src/atspi.rs
```

Expected output: four lines showing the four test names with line numbers near `:2003`, `:2015`, `:2027`, `:2039` (drift of ±2 lines is fine; subsequent edits use the test names as anchors, not line numbers).

- [ ] **Step 2: Extend `accessible_insert_runtime_helpers_classify_text_editor_targets`**

Find the test (around `:2002`-`:2012`). Replace its body with:

```rust
    #[test]
    fn accessible_insert_runtime_helpers_classify_text_editor_targets() {
        // Existing GNOME assertions (regression baseline — must not change).
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.gnome.TextEditor"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("gnome-text-editor"),
            FriendlyInsertTargetClass::TextEditor
        );

        // W6: KDE app entries (bare executable basename + reverse-DNS form).
        assert_eq!(
            friendly_insert_target_class_from_application_id("kate"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.kate"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("kwrite"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.kwrite"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("kontact"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.kontact"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("kmail"),
            FriendlyInsertTargetClass::TextEditor
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.kmail"),
            FriendlyInsertTargetClass::TextEditor
        );
    }
```

- [ ] **Step 3: Extend `accessible_insert_runtime_helpers_classify_browser_textarea_targets`**

Find the test (around `:2014`-`:2024`). Replace its body with:

```rust
    #[test]
    fn accessible_insert_runtime_helpers_classify_browser_textarea_targets() {
        // Existing assertions (regression baseline — must not change).
        assert_eq!(
            friendly_insert_target_class_from_application_id("browser-textarea"),
            FriendlyInsertTargetClass::BrowserTextarea
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("firefox"),
            FriendlyInsertTargetClass::BrowserTextarea
        );

        // W6: Falkon (KDE web browser).
        assert_eq!(
            friendly_insert_target_class_from_application_id("falkon"),
            FriendlyInsertTargetClass::BrowserTextarea
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.falkon"),
            FriendlyInsertTargetClass::BrowserTextarea
        );
    }
```

- [ ] **Step 4: Extend `accessible_insert_runtime_helpers_classify_terminal_targets`**

Find the test (around `:2026`-`:2036`). Replace its body with:

```rust
    #[test]
    fn accessible_insert_runtime_helpers_classify_terminal_targets() {
        // Existing assertions (regression baseline — must not change).
        assert_eq!(
            friendly_insert_target_class_from_application_id("gnome-terminal-server"),
            FriendlyInsertTargetClass::Terminal
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("ghostty"),
            FriendlyInsertTargetClass::Terminal
        );

        // W6: Konsole (KDE terminal).
        assert_eq!(
            friendly_insert_target_class_from_application_id("konsole"),
            FriendlyInsertTargetClass::Terminal
        );
        assert_eq!(
            friendly_insert_target_class_from_application_id("org.kde.konsole"),
            FriendlyInsertTargetClass::Terminal
        );
    }
```

- [ ] **Step 5: Add new pinning test for executable-name passthrough**

Locate the existing test `accessible_insert_runtime_helpers_preserve_unknown_executable_names` (around `:2039`). Insert a new test **immediately after** it (before the next test like `..._apply_insert_at_char_offset`):

```rust
    #[test]
    fn accessible_insert_runtime_helpers_preserve_kde_executable_names() {
        // Lock the assumption that KDE app basenames pass through
        // friendly_application_id_from_executable_name unchanged. If a future
        // GNOME-style remap shim accidentally remaps any of these, this test
        // catches it before insertion silently breaks for KDE.
        for kde_basename in [
            "kate", "kwrite", "konsole", "kontact", "kmail", "falkon",
        ] {
            assert_eq!(
                friendly_application_id_from_executable_name(kde_basename),
                kde_basename,
                "expected basename {kde_basename} to pass through unchanged"
            );
        }
    }
```

This test is small enough to use a loop over a slice — clearer than 6 individual `assert_eq!` calls. Failure messages identify the offending basename.

---

## Task 5: Run tests, see them fail (red phase verification)

- [ ] **Step 1: Run only the affected tests**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist
source $HOME/.cargo/env
cargo test -p pepperx-platform-gnome accessible_insert_runtime_helpers_classify 2>&1 | tail -40
```

Expected output: three test failures (one for each of the extended `_classify_*_targets` tests). Each failure prints something like:
```
assertion `left == right` failed
  left: Unsupported
 right: TextEditor
```
…showing that `friendly_insert_target_class_from_application_id("kate")` (etc.) currently returns `Unsupported` — exactly what TDD red phase expects before the production change lands.

- [ ] **Step 2: Confirm the new pinning test passes already**

```sh
cargo test -p pepperx-platform-gnome accessible_insert_runtime_helpers_preserve_kde_executable_names 2>&1 | tail -10
```

Expected output: `test result: ok. 1 passed; 0 failed`. (KDE basenames already pass through `friendly_application_id_from_executable_name` unchanged because there's no remap for them — the test is locking that fact, not creating new behavior.)

If this test FAILS at this point, that's a real surprise — surface to user; the spec assumed no remap was needed. Investigate `friendly_application_id_from_executable_name` at `atspi.rs:1110-1116` for an entry that shouldn't be there.

---

## Task 6: Edit the production whitelist (TDD green phase)

**Files:**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs:315-339` — extend the three relevant match arms.
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs:858-877` — add a `// TODO(W9):` comment.

- [ ] **Step 1: Replace `friendly_insert_target_class_from_application_id`**

Find the function (around `:315`-`:339`). Replace the entire function body with:

```rust
fn friendly_insert_target_class_from_application_id(
    application_id: &str,
) -> FriendlyInsertTargetClass {
    match application_id {
        "org.gnome.TextEditor"
        | "gnome-text-editor"
        | "kate"
        | "org.kde.kate"
        | "kwrite"
        | "org.kde.kwrite"
        | "kontact"
        | "org.kde.kontact"
        | "kmail"
        | "org.kde.kmail" => FriendlyInsertTargetClass::TextEditor,
        "browser-textarea"
        | "firefox"
        | "org.mozilla.firefox"
        | "chromium"
        | "chromium-browser"
        | "google-chrome"
        | "com.google.Chrome"
        | "brave-browser"
        | "com.brave.Browser"
        | "microsoft-edge"
        | "com.microsoft.Edge"
        | "vivaldi"
        | "com.vivaldi.Vivaldi"
        | "falkon"
        | "org.kde.falkon" => FriendlyInsertTargetClass::BrowserTextarea,
        "ghostty"
        | "xterm"
        | "gnome-terminal"
        | "gnome-terminal-server"
        | "ptyxis"
        | "konsole"
        | "org.kde.konsole" => FriendlyInsertTargetClass::Terminal,
        "wine" | "wine64-preloader" => FriendlyInsertTargetClass::Hostile,
        _ => FriendlyInsertTargetClass::Unsupported,
    }
}
```

- [ ] **Step 2: Add the W9 TODO comment**

Find `ensure_runtime_supported_backend` (around `:858`-`:877`). Insert this comment block on the line **immediately before** the function signature `fn ensure_runtime_supported_backend(`:

```rust
// TODO(W9): UINPUT_TEXT_BACKEND_NAME is silently rejected here with
// "is not implemented yet" — the upstream uinput-text fallback path
// is unreachable as a result. See docs/superpowers/specs/2026-04-30-
// tuxedoos-kde-viability-roadmap.md (W9 row) for the architectural
// fix. W6's whitelist additions work around this for whitelisted KDE
// apps but do not fix the underlying gate.
```

The comment is two-slash style (not `/* */`) to match Rust idiom and to avoid breaking on doc-comment parsing.

---

## Task 7: Run quality gates (green phase verification)

- [ ] **Step 1: Run the W6 tests**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist
source $HOME/.cargo/env
cargo test -p pepperx-platform-gnome accessible_insert_runtime_helpers 2>&1 | tail -20
```

Expected output: `test result: ok. <N> passed; 0 failed; 0 ignored`. The new W6 assertions all pass. Existing assertions (regression baseline) still pass.

- [ ] **Step 2: Run the full workspace test suite**

```sh
cargo test --workspace 2>&1 | tail -30
```

Expected output: every crate ends with `test result: ok.`; total passed count is unchanged from W1's 116 + the new entries we added (so likely 117+).

If any test fails that was passing on W1's main, the change accidentally regressed something — STOP and investigate. Do not proceed to clippy.

- [ ] **Step 3: Run clippy**

```sh
cargo clippy -p pepperx-platform-gnome -- -D warnings 2>&1 | tail -20
```

Expected output: clean — no warnings. `Finished` and exit 0.

If clippy fires on the new code (e.g. unused variables, style issues), fix it inline before proceeding. **Do NOT run `cargo clippy --workspace -- -D warnings`** — that would surface the upstream-code drift documented in W1 (W4c's job), which is unrelated to W6 and would block this gate spuriously. We only check our own crate here.

- [ ] **Step 4: cargo fmt --check is intentionally skipped**

Per spec done-criterion #3 and W1 findings, the upstream main fails `cargo fmt --check` against rustc 1.95.0. Running it here would fail on unrelated code. Skip the gate; W4c will fix the drift later.

If you really want to fmt-check just the new code, run `rustfmt --check crates/pepperx-platform-gnome/src/atspi.rs` standalone — but it's not gating.

---

## Task 8: Commit code + tests

**Files:**
- Modify: `crates/pepperx-platform-gnome/src/atspi.rs` (whitelist + 4 tests + W9 TODO comment)

- [ ] **Step 1: Stage and inspect the diff**

```sh
git add crates/pepperx-platform-gnome/src/atspi.rs
git status
git diff --cached crates/pepperx-platform-gnome/src/atspi.rs | head -100
```

Expected: only `atspi.rs` is staged. Diff shows: 12 added match arms in `friendly_insert_target_class_from_application_id`, 12 added assertions across three existing tests, one new test (`..._preserve_kde_executable_names`), one comment block before `ensure_runtime_supported_backend`. No unintended changes elsewhere.

- [ ] **Step 2: Commit**

```sh
git commit -m "$(cat <<'EOF'
W6: AT-SPI whitelist additions for KDE apps

Extend friendly_insert_target_class_from_application_id with 6 KDE
apps (Kate, KWrite, Konsole, Kontact, KMail, Falkon), each in both
bare-executable-basename and org.kde.<app> reverse-DNS forms. Apps
classified per the existing category model:
- TextEditor: Kate, KWrite, Kontact, KMail
- BrowserTextarea: Falkon
- Terminal: Konsole

Tests updated:
- Three existing classify_*_targets tests extended with the new
  KDE entries; pre-existing GNOME entries retained as regression
  assertions
- New accessible_insert_runtime_helpers_preserve_kde_executable_names
  test pins the assumption that KDE basenames pass through
  friendly_application_id_from_executable_name unchanged (locks
  against a future GNOME-style remap shim silently breaking KDE)

Auxiliary: TODO(W9) comment added at ensure_runtime_supported_backend
documenting the unreachable uinput-text fallback path. W6's whitelist
addition is a workaround; W9 is the architectural fix.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: one file changed, ~70-80 line additions / 0 deletions (depending on exact line counts).

---

## Task 9: Build release binary

- [ ] **Step 1: cargo build --release**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist
source $HOME/.cargo/env
cargo build --release 2>&1 | tail -10
```

Expected output: ends with `Finished \`release\` profile [optimized] target(s) in <time>` and exits 0. This rebuild is incremental (most crates are cached from W1); should take ~30 seconds to a few minutes depending on what's invalidated by the source-code change.

- [ ] **Step 2: Verify the new pepper-x binary exists**

```sh
ls -lh target/release/pepper-x
file target/release/pepper-x
```

Expected: ELF 64-bit executable, freshly built (newer mtime than the existing `/usr/local/bin/pepper-x`).

- [ ] **Step 3: Helper binaries — explicitly NOT rebuilding-relevant for install**

The `pepperx-uinput-helper` and `pepperx-cleanup-helper` binaries also rebuild but are functionally unchanged from W1 (no source touched in their crates). We do NOT reinstall them. The existing copies in `/usr/libexec/pepper-x/` from W1 stay.

---

## Task 10: Install the new pepper-x binary

This needs sudo. **The user runs this step.**

- [ ] **Step 1: Show the user the install command**

Print:
```
Please run, in any terminal:

  sudo install -m 755 /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist/target/release/pepper-x /usr/local/bin/

Reply when done.
```

- [ ] **Step 2: After user reports done, verify the install**

```sh
ls -l /usr/local/bin/pepper-x
file /usr/local/bin/pepper-x
```

Expected: file mtime matches the just-built binary (within seconds), executable, owned by root.

If the install put the binary somewhere unexpected (e.g. `/usr/local/sbin/`), surface to user.

---

## Task 11: Manual smoke — Kate test with discriminators

🛑 **User action required.** This is the heart of the gate; the discriminators are what make the test meaningful.

- [ ] **Step 1: Make sure no pepper-x is running yet**

```sh
pgrep -af pepper-x
```

Expected: empty (no PID). If there's an old instance, kill it: `pkill -f /usr/local/bin/pepper-x`.

- [ ] **Step 2: Launch pepper-x with stderr captured**

User runs in a fresh terminal (not Claude Code's process tree):
```sh
pepper-x 2>&1 | tee /tmp/w6-smoke.log
```

The GTK window opens; capture file at `/tmp/w6-smoke.log` accumulates pepper-x's stderr.

- [ ] **Step 3: Open Kate and prepare a document**

User opens Kate (`kate &`). Open or create a new document. Type a few words to confirm normal input works. Position the cursor at end of typed text.

- [ ] **Step 4: Dictate**

User holds Alt+Super, says *"hello world from kate"*, releases.

Within ~2 seconds, observed text behavior should be: cleaned text appears at the cursor.

- [ ] **Step 5: Discriminator (a) — `pgrep` for the helper**

In Claude Code's terminal (or any other terminal), immediately run:
```sh
pgrep -af pepperx-uinput-helper
```

**Expected: empty output (no PID).** That means no helper subprocess was spawned during the dictation cycle, which means insertion went through the AT-SPI path. This is the strong signal that W6's whitelist addition is doing its job.

If `pgrep` returns a PID, the helper WAS spawned — meaning insertion went through plain uinput typing, NOT AT-SPI. Smoke gate FAILED on Kate. See "If smoke fails" below.

- [ ] **Step 6: Discriminator (b) — log inspection**

```sh
tail -40 /tmp/w6-smoke.log
```

Look for lines around the dictation cycle. Expected: a log line indicating the AT-SPI / friendly-insert backend was selected — something like `friendly insertion backend friendly-insert succeeded` or `using atspi-editable-text` (exact wording is in pepper-x's source; substantively, the word "friendly" or "atspi" appears, and the word "uinput" does NOT appear in association with the insertion).

If the log shows `uinput-text` selection or `failed to find a focused target` followed by silent-failure perf log, the gate FAILED on Kate. See "If smoke fails" below.

- [ ] **Step 7: Decision**

Both discriminators pass → ✅ Kate side of W6 is verified.
Either discriminator fails → ❌ STOP. See "If smoke fails" below.

**If smoke fails:**
- AT-SPI registry visibility on KDE is still suspect even though the pre-flight (Task 3) passed. The pre-flight checked the *registry tree*; what may differ at dictation time is per-app accessibility plumbing (`QT_ACCESSIBILITY` env, Qt6 plugin path, etc.).
- Invoke `superpowers:systematic-debugging`. Possible diagnostic steps: launch Kate via `QT_ACCESSIBILITY=1 kate` and re-test; check `dpkg -l | grep -E 'qt6-base|qt-at-spi'` for the Qt6 AT-SPI plugin package.
- **Do not patch around.** If a quick env-var fix doesn't resolve it, surface to user with the recommendation to promote **W9** ahead of W2 (per spec's failure decision tree).

---

## Task 12: Manual smoke — Konsole test with discriminators

🛑 **User action required.** Same shape as Task 11, different target app.

- [ ] **Step 1: Open Konsole**

User opens Konsole (`konsole &`). Focus on a shell prompt; type a few characters to confirm normal input works; clear the line.

- [ ] **Step 2: Dictate**

User holds Alt+Super, says *"echo hello from konsole"*, releases.

Within ~2 seconds, observed text behavior: `echo hello from konsole` appears at the prompt. (Punctuation may differ slightly per cleanup model behavior — that's fine; what matters is the text and the discriminators.)

- [ ] **Step 3: Discriminator (a) — `pgrep` for the helper**

```sh
pgrep -af pepperx-uinput-helper
```

**Expected: empty.** Same as Kate.

- [ ] **Step 4: Discriminator (b) — log inspection**

```sh
tail -40 /tmp/w6-smoke.log
```

Look for the most recent dictation cycle. Expected: AT-SPI/friendly-insert backend selected.

- [ ] **Step 5: Decision**

Both discriminators pass → ✅ Konsole side of W6 is verified.
Either discriminator fails → ❌ STOP, same failure path as Kate.

---

## Task 13: Manual smoke — focus-switching test (Kate → Konsole → Kate)

🛑 **User action required.** Tests that the runtime focus-tracking dispatch correctly resolves the *currently* focused app, not a stale one. Suggested by review pass; lightweight added confidence.

- [ ] **Step 1: Both Kate and Konsole still open from Tasks 11-12**

User confirms both windows are still up.

- [ ] **Step 2: Sequence three dictations with focus changes**

Without restarting pepper-x:

1. Click into the Kate window. Hold Alt+Super, say *"first kate"*, release. Confirm text appears in Kate.
2. Alt-Tab (or click) to switch focus to Konsole. Hold Alt+Super, say *"middle konsole"*, release. Confirm text appears at the Konsole prompt.
3. Alt-Tab back to Kate. Hold Alt+Super, say *"last kate"*, release. Confirm text appears in the Kate document (NOT in Konsole).

- [ ] **Step 3: Discriminators**

After all three dictations:
```sh
pgrep -af pepperx-uinput-helper
tail -60 /tmp/w6-smoke.log
```

Expected: `pgrep` empty; log shows three dictation cycles, each selecting the AT-SPI / friendly-insert backend, with focus targets resolving to `kate`, `konsole`, `kate` respectively.

If the second dictation appeared in Kate (the previously-focused app) instead of Konsole, focus-tracking is broken — surface to user. Likely culprits: AT-SPI's "active focused frame" cache, or pepper-x's snapshot-time-of-keypress vs. snapshot-at-stop semantics.

- [ ] **Step 4: Decision**

All three dictations land in the right app and discriminators pass → ✅ focus-switching is verified.
Any dictation lands in the wrong app → ❌ STOP, surface to user. This is a real architectural concern but may not be blocking for W6's daily-driver bar (you can just keep focus stable when dictating). Ask user how to proceed.

---

## Task 14: Update roadmap — W6 done

- [ ] **Step 1: Update top status block**

In the worktree's `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`, replace the status block:

OLD:
```
current_workstream:   W6 — AT-SPI app whitelist additions for KDE
phase:                1
state:                in-progress
branch:               w6-atspi-kde-whitelist
worktree:             ../pepper-x.w6-atspi-kde-whitelist
last_updated:         2026-05-01
```

NEW (use today's date if execution spans multiple days):
```
current_workstream:   W2 — KDE Global Shortcut → D-Bus
phase:                1
state:                pending  (W6 done; W2 awaits brainstorming)
branch:               (none yet — created when W2 spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

- [ ] **Step 2: Update W6 row state to `done`**

Change W6's State column from `in-progress` to `done`. Spec/Plan/Branch columns stay populated.

- [ ] **Step 3: Append status-log entry**

At the bottom under "## Status log":
```
- `<today's date>` — W6 done. AT-SPI registry visibility for Kate and Konsole verified pre-flight; whitelist additions made; tests pass; manual smoke discriminators (pgrep empty + AT-SPI backend in stderr) confirm AT-SPI insertion path active in Kate and Konsole; focus-switching test passes. State: `in-progress` → `done`. Next: W2 brainstorm.
```

If smoke had unexpected results (e.g. needed `QT_ACCESSIBILITY=1` env to work; or focus-switch had a hiccup; or one specific dictation went via uinput unexpectedly), reflect that honestly in the log entry — future re-orientation depends on it.

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W6 done; advance current_workstream to W2"
```

---

## Task 15: Merge `w6-atspi-kde-whitelist` to main

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: working tree clean, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w6-atspi-kde-whitelist -m "Merge W6: AT-SPI KDE whitelist additions

W6 verified end-to-end on TuxedoOS 24.04 + KDE Plasma + Wayland.
Six KDE apps (Kate, KWrite, Konsole, Kontact, KMail, Falkon) added to
the AT-SPI app whitelist; smoke tests in Kate and Konsole confirmed
caret-aware AT-SPI insertion (pgrep pepperx-uinput-helper empty during
dictation; pepper-x stderr logs AT-SPI backend selection).

Auxiliary: TODO(W9) comment at atspi.rs:858-877 documents the still-
broken uinput-text fallback. Roadmap status table reflects W6 done;
'Refactor / enhancement ideas (deferred)' section seeded.

current_workstream advances to W2 (KDE Global Shortcut)."
```

Expected output: a merge commit summary listing changed files (`atspi.rs`, `tuxedoos-kde-viability-roadmap.md`).

- [ ] **Step 3: Do NOT push to origin**

The user's stated preference (across W1) is to keep work local until they decide to push. Skip `git push origin main`. Note in the conversation that there are now N commits ahead of origin/main (count via `git log --oneline origin/main..HEAD | wc -l`).

---

## Task 16: Cleanup — remove the worktree

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w6-atspi-kde-whitelist
git status
```

Expected: `nothing to commit, working tree clean`. If anything is uncommitted, STOP — that work would be lost on remove.

- [ ] **Step 2: Switch back to main checkout, remove worktree, delete branch**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w6-atspi-kde-whitelist
git worktree list
git branch -d w6-atspi-kde-whitelist
git branch
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout.
- `branch -d`: `Deleted branch w6-atspi-kde-whitelist (was <sha>).`
- `branch`: only `main` (with `*` marker).

If `git branch -d` refuses with "not fully merged", don't force with `-D`. Investigate why the merge looked successful but the branch reachability check disagrees.

---

## Done

When all 16 tasks are checked, W6 is complete:

- AT-SPI whitelist extended with 6 KDE apps × 2 ID forms.
- Three existing tests extended with new entries + regression assertions; one new pinning test added.
- TODO(W9) comment placed at the `ensure_runtime_supported_backend` gate so future readers know about the fallback gap.
- `cargo build --release`, `cargo test --workspace`, `cargo clippy -p pepperx-platform-gnome -- -D warnings` all green.
- `pepper-x` binary updated in `/usr/local/bin/`; helpers unchanged.
- Manual smoke gate passed in Kate AND Konsole using `pgrep` + log discriminators (proving AT-SPI path, not plain uinput typing).
- Focus-switching smoke test passed (Kate → Konsole → Kate sequence with each dictation landing in the correct app).
- Roadmap reflects W6=done, W2=pending+next, "Refactor / enhancement ideas (deferred)" section seeded.

**Next session-start re-orientation will pick up W2.** Per ways-of-working entry-point logic: `current_workstream: W2, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

The spec's risks table flagged several scenarios. Tasks above address them as follows:

| Risk | Where addressed |
|---|---|
| AT-SPI registry doesn't see KDE Qt apps | Task 3 (pre-flight `busctl tree` check, with explicit decision branch). |
| `org.kde.kmail` is the wrong D-Bus name | Low impact — Task 8's commit message documents the rationale; runtime uses bare basename anyway. |
| Existing entry regression | Task 4 retains pre-existing assertions in each extended test; Task 7 catches via `cargo test`. |
| Hotkey collision | Independent of W6; documented in `pepper-x-install.md`. |
| Clippy fires on new code | Task 7 Step 3 runs clippy; fixes go inline. |
| AT-SPI on KDE Wayland Qt6 plugin missing | Task 11/12 failure path includes `QT_ACCESSIBILITY=1` and `dpkg -l | grep qt6-at-spi` diagnostic. |
| Kontact masquerade (KMail-in-Kontact) | Documented limitation; spec's "Out of scope" item — not addressed in W6. |
| Falkon `BrowserTextarea` unverified | Falkon manual smoke deferred (Q2 Option C); ad-hoc testing only. Documented in spec. |
| Snap/Flatpak Kate/Konsole basename wrapping | Documented limitation; out of scope for distro-pkg KDE on TuxedoOS. |
