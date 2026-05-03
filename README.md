# Pepper X

Local dictation for Linux on Wayland. Hold a key combo, speak, release — your words appear in the focused app. Everything runs locally, no cloud.

Tested on GNOME 48+ (Ubuntu 25.04, Fedora 42+) and KDE Plasma 6 (TuxedoOS 24.04 / Ubuntu 24.04). Other Wayland desktops likely work via the universal uinput typing path.

## What it does

- **Hold Alt+Super** (configurable) to record, release to stop
- **Streaming transcription** via Nemotron 0.6B — text is ready the instant you stop talking
- **LLM cleanup** via Qwen 3.5 — fixes filler words, punctuation, capitalization, self-corrections
- **Text insertion** via uinput virtual keyboard — types directly into any focused app
- **Window OCR context** — captures screen text to help the cleanup model disambiguate names and terms
- **Speaker diarization** — filters out other voices (experimental)

## Performance

On an Intel Core Ultra 7 155U (no GPU):

```
record=3.2s  transcribe=0.0s  cleanup=0.5s  insert=0.2s  total=0.7s
```

Transcription happens during recording (streaming). Cleanup uses a pre-warmed KV cache.

## Install

### Prerequisites

Ubuntu 24.04+ (verified on TuxedoOS 24.04) or Fedora 42+. Wayland session on any modern Linux desktop. GNOME 48+ gets the polished experience (tray icon, AT-SPI caret-aware insertion); KDE Plasma 6 and other Wayland desktops use the universal uinput typing path.

```sh
# Ubuntu (24.04+, verified on TuxedoOS 24.04)
sudo apt install \
  build-essential cmake clang libclang-dev \
  libadwaita-1-dev libatspi2.0-dev libgirepository1.0-dev \
  libglib2.0-dev libgtk-4-dev \
  libpipewire-0.3-dev libssl-dev libvulkan-dev libxkbcommon-dev \
  pkg-config tesseract-ocr

# Fedora (list from upstream; not re-verified on this fork)
sudo dnf install \
  cmake gcc gcc-c++ clang clang-devel \
  at-spi2-core-devel glib2-devel gobject-introspection-devel \
  gtk4-devel libadwaita-devel libxkbcommon-devel vulkan-loader-devel \
  openssl-devel pipewire-devel \
  pkgconf-pkg-config tesseract
```

Notes on the apt deps:
- No `cargo` from apt — it's typically too old (apt's is 1.75 on noble). Install rustup instead (next step). `gtk4 0.11` requires Rust ≥ 1.92.
- `clang` + `libclang-dev` are needed by `llama-cpp-sys-4`'s bindgen step.
- `libssl-dev` is needed by `openssl-sys`, transitively pulled by `ort-sys`'s build script (downloads ONNX Runtime via `ureq` v3 → `native-tls`).
- `libpipewire-0.3-dev` is needed by the `pipewire` crate used in `pepperx-audio`.
- `libgtk4-layer-shell-dev` is intentionally NOT here — it isn't used by any crate.

Install rustup with a stable toolchain:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
source $HOME/.cargo/env
rustc --version  # should print 1.92.0 or higher
```

Your user must be in the `input` group for hotkey capture and text injection:

```sh
sudo usermod -aG input $USER
# Log out and back in (or reboot — see note below)
```

**KDE / systemd-logind note:** on some KDE Plasma + systemd-logind setups, "Log Out" doesn't fully end the user session, so the new group membership doesn't reach any user process (terminals, shells, pepper-x itself). If `groups` in a new terminal doesn't show `input` after logging back in, **reboot** — that reliably refreshes credentials across all processes.

A udev rule is needed for the virtual keyboard:

```sh
echo 'KERNEL=="uinput", GROUP="input", MODE="0660"' | sudo tee /etc/udev/rules.d/99-pepper-x-uinput.rules
echo 'SUBSYSTEM=="input", ATTRS{name}=="Pepper X virtual keyboard", ENV{ID_INPUT_KEYBOARD}="1"' | sudo tee /etc/udev/rules.d/99-pepper-x-keyboard.rules
sudo udevadm control --reload-rules
```

### Build and install

Network egress is required during `cargo build`: the `ort` crate downloads a prebuilt ONNX Runtime for the ASR engine.

```sh
cargo build --release
sudo install -m 755 target/release/pepper-x /usr/local/bin/
sudo mkdir -p /usr/libexec/pepper-x
sudo install -m 755 target/release/pepperx-uinput-helper /usr/libexec/pepper-x/
sudo install -m 755 target/release/pepperx-cleanup-helper /usr/libexec/pepper-x/
```

**On GNOME** (optional; gives you the tray icon and floating status pill):

```sh
bash scripts/dev-install-extension.sh
```
Log out and back in for the extension to load.

**On KDE Plasma or any non-GNOME Wayland desktop**: skip the GNOME extension (`scripts/dev-install-extension.sh` will exit with an error if `gnome-extensions` isn't on PATH). Create an autostart entry instead so the D-Bus service is available at session start:

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

The `OnlyShowIn=KDE;GNOME;XFCE;` line is defensive — some desktops ignore autostart entries without an explicit show list.

### Download models

Launch the app, go to the **Models** section, and click **Download Missing Models**. Or download manually:

- **ASR**: Nemotron 0.6B int8 (~850MB) — downloaded from HuggingFace on first run
- **Cleanup**: Qwen 3.5 0.8B Q4_K_M (~500MB) or 2B Q4_K_M (~1.3GB)

## Usage

```sh
pepper-x
```

That's it. The app:
1. On GNOME: registers with the `pepperx@obra` Shell extension (tray icon + status pill). On other desktops: just runs as a regular GTK4 app — the D-Bus service is still active and the dictation pipeline still works.
2. Pre-warms the cleanup model in the background
3. Listens for your trigger keys (Alt+Super by default)

You can also trigger recording from a system shortcut by binding to the D-Bus service (works on any desktop with a session bus):

```sh
gdbus call --session \
  --dest com.obra.PepperX.Service \
  --object-path /com/obra/PepperX \
  --method com.obra.PepperX.StartRecording shell-action
```

On KDE Plasma, this can be wired into a Global Shortcut via System Settings → Shortcuts → Custom Shortcuts.

### Settings

The app window is organized into sections:

- **Recording** — Shortcut recorders (hold-to-record + toggle-to-record), mic picker, sound effects, speaker filtering, test dictation
- **Cleanup** — Enable/disable, window context toggle, prompt profile, custom prompt editor
- **Corrections** — Editable preferred transcriptions and commonly misheard replacements
- **Models** — ASR and cleanup model selection with download progress
- **History** — Transcription lab with per-stage model pickers, inline prompt editor, word-level diff, audio playback, diarization timeline
- **General** — Launch at login
- **Diagnostics** — Runtime status

### CLI

```sh
# Transcribe a WAV file
pepper-x --transcribe-wav recording.wav

# Transcribe + cleanup
pepper-x --transcribe-wav-and-cleanup recording.wav

# Rerun an archived recording
pepper-x --rerun-archived-run <run-id>
```

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

## Architecture

- **`pepper-x`** — GTK4/libadwaita app, owns the recording pipeline, settings, history
- **`pepperx-cleanup-helper`** — Persistent subprocess running llama.cpp (llama-cpp-4) for Qwen 3.5 inference, isolated to avoid ONNX Runtime symbol collision with the ASR engine
- **`pepperx-uinput-helper`** — Persistent subprocess with XKB-aware virtual keyboard for text injection
- **`pepperx@obra` GNOME extension** — Tray icon, floating status pill overlay, D-Bus bridge. **Optional, GNOME-only.** On other desktops the D-Bus service runs without the extension; the autostart `.desktop` ensures pepper-x is launched at session start.

### Text-insertion strategy

The insertion path tries backends in order, falling through on failure:

1. **AT-SPI EditableText** (caret-aware insert) — fastest and cleanest path. Requires the focused app to register with AT-SPI. Works on GNOME apps and on Qt apps when the Qt AT-SPI accessibility bridge plugin is installed (Plasma 6 with `qt-at-spi` available, etc.).
2. **AT-SPI key-string injection** — fallback for terminals with limited editable-text support.
3. **Clipboard paste** — fallback when injection fails.
4. **uinput keystroke synthesis** — universal fallback. Kernel-level virtual keyboard via the `pepperx-uinput-helper` subprocess; types into whatever has keyboard focus. No caret-position knowledge, no readback, but works everywhere — including KDE/Plasma + Wayland setups where the Qt AT-SPI bridge isn't installed.

### Key crates

| Crate | Purpose |
|-------|---------|
| `pepperx-asr` | Streaming ASR via parakeet-rs (Nemotron 0.6B) |
| `pepperx-cleanup` | Cleanup prompt assembly, subprocess communication |
| `pepperx-cleanup-helper` | llama-cpp-4 inference (Qwen 3.5) |
| `pepperx-audio` | PipeWire recording with streaming chunk delivery |
| `pepperx-corrections` | Preferred transcriptions and misheard replacements store |
| `pepperx-models` | Model catalog, download, readiness checking |
| `pepperx-platform-gnome` | evdev modifier capture, AT-SPI text insertion, OCR context |
| `pepperx-ipc` | D-Bus service for extension communication |
| `pepperx-uinput-helper` | XKB-aware keystroke injection |

## Tests

```sh
cargo test --workspace
```

## License

See individual crate licenses.


