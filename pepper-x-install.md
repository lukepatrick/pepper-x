# Pepper X install on TuxedoOS

Tracking install of [obra/pepper-x](https://github.com/obra/pepper-x) — GNOME-first local dictation for Linux.

## System snapshot (2026-04-30)

- **OS**: TUXEDO OS 24.04 (Ubuntu 24.04 "noble" base)
- **Desktop**: KDE Plasma (`XDG_CURRENT_DESKTOP=KDE`)
- **Session**: Wayland
- **User groups**: no `input` group yet
- **Cargo / rustc**: not installed
- **udev rules**: none for pepper-x / uinput

## Verdict — ✅ should work, with caveats

After reading the source, **most of pepper-x is desktop-agnostic** and the build is fixable on noble with a few tweaks. You'll get working dictation. You won't get the GNOME tray icon / floating status pill.

### What you'll get on KDE Plasma

| Feature | Works? | Notes |
|---|---|---|
| Hold-to-record (modifier hotkey) | ✅ | Pure evdev (`/dev/input/event*`), DE-agnostic. Needs `input` group + udev rules. |
| Toggle-to-record shortcut | ✅ | Same evdev path. |
| uinput text injection | ✅ | Pure uinput + xkbcommon. Types into any focused app. |
| AT-SPI "friendly" insertion | ⚠️ partial | App whitelist is GNOME-centric (gnome-terminal, ptyxis, ghostty, mainstream browsers). Kate/KWrite/Konsole fall back to uinput typing — still functional, just no caret-aware insert. |
| OCR window context | ❌ | Currently **disabled in code on all platforms** (`context.rs:51-67` returns empty `SupportingContext::default()` — note: "triggers GNOME's screenshot sound"). No regression vs GNOME. |
| GTK4 + libadwaita app window | ✅ | Settings, history, models, diagnostics — all fine under Plasma. |
| CLI (`--transcribe-wav`) | ✅ | Desktop-independent. |
| D-Bus service `com.obra.PepperX` | ✅ | Standard session bus. Pokeable via `gdbus call …` from a KDE Global Shortcut. |
| GNOME Shell extension (tray icon, status pill) | ❌ | Hard requirement on `gnome-shell` 48+. **Skip the install step entirely.** |

### Project status notes (relevant context)

- **Brand new project** (~5 weeks old, created 2026-03-28). Only 2 contributors (obra + mvanhorn).
- **No releases, no pre-built binaries** — build-from-source is the only path.
- **Zero KDE-related issues/PRs/discussions** — you'd be early. Discussions disabled.
- **No reports of 24.04 builds** (working or failing) — also early.
- Maintainer responsive: merged the only 2 community PRs same day.

## Corrected install checklist (Ubuntu 24.04 + KDE)

### Prereqs

- [ ] Install build deps **(skip `cargo` and `libgtk4-layer-shell-dev` from README; add `clang`, `libclang-dev`, `libssl-dev`, `libpipewire-0.3-dev`)**:
  ```sh
  sudo apt install \
    build-essential cmake \
    libadwaita-1-dev libatspi2.0-dev libgirepository1.0-dev \
    libglib2.0-dev libgtk-4-dev \
    libvulkan-dev libxkbcommon-dev \
    libssl-dev libpipewire-0.3-dev \
    pkg-config tesseract-ocr \
    clang libclang-dev git
  ```
  Why the changes:
  - `cargo` from apt is 1.75 — too old. `gtk4 0.11` needs Rust 1.92, `libadwaita 0.9` needs edition 2024 (≥1.85), `ort` needs 1.88. → Use rustup instead.
  - `libgtk4-layer-shell-dev` is **not in noble repos** and **not actually used by any crate** (verified — no `gtk4-layer-shell` in any Cargo.toml or Cargo.lock). Safe to drop.
  - `clang`/`libclang-dev` is needed by `llama-cpp-sys-4`'s bindgen step. The README forgets this on every distro.
  - `libssl-dev` is needed transitively: `ort-sys`'s build script uses `ureq` v3, which uses `native-tls`, which needs system OpenSSL. Discovered when W1's build failed with `openssl-sys: Could not find directory of OpenSSL installation`. Missing from upstream README on every distro.
  - `libpipewire-0.3-dev` is needed by the `pipewire` crate (used in `pepperx-audio`). Discovered when W1's build failed at `libspa-sys` with `The system library libpipewire-0.3 required by crate libspa-sys was not found`. Missing from upstream README on every distro.
- [ ] Install rustup (stable toolchain ≥ 1.92):
  ```sh
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  source $HOME/.cargo/env
  rustup default stable
  ```
- [ ] Add `$USER` to `input` group (sudo):
  ```sh
  sudo usermod -aG input $USER
  ```
- [ ] **Log out / back in** (or reboot) for group to take effect.
- [ ] Add udev rules (sudo):
  ```sh
  echo 'KERNEL=="uinput", GROUP="input", MODE="0660"' \
    | sudo tee /etc/udev/rules.d/99-pepper-x-uinput.rules
  echo 'SUBSYSTEM=="input", ATTRS{name}=="Pepper X virtual keyboard", ENV{ID_INPUT_KEYBOARD}="1"' \
    | sudo tee /etc/udev/rules.d/99-pepper-x-keyboard.rules
  sudo udevadm control --reload-rules
  ```

### Build

- [ ] Clone:
  ```sh
  git clone https://github.com/obra/pepper-x ~/projects/pepper-x
  cd ~/projects/pepper-x
  ```
- [ ] `cargo build --release` — needs network egress (`ort` downloads a prebuilt ONNX Runtime). Expect a long compile (llama.cpp + GTK + bindgen).
- [ ] Install binaries (sudo):
  ```sh
  sudo install -m 755 target/release/pepper-x /usr/local/bin/
  sudo mkdir -p /usr/libexec/pepper-x
  sudo install -m 755 target/release/pepperx-uinput-helper /usr/libexec/pepper-x/
  sudo install -m 755 target/release/pepperx-cleanup-helper /usr/libexec/pepper-x/
  ```
- [ ] **Skip** `bash scripts/dev-install-extension.sh` — it requires `gnome-extensions` and exits with an error on KDE. Confirmed via `scripts/dev-install-extension.sh:161-164` (the `command -v gnome-extensions` guard).

### First run

- [ ] Create the KDE autostart entry so the D-Bus service is available at login (replaces the GNOME Shell extension's role on KDE):
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
- [ ] Launch `pepper-x` from a terminal (so we see logs).
- [ ] Open the GTK window → **Models** section → **Download Missing Models**:
  - ASR: Nemotron 0.6B int8 (~850 MB from HuggingFace)
  - Cleanup: Qwen 3.5 0.8B Q4_K_M (~500 MB) — start with the smaller one
- [ ] Configure shortcut (default: Alt+Super hold-to-record). Test in a text editor.
- [ ] **Important on KDE**: text-insertion into KDE-native apps (Kate/Konsole/etc.) is broken in upstream main as of 2026-05-01 — see "W1 execution findings" below. The hold-to-record trigger fires, audio is captured, ASR transcribes correctly, but no text appears in the focused app because pepper-x's uinput-helper fallback path is gated off for non-whitelisted apps. **W6 (AT-SPI whitelist) is required for daily-driver use** and has been promoted to Phase 1 of the roadmap.

### Optional KDE polish

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

For ad-hoc invocation without the binding (one-off testing):

```sh
gdbus call --session --timeout 1 \
  --dest com.obra.PepperX.Service \
  --object-path /com/obra/PepperX \
  --method com.obra.PepperX.StartRecording shell-action
```

## Things to watch for during the build

- **No network at build time** → `ort` will fail to download ONNX Runtime.
- **bindgen errors mentioning `clang`** → `libclang-dev` missing.
- **`error[E####]: edition 2024` / "rustc too old"** → still on apt cargo, need rustup.
- **`openssl-sys: Could not find directory of OpenSSL installation`** → `libssl-dev` missing.
- **`libspa-sys ... libpipewire-0.3 ... not found`** → `libpipewire-0.3-dev` missing.
- **Linker errors on libadwaita / gtk4** → noble ships gtk4 4.14.2 / libadwaita 1.5.0, both at the floor. Should pass; if not, kisak-mesa or a GNOME PPA could backport.
- **`cargo fmt --check` and `cargo clippy -- -D warnings` fail on upstream `main`** with rustc 1.95.0 — the upstream code itself doesn't pass its own CI gates against the latest stable Rust. Upstream CI uses `dtolnay/rust-toolchain@stable` (no toolchain pin), so their main drifts as Rust evolves. Not a viability issue; pure CI hygiene. Adding a `rust-toolchain.toml` upstream would lock this down.
- **Smoke tests in `tests/smoke/*.sh` are NOT wired into `cargo test`.** They require manual setup (e.g. `PEPPERX_CLEANUP_MODEL_PATH` pointing at a downloaded GGUF model, a real audio device) and are not gating for a "build green" check. Don't rely on `cargo test --workspace` to exercise live audio or hotkey paths.

## Source-code map (for the fork session)

File:line references collected during research — the next session shouldn't need to re-discover these.

### Hotkey + input (desktop-agnostic, already works on KDE)

- `crates/pepperx-platform-gnome/src/evdev_modifier_capture.rs`
  - `find_keyboard_devices()` — lines ~570-595. Reads `/dev/input` directly.
  - Main epoll loop — lines ~625-735. Pure kernel evdev.
  - `TriggerMode::Toggle` state machine — lines ~325-340, ~690-720.
- `crates/pepperx-uinput-helper/src/main.rs` — lines 1-120. Pure `evdev::uinput::VirtualDevice` + xkbcommon. Reads `PEPPERX_XKB_LAYOUT` env or auto-detects layout. No GNOME calls.

### KDE-relevant friction points (candidates for fork patches)

- `crates/pepperx-platform-gnome/src/atspi.rs:315-339`
  - `friendly_insert_target_class_from_application_id()` whitelist is GNOME-centric.
  - **Patch idea (W6)**: add Kate, KWrite, Konsole, Kontact, Falkon, Dolphin entries so they get caret-aware AT-SPI insertion (KDE ships `qt-at-spi`).
  - **W1 finding (2026-05-01)**: this is more critical than originally classified. With KDE apps absent from the whitelist, the entire insertion path silently fails — see "W1 execution findings" section below.
- `crates/pepperx-platform-gnome/src/atspi.rs:858-877`
  - `ensure_runtime_supported_backend()` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet" before the spawn-helper fallback in `app/src/transcription.rs:557` can fire. Combined with the whitelist gap above, this is why nothing types into Kate/Konsole on KDE during W1's smoke test. Architectural fix candidate; bigger than W6.
- `crates/pepperx-platform-gnome/src/screenshot.rs` lines 104-150
  - Uses XDG portal but then **scrapes `~/Pictures/` for the latest `Screenshot*.png`** — that's a GNOME-Shell save convention.
  - **Patch idea**: use the portal response signal correctly; would unlock OCR context on every desktop.
- `crates/pepperx-platform-gnome/src/context.rs` lines 51-67
  - `capture_supporting_context()` returns empty `SupportingContext::default()` unconditionally with comment about GNOME's screenshot sound.
  - Cleanup model gets no window OCR context on any platform right now. Fixing this benefits everyone, not just KDE.
- `crates/pepperx-platform-gnome/src/service.rs` lines 38-50, 96-100
  - D-Bus service definition. `set_modifier_only_supported(false)` path suggests architecture is intentional about graceful degradation.
- `crates/pepperx-ipc/src/lib.rs` lines 50-58
  - `Capabilities::shell_default()` defaults `extension_connected = false` — code paths gated on this flag stay quiet on KDE rather than crashing.

### GNOME-only (skip / replace for KDE)

- `gnome-extension/extension.js` — PanelMenu.Button (tray icon, lines 30-78), status overlay polling. Hard requirement on `gnome-shell`.
- `gnome-extension/metadata.json` — `shell-version: ["48","49","50"]`.
- `scripts/dev-install-extension.sh` lines 90-93 — exits with error if `gnome-extensions` not on PATH. Just don't run this script.

### Build-relevant

- Workspace `Cargo.toml`, `app/Cargo.toml`, `crates/pepperx-asr/Cargo.toml`, `crates/pepperx-cleanup-helper/Cargo.toml`, `crates/pepperx-platform-gnome/Cargo.toml`, `Cargo.lock`
- `.github/workflows/ci.yml` — confirms apt deps used in CI; missing `clang` / `libclang-dev` (latent bug).
- `app/src/main.rs` lines 21-32 — CLI short-circuits before `app::run()` for `StartupMode != Gui`.

### Verified facts (don't re-research)

- `gtk4` 0.11.1 declares `rust-version = "1.92"` (so apt cargo 1.75 cannot build).
- `libadwaita` 0.9.1 uses `edition = "2024"` (needs ≥ 1.85).
- `ort` 2.0.0-rc.12 declares `rust-version = "1.88"` and is configured with `download-binaries` enabled — fetches prebuilt ONNX Runtime at build time (needs network).
- `llama-cpp-sys-4` 0.2.23 vendors llama.cpp; build.rs uses bindgen 0.72 → needs `libclang`.
- No crate in the workspace depends on `gtk4-layer-shell` (verified absent from Cargo.toml and Cargo.lock).
- Native lib floors satisfied by Ubuntu 24.04 (noble): gtk4 ≥ 4.14 (have 4.14.2), libadwaita ≥ 1.5 (have 1.5.0), glib ≥ 2.80 (have 2.80.0), pango ≥ 1.52 (have 1.52.1), libpipewire ≥ 1.0 (have 1.0.5).
- Project state on 2026-04-30: created 2026-03-28, last commit 2026-04-07, ~5 weeks old, 2 contributors, 0 releases, 1 fork (`mvanhorn/pepper-x`), discussions disabled, 6 issues all maintainer-authored launch checklist items, 20 stars.

## Recommended upstream fixes (PR candidates)

Independent of our local needs — these would help any non-25.04 / non-GNOME user. Updated 2026-05-01 with W1 execution findings.

1. **README + CI**: `apt install` list adjustments
   - Drop `cargo` (too old on most LTS); recommend rustup explicitly.
   - Drop `libgtk4-layer-shell-dev` (not in Cargo.toml, not in noble repos, dead reference).
   - Add `clang` + `libclang-dev` (required by `llama-cpp-sys-4` bindgen on every distro).
   - **Add `libssl-dev`** (required by `openssl-sys` pulled in transitively via `ort-sys` build deps → `ureq` → `native-tls`). Discovered during W1.
   - **Add `libpipewire-0.3-dev`** (required by `pipewire` crate used in `pepperx-audio`). Discovered during W1.
   - Note Ubuntu 24.04 also works (native lib floors are met).
2. **`scripts/dev-install-extension.sh`**: detect non-GNOME early and print a friendly "skipping — no GNOME Shell detected, app will run without tray icon" message instead of an error. Real check is at `:161-164`, not `:90-93`.
3. **Platform crate naming**: `pepperx-platform-gnome` does the desktop-agnostic evdev + uinput work too. Splitting into `pepperx-platform-linux` (evdev, uinput, AT-SPI) + a thinner `pepperx-platform-gnome` (extension D-Bus glue, GNOME app whitelist) would clarify what's portable.
4. **OCR context fix**: `capture_supporting_context()` is currently a no-op on all platforms (`crates/pepperx-platform-gnome/src/context.rs:50-68`). Wire up the portal response signal correctly — wins on GNOME *and* enables KDE. Note: `screenshot.rs` and the public `capture_supporting_context` aren't currently wired together; only a test-only helper calls `screenshot_window`. Fix requires editing both.
5. **AT-SPI app whitelist**: extend `friendly_insert_target_class_from_application_id` (`crates/pepperx-platform-gnome/src/atspi.rs:315-339`) with KDE app IDs (Kate, KWrite, Konsole, Kontact, Falkon, Dolphin).
6. **NEW: pin Rust toolchain version** via `rust-toolchain.toml` so upstream's `main` doesn't drift fmt/clippy-clean status as rustup updates roll forward. CI currently uses `dtolnay/rust-toolchain@stable` — a moving target. With rustc 1.95.0 the upstream main fails both `cargo fmt --check` and `cargo clippy -- -D warnings`.
7. **NEW: fix the uinput-text fallback path.** The architecture in `app/src/transcription.rs:474-481` is *designed* to fall back to a uinput helper when the AT-SPI path fails, but `ensure_runtime_supported_backend()` in `crates/pepperx-platform-gnome/src/atspi.rs:858-877` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet". The result: on every desktop where the focused app isn't in the AT-SPI whitelist, **insertion silently fails after a successful transcription** (`insert=0.2s` is logged but the helper is never spawned). Verified empirically during W1 on KDE: hotkey trigger fires, ASR transcribes, cleanup runs, `pgrep pepperx-uinput-helper` returns empty, no text appears in the focused app. This is bigger than W6's whitelist additions and is the headline architectural problem on KDE.
8. **Helper stderr is suppressed (`fc04b8b`).** Combined with #7, this means the failure mode is invisible: the uinput-helper would log `[Pepper X uinput] detected layout from /etc/default/keyboard: us` and `[Pepper X uinput] XKB layout 'us' loaded, 99 characters mapped` to stderr if it were spawned, but the suppression makes diagnosis harder. Reconsider whether stderr suppression should be conditional (e.g. only in release builds, or behind an env var like `PEPPERX_HELPER_STDERR=1`).

## W1 execution findings (2026-05-01)

What worked end-to-end on TuxedoOS 24.04 + KDE Plasma + Wayland:

- ✅ `cargo build --release` (after adding `libssl-dev` and `libpipewire-0.3-dev` to apt list)
- ✅ `cargo test --workspace` (116/116 tests pass, 4 ignored, 0 failures)
- ✅ Helper binary install to `/usr/libexec/pepper-x/`
- ✅ `input` group + udev rules (with caveat: the user's existing session ACL on `/dev/uinput` already grants luke direct access; group membership matters for fresh boots)
- ✅ KDE autostart `.desktop` validates via `desktop-file-validate`
- ✅ pepper-x GUI launches; model download via Models section works (Nemotron 0.6B int8 + Qwen 3.5 0.8B Q4_K_M)
- ✅ D-Bus service registers (`com.obra.PepperX.Service`, methods `StartRecording`/`StopRecording`/`Ping`/`ShowSettings`/`ShowHistory`/`GetCapabilities`/`GetLiveStatus`)
- ✅ Hotkey detection (Alt+Super) — `[Pepper X] modifier-only start` fires correctly, **but only when pepper-x is launched with input-group access** (`sg input -c '/usr/local/bin/pepper-x'`). Without it, evdev reads silently fail and the hotkey never triggers.
- ✅ Audio capture via PipeWire
- ✅ Streaming ASR (Nemotron) — transcripts logged correctly: `using streaming transcript: "Echo Hello World"`
- ✅ Cleanup pipeline (Qwen via `pepperx-cleanup-helper`) — runs in ~500-800ms

What did NOT work:

- ❌ **End-to-end text insertion into KDE apps (Kate, Konsole)**. Pepper-x logs `insert=0.2s` and `perf: ... total=1.0s` as if insertion succeeded, but no text appears at the cursor. Root cause is upstream architectural; see #7 above. The uinput-helper subprocess is never spawned (`pgrep pepperx-uinput-helper` returns empty), the kernel never registers a "Pepper X virtual keyboard" device for that pepper-x instance, and the JSON IPC call to a non-existent helper fails silently.

What was deferred from W1:

- AT-SPI app whitelist additions (W6) — was Phase 2, now promoted to Phase 1 because it's necessary for W1's "daily-driver bar" to be met.
- The architectural fix for the uinput-text fallback path (#7) — not on the original roadmap; will be added.

Group-membership re-login caveat:

- TuxedoOS / KDE Plasma 6 retained the user session across "Log Out" in our session — `usermod -aG input luke` did NOT reach any user process (terminal, Claude Code, even fresh KDE terminals all lacked the group). `getent group input` showed luke in the group at the system level; `sg input -c '...'` worked as a per-invocation workaround. **The reliable durable fix is a full reboot.** Without it, you cannot launch pepper-x normally and have it capture hotkeys; you must use `sg input -c '/usr/local/bin/pepper-x'` or wait until next reboot.

## Brief for the next session (in the fork)

Goal: get pepper-x running on TuxedoOS 24.04 + KDE Plasma + Wayland.

**State at handoff**:
- This file (`/home/luke/projects/scratch/pepper-x-install.md`) has full context. Read it first.
- No code has been written yet. No fork has been created yet — Luke will fork manually.
- System has nothing installed: no rustup, no apt build deps, not in `input` group, no udev rules.

**Suggested order in the next session**:
1. Confirm the fork path and `cd` into it.
2. Walk Luke through the apt install (corrected list), rustup install, group add, udev rules. He'll run sudo commands.
3. `cargo build --release` and triage any failures using the "Things to watch for" table above.
4. Install binaries; **skip** the GNOME extension script.
5. First-run smoke test: launch from terminal, download smaller Qwen 0.8B + Nemotron, test hold-to-record into a Konsole or Kate window.
6. If the basic flow works: decide which of the "Recommended upstream fixes" to attempt as PRs vs keep as fork-only patches.

**What to ask Luke up front in the new session**:
- Confirm fork URL and local path.
- Whether he wants to upstream fixes (PRs against `obra/pepper-x`) or just maintain a personal fork.
- Whether to try the larger Qwen 2B cleanup model (1.3 GB) or stick with 0.8B (500 MB).

**Don't re-do**:
- Don't re-fetch the README or re-grep the source for what's GNOME-specific — citations are above.
- Don't re-check Ubuntu 24.04 package versions — verified above.
- Don't re-check project liveness / issues / forks — verified above.

## Notes / log

- _(append step results here as we go)_
