# W2 — KDE Global Shortcut → D-Bus Trigger

Workstream W2 of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). Adds a KDE-native trigger surface for pepper-x's existing D-Bus service. **No production-code changes** — this is a packaging + docs change.

## Goal

Let users on KDE Plasma trigger pepper-x dictation via a Custom Global Shortcut bound through System Settings. Achieved by extending `packaging/deb/pepper-x.desktop` with `[Desktop Action ...]` blocks for the four user-facing D-Bus methods, plus README/install-md sections documenting the binding workflow.

## Background — what changed since the original W2 framing

The original W2 row in the roadmap was scoped as "KDE Global Shortcut → D-Bus binding so dictation can be triggered without the GNOME tray icon." Two adjustments after the W2 brainstorm:

1. **Hotkey path already works on KDE.** W1's evdev modifier-capture path (Alt+Super by default) was verified working in the W9 smoke. Daily-driver dictation is not blocked by the absence of a KDE shortcut. W2 is *polish* — an alternative trigger UX, not a viability fix.
2. **Tray icon, status pill, first-run wizard, screenshot service deferred to a new W10.** The brainstorm considered three scopes (A — minimal binding only, B — tray icon, C — full GNOME-extension equivalent). Per user decision, W2 is scope A; B + C + a first-run-wizard alternative go into a new **W10 — KDE-native UX** workstream at Phase 4 (review after all other Phase 2 work settles).

The KDE shortcut path does NOT bypass the `input` group requirement — the uinput helper still writes to `/dev/uinput`. W2 only changes the *trigger* mechanism, not the underlying permissions for keystroke insertion. Documented in the README addition so users don't expect "shortcut works but I'm not in the input group" to also produce text.

## Done criteria

1. **`packaging/deb/pepper-x.desktop` extended** with:
   - `Actions=StartRecording;StopRecording;ShowSettings;ShowHistory;`
   - Four `[Desktop Action ...]` blocks, one per user-facing D-Bus method, each with a single-line `Exec=gdbus call ...` invocation.
   - `Categories=Utility;AudioVideo;` (was `Utility;GNOME;`) — drops the GNOME-only tag for cross-DE menu compatibility.
   - `Comment=Local Linux dictation shell with hold-to-record and KDE Global Shortcut support` (was `GNOME-first local Linux dictation shell`) — matches the W4a README refresh framing.
2. **`desktop-file-validate` passes** on the modified file.
3. **README addition** under `## Usage`: a "Triggering dictation from a KDE Global Shortcut" subsection covering per-user install (`~/.local/share/applications/pepper-x.desktop`), the System Settings → Custom Shortcuts walkthrough, the Meta+V default suggestion, and the `input`-group caveat.
4. **`pepper-x-install.md` addition**: same content as the README addition, placed near the existing "Optional KDE polish" section.
5. **Manual smoke pass**: shortcut bound to "Start dictation", pressed in Kate, dictation starts, text appears at cursor. All four Actions verified visible in System Settings → Custom Shortcuts → Pepper X. Pepper-x's evdev Alt+Super hotkey ALSO still fires (coexistence verified). `StopRecording` invoked while NOT recording either no-ops silently or surfaces a benign D-Bus error — outcome documented in status log either way (no production-code change needed; this is observed behavior, not a contract).
6. **Roadmap updates**: W2 row state transitions `pending` → `planned` → `in-progress` → `done`; status log captures smoke outcome; `current_workstream:` advances to the next pending workstream (W4b).
7. **W10 created in the roadmap**: new row at Phase 4 (`pending`, trigger: "review after all Phase 2 work is settled — W4b, W4c, W5, W7"). Phase description added: "Phase 4 — Future polish: KDE-native UX work to bring the experience to feature-parity with the GNOME extension. Single workstream (W10) for now; may decompose into W10a/b/c if brainstormed and split." Notes for W10 reference this W2 spec for the deferred B/C/(d) options.
8. **Branch merged to main**: `w2-kde-global-shortcut` worktree branch merges via `--no-ff`, then is removed.

## Approach

### Final `packaging/deb/pepper-x.desktop`

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

Each `Exec=` is a single line in the actual file (the spec's freedesktop.org grammar requires it). The `gdbus call` invocations match the verified D-Bus surface from W1 (bus name `com.obra.PepperX.Service`, object path `/com/obra/PepperX`, methods PascalCase per zbus auto-conversion).

**`--timeout 1` is included on every `gdbus call`** so a hung pepper-x doesn't block the KDE shortcut caller for the default 25 seconds. One-second is plenty for a method that just queues an internal event; if pepper-x isn't responsive in 1s, surfacing the error fast is better than blocking.

**`StartRecording` passes `shell-action` as the trigger_source.** Per the codebase trace during the W2 review pass: `trigger_source` is metadata only — it's stamped into history records (`app/src/history_store.rs:339`) and surfaced for telemetry, but no runtime branch in `app/src/session_runtime.rs` or `app/src/transcription.rs` switches behavior on the variant. The "wait for modifier release" semantics live entirely in the evdev capture path and never go through D-Bus. Using `shell-action` is therefore correct for **history attribution / telemetry**, not for behavioral selection. A typo or wrong value (e.g. `"foo"`) would be rejected by `parse_trigger_source` (`crates/pepperx-ipc/src/lib.rs:184-191`) and surface as `fdo::Error::Failed` — silent from a KDE shortcut context (see "Errors invisible" in the troubleshooting section).

### README addition

A new subsection placed under `## Usage`:

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

### `pepper-x-install.md` addition

Same content as the README addition, placed near the existing "Optional KDE polish" section that already mentions invoking `gdbus call` directly. The new content effectively replaces / extends that one-liner with the full per-user install + System Settings walkthrough.

## Testing strategy

### Unit tests

None. No production code changes.

### Manual smoke gate

1. Install desktop file per-user: `install -m 644 packaging/deb/pepper-x.desktop ~/.local/share/applications/`
2. `desktop-file-validate ~/.local/share/applications/pepper-x.desktop` — empty output (= valid).
3. `gtk-launch pepper-x` — sanity check; pepper-x window opens.
4. System Settings → Shortcuts → Custom Shortcuts → bind a key combo (e.g. Meta+V) to "Pepper X → Start dictation".
5. Make sure pepper-x is running (`pgrep -af pepper-x`); capture stderr: launch `pepper-x 2>&1 | tee /tmp/w2-smoke.log` if not already running.
6. Focus a Kate document; press the bound shortcut.
7. Speak, press the shortcut again to stop.

**Discriminators:**
- **Primary**: text appears in Kate.
- **Secondary**: `/tmp/w2-smoke.log` shows a perf line for the cycle and a `shell-action` trigger trace.
- **Liveness**: `pgrep -af pepperx-uinput-helper` returns a PID.
- **Coexistence**: pepper-x's evdev hotkey (Alt+Super) ALSO still works in a separate dictation cycle.

If smoke fails:
- Action doesn't show in System Settings → run `kbuildsycoca6` (rebuilds KDE's KService cache) and re-check; or just relogin.
- Action appears but shortcut does nothing → copy the `Exec=` to a terminal, run it standalone, see what `gdbus call` reports.
- Shortcut fires but no recording → check pepper-x stderr; the D-Bus method may have errored.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| KDE Plasma 5 vs 6 handle Custom Shortcuts UI differently | Low–Medium | Smoke covers Plasma 6 (TuxedoOS). Spec marks Plasma 5 as unverified; documented in the README addition. |
| `gdbus call`'s long Exec line breaks `.desktop` parser | Low | `desktop-file-validate` catches syntax issues during smoke step 2. Action format is stable across freedesktop.org spec versions. |
| Shortcut fires while pepper-x isn't running → silent failure | Medium | Documented dependency on autostart `.desktop` from W1. Listed in README addition's "This trigger path requires" block. Future W10 (first-run wizard) could make this prettier. |
| User binds shortcut colliding with pepper-x's evdev hotkey → double-trigger | Low | Documented in the README addition (suggested default Meta+V chosen specifically to avoid Alt+Super). |
| KDE service cache stale → KDE doesn't see the Actions | Low | Documented in README addition as `kbuildsycoca6` step (NOT `update-desktop-database`, which updates the mime cache, not the KService registry KDE actually uses for Actions). Most desktops re-scan on next session login. |
| User expects shortcut path to bypass `input` group → confusion | Medium | Explicit caveat in README addition: "does NOT bypass the `input` group requirement." |
| Shortcut errors invisible (no controlling terminal) | Medium | Documented in README troubleshooting block: copy the `Exec=` line into a terminal to surface `gdbus` error output. |
| Per-user vs system-wide install conflict on deb upgrade | Low | Documented in README: per-user copy shadows system one and is NOT updated by `apt upgrade`. Pick one route or manually re-sync after upgrade. |
| `gdbus` not installed on minimal KDE images | Low | Documented prerequisite (`libglib2.0-bin`); usually already present on standard Plasma installs. |
| Default `gdbus` timeout (25s) blocks shortcut caller if pepper-x is hung | Low | All four Exec lines include `--timeout 1`. One second is generous for a method that just queues an event. |
| `StopRecording` invoked while NOT recording → D-Bus error | Low | Behavior in `service.rs:163-177`: returns `fdo::Error::Failed` if runtime errors, swallows `DuplicateStop` silently. KDE shortcut surface won't show the error (no controlling terminal). Manual smoke verifies the no-op-while-idle case. If it surfaces a notification toast, document as known-cosmetic. |
| Single-toggle UX expectation vs separate Start/Stop bindings | Medium (UX) | Most users want a single key that toggles. W2 ships separate Start/Stop because the D-Bus surface is method-pair-shaped. **Acknowledged limitation** — promote a "ToggleRecording" D-Bus method or a wrapper script as a future workstream (likely folded into W10's first-run-wizard scope). For now, users can either use pepper-x's own toggle hotkey (Alt+Space+Super by default) OR bind the same KDE shortcut to a small script that introspects state and calls Start/Stop accordingly. |
| `Categories=Utility;AudioVideo;` shifts pepper-x to "Multimedia" group in KDE menus | Low | Cosmetic surprise for current GNOME users. Documented in this risks table; can be tweaked back if disruptive. |
| `desktop-file-validate` only runs as a manual smoke step, not as a committed CI gate | Low | Future regressions to `pepper-x.desktop` (Action typo, mismatched bus name) won't be caught by `cargo test`. Could be added as a `tests/smoke/test_desktop_file.sh` script in a follow-up; tracked as a deferred-ideas candidate. |

## Out of scope

- **Tray icon, status pill, screenshot service, first-run shortcut wizard** — all deferred to **W10 — KDE-native UX** at Phase 4. W2 is the minimal binding-only delivery; W10 is the polish workstream.
- **Plasma 5 verification** — no environment available; documented as untested.
- **Source-code changes** — none. The D-Bus surface already supports everything W2 needs.
- **System-wide install of the desktop file** — going per-user (`~/.local/share/applications/`). System-wide via the deb pipeline is the alternative; documented as a conflict scenario in the README addition.
- **D-Bus session `.service` activation file** — would let `gdbus call` auto-launch pepper-x if it's not running, fixing the "shortcut fires while pepper-x isn't running → silent failure" risk properly. Out of W2 scope; surfaced as a deferred-ideas candidate (see auxiliary deliverables).
- **A `ToggleRecording` D-Bus method or wrapper script** — would smooth the single-toggle-key UX expectation. Out of W2 scope; deferred-ideas candidate.
- **`desktop-file-validate` as a committed CI gate** — currently a manual smoke step. Could be a shell test in `tests/smoke/`; deferred-ideas candidate.
- **Upstream PR filing** — local-only per established pattern.

## Auxiliary deliverables (folded into W2's commits)

Per architect-review feedback, **W10 creation gets its own commit** on the W2 branch (separable from the main packaging+docs deliverables) so reviewers can revert it independently if desired.

1. **Roadmap state transitions** — W2 row `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populate. `current_workstream:` advances to W4b on completion.
2. **Status-log entries** — one line per transition; final entry captures smoke outcomes (especially `StopRecording`-while-idle behavior verified during smoke).
3. **New workstream W10** added to the workstream table (separate commit):
   - `W10 | 4 | KDE-native UX (tray icon + status pill + screenshot service + first-run shortcut wizard) | pending | — | — | — | maybe | Comprises B + C + (d) deferred from W2 brainstorm. Trigger: review after all Phase 2 work is settled (W4b, W4c, W5, W7). Decomposes into tray icon (B), status pill via gtk4-layer-shell (C), KDE screenshot bridge (C), first-run shortcut wizard (d) — may split into W10a/b/c/d when brainstormed.`
4. **New "Phase 4 — Future polish" description** added to the Phase descriptions section (same separate commit as W10 creation).
5. **Status-log entry on creation of W10** — captures the brainstorm context so future-me re-orienting at the W10 brainstorm has the rationale.
6. **Three new entries appended to "Refactor / enhancement ideas (deferred)"** (separate commit), surfaced by the W2 architect-review:
   - **D-Bus session `.service` activation file** — ship a `com.obra.PepperX.Service.service` file under `/usr/share/dbus-1/services/` (or `~/.local/share/dbus-1/services/` for per-user) that lets `gdbus call` auto-launch pepper-x if it isn't running. Fixes the "shortcut fires while pepper-x isn't running → silent failure" UX gap properly. Trigger: a user complains about the shortcut "doing nothing" when pepper-x has crashed/exited.
   - **`ToggleRecording` D-Bus method or wrapper script** — smooth the single-toggle-key UX expectation. Either a new method that introspects state and dispatches Start/Stop, or a small shipped script (`packaging/kde/pepper-x-toggle.sh`) that does the same in userspace. Trigger: more than one user asks for "one key, not two" in feedback.
   - **`desktop-file-validate` as a committed CI gate** — add `tests/smoke/test_desktop_file.sh` (or a `cargo test` integration test) that runs `desktop-file-validate` against `packaging/deb/pepper-x.desktop` on every CI run. Catches Action typos, mismatched bus names, syntax drift. Trigger: someone introduces a `.desktop` regression that ships before being caught manually.

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. Plan will translate the four-stage structure (worktree setup → file edits + commit → manual smoke gate → wrap-up incl. W10 creation) into ordered, individually-verifiable tasks. Manual smoke is the hard sync point requiring user action; everything else is autonomous-friendly.
