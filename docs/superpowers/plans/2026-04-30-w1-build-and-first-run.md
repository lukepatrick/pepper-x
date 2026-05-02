# W1 — Build + First-Run on TuxedoOS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Get `pepper-x` building from source on TuxedoOS 24.04 (Ubuntu noble base) + KDE Plasma + Wayland and producing dictated text into focused KDE apps (Kate, Konsole) end-to-end.

**Architecture:** Three-process design at runtime — main `pepper-x` GUI app (GTK4/libadwaita) plus two persistent helper subprocesses (`pepperx-uinput-helper` for keystroke injection, `pepperx-cleanup-helper` for llama.cpp Qwen inference). Helpers exist for dynamic-linker isolation between ONNX Runtime (ASR) and llama.cpp (cleanup). The KDE viability work in W1 is mostly **system setup** (apt deps, rustup, group/udev, helper install, autostart `.desktop`) — almost no source-code changes.

**Tech Stack:**
- Build: rustup + Rust ≥ 1.92, `cargo`, apt (`build-essential`, `cmake`, `clang`, `libclang-dev`, `libgtk-4-dev`, `libadwaita-1-dev`, `libatspi2.0-dev`, `libxkbcommon-dev`, `libvulkan-dev`, `tesseract-ocr`)
- Runtime: kernel `/dev/uinput`, `input` group, udev rules, D-Bus session bus (`com.obra.PepperX.Service`), KDE Plasma autostart via `~/.config/autostart/*.desktop`
- Target host: TuxedoOS 24.04 (Ubuntu noble), KDE Plasma, Wayland session

**Source spec:** `docs/superpowers/specs/2026-04-30-w1-build-and-first-run-design.md`

---

## File Structure

This plan modifies/creates these files. Most "creates" are outside the repo (system files); only the documentation and roadmap edits live in git.

**In-repo (committed on the `w1-build-and-first-run` branch):**
- Modify: `pepper-x-install.md` — line-citation fixes + autostart step + smoke-test note (Stage F of spec).
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — workstream state transitions (`spec` → `planned` → `in-progress` → `done`) and status-log entries.

**Out-of-repo (system, written by sudo or in user's HOME):**
- Create: `/etc/udev/rules.d/99-pepper-x-uinput.rules` (sudo)
- Create: `/etc/udev/rules.d/99-pepper-x-keyboard.rules` (sudo)
- Modify: `/etc/group` via `usermod -aG input` (sudo)
- Create: `/usr/local/bin/pepper-x` via `sudo install` (sudo)
- Create: `/usr/libexec/pepper-x/pepperx-uinput-helper` (sudo)
- Create: `/usr/libexec/pepper-x/pepperx-cleanup-helper` (sudo)
- Create: `~/.config/autostart/pepper-x.desktop`
- Created by `rustup` installer: `~/.cargo/`, `~/.rustup/`

**Worktree note:** W1 is mostly system setup, not code. We still create the `w1-build-and-first-run` branch and worktree per the standard rhythm — it serves as the commit container for the in-repo doc edits and keeps `main` clean during the build. No actual source-code isolation is needed for this workstream.

---

## Task 1: Create W1 worktree and branch

**Files:**
- (filesystem) Create worktree at `/home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run/`
- (git) Create branch: `w1-build-and-first-run` from `main`

- [ ] **Step 1: Verify clean main**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected output:
- `git status`: shows untracked `pepper-x-install.md` and `CLAUDE.md` only (no staged or modified tracked files); on branch `main`.
- `git branch --show-current`: `main`

- [ ] **Step 2: Create worktree**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree add -b w1-build-and-first-run ../pepper-x.w1-build-and-first-run main
```

Expected output:
```
Preparing worktree (new branch 'w1-build-and-first-run')
HEAD is now at <sha> Add TuxedoOS/KDE viability roadmap and W1 spec
```

- [ ] **Step 3: Verify worktree**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
git worktree list
git branch --show-current
```

Expected output:
- `git worktree list`: two entries — main checkout at `pepper-x` on `main`, and the new worktree at `pepper-x.w1-build-and-first-run` on `w1-build-and-first-run`.
- `git branch --show-current`: `w1-build-and-first-run`

**From here on, all repo-relative commands run in `/home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run/` unless explicitly noted.**

---

## Task 2: Update roadmap — W1 state to in-progress

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — change `state:` block and W1 row in status table; add status-log entry.

- [ ] **Step 1: Update the `state:` block at top**

In `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`, replace the status block:

OLD:
```
current_workstream:   W1 — Build + first-run on TuxedoOS
phase:                1
state:                spec  (spec written; plan not yet created)
branch:               (none yet — created when plan starts)
worktree:             (none yet)
last_updated:         2026-04-30
```

NEW:
```
current_workstream:   W1 — Build + first-run on TuxedoOS
phase:                1
state:                in-progress
branch:               w1-build-and-first-run
worktree:             ../pepper-x.w1-build-and-first-run
last_updated:         2026-04-30
```

- [ ] **Step 2: Update W1 row in workstream table**

In the same file, in the workstream table, change the W1 row's `State` column from `spec` to `in-progress` and the `Branch` column from `—` to `w1-build-and-first-run` and the `Plan` column from `—` to `2026-04-30-w1-build-and-first-run.md`.

- [ ] **Step 3: Append status-log entry**

At the very bottom of the file (under "## Status log"), add a new bullet:
```
- `2026-04-30` — W1 plan written and execution begun. State: `planned` → `in-progress`. Branch: `w1-build-and-first-run`.
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W1 in-progress in roadmap"
```

Expected output: one file changed, ~6 insertions, ~3 deletions.

---

## Task 3: Pre-flight system inspection

Run a baseline so we know what state the machine is in before we touch anything. Each item below is expected to FAIL (or report "missing"). This is the "write the failing verification" step.

- [ ] **Step 1: Inspect group, rustup, helpers, autostart**

Run:
```sh
groups
which rustc cargo 2>&1 || true
ls /dev/uinput 2>&1 || true
ls /etc/udev/rules.d/99-pepper-x-* 2>&1 || true
ls /usr/local/bin/pepper-x /usr/libexec/pepper-x/ 2>&1 || true
ls ~/.config/autostart/pepper-x.desktop 2>&1 || true
```

Expected output (what we expect to see PRE-install, per `pepper-x-install.md` "System snapshot"):
- `groups`: does NOT contain `input`.
- `which rustc cargo`: not found, OR `cargo` resolves to `/usr/bin/cargo` (apt's, version 1.75 — too old).
- `ls /dev/uinput`: shows the device exists (kernel module loaded) but with default group/perms.
- `ls /etc/udev/rules.d/99-pepper-x-*`: "No such file or directory".
- `ls /usr/local/bin/pepper-x ...`: "No such file or directory".
- `ls ~/.config/autostart/pepper-x.desktop`: "No such file or directory".

Record any deviations from this baseline — they may indicate a partial prior install that affects later tasks.

---

## Task 4: Install apt build dependencies

**Files:** none in repo. Modifies system package state via `apt`.

- [ ] **Step 1: Install the corrected dependency list**

Run (will prompt for sudo password):
```sh
sudo apt update
sudo apt install -y \
  build-essential cmake \
  libadwaita-1-dev libatspi2.0-dev libgirepository1.0-dev \
  libglib2.0-dev libgtk-4-dev \
  libvulkan-dev libxkbcommon-dev \
  pkg-config tesseract-ocr \
  clang libclang-dev git
```

**Notable omissions vs. upstream README:**
- No `cargo` (apt's is 1.75; we'll use rustup).
- No `libgtk4-layer-shell-dev` (not in noble repos and absent from `Cargo.toml` / `Cargo.lock`).

**Notable additions vs. upstream README:**
- `clang` and `libclang-dev` — needed by `llama-cpp-sys-4`'s bindgen step.

- [ ] **Step 2: Verify all packages installed**

Run:
```sh
dpkg -l build-essential cmake libadwaita-1-dev libatspi2.0-dev libgirepository1.0-dev libglib2.0-dev libgtk-4-dev libvulkan-dev libxkbcommon-dev pkg-config tesseract-ocr clang libclang-dev git 2>&1 | awk '/^ii/{c++} END{print c}'
```

Expected output: `14` (the count of installed packages, matching the count in the install command).

If any package is missing, the line count will be less than 14. Re-run the install command and check for apt errors.

- [ ] **Step 3: Verify clang specifically (the latent bug)**

Run:
```sh
clang --version
ls /usr/lib/llvm-*/lib/libclang.so* 2>&1
```

Expected output:
- `clang --version`: prints a clang version (likely 18 or 19 on noble).
- `ls`: shows at least one `libclang.so*` file.

If clang is missing, `cargo build` will later fail at the `llama-cpp-sys-4` bindgen step.

---

## Task 5: Install rustup + stable toolchain

**Files:** none in repo. Creates `~/.cargo/`, `~/.rustup/`, `~/.cargo/bin/`.

- [ ] **Step 1: Pre-flight — confirm rustup not yet installed**

Run:
```sh
ls ~/.cargo/bin/rustup 2>&1 || echo "no rustup"
```

Expected output: `no rustup` OR "No such file or directory".

If rustup IS already installed, skip Step 2 and run `rustup default stable && rustup update stable` instead, then jump to Step 3.

- [ ] **Step 2: Install rustup**

Run:
```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
```

Expected output: ends with `Rust is installed now. Great!` and instructions to source `~/.cargo/env`.

- [ ] **Step 3: Source cargo env into the current shell**

Run:
```sh
source $HOME/.cargo/env
```

No output expected. (Add this line to `~/.bashrc` or `~/.zshrc` if not already present, so future shells get it automatically.)

- [ ] **Step 4: Verify rustc and cargo come from rustup**

Run:
```sh
which rustc cargo
rustc --version
cargo --version
```

Expected output:
- `which rustc cargo`: both should resolve to under `~/.cargo/bin/` (NOT `/usr/bin/cargo`).
- `rustc --version`: `rustc 1.92.x` or higher (1.92 is the floor; gtk4 0.11 requires it).
- `cargo --version`: matching cargo version.

If `which cargo` returns `/usr/bin/cargo`, the apt cargo is shadowing rustup. Either remove apt cargo (`sudo apt remove cargo`) or ensure `~/.cargo/bin` is earlier in `$PATH` than `/usr/bin`.

---

## Task 6: Add user to input group

**Files:** modifies `/etc/group` via `usermod`.

- [ ] **Step 1: Pre-flight verification**

Run:
```sh
groups | grep -q input && echo "already in input" || echo "not in input"
```

Expected output: `not in input` (per Task 3 baseline).

- [ ] **Step 2: Add user to input group**

Run:
```sh
sudo usermod -aG input $USER
```

No output expected on success.

- [ ] **Step 3: Confirm /etc/group updated**

Run:
```sh
getent group input
```

Expected output: `input:x:NNN:luke` (or whatever your username is) — your username appears in the comma-separated members list.

**IMPORTANT:** This change does NOT take effect in the current login session. Continue to Task 7 for the mandatory logout/login.

---

## Task 7: 🛑 STOP — User Action Required: Logout / Login

This is a hard sync point. Group changes don't apply to existing sessions; you must log out fully and log back in.

- [ ] **Step 1: User logs out of KDE Plasma session**

Use the KDE menu → Power → Log Out. A reboot also works but is overkill. Closing the terminal is NOT sufficient.

- [ ] **Step 2: User logs back in**

Resume the same Plasma session.

- [ ] **Step 3: Re-open a terminal and re-source rustup env**

Run:
```sh
source $HOME/.cargo/env
```

No output expected. (You may not need this if your shell rc file already sources it, but doing it is safe.)

- [ ] **Step 4: Verify group membership now active**

Run:
```sh
groups | grep -q input && echo "OK" || echo "FAIL: still not in input"
id -nG | tr ' ' '\n' | grep input
```

Expected output: `OK`, and `input` printed.

If still missing, the logout was not complete (some KDE setups keep the Wayland session alive across "Log Out"). Reboot and try again.

---

## Task 8: Install udev rules

**Files:**
- Create: `/etc/udev/rules.d/99-pepper-x-uinput.rules`
- Create: `/etc/udev/rules.d/99-pepper-x-keyboard.rules`

- [ ] **Step 1: Pre-flight — confirm rules absent**

Run:
```sh
ls /etc/udev/rules.d/99-pepper-x-* 2>&1
```

Expected output: "No such file or directory".

- [ ] **Step 2: Write both udev rules**

Run:
```sh
echo 'KERNEL=="uinput", GROUP="input", MODE="0660"' \
  | sudo tee /etc/udev/rules.d/99-pepper-x-uinput.rules
echo 'SUBSYSTEM=="input", ATTRS{name}=="Pepper X virtual keyboard", ENV{ID_INPUT_KEYBOARD}="1"' \
  | sudo tee /etc/udev/rules.d/99-pepper-x-keyboard.rules
```

Expected output: each `tee` echoes the line back to stdout once.

- [ ] **Step 3: Reload udev rules**

Run:
```sh
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=input
```

No output expected.

- [ ] **Step 4: Verify /dev/uinput permissions**

Run:
```sh
ls -l /dev/uinput
```

Expected output: file exists with **group `input` and mode `0660`** — looks like `crw-rw---- 1 root input 10, 223 ...`.

If group is still `root` or mode is different, the udev rule didn't apply. Try `sudo udevadm trigger` (no filter) or reboot.

---

## Task 9: Verify network egress for ONNX download

`ort` 2.0-rc.12 fetches a prebuilt ONNX Runtime during `cargo build`. Confirming egress now prevents a 30-minute build from failing at the end.

- [ ] **Step 1: Test HTTPS connectivity to the likely host**

Run:
```sh
curl -sSI https://github.com/microsoft/onnxruntime/releases/latest -o /dev/null -w "%{http_code}\n"
curl -sSI https://huggingface.co/ -o /dev/null -w "%{http_code}\n"
```

Expected output: two lines, each `200` or `302` (the latter is GitHub redirecting to the latest release page).

If either returns `000` (connection failed), no network — `cargo build` will fail at the `ort` build step. Surface this to the user; it's not a bug to work around.

---

## Task 10: cargo build --release

**Files:** none in repo. Creates `target/release/pepper-x`, `target/release/pepperx-uinput-helper`, `target/release/pepperx-cleanup-helper`.

- [ ] **Step 1: Pre-flight — clean any stale target dir**

Run (in the worktree):
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
ls target 2>&1 || echo "no target dir"
```

If a `target/` exists and is on a different rustc version, you may want `cargo clean`. Otherwise leave it.

- [ ] **Step 2: Build (long-running — expect 5-30 minutes)**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
cargo build --release 2>&1 | tee /tmp/w1-cargo-build.log
```

Expected output: ends with `Finished \`release\` profile [optimized] target(s) in <time>` and `cargo` exits 0.

**Common failure modes (per spec Stage B.1):**
| Symptom | Root cause | Fix |
|---|---|---|
| `error: failed to parse manifest` referencing edition 2024 | apt cargo still in PATH | `which cargo` should be `~/.cargo/bin/cargo`. If not, `source $HOME/.cargo/env`. |
| `error: failed to run custom build command for llama-cpp-sys-4` mentioning `clang` or `libclang.so` | `libclang-dev` missing | `sudo apt install libclang-dev` (Task 4). |
| `error: failed to run custom build command for ort-sys` with HTTP/curl errors | network egress blocked at build time | Resolve network before re-running. |
| `linker errors` referencing `libadwaita` or `gtk4` | unlikely on noble (4.14.2 / 1.5.0); if seen, surface to user | n/a |

If the build fails, invoke `superpowers:systematic-debugging` and `error-debugging:debugger` per the ways-of-working table. Do NOT patch around symptoms (e.g. by disabling features in `Cargo.toml`).

- [ ] **Step 3: Verify all three binaries built**

Run:
```sh
ls -lh target/release/pepper-x target/release/pepperx-uinput-helper target/release/pepperx-cleanup-helper
```

Expected output: three files, all executable, with non-zero size (typically 50-200 MB each due to bundled native libs).

---

## Task 11: Run quality gates (test + fmt + clippy)

These are the three CI gates. They must all pass before installing binaries.

- [ ] **Step 1: cargo test --workspace**

Run:
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
cargo test --workspace 2>&1 | tee /tmp/w1-cargo-test.log
```

Expected output: ends with `test result: ok.` for every crate, and `cargo` exits 0.

**Note:** This does NOT run shell-based smoke tests in `tests/smoke/*.sh`. Those are manual and not gating for W1 (some require `PEPPERX_CLEANUP_MODEL_PATH` and a real audio device).

- [ ] **Step 2: cargo fmt --check**

Run:
```sh
cargo fmt --check
```

Expected output: empty stdout, exit 0. Any output = formatting drift; investigate which file (it shouldn't be possible since we haven't edited any Rust source).

- [ ] **Step 3: cargo clippy -- -D warnings**

Run:
```sh
cargo clippy -- -D warnings 2>&1 | tee /tmp/w1-cargo-clippy.log
```

Expected output: ends with `Finished` and exits 0 with no warnings.

If clippy fires on warnings introduced by a newer rustc than upstream tested with, surface to user — do NOT silence individual warnings; this is upstream's code, not ours.

---

## Task 12: Install binaries

**Files:**
- Create: `/usr/local/bin/pepper-x`
- Create: `/usr/libexec/pepper-x/pepperx-uinput-helper`
- Create: `/usr/libexec/pepper-x/pepperx-cleanup-helper`

- [ ] **Step 1: Pre-flight — confirm absent**

Run:
```sh
ls /usr/local/bin/pepper-x /usr/libexec/pepper-x/ 2>&1
```

Expected output: "No such file or directory".

- [ ] **Step 2: Install all three with sudo**

Run (from the worktree where `target/release/` lives):
```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
sudo install -m 755 target/release/pepper-x /usr/local/bin/
sudo mkdir -p /usr/libexec/pepper-x
sudo install -m 755 target/release/pepperx-uinput-helper /usr/libexec/pepper-x/
sudo install -m 755 target/release/pepperx-cleanup-helper /usr/libexec/pepper-x/
```

No output expected.

**Do NOT run `bash scripts/dev-install-extension.sh`** — it will exit 1 on KDE because `gnome-extensions` is not on PATH (real check at `scripts/dev-install-extension.sh:161-164`). W5 will fix this for everyone; for W1, just don't run it.

- [ ] **Step 3: Verify install**

Run:
```sh
which pepper-x
ls -l /usr/libexec/pepper-x/
file /usr/local/bin/pepper-x
```

Expected output:
- `which`: `/usr/local/bin/pepper-x`
- `ls`: shows `pepperx-uinput-helper` and `pepperx-cleanup-helper`, both executable, owned by root.
- `file`: `... ELF 64-bit LSB pie executable, x86-64 ...`

---

## Task 13: Create autostart `.desktop` file

**Files:**
- Create: `~/.config/autostart/pepper-x.desktop`

This is the KDE-specific replacement for the GNOME Shell extension's app-launching role. It ensures `pepper-x` is running at session start so the D-Bus service is available when the KDE Global Shortcut (W2) fires.

- [ ] **Step 1: Pre-flight — confirm absent**

Run:
```sh
ls ~/.config/autostart/pepper-x.desktop 2>&1 || echo "absent"
```

Expected output: `absent` or "No such file or directory".

- [ ] **Step 2: Write the file**

Run (heredoc preserves exact contents):
```sh
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/pepper-x.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Pepper X
Comment=Local dictation
Exec=/usr/local/bin/pepper-x
Icon=pepper-x
StartupNotify=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
OnlyShowIn=KDE;GNOME;XFCE;
EOF
```

The `OnlyShowIn=KDE;GNOME;XFCE;` line is defensive — some desktops ignore autostart entries without an explicit show list. Including KDE first makes this work on KDE without breaking GNOME users who copy the same file.

- [ ] **Step 3: Verify file is well-formed**

Run:
```sh
desktop-file-validate ~/.config/autostart/pepper-x.desktop
cat ~/.config/autostart/pepper-x.desktop
```

Expected output:
- `desktop-file-validate`: empty (no errors). If `desktop-file-validate` is not installed, `sudo apt install desktop-file-utils` and retry.
- `cat`: shows the contents written above.

---

## Task 14: 🛑 STOP — User Action Required: First launch + model download

This step is GUI interaction; cannot be automated. The agent should NOT attempt to launch `pepper-x` headlessly or click through models — it must hand control back to the user.

- [ ] **Step 1: User launches pepper-x from a terminal**

User runs:
```sh
pepper-x
```

The GTK4 window should open. Logs print to the terminal — check for any startup errors related to D-Bus registration, audio (PipeWire), or the helper binaries.

- [ ] **Step 2: User opens Models section in the GUI**

In the pepper-x window, find the **Models** section (left sidebar). Click **Download Missing Models**.

- [ ] **Step 3: User downloads the smaller cleanup model first**

- ASR: **Nemotron 0.6B int8** (~850 MB from HuggingFace) — required.
- Cleanup: **Qwen 3.5 0.8B Q4_K_M** (~500 MB) — sufficient. (User can later upgrade to the 2B model ~1.3 GB if quality is insufficient.)

Wait for both downloads to complete. Watch the progress bars. If a download fails partway, restart it from the same screen.

- [ ] **Step 4: User confirms models loaded**

After download completes, the Models section should show both as "ready" / "loaded." There may be a model-warmup pause before the first dictation.

- [ ] **Step 5: User leaves pepper-x running for the next tasks**

Don't close the window — Tasks 15-17 need the D-Bus service registered and the app warm.

---

## Task 15: Verify D-Bus service registered

- [ ] **Step 1: Introspect the service**

Run (in any terminal):
```sh
gdbus introspect --session \
  --dest com.obra.PepperX.Service \
  --object-path /com/obra/PepperX
```

Expected output: a multi-line introspection dump. Look for the line containing `start_recording`. The full method signature should look like:

```
method start_recording(string trigger_source) -> ()
```

(Or similar — the exact zbus serialization may vary.)

- [ ] **Step 2: Confirm method invocation works (dry-run)**

Run:
```sh
gdbus call --session \
  --dest com.obra.PepperX.Service \
  --object-path /com/obra/PepperX \
  --method com.obra.PepperX.StartRecording \
  shell-action
```

Expected output: typically `()` or empty success. If you see `Error: GDBus.Error:org.freedesktop.DBus.Error.UnknownMethod`, the service did register but the method name needs case-fixing — try `start_recording` (snake_case) instead of `StartRecording`.

**Note:** This call actually triggers a recording session. If `pepper-x` starts recording briefly, that's the correct behavior — release any active hotkey and let it stop. This proves W2's eventual KDE Global Shortcut wiring will work.

If the introspection in Step 1 fails with "service not running," then `pepper-x` isn't registered on the bus. Check the terminal where you launched it for errors.

---

## Task 16: 🛑 STOP — User Action Required: Smoke test in Kate

This is the core "does dictation work?" test. Must be done by a human with a microphone.

- [ ] **Step 1: User opens Kate, places cursor in a new document**

```sh
kate &
```

Type a few words first to confirm the editor is taking input normally. Place cursor at end.

- [ ] **Step 2: User performs hold-to-record dictation**

Hold the configured hotkey (default: **Alt+Super**). Speak: *"Hello world, this is a test of pepper-x dictation."* Release the hotkey.

- [ ] **Step 3: User verifies cleaned text appears in Kate**

Within ~2 seconds of releasing the hotkey, the cleaned text should appear at the cursor. Plain uinput typing is acceptable — the text will appear character-by-character.

**Pass criteria:** The dictated phrase (or a close transcription with cleanup-applied capitalization/punctuation) appears in Kate.

**Fail troubleshooting (do not patch around — invoke `superpowers:systematic-debugging`):**

| Symptom | Likely cause |
|---|---|
| Hotkey does nothing | KDE Global Shortcut already bound to Alt+Super; reconfigure pepper-x's hotkey in Settings → Recording. |
| Audio not captured (no transcription) | PipeWire mic permission; check `pavucontrol` recording tab while holding the hotkey. |
| Text appears garbled / wrong layout | XKB layout mismatch in `pepperx-uinput-helper`; set `PEPPERX_XKB_LAYOUT` env. |
| Text doesn't appear at all | uinput device not opened; verify `ls -l /dev/uinput` again (Task 8 Step 4). |
| ASR completes but cleanup hangs | cleanup-helper subprocess; check `pgrep pepperx-cleanup-helper`. |

---

## Task 17: 🛑 STOP — User Action Required: Smoke test in Konsole

Same as Task 16 but in a different app, to confirm uinput injection works in a terminal context.

- [ ] **Step 1: User opens Konsole, focuses it at a shell prompt**

```sh
konsole &
```

- [ ] **Step 2: User performs hold-to-record dictation**

Hold Alt+Super. Speak: *"echo testing pepper x in konsole"*. Release.

- [ ] **Step 3: User verifies text appears at the prompt**

The text should appear at the shell prompt as if typed. Pressing Enter should execute it (proving the keystrokes really did reach the terminal as input, not as some overlay).

**If Konsole works but Kate didn't (or vice versa):** that's a real diagnostic data point. Surface to user; do NOT proceed to Task 18 until the root cause is understood.

---

## Task 18: Update `pepper-x-install.md`

**Files:**
- Modify: `pepper-x-install.md` (in the worktree) — line citation fixes, autostart step, smoke-test caveat.

`pepper-x-install.md` is the user's pre-existing research note. At plan execution start it was untracked in `main` (per Task 1 Step 1 baseline). We update it inside this workstream so its claims match reality post-W1, then commit it on the W1 branch so the merge-to-main puts it in version control. The corresponding upstream-candidate version of these doc changes lives in `README.md` and is W4a's concern, NOT this task.

- [ ] **Step 1: Ensure `pepper-x-install.md` exists in the worktree**

The worktree was created from `main` at a point where `pepper-x-install.md` was untracked, so the file isn't in the worktree. Copy it from the main checkout:

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
test -f pepper-x-install.md && echo "already present" || cp /home/luke/projects/github/lukepatrick/pepper-x/pepper-x-install.md ./
ls -l pepper-x-install.md
```

Expected output: `ls -l` shows the file exists in the worktree.

- [ ] **Step 2: Confirm stale citations are present**

```sh
grep -n "scripts/dev-install-extension.sh:90-93\|atspi.rs:350-370\|atspi.rs ≈ lines 350-370" pepper-x-install.md
```

Expected output: line numbers showing one or both stale citations. If grep returns empty, the citations may have already been corrected; verify by reading the file before assuming the task is a no-op.

- [ ] **Step 3: Fix the script line citation**

Replace every occurrence of `scripts/dev-install-extension.sh:90-93` (and the bare form `dev-install-extension.sh:90-93`) with `scripts/dev-install-extension.sh:161-164`. The new location is the actual line of the `command -v gnome-extensions` guard, verified by the fact-check agent during brainstorming.

- [ ] **Step 4: Fix the atspi.rs line citation**

Replace every occurrence of `atspi.rs ≈ lines 350-370` and `atspi.rs:350-370` with `atspi.rs:315-339`. The new location is the actual span of `friendly_insert_target_class_from_application_id`, also verified by the fact-check agent.

- [ ] **Step 5: Add the KDE autostart step**

In the install section of `pepper-x-install.md` (suggested location: under "### Build" right after the binary-install commands, OR at the start of "### First run"), add the following block. This replaces the GNOME Shell extension's role on KDE — the extension is what kept the app running on GNOME; on KDE we use a `.desktop` file.

```markdown
- [ ] Create the KDE autostart entry so the D-Bus service is available at session start (replaces the GNOME Shell extension's role):
  ```sh
  mkdir -p ~/.config/autostart
  cat > ~/.config/autostart/pepper-x.desktop <<'EOF'
  [Desktop Entry]
  Type=Application
  Name=Pepper X
  Comment=Local dictation
  Exec=/usr/local/bin/pepper-x
  Icon=pepper-x
  StartupNotify=false
  NoDisplay=false
  X-GNOME-Autostart-enabled=true
  OnlyShowIn=KDE;GNOME;XFCE;
  EOF
  desktop-file-validate ~/.config/autostart/pepper-x.desktop
  ```
  Verification: at next login, `pgrep pepper-x` should show the process running.
```

- [ ] **Step 6: Add the smoke-test caveat**

Append (or insert into the "Things to watch for" section) a short note:

```markdown
- `cargo test --workspace` does NOT include the shell-based smoke tests in `tests/smoke/`. Those are manual and require setup (e.g. `PEPPERX_CLEANUP_MODEL_PATH` pointing at a downloaded GGUF model, a real audio device). They are not gating for a "build green" check.
```

- [ ] **Step 7: Verify the file still reads clearly**

Read the modified file end-to-end. Confirm:
- The new autostart block flows naturally with surrounding install steps (no contradictions, no orphaned cross-references).
- The smoke-test caveat doesn't duplicate something that's already there.
- All "skip dev-install-extension.sh" guidance reflects the corrected `:161-164` line, not `:90-93`.

---

## Task 19: Commit the documentation pass

**Files committed:**
- `pepper-x-install.md` (the edits above)

- [ ] **Step 1: Check what's staged**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
git status
git diff pepper-x-install.md
```

Expected output: `pepper-x-install.md` shown as untracked or modified (depending on whether the file was committed previously). The diff should show only the line-citation fixes, autostart step, and smoke-test caveat.

- [ ] **Step 2: Stage and commit**

```sh
git add pepper-x-install.md
git commit -m "Update pepper-x-install.md after W1 verification

- Fix stale line citations: scripts/dev-install-extension.sh :90-93 -> :161-164
- Fix stale line citations: atspi.rs :350-370 -> :315-339
- Add KDE autostart .desktop step (replaces GNOME Shell extension's role on KDE)
- Note that cargo test --workspace excludes shell-based smoke tests in tests/smoke/

These corrections reflect what was actually verified on TuxedoOS 24.04 + KDE
during W1 execution."
```

Expected output: one file changed.

---

## Task 20: Update roadmap — W1 state to done

**Files:**
- Modify: `docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md` — flip W1 to done, set current_workstream to next.

- [ ] **Step 1: Update top status block**

In the roadmap file, replace:

OLD:
```
current_workstream:   W1 — Build + first-run on TuxedoOS
phase:                1
state:                in-progress
branch:               w1-build-and-first-run
worktree:             ../pepper-x.w1-build-and-first-run
last_updated:         2026-04-30
```

NEW (use today's date if execution spans multiple days):
```
current_workstream:   W2 — KDE Global Shortcut → D-Bus
phase:                1
state:                pending  (W1 done; W2 awaits brainstorming)
branch:               (none yet — will be created when W2 spec is written)
worktree:             (none yet)
last_updated:         <today's date>
```

- [ ] **Step 2: Update W1 row in workstream table**

Change W1's `State` column from `in-progress` to `done`.

- [ ] **Step 3: Append status-log entry**

At the bottom of the file:
```
- `<today's date>` — W1 done. Build green, helpers installed, autostart wired, dictation verified end-to-end in Kate and Konsole. State: `in-progress` → `done`. Next: W2 brainstorm.
```

- [ ] **Step 4: Commit**

```sh
git add docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md
git commit -m "Mark W1 done; advance current_workstream to W2"
```

---

## Task 21: Merge `w1-build-and-first-run` to main

- [ ] **Step 1: Switch back to main checkout**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git status
git branch --show-current
```

Expected: working tree clean, on `main`.

- [ ] **Step 2: Merge with no-fast-forward**

Per ways-of-working: every workstream gets a merge commit so the branch boundary is visible in history.

```sh
git merge --no-ff w1-build-and-first-run -m "Merge W1: build + first-run on TuxedoOS"
```

Expected output: a merge commit summary listing the changed files (`docs/superpowers/specs/2026-04-30-tuxedoos-kde-viability-roadmap.md`, `pepper-x-install.md`).

- [ ] **Step 3: Push to fork**

```sh
git push origin main
```

Expected output: confirmation that origin/main is updated.

---

## Task 22: Cleanup — remove the worktree

The branch is merged; the worktree no longer needs to exist.

- [ ] **Step 1: Verify nothing uncommitted in the worktree**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x.w1-build-and-first-run
git status
```

Expected: `nothing to commit, working tree clean`. If anything is uncommitted, STOP and surface to user — that work would be lost.

- [ ] **Step 2: Switch back to main checkout, then remove worktree**

```sh
cd /home/luke/projects/github/lukepatrick/pepper-x
git worktree remove ../pepper-x.w1-build-and-first-run
git worktree list
```

Expected output:
- `worktree remove`: no output on success.
- `worktree list`: only the main checkout remains.

- [ ] **Step 3: Delete the merged branch**

```sh
git branch -d w1-build-and-first-run
```

Expected output: `Deleted branch w1-build-and-first-run (was <sha>).` (If git refuses with "not fully merged," something went wrong with the merge — investigate, do NOT use `-D` to force.)

---

## Done

When all 22 tasks are checked, W1 is complete:
- Build is green; helpers are installed; udev/group/autostart are wired.
- Dictation works end-to-end in Kate and Konsole.
- The roadmap reflects W1=done and points at W2 as next.
- The pepper-x-install.md has been corrected and committed.

**Next session-start re-orientation will pick up W2.** The roadmap's entry-point logic says: `no current_workstream pending` (well, current_workstream=W2 with state=pending) → "pick next pending Wn from table; invoke superpowers:brainstorming" → that brainstorming session writes the W2 spec, which writing-plans then translates into a W2 plan, and so on.
