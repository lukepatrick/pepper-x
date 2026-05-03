# W4b — CI Dependency-List Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update `.github/workflows/ci.yml` with four missing apt packages (`clang`, `libclang-dev`, `libssl-dev`, `libpipewire-0.3-dev`), one removal (`libgtk4-layer-shell-dev`), `--locked` on cargo invocations, a new `cargo build --release` step, and four CI-hygiene additions (timeout, permissions, concurrency, CARGO_TERM_COLOR).

**Architecture:** Single-file edit + roadmap updates. Zero Rust source changes. Verification requires a real GitHub Actions run, so the smoke is a soft user-action checkpoint after pushing.

**Tech Stack:**
- GitHub Actions (`actions/checkout@v4`, `dtolnay/rust-toolchain@stable`)
- `apt-get` on `ubuntu-latest` runners
- `cargo` (fmt / clippy / test / build) with `--locked`
- YAML syntax validation: `python3 -c 'import yaml; yaml.safe_load(open(...))'` (no extra deps)

**Source spec:** `docs/superpowers/specs/2026-05-02-w4b-ci-dependency-fix-design.md`

---

## File Structure

This plan modifies/creates these files. All in-repo on the W4b branch.

**In-repo (committed on `w4b-ci-dependency-fix`):**
- Modify: `.github/workflows/ci.yml` — full replacement with the W4b content (apt list + cargo flags + hygiene additions).
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions for W4b; new W11 row at Phase 4; one new entry in "Refactor / enhancement ideas (deferred)"; status-log entries.

**No out-of-repo changes.** No system file edits, no per-user installs. The verification step requires GitHub Actions to actually run the new workflow on a push, which means the user must `git push` after merging — but the push itself is not part of W4b's deliverable.

---

## Task 1: Create W4b worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w4b-ci-dependency-fix/`
- (git) Create branch: `w4b-ci-dependency-fix` from `main`

- [ ] **Step 1: Verify clean main**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
git worktree list
```

Expected: clean working tree, on `main`. The W6 branch and any backup branches may exist as non-worktree branches — fine.

- [ ] **Step 2: Create the worktree**

```sh
git worktree add -b w4b-ci-dependency-fix ../pepper-x.w4b-ci-dependency-fix main
```

Expected: `Preparing worktree (new branch 'w4b-ci-dependency-fix')` then `HEAD is now at <sha>`.

- [ ] **Step 3: Verify**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w4b-ci-dependency-fix
git worktree list
git branch --show-current
```

Expected: two worktrees listed; current branch `w4b-ci-dependency-fix`.

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w4b-ci-dependency-fix/`.**

---

## Task 2: Update roadmap — W4b to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — top status block, W4b row, status log.

- [ ] **Step 1: Update top status block**

Replace:

```
current_workstream:   W4b — CI dependency-list fix
phase:                2
state:                pending  (W2 done; W4b awaits brainstorming)
branch:               (none yet — created when W4b spec is written)
worktree:             (none yet)
last_updated:         2026-05-02
```

With:

```
current_workstream:   W4b — CI dependency-list fix
phase:                2
state:                in-progress
branch:               w4b-ci-dependency-fix
worktree:             ../pepper-x.w4b-ci-dependency-fix
last_updated:         2026-05-02
```

- [ ] **Step 2: Update W4b row in workstream table**

Find W4b's row. Change `State` from `pending` to `in-progress`; populate `Branch` (`w4b-ci-dependency-fix`), `Spec` (`2026-05-02-w4b-ci-dependency-fix-design.md`), `Plan` (`2026-05-02-w4b-ci-dependency-fix.md`).

- [ ] **Step 3: Append status-log entry**

At the bottom under "## Status log":

```
- `2026-05-02` — W4b plan written and execution begun. State: `pending` → `in-progress`. Branch: `w4b-ci-dependency-fix`. Reviewer pass folded in four CI-hygiene additions (timeout-minutes, permissions: contents: read, concurrency cancel-in-progress, CARGO_TERM_COLOR=always) and surfaced one deferred-ideas entry (defensive apt listing for runner-rollforward protection).
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W4b in-progress"
```

Expected: one file changed.

---

## Task 3: Replace `.github/workflows/ci.yml`

**Files:**
- Modify: `.github/workflows/ci.yml` — full replacement.

- [ ] **Step 1: Read the existing file to confirm baseline**

```sh
cat .github/workflows/ci.yml
```

Expected: 45 lines matching the spec's "before" state — apt list with `libgtk4-layer-shell-dev` and missing `clang`/`libclang-dev`/`libssl-dev`/`libpipewire-0.3-dev`; no top-level `permissions:`/`concurrency:`/`env:`; no `--locked` on cargo invocations; no `cargo build --release` step.

- [ ] **Step 2: Replace the file**

Replace `.github/workflows/ci.yml` with exactly:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  CARGO_TERM_COLOR: always

jobs:
  ci:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            build-essential \
            clang \
            cmake \
            libadwaita-1-dev \
            libatspi2.0-dev \
            libclang-dev \
            libgirepository1.0-dev \
            libglib2.0-dev \
            libgtk-4-dev \
            libpipewire-0.3-dev \
            libssl-dev \
            libvulkan-dev \
            libxkbcommon-dev \
            pkg-config \
            tesseract-ocr

      - name: Set up Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - name: Check formatting
        run: cargo fmt --check

      - name: Lint with clippy
        run: cargo clippy --locked -- -D warnings

      - name: Run tests
        run: cargo test --locked --workspace

      - name: Build release
        run: cargo build --locked --release
```

Note the apt list is alphabetized for diff readability.

---

## Task 4: Verify YAML syntax

The file is now valid GitHub Actions YAML — but a syntax error here would silently break CI on next push. Verify with the tools that exist on the host (no extra installs).

- [ ] **Step 1: YAML parse check via Python**

```sh
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "YAML OK"
```

Expected: `YAML OK` and no exception. If `python3` is not available or `pyyaml` isn't installed, fall back to:

```sh
yq '.' .github/workflows/ci.yml > /dev/null && echo "yq OK"
```

If neither works, just visually inspect the file. The structure is shallow and YAML-friendly (no tab characters, consistent 2-space indentation).

- [ ] **Step 2: Sanity-check the action references and step count**

```sh
grep -E "^      - " .github/workflows/ci.yml
```

Expected: 7 step entries (checkout, install deps, set up Rust, fmt, clippy, test, build).

```sh
grep -E "^\s*uses:" .github/workflows/ci.yml
```

Expected: 2 action uses — `actions/checkout@v4` and `dtolnay/rust-toolchain@stable`.

- [ ] **Step 3: Confirm critical changes are present**

```sh
grep -E "libssl-dev|libpipewire-0.3-dev|libclang-dev|^\s*clang\s|--locked|cargo build --locked --release|timeout-minutes|cancel-in-progress|CARGO_TERM_COLOR|permissions:" .github/workflows/ci.yml
```

Expected: 11+ matches (each addition shows up at least once). If any expected substring is missing, re-check the file content from Task 3 Step 2.

```sh
grep -E "libgtk4-layer-shell-dev" .github/workflows/ci.yml
```

Expected: zero matches (dead reference removed).

---

## Task 5: Commit the ci.yml change

**Files committed:**
- `.github/workflows/ci.yml`

- [ ] **Step 1: Inspect the staged diff**

```sh
git add .github/workflows/ci.yml
git status
git diff --cached .github/workflows/ci.yml | head -120
```

Expected: only `.github/workflows/ci.yml` staged. Diff shows the full replacement as expected.

- [ ] **Step 2: Commit**

```sh
git commit -m "$(cat <<'EOF'
W4b: fix CI dependency list, add release-build verification, harden workflow

apt install list: add clang, libclang-dev, libssl-dev, libpipewire-0.3-dev
(all four are required by the build but were pre-installed on
ubuntu-latest runners by accident); remove libgtk4-layer-shell-dev
(dead reference — not used by any crate).

Cargo invocations now use --locked: enforces Cargo.lock consistency
and fails CI fast if dependencies would silently drift. New
"cargo build --locked --release" step exercises bindgen-driven sys
crates more aggressively than test/clippy do — would have caught the
W1 openssl-sys / libspa-sys missing-package failures pre-developer.

Workflow hardening (deployment-engineer review): timeout-minutes: 30
caps runaway builds (default 6h is dangerous when bindgen/llama-cpp
can hang); permissions: contents: read enforces least-privilege;
concurrency cancel-in-progress drops stale runs on rapid PR pushes;
CARGO_TERM_COLOR=always restores readable cargo output in run logs;
pull_request branch filter scopes the trigger to main.
EOF
)"
```

Expected: one file changed.

---

## Task 6: Add W11 (Phase 4 — CI performance + matrix expansion) and one new deferred-ideas entry

Per the W2 pattern, this goes in a separate commit so reviewers can revert independently.

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Add W11 row to the workstream table**

Find the existing W10 row (Phase 4). Insert a new row immediately after W10:

```
| W11 | 4 | CI performance + matrix expansion | `pending` | — | — | — | maybe | **New 2026-05-02 from W4b brainstorm.** Comprises C deferred from W4b: cache `target/` between runs (`Swatinem/rust-cache` or manual), split jobs (build/test/lint) for parallelism, matrix testing on multiple Ubuntu versions. Trigger: CI runtime becomes painful, OR a runner-image change breaks the build and we want defensive pre-installation. |
```

- [ ] **Step 2: Update Phase 4 description**

Find the existing "Phase 4 — Future polish" paragraph in the "## Phase descriptions" section. Update it to acknowledge W11:

OLD:
```
**Phase 4 — Future polish.** KDE-native UX work to bring the experience to feature-parity with the GNOME extension. Single workstream (W10) for now; may decompose into W10a/b/c/d if it gets brainstormed and split. Off the critical path; review after all Phase 2 work has settled.
```

NEW:
```
**Phase 4 — Future polish.** Off-critical-path improvements: KDE-native UX (W10) and CI performance + matrix expansion (W11). Both review after Phase 2 work has settled. Either may decompose into sub-workstreams when brainstormed.
```

- [ ] **Step 3: Append one new deferred-ideas entry**

Find the "## Refactor / enhancement ideas (deferred)" section. After the last existing entry and before the closing line ("Future workstreams that surface deferred ideas append entries here..."), add:

```markdown
- **Defensive apt listing for runner-rollforward protection** — explicitly add `libudev-dev` and `ca-certificates` to the CI apt list (currently pre-installed on `ubuntu-latest` but a future image change could break the build silently). Same trigger as W11's runner-image-change clause: when an image change causes a CI break, this is the small fix to apply alongside the bigger W11 caching/matrix work.
```

- [ ] **Step 4: Append a status-log entry**

At the bottom under "## Status log":

```
- `2026-05-02` — Created W11 (CI performance + matrix expansion) at Phase 4 alongside W10; updated Phase 4 description to cover both; appended one new deferred-ideas entry (defensive apt listing). Separate commit per architect-review's "scope creep" pattern so reviewers can revert independently of W4b's main ci.yml change.
```

- [ ] **Step 5: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "$(cat <<'EOF'
Add W11 (CI performance + matrix expansion) and 1 deferred-ideas entry

W11 captures the C option deferred from W4b brainstorm: cache target/
between runs (Swatinem/rust-cache), split jobs (build/test/lint) for
parallelism, matrix testing on multiple Ubuntu versions. Phase 4 —
Future polish, alongside W10 (KDE-native UX). Trigger: CI runtime
becomes painful, OR a runner-image change breaks the build.

One new deferred-ideas entry: defensive apt listing for libudev-dev /
ca-certificates (currently pre-installed on ubuntu-latest, but future
image change could break the build silently).

Separate commit per the W2 pattern so reviewers can revert this
independently of W4b's main ci.yml change.
EOF
)"
```

Expected: one file changed.

---

## Task 7: Merge `w4b-ci-dependency-fix` to main

This puts the W4b changes on the main branch, ready for the next push to verify CI.

- [ ] **Step 1: Switch to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: clean working tree, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

```sh
git merge --no-ff w4b-ci-dependency-fix -m "$(cat <<'EOF'
Merge W4b: CI dependency-list fix + lightweight hardening

apt list reconciled with W4a's README list (add clang, libclang-dev,
libssl-dev, libpipewire-0.3-dev; remove libgtk4-layer-shell-dev).
Cargo invocations now use --locked. New cargo build --release step
catches bindgen-driven sys-crate failures pre-developer.

Workflow hardening: timeout-minutes: 30, permissions: contents: read,
concurrency cancel-in-progress, CARGO_TERM_COLOR=always.

W11 (CI performance + matrix expansion) created at Phase 4 in a
separate commit; one new deferred-ideas entry (defensive apt
listing for libudev-dev / ca-certificates).

CI green verification deferred to the next push to origin/main.

current_workstream advances to W4c (Pin Rust toolchain version) on
the wrap-up commit after CI is verified green.
EOF
)"
```

Expected: merge commit listing the changed files.

- [ ] **Step 3: Do NOT push yet**

Per established pattern (W1 / W6 / W9 / W2): work stays local until the user decides to push. The push is the verification step — see Task 8.

---

## Task 8: Cleanup — remove the worktree

The branch is merged; the worktree no longer needs to exist. Branch stays for now (delete after CI is verified green).

- [ ] **Step 1: Verify worktree clean**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w4b-ci-dependency-fix
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Switch back to main checkout, remove worktree**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w4b-ci-dependency-fix
git worktree list
```

Expected:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout (existing W6 / backup branches still listed if present).

- [ ] **Step 3: Branch stays for now**

Don't delete `w4b-ci-dependency-fix` yet — keep it as a recovery point until the user has pushed and CI is verified green. The branch is fully merged into main, so `git branch -d w4b-ci-dependency-fix` would succeed; we just defer that to Task 10.

---

## Task 9: 🛑 User action required — push and verify CI run

This is the W4b smoke gate. **The user must push to origin (their preferred squash-or-merge workflow applies) and watch the GitHub Actions run on `origin/main`.**

- [ ] **Step 1: User pushes to origin**

User runs whichever of these matches their squash workflow (consistent with W1/W6/W9/W2 pattern):

Option A (preserve all commits as-is):
```sh
git push origin main
```

Option B (squash W4b's commits into a single workstream-shaped commit before pushing — matches the established pattern):
```sh
# (User's existing squash workflow — see memory: feedback_workstream_commit_style.md)
git branch backup-pre-w4b-squash HEAD
git reset --soft <pre-W4b-base>
git commit -m "W4b: <workstream-shaped condensed message>"
git push origin main
```

The autonomous portion of W4b makes no assumption about which the user picks.

- [ ] **Step 2: User watches the GitHub Actions run**

After push, the user goes to **github.com/lukepatrick/pepper-x → Actions → CI** (the most recent run on `main`).

Expected outcome:
- Run kicks off automatically (push to main triggers the workflow).
- All seven steps complete in green: checkout, install dependencies, Set up Rust, Check formatting, Lint with clippy, Run tests, Build release.
- Total wall-clock time inside `timeout-minutes: 30` (typically 10-20 minutes for a cold cargo build of this codebase).

- [ ] **Step 3: User reports outcome**

User reports back with one of:
- "CI green" — proceed to Task 10.
- "CI red — <step that failed and the relevant error message>" — STOP. Triage with `superpowers:systematic-debugging`. Likely failure modes:
  - **`fmt`/`clippy` red on upstream code**: this would mirror W1's W4c-territory drift (rustc version newer than upstream's CI ran on). Out of W4b scope to fix; could mean we need to defer the `--locked`/`--release` parts of W4b until W4c lands a `rust-toolchain.toml`.
  - **`cargo test --locked` red because Cargo.lock drift**: the fact-check during W4b brainstorm verified the lockfile was consistent at HEAD; if it's drifted between the brainstorm and the push, run `cargo update --workspace` locally and retry.
  - **`cargo build --release` red on a missing apt package**: surface the package name; we missed something in the apt list. The fix is to add it, commit on a follow-up branch, and push again.
  - **`timeout-minutes: 30` exceeded**: the cold build took longer than expected. Either bump the timeout or accept that caching (W11) needs to be promoted ahead of schedule.

If the failure is W4c-territory drift, the user may decide to:
- Mark W4b "done with caveats" similar to W1 (the deps-fix part landed; the `--locked` / `--release` / clippy parts await W4c), OR
- Revert the `--locked` flags and re-push, leaving W4b at "deps-fix only" and accepting CI doesn't gate `--locked` until W4c.

This is a runtime decision; the spec doesn't pre-commit to one path.

---

## Task 10: Update roadmap — W4b done

Only fires after the user reports CI green.

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`

- [ ] **Step 1: Update top status block**

Replace W4b's in-progress block with:

```
current_workstream:   W4c — Pin Rust toolchain version
phase:                2
state:                pending  (W4b done; W4c awaits brainstorming)
branch:               (none yet — created when W4c spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

- [ ] **Step 2: Mark W4b row as done**

Change W4b's State column from `in-progress` to `done`. Spec/Plan/Branch columns stay populated (Branch becomes `w4b-ci-dependency-fix (merged)` to match the W1/W6/W9 pattern).

- [ ] **Step 3: Append status-log entry**

```
- `<today's date>` — W4b done. ci.yml updated with the four missing apt packages, libgtk4-layer-shell-dev removed, --locked added to cargo invocations, new cargo build --release step, plus four CI-hygiene additions (timeout-minutes, permissions, concurrency, CARGO_TERM_COLOR). Verified green on a real GitHub Actions run on origin/main: <one-liner about the run, e.g. "all seven steps green; release build completed in <N> minutes; no apt-install warnings">. State: `in-progress` → `done`. **current_workstream advances to W4c** (Pin Rust toolchain version).
```

If CI was red and W4b shipped "done with caveats", reflect that honestly in the status log: which sub-deliverables landed, which were deferred, and to which workstream.

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W4b done; advance current_workstream to W4c"
```

- [ ] **Step 5: Delete the merged branch**

```sh
git branch -d w4b-ci-dependency-fix
git branch
```

Expected: `Deleted branch w4b-ci-dependency-fix (was <sha>).`

If `git branch -d` refuses with "not fully merged", don't force with `-D` — investigate. The branch was merged in Task 7; refusal here would suggest the user's squash workflow somehow dropped commits.

- [ ] **Step 6: Reminder to push the done-state commit**

If the user pushed in Task 9 with a squashed flow that combined the in-progress/main-change/W11/done-state commits into one, this Task 10 step is redundant — the done state already reached origin via the Task 9 push.

If the user pushed in Task 9 BEFORE marking done (the linear flow), this Task 10's commit is now ahead of origin/main again. They'll do another push at their convenience.

No autonomous push. Note in the conversation if there are now N commits ahead of origin/main after this task.

---

## Done

When all 10 tasks are checked, W4b is complete:

- `.github/workflows/ci.yml` updated with four apt adds, one apt remove, `--locked` on three cargo invocations, new `cargo build --release` step, four CI-hygiene additions.
- W11 (CI performance + matrix expansion) added at Phase 4; one new deferred-ideas entry (defensive apt listing).
- Branch merged to main; worktree removed; `w4b-ci-dependency-fix` branch deleted after CI verified green.
- Roadmap reflects W4b=done, W4c=pending+next.
- Real GitHub Actions CI run verified all seven steps green.

**Daily-driver CI hygiene improved.** Future contributors / future runner-image rolls protected against the four package omissions; release build verified on every PR; runs cancel cleanly on rapid pushes; timeout caps runaway builds.

**Next session-start re-orientation will pick up W4c.** Per ways-of-working entry-point logic: `current_workstream: W4c, state: pending` → "pick next pending Wn → invoke `superpowers:brainstorming`."

---

## Appendix: Risks acknowledged in the spec, surfaced in this plan

| Risk | Where addressed |
|---|---|
| `cargo build --release` ~2x runtime in dep graph compile | Documented as known cost; caching deferred to W11. Plan's `timeout-minutes: 30` provides headroom. |
| `--locked` causes CI failure when Cargo.lock drifts | Task 4 Step 3 verifies the changes are present; Task 9 Step 3 covers the lockfile-drift failure mode in triage. |
| `concurrency: cancel-in-progress: true` cancels in-flight runs | Documented as intended behavior. |
| `permissions: contents: read` breaks something needing write | Plan's verification (Task 9) catches this on the first run — if any step needed write, the run fails and we bump permissions. |
| GitHub `ubuntu-latest` rolls forward and changes pre-installed package set | Captured as a deferred-ideas entry in Task 6 Step 3. |
| ci.yml YAML syntax error | Task 4 Steps 1-3 verify before commit. |
| Upstream-code drift causes `--locked` / clippy / fmt to fail in W4c-territory | Task 9 Step 3 explicit triage option to mark W4b "done with caveats" similar to W1. |
