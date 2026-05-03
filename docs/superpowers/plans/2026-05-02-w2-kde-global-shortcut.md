# W2 — KDE Global Shortcut Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `packaging/deb/pepper-x.desktop` with `[Desktop Action ...]` blocks for the four user-facing D-Bus methods, plus document the System Settings → Custom Shortcuts binding workflow. Adds an alternative trigger surface for KDE users alongside pepper-x's existing evdev hotkey capture.

**Architecture:** Packaging + docs only. **Zero Rust source changes.** The D-Bus surface (`com.obra.PepperX.Service` → `StartRecording`/`StopRecording`/`ShowSettings`/`ShowHistory`) already supports everything W2 needs. Each `.desktop` Action wraps a `gdbus call --timeout 1` invocation; KDE's Custom Shortcuts UI surfaces them automatically once the file is on the user's `XDG_DATA_DIRS` path.

**Tech Stack:**
- `desktop-file-utils` (already installed: `desktop-file-validate`, `update-desktop-database`)
- `kbuildsycoca6` (from `plasma-workspace`; rebuilds KDE's KService cache after `.desktop` changes)
- `gdbus` (from `libglib2.0-bin`; usually present on standard KDE installs)
- KDE Plasma 6 System Settings → Shortcuts → Custom Shortcuts

**Source spec:** `docs/superpowers/specs/2026-05-02-w2-kde-global-shortcut-design.md`

---

## File Structure

This plan modifies/creates these files. All in-repo on the W2 branch.

**In-repo (committed on `w2-kde-global-shortcut`):**
- Modify: `packaging/deb/pepper-x.desktop` — extend with `Actions=...` line + four `[Desktop Action ...]` blocks; broaden `Categories=` to drop `GNOME;`; update `Comment=` to drop "GNOME-first" framing.
- Modify: `README.md` — add a new subsection under `## Usage` titled "Triggering dictation from a KDE Global Shortcut".
- Modify: `pepper-x-install.md` — add the same KDE Global Shortcut binding content near the existing "Optional KDE polish" section.
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W2; new W10 row; new Phase 4 description; 3 new entries in "Refactor / enhancement ideas (deferred)"; status-log entries.

**Out-of-repo (system, written without sudo):**
- Install: `~/.local/share/applications/pepper-x.desktop` (per-user copy, populated from the worktree's edited file).

**Worktree-only / ephemeral:**
- `/tmp/w2-smoke.log` — pepper-x stderr capture during manual smoke (not committed).

---

## Task 1: Create W2 worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w2-kde-global-shortcut/`
- (git) Create branch: `w2-kde-global-shortcut` from `main`

- [ ] **Step 1: Verify clean main**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
git worktree list
```

Expected: clean working tree, on `main`, only the main checkout in worktree list (the `w6-atspi-kde-whitelist` branch may still exist as a non-worktree resumption point — that's fine).

- [ ] **Step 2: Create the worktree**

```sh
git worktree add -b w2-kde-global-shortcut ../pepper-x.w2-kde-global-shortcut main
```

Expected: `Preparing worktree (new branch 'w2-kde-global-shortcut')` then `HEAD is now at <sha>`.

- [ ] **Step 3: Verify**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w2-kde-global-shortcut
git worktree list
git branch --show-current
```

Expected: two worktrees listed; current branch `w2-kde-global-shortcut`.

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w2-kde-global-shortcut/`.**

---

## Task 2: Update roadmap — W2 to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W2 row, status log.

- [ ] **Step 1: Update top status block**

Replace:

```
current_workstream:   W2 — KDE Global Shortcut → D-Bus
phase:                1
state:                pending  (W9 done; W2 awaits brainstorming)
branch:               (none yet — created when W2 spec is written)
worktree:             (none yet)
last_updated:         2026-05-02
```

With:

```
current_workstream:   W2 — KDE Global Shortcut → D-Bus
phase:                1
state:                in-progress
branch:               w2-kde-global-shortcut
worktree:             ../pepper-x.w2-kde-global-shortcut
last_updated:         2026-05-02
```

- [ ] **Step 2: Update W2 row in workstream table**

Find W2's row. Change `State` from `pending` to `in-progress`; populate `Branch` (`w2-kde-global-shortcut`), `Spec` (`2026-05-02-w2-kde-global-shortcut-design.md`), `Plan` (`2026-05-02-w2-kde-global-shortcut.md`).

- [ ] **Step 3: Append status-log entry**

At the bottom under "## Status log":
```
- `2026-05-02` — W2 plan written and execution begun. State: `pending` → `in-progress`. Branch: `w2-kde-global-shortcut`. Six must-fix corrections applied to W2 spec from review pass (kbuildsycoca6 not update-desktop-database; --timeout 1 on gdbus calls; gdbus prerequisite documented; per-user vs system install conflict documented; errors-invisible-from-shortcut-context documented; shell-action rationale corrected from "behavior switch" to "history/telemetry attribution"). Three new deferred-ideas surfaced (D-Bus .service activation file; ToggleRecording method/script; desktop-file-validate as CI gate).
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W2 in-progress"
```

Expected: one file changed.

---

## Task 3: Edit `packaging/deb/pepper-x.desktop`

**Files:**
- Modify: `packaging/deb/pepper-x.desktop` — replace entire file with the W2 content.

- [ ] **Step 1: Read the existing file to confirm baseline**

```sh
cat packaging/deb/pepper-x.desktop
```

Expected: 11 lines matching the spec's "before" state — `Categories=Utility;GNOME;`, `Comment=GNOME-first local Linux dictation shell`, no `Actions=` line.

- [ ] **Step 2: Replace the file with the W2 content**

Replace `packaging/deb/pepper-x.desktop` with exactly:

```
[Desktop Entry]
Type=Application
Version=1.0
Name=Pepper X
Comment=Local Linux dictation shell with hold-to-record and KDE Global Shortcut support
Exec=pepper-x
Icon=com.obra.PepperX
StartupWMClass=com.obra.PepperX
Categories=Utility;AudioVideo;
Terminal=false
Actions=StartRecording;StopRecording;ShowSettings;ShowHistory;

[Desktop Action StartRecording]
Name=Start dictation
Exec=gdbus call --session --timeout 1 --dest com.obra.PepperX.Service --object-path /com/obra/PepperX --method com.obra.PepperX.StartRecording shell-action

[Desktop Action StopRecording]
Name=Stop dictation
Exec=gdbus call --session --timeout 1 --dest com.obra.PepperX.Service --object-path /com/obra/PepperX --method com.obra.PepperX.StopRecording

[Desktop Action ShowSettings]
Name=Open Pepper X settings
Exec=gdbus call --session --timeout 1 --dest com.obra.PepperX.Service --object-path /com/obra/PepperX --method com.obra.PepperX.ShowSettings

[Desktop Action ShowHistory]
Name=Open Pepper X history
Exec=gdbus call --session --timeout 1 --dest com.obra.PepperX.Service --object-path /com/obra/PepperX --method com.obra.PepperX.ShowHistory
```

Each `Exec=` is a single physical line (no line continuations). The freedesktop.org `.desktop` grammar requires it.

- [ ] **Step 3: Validate the file syntactically**

```sh
desktop-file-validate packaging/deb/pepper-x.desktop
```

Expected: empty output (= valid). If `desktop-file-validate` complains:
- "value 'Utility;AudioVideo;' for key 'Categories' contains a registered category 'AudioVideo' but registered categories are present" → ignore (false positive in some `desktop-file-utils` versions).
- "Required key 'Name' missing in '[Desktop Action ...]' group" → check that each Action block has a `Name=` line.
- Action name in `Actions=StartRecording;StopRecording;...` doesn't match a `[Desktop Action <name>]` block → check capitalization and spelling.

If validation reports a real syntax error, fix inline before proceeding.

---

## Task 4: Update `README.md` with the KDE Global Shortcut subsection

**Files:**
- Modify: `README.md` — add new subsection under `## Usage`.

- [ ] **Step 1: Find the insertion point**

```sh
grep -n "^### CLI\|^### Settings\|^## Architecture" README.md | head -5
```

Expected: line numbers for the existing subsections under `## Usage`. The new "Triggering dictation from a KDE Global Shortcut" subsection goes between `### CLI` and `## Architecture` (i.e. last subsection of Usage).

- [ ] **Step 2: Insert the new subsection**

After the last line of the `### CLI` subsection (the closing ``` of the CLI examples block) and before `## Architecture`, add:

```markdown
### Triggering dictation from a KDE Global Shortcut

Pepper X ships `.desktop` Actions for its main D-Bus methods. KDE's System Settings discovers them automatically once the desktop file is on the user's `XDG_DATA_DIRS` path. Per-user install (no sudo):

```sh
mkdir -p ~/.local/share/applications
install -m 644 packaging/deb/pepper-x.desktop ~/.local/share/applications/
kbuildsycoca6  # rebuild KDE service cache so the Actions appear immediately (or just relogin)
```

Prerequisites: `gdbus` (from `libglib2.0-bin` on Ubuntu — usually already present on KDE) and `kbuildsycoca6` (ships with `plasma-workspace`). Both are present on standard KDE Plasma 6 installs; if `gdbus` is missing, install with `sudo apt install libglib2.0-bin`.

Then in **System Settings → Shortcuts → Custom Shortcuts**:
1. Click **Add Custom Shortcut** → **Application**.
2. Browse to **Pepper X**. KDE shows the four available Actions:
   - Start dictation
   - Stop dictation
   - Open Pepper X settings
   - Open Pepper X history
3. Pick **Start dictation**, assign your preferred key combo (e.g. `Meta+V`), apply.

Pressing the bound shortcut now triggers dictation via the D-Bus service, independently of pepper-x's own evdev hotkey capture (the two trigger paths coexist; you can use either or both).

This trigger path requires:
- pepper-x running (the autostart `.desktop` from the install steps above ensures it). If you press the shortcut while pepper-x isn't running, `gdbus call` fails silently — there is no D-Bus service-activation file shipped (yet).
- D-Bus session bus available (any KDE Plasma session has this).

It does NOT bypass the `input` group requirement for keystroke insertion — the uinput helper still writes to `/dev/uinput`. If your dictation triggers but no text appears, see the udev / `input` group setup above.

**If the shortcut does nothing**: KDE Custom Shortcuts run their `Exec` without a controlling terminal, so any error from `gdbus call` is invisible. To debug, copy the `Exec=` line from `~/.local/share/applications/pepper-x.desktop` (look for the `[Desktop Action StartRecording]` block) and run it directly in a terminal. Common failures: pepper-x not running (`org.freedesktop.DBus.Error.ServiceUnknown`), wrong service name (typo), or a transient D-Bus issue.

**System-wide vs per-user install conflict**: if you also install pepper-x via the deb package (which writes `/usr/share/applications/pepper-x.desktop` system-wide), the per-user copy at `~/.local/share/applications/pepper-x.desktop` shadows it. After a deb upgrade, the per-user copy is NOT updated — you'll see the old Actions list. Either remove the per-user copy after deb installs, or stay per-user and skip the deb route.
```

- [ ] **Step 3: Verify the README still parses cleanly**

```sh
wc -l README.md
grep "^##\|^###" README.md
```

Expected: line count grew by ~30; new `### Triggering dictation from a KDE Global Shortcut` heading appears between `### CLI` and `## Architecture`.

---

## Task 5: Update `pepper-x-install.md` with the same content

**Files:**
- Modify: `pepper-x-install.md` — replace/extend the existing "Optional KDE polish" section.

- [ ] **Step 1: Find the existing section**

```sh
grep -n "^### Optional KDE polish\|^## " pepper-x-install.md | head -10
```

Expected: a `### Optional KDE polish` heading around line 103 (per W6 fact-check; may have drifted slightly). The current section has a brief `gdbus call` example.

- [ ] **Step 2: Replace the existing "Optional KDE polish" section**

Find the section starting with `### Optional KDE polish` and ending at the next `##` heading (`## Things to watch for during the build`). Replace its body with the same content as the README addition from Task 4 Step 2 (titled "Triggering dictation from a KDE Global Shortcut" — but the heading should remain `### Optional KDE polish` for backward consistency in this internal-research doc).

The replacement body (under the unchanged `### Optional KDE polish` heading):

```markdown
Pepper X ships `.desktop` Actions for its main D-Bus methods. KDE's System Settings discovers them automatically once the desktop file is on the user's `XDG_DATA_DIRS` path. Per-user install (no sudo):

```sh
mkdir -p ~/.local/share/applications
install -m 644 packaging/deb/pepper-x.desktop ~/.local/share/applications/
kbuildsycoca6  # rebuild KDE service cache so the Actions appear immediately (or just relogin)
```

Prerequisites: `gdbus` (from `libglib2.0-bin` on Ubuntu — usually already present on KDE) and `kbuildsycoca6` (ships with `plasma-workspace`).

Then in **System Settings → Shortcuts → Custom Shortcuts**:
1. Click **Add Custom Shortcut** → **Application**.
2. Browse to **Pepper X**. KDE shows the four available Actions: Start dictation, Stop dictation, Open Pepper X settings, Open Pepper X history.
3. Pick **Start dictation**, assign your preferred key combo (e.g. `Meta+V`), apply.

The KDE shortcut path is independent of pepper-x's own evdev hotkey capture; both trigger paths coexist. It does NOT bypass the `input` group requirement for keystroke insertion (the uinput helper still writes to `/dev/uinput`).

**If the shortcut does nothing**: KDE Custom Shortcuts run their `Exec` without a controlling terminal, so any error from `gdbus call` is invisible. Copy the `Exec=` line from `~/.local/share/applications/pepper-x.desktop` and run it directly in a terminal to surface the error.

**System-wide vs per-user install conflict**: if you also install via the deb package, the per-user copy shadows the system one and is NOT updated by `apt upgrade`. Pick one route or manually re-sync after upgrade.
```

(Keep the same content tone/length as the README addition, just dropping the "above" cross-references that don't exist in this document.)

- [ ] **Step 3: Verify**

```sh
grep -n "^### Optional KDE polish\|^## Things to watch" pepper-x-install.md
```

Expected: section heading at the same line number range; next `##` heading still present immediately after the new content.

---

## Task 6: Commit the deliverables (desktop file + docs)

**Files committed:**
- `packaging/deb/pepper-x.desktop`
- `README.md`
- `pepper-x-install.md`

- [ ] **Step 1: Inspect the staged diff**

```sh
git add packaging/deb/pepper-x.desktop README.md pepper-x-install.md
git status
git diff --cached --stat
```

Expected: three files staged; insertion-heavy diffs (no large deletions in the .desktop file beyond the small Comment + Categories changes).

- [ ] **Step 2: Commit**

```sh
git commit -m "$(cat <<'EOF'
W2: KDE Global Shortcut .desktop Actions and docs

Extend packaging/deb/pepper-x.desktop with Actions=StartRecording;
StopRecording;ShowSettings;ShowHistory; and four [Desktop Action ...]
blocks wrapping gdbus call invocations against the existing D-Bus
surface. Each Exec uses --timeout 1 so a hung pepper-x doesn't block
the shortcut caller for the default 25s. Categories=Utility;AudioVideo;
(was Utility;GNOME;) and a refreshed Comment= drop the GNOME-only
framing post-W4a.

README and pepper-x-install.md gain a "Triggering dictation from a KDE
Global Shortcut" section: per-user install path, kbuildsycoca6 cache
refresh, System Settings walkthrough, troubleshooting block (errors-
invisible-from-shortcut-context, per-user-vs-system install conflict,
input-group caveat).

No production-code changes. The D-Bus methods (StartRecording,
StopRecording, ShowSettings, ShowHistory) already exist; this just
exposes them as KDE-discoverable Actions.
EOF
)"
```

Expected: three files changed.

---

## Task 7: Add W10, Phase 4, and three new deferred-ideas entries (separate commit)

Per the architect-review's "scope creep" feedback, the W10 creation goes in a SEPARATE commit so reviewers can revert it independently of the main W2 deliverables.

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Add the W10 row to the workstream table**

Find the existing workstream table; insert a new row after W9's row (or wherever the table's row ordering rules dictate — currently the table is in roughly id-numerical-with-phase-grouping order). Add:

```
| W10 | 4 | KDE-native UX (tray icon + status pill + screenshot service + first-run shortcut wizard) | `pending` | — | — | — | maybe | **New 2026-05-02 from W2 brainstorm.** Comprises B + C + (d) deferred from W2 (tray icon via SNI/StatusNotifierItem, status pill via gtk4-layer-shell on KDE, KDE screenshot bridge equivalent of GNOME extension's `com.obra.PepperX.Screenshot`, first-run shortcut wizard). Trigger: review after all Phase 2 work is settled (W4b, W4c, W5, W7). May decompose into W10a/b/c/d when brainstormed. |
```

- [ ] **Step 2: Add the Phase 4 description**

Find the "## Phase descriptions" section. After the existing "Phase 3 — Architectural (conditional)" paragraph, add:

```markdown
**Phase 4 — Future polish.** KDE-native UX work to bring the experience to feature-parity with the GNOME extension. Single workstream (W10) for now; may decompose into W10a/b/c/d if it gets brainstormed and split. Off the critical path; review after all Phase 2 work has settled.
```

- [ ] **Step 3: Append three new deferred-ideas entries**

Find the "## Refactor / enhancement ideas (deferred)" section. After the last existing entry and before the "Future workstreams that surface deferred ideas append entries here..." closing line, add:

```markdown
- **D-Bus session `.service` activation file** — ship a `com.obra.PepperX.Service.service` file under `/usr/share/dbus-1/services/` (or `~/.local/share/dbus-1/services/` for per-user) that lets `gdbus call` auto-launch pepper-x if it isn't running. Fixes the "shortcut fires while pepper-x isn't running → silent failure" UX gap properly. Trigger: a user complains about the shortcut "doing nothing" when pepper-x has crashed/exited.
- **`ToggleRecording` D-Bus method or wrapper script** — smooth the single-toggle-key UX expectation. Either a new method that introspects state and dispatches Start/Stop, or a small shipped script (`packaging/kde/pepper-x-toggle.sh`) that does the same in userspace. Trigger: more than one user asks for "one key, not two" in feedback.
- **`desktop-file-validate` as a committed CI gate** — add `tests/smoke/test_desktop_file.sh` (or a `cargo test` integration test) that runs `desktop-file-validate` against `packaging/deb/pepper-x.desktop` on every CI run. Catches Action typos, mismatched bus names, syntax drift. Trigger: someone introduces a `.desktop` regression that ships before being caught manually.
```

- [ ] **Step 4: Append a status-log entry**

At the bottom under "## Status log":
```
- `2026-05-02` — Created W10 (KDE-native UX) at Phase 4; added Phase 4 description; appended three new deferred-ideas entries from W2 architect-review (D-Bus .service activation file; ToggleRecording method/script; desktop-file-validate CI gate). Separate commit per architect-review's "scope creep" feedback.
```

- [ ] **Step 5: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "$(cat <<'EOF'
Add W10 (KDE-native UX) workstream and 3 deferred-ideas

W10 captures the B + C + (d) options deferred from W2 brainstorm:
KDE tray icon (SNI/StatusNotifierItem), status pill via gtk4-layer-
shell on KDE, KDE screenshot-bridge equivalent of the GNOME extension's
Screenshot service, and a first-run shortcut wizard. Phase 4 — Future
polish, off the critical path; trigger is "review after all Phase 2
work is settled (W4b, W4c, W5, W7)."

Three new deferred-ideas entries surfaced by W2 architect-review:
D-Bus .service activation file (auto-launch pepper-x on shortcut
press); ToggleRecording method/script (single-toggle UX); and
desktop-file-validate as a committed CI gate.

Separate commit so reviewers can revert this independently of W2's
main deliverables.
EOF
)"
```

Expected: one file changed.

---

## Task 8: Install the desktop file per-user and refresh KDE service cache

This is autonomous (no sudo, no user interaction). Sets up the per-user copy that the smoke test will use.

- [ ] **Step 1: Pre-flight — make sure no stale per-user copy exists**

```sh
ls ~/.local/share/applications/pepper-x.desktop 2>&1 || echo "no existing per-user copy"
```

If a copy exists from a previous session, note its mtime (we'll overwrite it in Step 2).

- [ ] **Step 2: Install the per-user copy**

```sh
mkdir -p ~/.local/share/applications
install -m 644 packaging/deb/pepper-x.desktop ~/.local/share/applications/
ls -l ~/.local/share/applications/pepper-x.desktop
```

Expected: file installed with mode 644, same content as the worktree's edited version.

- [ ] **Step 3: Validate the installed file**

```sh
desktop-file-validate ~/.local/share/applications/pepper-x.desktop
```

Expected: empty output (= valid).

- [ ] **Step 4: Refresh KDE service cache**

```sh
kbuildsycoca6 2>&1 | tail -5
```

Expected: brief output (mostly informational); exit 0. KDE's KService registry now sees the four Actions.

If `kbuildsycoca6` is not found (`Command not found`), the system might be on Plasma 5 (`kbuildsycoca5`) or a stripped-down Plasma. Surface to user.

---

## Task 9: 🛑 User action required — bind shortcut and smoke test

The discriminator priority (per the spec): primary = text appears in Kate; secondary = stderr log; liveness = `pgrep`.

- [ ] **Step 1: Make sure pepper-x is running with stderr captured**

User runs (in a fresh terminal, NOT this Claude Code terminal):

```sh
pkill -f /usr/local/bin/pepper-x 2>/dev/null
pkill -f /usr/libexec/pepper-x/pepperx-uinput-helper 2>/dev/null
sleep 1
pepper-x 2>&1 | tee /tmp/w2-smoke.log
```

GTK window opens; capture file at `/tmp/w2-smoke.log` accumulates pepper-x's stderr.

- [ ] **Step 2: Verify all four Actions are visible to KDE**

User opens **System Settings → Shortcuts → Custom Shortcuts → Add Custom Shortcut → Application**, then browses for **Pepper X**.

User confirms KDE displays all four Actions in the picker:
- Start dictation
- Stop dictation
- Open Pepper X settings
- Open Pepper X history

If all four don't appear: run `kbuildsycoca6` again, log out and log back in, OR check whether `desktop-file-validate ~/.local/share/applications/pepper-x.desktop` still passes.

- [ ] **Step 3: Bind a shortcut to "Start dictation"**

User picks **Start dictation**, assigns `Meta+V` (or any non-conflicting combo — should NOT be Alt+Super since pepper-x's evdev capture already uses that), applies.

- [ ] **Step 4: Trigger via the bound shortcut (primary smoke)**

User opens Kate, focuses a document, presses `Meta+V` (or whatever combo was bound), speaks "hello world from KDE shortcut", presses `Meta+V` again to stop.

User confirms text appears at the cursor in Kate.

- [ ] **Step 5: Verify discriminators**

In another terminal, user runs:

```sh
grep -E '\[Pepper X uinput\]|\[Pepper X\] perf:|\[Pepper X\] modifier-only|shell-action|Access\(' /tmp/w2-smoke.log | tail -20
pgrep -af pepperx-uinput-helper
```

Expected:
- Stderr log shows a `[Pepper X] perf:` line for the cycle.
- Stderr log shows the `shell-action` trigger source somewhere (or absence of `modifier-only` for THIS cycle, since it came via D-Bus, not evdev).
- `pgrep` shows the helper is alive.

- [ ] **Step 6: Coexistence test — pepper-x's evdev hotkey still works**

User releases focus to Kate, holds **Alt+Super** (pepper-x's own default hotkey), speaks "this is via evdev", releases.

User confirms text appears at the cursor.

This proves the two trigger paths coexist — both fire `start_session`, neither blocks the other.

- [ ] **Step 7: StopRecording-while-idle observation**

User opens System Settings → Custom Shortcuts again, binds a SECOND shortcut (e.g. `Meta+Shift+V`) to **Stop dictation**.

User makes sure no recording is currently active, then presses `Meta+Shift+V`.

User observes:
- Best case: nothing happens (silent no-op) — `service.rs:163-177` swallows `DuplicateStop` errors.
- Possible: brief KDE notification toast about a D-Bus error.

Either way is acceptable. User reports observed behavior — it goes into the W2 status-log entry as observed-during-smoke (not a contract).

- [ ] **Step 8: Decision**

Steps 4-6 all pass → ✅ W2 is verified.
Step 4 fails → ❌ STOP. Per spec troubleshooting:
- Action doesn't appear in System Settings → recheck `desktop-file-validate`, run `kbuildsycoca6` again, consider relogin.
- Shortcut doesn't fire → copy the `Exec=` line from `~/.local/share/applications/pepper-x.desktop` and run in a terminal; surface what `gdbus call` actually says.
- Recording starts but no text → check pepper-x stderr for D-Bus or insertion errors; the W9 path should be intact.

Surface findings; invoke `superpowers:systematic-debugging` if the issue isn't a quick documentation fix.

---

## Task 10: Update roadmap — W2 done

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Update top status block**

Replace W2's in-progress block with:

```
current_workstream:   W4b — CI dependency-list fix
phase:                2
state:                pending  (W2 done; W4b awaits brainstorming)
branch:               (none yet — created when W4b spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

- [ ] **Step 2: Mark W2 row as done**

Change W2's State column from `in-progress` to `done`. Spec/Plan/Branch columns stay populated.

- [ ] **Step 3: Append status-log entry**

```
- `<today's date>` — W2 done. .desktop Actions added; KDE Custom Shortcut bound to "Start dictation" (Meta+V); manual smoke verified end-to-end on TuxedoOS 24.04 + KDE Plasma + Wayland: dictation triggered via the KDE shortcut produces text in Kate. Coexistence with pepper-x's evdev Alt+Super hotkey verified. StopRecording-while-idle observed-during-smoke: <fill in actual observed behavior — no-op silently OR brief D-Bus error toast>. State: `in-progress` → `done`. **current_workstream advances to W4b** (CI dependency-list fix).
```

(If smoke had unexpected results — e.g., needed extra `kbuildsycoca6` runs, focus-tracking hiccup, one Action mis-displayed in System Settings — reflect honestly; future re-orientation depends on it.)

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W2 done; advance current_workstream to W4b"
```

---

## Task 11: Merge `w2-kde-global-shortcut` to main

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: clean working tree, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w2-kde-global-shortcut -m "Merge W2: KDE Global Shortcut → D-Bus trigger

.desktop Actions for the four user-facing D-Bus methods (StartRecording,
StopRecording, ShowSettings, ShowHistory), each wrapping gdbus call
--timeout 1 against com.obra.PepperX.Service. KDE Custom Shortcuts UI
discovers them automatically. README + pepper-x-install.md gain the
binding workflow.

No production-code changes; the D-Bus surface already supported
everything. W2 verified end-to-end on TuxedoOS 24.04 + KDE Plasma 6:
shortcut bound to Meta+V triggers dictation; text appears in Kate;
coexistence with evdev Alt+Super hotkey confirmed.

W10 (KDE-native UX — tray icon, status pill, screenshot bridge,
first-run wizard) created at Phase 4 in a separate commit;
three new deferred-ideas entries surfaced by review.

current_workstream advances to W4b (CI dependency-list fix)."
```

Expected: merge commit with the changed files.

- [ ] **Step 3: Do NOT push**

Per established pattern across W1/W6/W9, work stays local until the user decides to push. Note in the conversation that there are now N commits ahead of `origin/main` (`git log --oneline origin/main..HEAD | wc -l`).

---

## Task 12: Cleanup — remove the worktree

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w2-kde-global-shortcut
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Switch back to main checkout, remove worktree, delete branch**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w2-kde-global-shortcut
git worktree list
git branch -d w2-kde-global-shortcut
git branch
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout (the `w6-atspi-kde-whitelist` branch may still exist as a non-worktree resumption point).
- `branch -d`: `Deleted branch w2-kde-global-shortcut (was <sha>).`
- `branch`: shows `* main` (with the W6 branch alongside).

If `git branch -d` refuses with "not fully merged", don't force with `-D`. Investigate.

---

## Done

When all 12 tasks are checked, W2 is complete:

- `packaging/deb/pepper-x.desktop` has four Actions wrapping gdbus call invocations.
- README + pepper-x-install.md have the KDE Global Shortcut binding section.
- KDE Custom Shortcut bound to "Start dictation" verified end-to-end on TuxedoOS 24.04 + KDE Plasma 6.
- Coexistence with pepper-x's evdev Alt+Super hotkey verified.
- W10 (KDE-native UX) created at Phase 4 with the deferred B+C+(d) options captured.
- Three new deferred-ideas entries (D-Bus .service activation, ToggleRecording, desktop-file-validate CI gate) added.
- Roadmap reflects W2=done, W4b=pending+next.

**Daily-driver UX broadened on KDE.** Users now have two trigger surfaces (evdev hotkey OR KDE Custom Shortcut), can pick whichever fits their workflow, and the underlying dictation pipeline (W9 universal uinput fallback) handles delivery in either case.

**Next session-start re-orientation will pick up W4b.** Per ways-of-working entry-point logic: `current_workstream: W4b, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

| Risk | Where addressed |
|---|---|
| KDE Plasma 5/6 differences in Custom Shortcuts UI | Plan covers Plasma 6; Plasma 5 documented as untested. |
| `gdbus` Exec format / shell escaping | Task 3 uses bare-token args (no `%` / `$` / quoting issues). |
| Shortcut fires while pepper-x isn't running → silent failure | Documented in README addition (Task 4 Step 2); `.service` activation file deferred to deferred-ideas. |
| `kbuildsycoca6` vs `update-desktop-database` confusion | Plan + README use `kbuildsycoca6` consistently. |
| User binds shortcut colliding with evdev Alt+Super | Task 9 Step 3 explicitly suggests Meta+V; documented in README. |
| Errors invisible from shortcut context | README troubleshooting block added (Task 4 Step 2). |
| Per-user vs system install conflict | README addition explicitly documents the conflict (Task 4 Step 2). |
| `StopRecording`-while-idle behavior | Task 9 Step 7 observes-and-records during smoke; not a contract. |
| Single-toggle UX expectation vs separate Start/Stop | Documented as known limitation; ToggleRecording deferred-ideas entry created (Task 7 Step 3). |
| `Categories=Utility;AudioVideo;` shifts pepper-x to "Multimedia" group | Cosmetic; documented in spec risks; plan doesn't roll back. |
