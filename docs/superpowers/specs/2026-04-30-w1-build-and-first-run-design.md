# W1 — Build + First-Run on TuxedoOS

Workstream W1 of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). First Phase 1 workstream; blocks everything else.

## Goal

Get `pepper-x` building from source and producing dictated text into a focused KDE app on TuxedoOS 24.04 + KDE Plasma + Wayland.

## Done criteria

All of these must be true simultaneously, verified by running the listed command and observing the listed output:

1. **Build green**:
   - `cargo build --release` exits 0.
   - `cargo test --workspace` exits 0.
   - `cargo fmt --check` exits 0.
   - `cargo clippy -- -D warnings` exits 0.
2. **Binaries installed**: `which pepper-x` returns `/usr/local/bin/pepper-x`; `/usr/libexec/pepper-x/pepperx-uinput-helper` and `/usr/libexec/pepper-x/pepperx-cleanup-helper` are present and executable.
3. **System integration live**: `groups` includes `input`; `ls -l /dev/uinput` shows group `input` with mode `0660`; the udev rule files exist in `/etc/udev/rules.d/`.
4. **Models downloaded**: launching `pepper-x`, opening Models, and clicking "Download Missing Models" succeeds for ASR (Nemotron 0.6B int8) and at least one cleanup model (Qwen 3.5 0.8B Q4_K_M).
5. **Autostart wired**: `~/.config/autostart/pepper-x.desktop` exists and is valid (verified via `gtk-launch pepper-x` or by logging out and back in — `pepper-x` is in the process list at next login).
6. **D-Bus service available**: `gdbus introspect --session --dest com.obra.PepperX.Service --object-path /com/obra/PepperX` returns the interface; `start_recording` is listed.
7. **End-to-end dictation works**: hold-to-record (default Alt+Super) while focus is in **Kate**, speak a short phrase, release. Cleaned text appears at the cursor position. Repeat in **Konsole** — text appears at the prompt. (Plain uinput typing is acceptable; caret-aware AT-SPI is W6's concern, not W1's.)
8. **Documentation updated**: `pepper-x-install.md` line citations corrected (script `:161-164`, `atspi.rs :315-339`); the manual "skip dev-install-extension.sh on KDE" step is documented in the install path; obvious upstream-candidate doc deltas (clang/libclang/Ubuntu 24.04/dropping libgtk4-layer-shell-dev) are flagged for future W4a but **not** filed here.

## Approach

Linear, in this order:

### Stage A — Toolchain and system prep

1. Install apt build dependencies — corrected list per `pepper-x-install.md`:
   ```
   sudo apt install build-essential cmake \
     libadwaita-1-dev libatspi2.0-dev libgirepository1.0-dev \
     libglib2.0-dev libgtk-4-dev \
     libvulkan-dev libxkbcommon-dev \
     pkg-config tesseract-ocr \
     clang libclang-dev git
   ```
   Notably **not** installing `cargo` (apt's is too old) or `libgtk4-layer-shell-dev` (dead reference; absent from `Cargo.toml` and `Cargo.lock`).
2. Install rustup + stable toolchain (≥ 1.92 needed for `gtk4` 0.11):
   ```
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source $HOME/.cargo/env && rustup default stable
   ```
3. Add `$USER` to `input` group; **log out / log back in** (group is the most common silent-failure point — verify via `groups` after re-login).
4. Install udev rules and reload:
   ```
   echo 'KERNEL=="uinput", GROUP="input", MODE="0660"' \
     | sudo tee /etc/udev/rules.d/99-pepper-x-uinput.rules
   echo 'SUBSYSTEM=="input", ATTRS{name}=="Pepper X virtual keyboard", ENV{ID_INPUT_KEYBOARD}="1"' \
     | sudo tee /etc/udev/rules.d/99-pepper-x-keyboard.rules
   sudo udevadm control --reload-rules
   ```
5. Verify network egress at build time is available — `ort` 2.0-rc.12 fetches a prebuilt ONNX Runtime during `cargo build` and will fail offline.

### Stage B — Build

1. `cargo build --release` from the repo root. Expected: long compile (llama.cpp + GTK + bindgen). Common failure modes:
   - `error[E####]: edition 2024` → still on apt cargo; rustup didn't take. Re-source `$HOME/.cargo/env`.
   - bindgen errors mentioning `clang` → `libclang-dev` missing.
   - linker errors on libadwaita/gtk4 → noble's gtk4 4.14.2 should pass; if not, surface to user.
   - `ort` download failure → no network.
2. `cargo test --workspace`, `cargo fmt --check`, `cargo clippy -- -D warnings`. **Note**: smoke tests in `tests/smoke/*.sh` are NOT wired into `cargo test` and run only as a manual verification step (some need env vars like `PEPPERX_CLEANUP_MODEL_PATH`); they are not gating for W1.

### Stage C — Install binaries

```
sudo install -m 755 target/release/pepper-x /usr/local/bin/
sudo mkdir -p /usr/libexec/pepper-x
sudo install -m 755 target/release/pepperx-uinput-helper /usr/libexec/pepper-x/
sudo install -m 755 target/release/pepperx-cleanup-helper /usr/libexec/pepper-x/
```

**Skip** `bash scripts/dev-install-extension.sh` — it errors on non-GNOME (real check at `:161-164`). W5 will fix this for everyone; for W1, just don't run it. Document the skip in our install notes.

### Stage D — Models

Launch `pepper-x` from a terminal so logs are visible. Open the GTK window → Models → Download Missing Models. Start with the smaller Qwen 0.8B cleanup model (~500 MB) before deciding whether to grab Qwen 2B (~1.3 GB).

### Stage E — KDE autostart

Create `~/.config/autostart/pepper-x.desktop` so the D-Bus service is available after a fresh login (no GNOME extension to start it). Reference shape:

```
[Desktop Entry]
Type=Application
Name=Pepper X
Exec=/usr/local/bin/pepper-x
X-GNOME-Autostart-enabled=true
StartupNotify=false
NoDisplay=false
```

Verify by `gtk-launch pepper-x` (or by logging out + back in and checking `pgrep pepper-x`).

### Stage F — Documentation pass

Inside this workstream, also update `pepper-x-install.md`:

- Fix `scripts/dev-install-extension.sh:90-93` reference to `:161-164` (verified via fact-check agent).
- Fix `atspi.rs:350-370` reference to `:315-339`.
- Add the autostart `.desktop` step.
- Note that `cargo test --workspace` does **not** include the shell-based smoke tests in `tests/smoke/`.

This documentation pass is **inside W1**, not a separate workstream. The upstream-candidate version of these doc fixes (for `README.md`, not `pepper-x-install.md`) lives in W4a and is filed separately when/if you decide to upstream.

### Stage G — End-to-end smoke

Manually:

1. Confirm `pepper-x` is running (`pgrep pepper-x`).
2. Open Kate, place cursor in a document.
3. Hold Alt+Super, say "hello world this is a test", release.
4. Confirm the cleaned-up text appears in Kate.
5. Repeat in Konsole at a shell prompt.

If any step fails, invoke `superpowers:systematic-debugging` — do not patch around symptoms.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| `ort` ONNX download fails (no network) | Low (you have egress) | Surface immediately — not workaround-able. |
| `libclang-dev` missing causes bindgen failure | Low (in apt list) | Caught by Stage B; install if missed. |
| Group membership doesn't take effect without full logout | High | Explicit re-login step; verify `groups` after. |
| KDE doesn't honor the autostart `.desktop` because `OnlyShowIn` defaults | Medium | Test by logging out/in; if needed add `NotShowIn=` or `OnlyShowIn=KDE;GNOME;`. |
| Hold-to-record evdev path needs `input` group AND uinput access — partial setup silently fails | Medium | Each udev rule + group addition is verified independently in Stage A. |
| Hold-to-record collides with KDE's existing Alt+Super shortcuts | Low–Medium | Configure pepper-x's hotkey in the GTK Settings → Recording section to a non-colliding combo if needed; document in install notes. |
| Cleanup helper crashes silently | Low | Recently fixed in upstream commit `fc04b8b "Suppress cleanup helper stderr output"` — verified merged. Run `pepperx-cleanup-helper` standalone if suspect. |

## Out of scope for W1

These are explicitly deferred to other workstreams or non-goals:

- AT-SPI caret-aware insertion in KDE apps → **W6**.
- OCR window context for cleanup model → **W7**.
- Upstream PRs for any doc/CI/code change → manual decision later, drawing from "Potential upstream contributions" in the roadmap.
- Replacing the GNOME tray icon with a KDE system-tray equivalent → not on the roadmap; KDE Global Shortcut (W2) is the trigger path instead.
- Whisper / non-Parakeet ASR backends → non-goal.
- Sandboxed packaging → non-goal.

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. The plan will translate Stages A–G into ordered, individually-verifiable steps with verification commands and a clear stop-on-failure rule.
