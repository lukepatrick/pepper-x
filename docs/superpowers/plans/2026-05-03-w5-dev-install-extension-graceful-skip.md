# W5 — `dev-install-extension.sh` Graceful Skip on Non-GNOME Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Change `scripts/dev-install-extension.sh:161-164` from `exit 1` to `exit 0` with a friendlier message when `gnome-extensions` is missing, and refresh seven stale doc references that promise the old "exits with error" behavior.

**Architecture:** Single-file behavior change (1 line + new wording) + 3-file doc sweep (CLAUDE.md, README.md, pepper-x-install.md × 5 sites). No code paths reorder; the change happens inside the existing `! command -v gnome-extensions` branch which is unreachable when the CLI is present. Manual KDE smoke is the only verification — this script isn't called from CI.

**Tech Stack:** bash; project markdown docs.

**Source spec:** `docs/superpowers/specs/2026-05-03-w5-dev-install-extension-graceful-skip-design.md`

---

## File Structure

This plan modifies these files. All in-repo on the W5 branch.

**In-repo (committed on `w5-dev-install-extension-graceful-skip`):**
- Modify: `scripts/dev-install-extension.sh` — replace lines 161-164 (the `exit 1` guard) with the exit-0 friendly-skip block.
- Modify: `CLAUDE.md` — refresh line 79's "exits with an error" note.
- Modify: `README.md` — refresh line 101's KDE warning.
- Modify: `pepper-x-install.md` — refresh five lines (30, 96, 204, 233, 283) that documented the old skip-this-step workaround.
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W5, status-log entries.

**No out-of-repo changes.** No CI changes (this script isn't invoked from CI). No new dependencies.

---

## Task 1: Create W5 worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w5-dev-install-extension-graceful-skip/`
- (git) Create branch: `w5-dev-install-extension-graceful-skip` from `main`

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
git worktree add -b w5-dev-install-extension-graceful-skip ../pepper-x.w5-dev-install-extension-graceful-skip main
```

Expected: `Preparing worktree (new branch 'w5-dev-install-extension-graceful-skip')` then `HEAD is now at <sha>`.

- [ ] **Step 3: Verify**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w5-dev-install-extension-graceful-skip
git worktree list
git branch --show-current
```

Expected: two worktrees listed; current branch `w5-dev-install-extension-graceful-skip`.

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w5-dev-install-extension-graceful-skip/`.**

---

## Task 2: Update roadmap — W5 to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W5 row, status log.

- [ ] **Step 1: Update top status block**

Find the existing status block at the top of the file (under "## Status"). Replace:

```
current_workstream:   W5 — dev-install-extension.sh graceful skip on non-GNOME
phase:                2
state:                pending  (W4c done; W5 awaits brainstorming)
branch:               (none yet — created when W5 spec is written)
worktree:             (none yet)
last_updated:         2026-05-03
```

With:

```
current_workstream:   W5 — dev-install-extension.sh graceful skip on non-GNOME
phase:                2
state:                in-progress
branch:               w5-dev-install-extension-graceful-skip
worktree:             ../pepper-x.w5-dev-install-extension-graceful-skip
last_updated:         2026-05-03
```

- [ ] **Step 2: Update W5 row in workstream table**

Find W5's row in the workstream table. Change `State` from `pending` to `in-progress`; populate `Branch` (`w5-dev-install-extension-graceful-skip`), `Spec` (`2026-05-03-w5-dev-install-extension-graceful-skip-design.md`), `Plan` (`2026-05-03-w5-dev-install-extension-graceful-skip.md`).

- [ ] **Step 3: Append status-log entry**

Under "## Status log", append at the bottom:

```
- `2026-05-03` — W5 plan written and execution begun. State: `pending` → `in-progress`. Branch: `w5-dev-install-extension-graceful-skip`. Reviewer pass surfaced two more stale `pepper-x-install.md` references the original brainstorm missed (`:30`, `:283`); doc-cleanup count updated 3 → 7 sites total. Reviewer also surfaced the `python3` dependency in `check_extension` (`:19-103`) — script still exits 1 on barebones non-GNOME boxes without python3 because `check_extension` runs before the new graceful-skip; accepted (python3 ~always present on Linux desktops; restructuring would break `--check` mode's deliberate "validate extension code on any platform" semantics). Message wording adjusted from "Custom Shortcut bound to com.obra.PepperX.Service" to "see README → 'Triggering dictation from a KDE Global Shortcut'" so the script doesn't drift if W2's recipe evolves.
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W5 in-progress"
```

---

## Task 3: Edit `scripts/dev-install-extension.sh`

**Files:**
- Modify: `scripts/dev-install-extension.sh:161-164`

- [ ] **Step 1: Read the current state of the file**

```sh
sed -n '155,170p' scripts/dev-install-extension.sh
```

Expected (around lines 157-164):

```bash
if [[ "${1:-}" == "--check" ]]; then
    exit 0
fi

if ! command -v gnome-extensions >/dev/null 2>&1; then
    echo "gnome-extensions is required to install the Pepper X extension" >&2
    exit 1
fi
```

- [ ] **Step 2: Apply the edit**

Replace the four-line block at lines 161-164 with the new exit-0 friendly-skip block. The exact replacement:

```bash
# Before:
if ! command -v gnome-extensions >/dev/null 2>&1; then
    echo "gnome-extensions is required to install the Pepper X extension" >&2
    exit 1
fi

# After:
if ! command -v gnome-extensions >/dev/null 2>&1; then
    echo "GNOME Shell extension tooling not detected — skipping extension install." >&2
    echo "On KDE, see README → 'Triggering dictation from a KDE Global Shortcut' for the .desktop-Action setup." >&2
    exit 0
fi
```

The `--check` branch at lines 157-159 stays untouched; `check_extension` at line 103 stays untouched.

- [ ] **Step 3: Verify the diff is narrow**

```sh
git diff scripts/dev-install-extension.sh
```

Expected: 4 deletions + 5 insertions, all inside one contiguous block (the `! command -v gnome-extensions` branch). No other lines touched.

- [ ] **Step 4: Sanity-check the script still parses**

```sh
bash -n scripts/dev-install-extension.sh && echo "syntax OK"
```

Expected: `syntax OK` (exit 0). `bash -n` is a no-execute syntax check.

- [ ] **Step 5: Commit**

```sh
git add scripts/dev-install-extension.sh
git commit -m "W5: dev-install-extension.sh exits 0 with friendly skip on non-GNOME"
```

---

## Task 4: Doc cleanup — CLAUDE.md + README.md + pepper-x-install.md (7 sites)

**Files:**
- Modify: `CLAUDE.md:79`
- Modify: `README.md:101`
- Modify: `pepper-x-install.md` lines 30, 96, 204, 233, 283

- [ ] **Step 1: Read each file's current state at the relevant lines**

```sh
echo "=== CLAUDE.md:79 ==="
sed -n '75,85p' CLAUDE.md
echo "=== README.md:101 ==="
sed -n '95,110p' README.md
echo "=== pepper-x-install.md:30 ==="
sed -n '27,33p' pepper-x-install.md
echo "=== pepper-x-install.md:96 ==="
sed -n '93,99p' pepper-x-install.md
echo "=== pepper-x-install.md:204 ==="
sed -n '201,207p' pepper-x-install.md
echo "=== pepper-x-install.md:233 ==="
sed -n '230,237p' pepper-x-install.md
echo "=== pepper-x-install.md:283 ==="
sed -n '280,286p' pepper-x-install.md
```

This dumps each context window to confirm the lines match what the spec described. Line numbers may have drifted slightly if the file was edited since the spec was written — use grep to locate by content if needed.

- [ ] **Step 2: Update `CLAUDE.md:79`**

Find the line containing *"the script exits with an error if `gnome-extensions` isn't on PATH"* and replace that phrase with *"the script exits cleanly with a skip message on non-GNOME (W5 fix)"*.

The full sentence in `CLAUDE.md:79` currently reads:

> `gnome-extension/` and `scripts/dev-install-extension.sh` are hard-bound to `gnome-shell` 48+ — **don't run the install script on KDE**, the script exits with an error if `gnome-extensions` isn't on PATH.

Update to:

> `gnome-extension/` and `scripts/dev-install-extension.sh` are hard-bound to `gnome-shell` 48+ — running the install script on KDE is harmless; it exits cleanly with a skip message on non-GNOME (W5 fix).

- [ ] **Step 3: Update `README.md:101`**

Find the line beginning with *"On KDE Plasma or any non-GNOME Wayland desktop"* and the *"will exit with an error"* clause inside it.

Current text (around line 101):

> **On KDE Plasma or any non-GNOME Wayland desktop**: skip the GNOME extension (`scripts/dev-install-extension.sh` will exit with an error if `gnome-extensions` isn't on PATH). Create an autostart entry instead so the D-Bus service is available at session start:

Update to:

> **On KDE Plasma or any non-GNOME Wayland desktop**: the GNOME extension is unused (`scripts/dev-install-extension.sh` is safe to run — it skips cleanly with an informational message). Create an autostart entry instead so the D-Bus service is available at session start:

- [ ] **Step 4: Update `pepper-x-install.md:30`**

Find the row in `pepper-x-install.md` describing the `dev-install-extension.sh` step. The current "Skip" entry says something like *"Hard requirement on `gnome-shell` 48+. **Skip the install step entirely.**"*.

Update to:

> Auto-skipped on non-GNOME after W5; running the script on KDE is a no-op.

- [ ] **Step 5: Update `pepper-x-install.md:96`**

Find the checklist line that currently says (paraphrasing):

> - [ ] **Skip** `bash scripts/dev-install-extension.sh` — it requires `gnome-extensions` and exits with an error on KDE. Confirmed via `scripts/dev-install-extension.sh:161-164` (the `command -v gnome-extensions` guard).

Update to:

> - [ ] `bash scripts/dev-install-extension.sh` — on KDE the script auto-skips after W5 with an informational message; running it is harmless.

(Note: the leading checkbox stays; the "Skip" semantics are no longer needed.)

- [ ] **Step 6: Update `pepper-x-install.md:204`**

Find the bullet currently saying:

> - `scripts/dev-install-extension.sh` lines 90-93 — exits with error if `gnome-extensions` not on PATH. Just don't run this script.

Update to:

> - `scripts/dev-install-extension.sh:161-164` — auto-skips on non-GNOME after W5 (exits 0 with informational message).

- [ ] **Step 7: Update `pepper-x-install.md:233`**

Find the line describing the W5 design proposal — currently something like:

> 2. **`scripts/dev-install-extension.sh`**: detect non-GNOME early and print a friendly "skipping — no GNOME Shell detected, app will run without tray icon" message instead of an error. Real check is at `:161-164`, not `:90-93`.

Update to:

> 2. **`scripts/dev-install-extension.sh`**: ✅ done in W5 (2026-05-03). Script auto-skips on non-GNOME with an exit-0 informational message. Edited block at `:161-164`.

- [ ] **Step 8: Update `pepper-x-install.md:283`**

Find the line currently saying (paraphrasing):

> Install binaries; **skip** the GNOME extension script.

Update to:

> Install binaries; the GNOME extension script auto-skips on non-GNOME (W5).

- [ ] **Step 9: Verify all seven edits**

```sh
git diff --stat
```

Expected: 3 files changed (`CLAUDE.md`, `README.md`, `pepper-x-install.md`); insertions + deletions in single digits per file.

```sh
git diff -- CLAUDE.md README.md pepper-x-install.md
```

Verify all seven edits look right; no unintended whitespace or unrelated changes.

- [ ] **Step 10: Confirm no remaining stale references**

```sh
grep -rn "exit.*1\|will exit with an error\|gnome-extensions is required" CLAUDE.md README.md pepper-x-install.md gnome-extension/README.md 2>&1 | grep -i "gnome-ext\|dev-install"
```

Expected: no output, or only references that legitimately describe historical behavior (e.g. "before W5, the script used to exit 1"). If new stale references surface, decide whether to fold in or defer.

- [ ] **Step 11: Commit**

```sh
git add CLAUDE.md README.md pepper-x-install.md
git commit -m "W5: refresh doc references to graceful-skip behavior"
```

---

## Task 5: 🛑 USER ACTION — manual KDE smoke

**Files:** none modified; pure verification.

The script doesn't run in CI; manual smoke on the dev box (TuxedoOS 24.04 + KDE Plasma) is the only verification.

- [ ] **Step 1: User runs the no-flag form**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w5-dev-install-extension-graceful-skip
bash scripts/dev-install-extension.sh
echo "exit=$?"
```

Expected:
- Two-line message printed to stderr:
  ```
  GNOME Shell extension tooling not detected — skipping extension install.
  On KDE, see README → 'Triggering dictation from a KDE Global Shortcut' for the .desktop-Action setup.
  ```
- `exit=0`

- [ ] **Step 2: User confirms no files were installed**

```sh
ls ~/.local/share/gnome-shell/extensions/pepperx@obra/ 2>&1
```

Expected: `ls: cannot access '...': No such file or directory` (the directory should not exist; the script's mkdir + cp at lines 166-172 should have been bypassed by the early exit).

If the directory exists and contains files (e.g. from a previous accidental run), the test is inconclusive — clean up and re-run, or just note that the smoke is verifying behavior of THIS run, not historical state.

- [ ] **Step 3: User runs `--check` mode**

```sh
bash scripts/dev-install-extension.sh --check
echo "exit=$?"
```

Expected: `exit=0` and no output (or only `check_extension`'s success — `check_extension` is silent on success). Behavior unchanged from before W5.

- [ ] **Step 4: User runs the existing wrapper test**

```sh
bash scripts/verify-extension-install.sh 2>&1 | tail -5
echo "exit=$?"
```

Expected: `exit=0`. This wrapper invokes the script with `--check` (per `verify-extension-install.sh:19`), so it short-circuits at line 157-159 before reaching the W5-edited block.

- [ ] **Step 5: User reports back**

User reports one of:
- **"All 4 smoke checks pass"** → proceed to Task 6.
- **"Smoke <N> failed: <output>"** → triage; common causes:
  - Step 1 prints the OLD message ("gnome-extensions is required...") → the script edit didn't apply or the wrong file is being run; check `git log --oneline` and `git diff main HEAD`.
  - Step 1 exits 1 → the edit is incomplete or syntax-broken; re-read `bash -n` output.
  - Step 2 shows files exist → the early-exit isn't actually short-circuiting the install; verify the `exit 0` line and that there's no fall-through.
  - Step 3 / Step 4 fail → unrelated to W5 (these paths are untouched); investigate independently.

---

## Task 6: Merge `w5-dev-install-extension-graceful-skip` to main

After user confirms smoke green.

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: clean working tree, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w5-dev-install-extension-graceful-skip -m "$(cat <<'EOF'
Merge W5: dev-install-extension.sh graceful skip on non-GNOME

Replace the exit-1 guard at scripts/dev-install-extension.sh:161-164
with an exit-0 friendly-skip block. On non-GNOME desktops (KDE,
Sway, headless CI), the script now prints a two-line informational
message and exits cleanly. Refreshes seven stale doc references
(CLAUDE.md:79, README.md:101, pepper-x-install.md lines 30/96/204/
233/283) that documented the old "skip this step or it errors" workaround.

check_extension (the static-validation block at :19-103) keeps running
on all platforms — useful as a sanity gate, and required by --check
mode (which deliberately bypasses the GNOME-tooling guard for
pre-deployment validation use cases).

current_workstream advances to W7 on the wrap-up commit.
EOF
)"
```

Expected: merge commit with the changed files.

- [ ] **Step 3: Do NOT push yet**

User pushes when ready (consistent with established workflow).

---

## Task 7: Cleanup — remove the worktree, mark W5 done, advance to W7

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w5-dev-install-extension-graceful-skip
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Switch back to main, remove worktree**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w5-dev-install-extension-graceful-skip
git worktree list
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout (W6 + backup branches still listed if present).

- [ ] **Step 3: Update roadmap top status block**

Edit `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`. Replace the W5 in-progress block with:

```
current_workstream:   W7 — OCR portal-response fix + rewire
phase:                2
state:                pending  (W5 done; W7 awaits brainstorming)
branch:               (none yet — created when W7 spec is written)
worktree:             (none yet)
last_updated:         2026-05-03
```

- [ ] **Step 4: Mark W5 row as done**

In the workstream table, change W5's State column from `in-progress` to `done`. Append to the Notes column: "**Done 2026-05-03**: script exits 0 with informational message on non-GNOME; seven stale doc references refreshed; manual KDE smoke pass."

- [ ] **Step 5: Append a status-log entry**

Append to "## Status log":

```
- `2026-05-03` — **W5 done.** `scripts/dev-install-extension.sh:161-164` now exits 0 with a two-line informational message on non-GNOME instead of `exit 1`. Seven stale doc references refreshed (`CLAUDE.md:79`, `README.md:101`, `pepper-x-install.md` lines 30/96/204/233/283). Manual KDE smoke confirmed: no-flag form exits 0 with new message and no files installed in `~/.local/share/gnome-shell/extensions/pepperx@obra/`; `--check` and `verify-extension-install.sh` paths unchanged. The W1-era doc-fix loop closes here — install.md's line citation `:90-93` → `:161-164` correction made during W1 was the documentation prerequisite for W5; now both the doc and the script behavior agree. State: `in-progress` → `done`. **current_workstream advances to W7** (OCR portal-response fix + rewire).
```

- [ ] **Step 6: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W5 done; advance current_workstream to W7"
```

- [ ] **Step 7: Delete the merged branch**

```sh
git branch -d w5-dev-install-extension-graceful-skip
git branch
```

If `git branch -d` refuses with "not fully merged" (likely if the user squashed at push time), force-delete with `-D` after verifying content equivalence:

```sh
git diff w5-dev-install-extension-graceful-skip <squashed-commit-sha> --stat
# Should be empty diff. Then:
git branch -D w5-dev-install-extension-graceful-skip
```

---

## Task 8: 🛑 USER ACTION — push to origin

- [ ] **Step 1: User pushes**

User runs whichever matches their squash workflow (no CI gate to verify, but pushing keeps origin/main current):

Option A (preserve all commits as-is):
```sh
git push origin main
```

Option B (squash W5's commits into a single workstream-shaped commit before pushing — matches W1/W2/W4b/W4c pattern):
```sh
git branch backup-pre-w5-squash HEAD
git reset --soft <pre-W5-commit-sha>
git commit -m "W5: dev-install-extension.sh graceful skip on non-GNOME"
git commit -m "Mark W5 done; advance current_workstream to W7"   # if state-transition stays separate
git push --force-with-lease origin main
```

The autonomous portion of W5 makes no assumption about which the user picks.

This script doesn't run in CI, so there's no post-push verification step. Once pushed, W5 is fully complete.

---

## Done

When all 8 tasks are checked, W5 is complete:

- `scripts/dev-install-extension.sh:161-164` exits 0 with a two-line informational message on non-GNOME.
- Seven stale doc references refreshed across `CLAUDE.md`, `README.md`, and `pepper-x-install.md`.
- Manual KDE smoke confirmed: no-flag form exits 0, no files installed; `--check` and `verify-extension-install.sh` paths unchanged.
- Roadmap reflects W5=done, W7=pending+next.

**Next session-start re-orientation will pick up W7.** Per ways-of-working entry-point logic: `current_workstream: W7, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

| Risk | Where addressed |
|---|---|
| `check_extension` calls `python3` and runs unconditionally before the new graceful-skip; barebones non-GNOME boxes without python3 still exit 1 | Spec accepts as-is (python3 ~always present on Linux desktops); plan does not restructure. |
| Genuinely-broken GNOME install (CLI removed, Shell present) now silently skips | Spec accepts as the trade-off of Approach A; plan does not add diagnostic for this edge case. |
| `set -euo pipefail` interaction with the new branch | Spec confirms the `if !` swallows the failure exit cleanly; plan's `bash -n` syntax check (Task 3 Step 4) is the verification. |
| Doc-update sweep misses a stale reference | Task 4 Step 10's grep audit catches this; if found, executor decides whether to fold in or defer. |
| Smoke test inconclusive if `~/.local/share/gnome-shell/extensions/pepperx@obra/` already exists from prior run | Task 5 Step 2 acknowledges; user can clean up or note that the test verifies THIS run, not historical state. |
