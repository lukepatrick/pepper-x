# TuxedoOS/KDE Viability Roadmap

> **READ FIRST** every session. Single source of truth for "what's next" on this fork.

## Status

```
current_workstream:   W4b — CI dependency-list fix
phase:                2
state:                pending  (W2 done; W4b awaits brainstorming)
branch:               (none yet — created when W4b spec is written)
worktree:             (none yet)
last_updated:         2026-05-02
```

## Goal

Make `obra/pepper-x` viable on **TuxedoOS 24.04 + KDE Plasma + Wayland**. Daily-driver bar:

- `cargo build --release` succeeds.
- Hold-to-record types into Kate, KWrite, Konsole, Falkon/Firefox, Kontact. **Note (revised 2026-05-01, revised again later same day from W6 pre-flight findings):** the plain-uinput fallback path is broken upstream for non-whitelisted apps (`ensure_runtime_supported_backend()` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet"). The original plan was W6 (AT-SPI whitelist additions) so KDE apps take the AT-SPI path. **W6 is now blocked** — TuxedoOS's `libqt6gui6` ships without the Qt AT-SPI accessibility bridge plugin, so Kate/Konsole never register with AT-SPI regardless of any whitelist. **W9 (architectural fix to the uinput-text fallback)** is now Phase 1 — it makes the universal fallback work and bypasses the AT-SPI dependency entirely.
- KDE Global Shortcut bound to the D-Bus service `com.obra.PepperX.Service` so dictation can be triggered without the GNOME tray icon.
- Pepper X autostarts on KDE login so the D-Bus service is available when the shortcut fires.

All work merges to `lukepatrick/pepper-x:main`. **No upstream PRs filed from this workflow** — upstreaming is a separate manual decision deferred until local vetting is complete (see "Potential upstream contributions" below).

## Workstreams

| ID | Phase | Title | State | Branch | Spec | Plan | Upstream candidate | Notes |
|----|-------|-------|-------|--------|------|------|---|---|
| W1 | 1 | Build + first-run on TuxedoOS | `done (caveats)` | `w1-build-and-first-run` (merged) | `2026-04-30-w1-build-and-first-run-design.md` | `2026-04-30-w1-build-and-first-run.md` | partial | Build, ASR, cleanup pipeline all verified end-to-end on TuxedoOS/KDE. Hotkey detection works (with `input` group; reboot required for it to take effect after `usermod`). **End-to-end text-insertion into KDE apps does NOT work** — uinput-helper fallback is gated off upstream. See W1 status-log entry 2026-05-01 and `pepper-x-install.md` "W1 execution findings" section. Apt-deps additions: `libssl-dev`, `libpipewire-0.3-dev` (also missing from upstream README). |
| W6 | 1 | AT-SPI app whitelist additions for KDE | `blocked` | `w6-atspi-kde-whitelist` (merged with no production change) | `2026-05-01-w6-atspi-kde-whitelist-design.md` | `2026-05-01-w6-atspi-kde-whitelist.md` | yes (when unblocked) | **Blocked 2026-05-01.** Pre-flight AT-SPI registry check (`busctl --address=$A11Y_BUS tree`) confirmed neither Kate nor Konsole register with AT-SPI on TuxedoOS — `libqt6gui6:amd64` 6.9.2 ships without the AT-SPI accessibility bridge plugin (no `accessible/` subdirectory under `/usr/lib/x86_64-linux-gnu/qt6/plugins/`); `apt-cache search atspi` finds no Qt-side bridge package in the repos. The whitelist patch alone cannot route Qt apps to the AT-SPI path that doesn't see them. **Unblock condition**: a way to install or build the Qt AT-SPI bridge plugin surfaces (KDE Neon backport / PPA / source build / vendor patch). Casual side research; not blocking the roadmap. Resume W6 from this branch when unblocked. |
| W2 | 1 | KDE Global Shortcut → D-Bus | `done` | `w2-kde-global-shortcut` (merged) | `2026-05-02-w2-kde-global-shortcut-design.md` | `2026-05-02-w2-kde-global-shortcut.md` | no | Adds KDE-native trigger surface (Custom Shortcut → D-Bus) alongside the existing evdev hotkey. Packaging + docs only; no production-code changes. KDE-specific glue; not relevant to GNOME users. |
| W4a | 2 | README refresh | `done` | (direct on `main` — quick refresh outside workstream framework per user request) | — | — | yes | **Done 2026-05-02 as a quick refresh.** Scope grew beyond original "doc-only deps fix" to include: rustup-instead-of-apt-cargo install step, KDE autostart `.desktop` (alternative to GNOME extension), broadened platform-support claim (Ubuntu 24.04+, KDE Plasma 6, generic Wayland desktops), new "Text-insertion strategy" subsection in Architecture explaining the AT-SPI → uinput fallback chain, KDE/systemd-logind logout-vs-reboot caveat, D-Bus shortcut-binding example. Apt deps fixed (drop `cargo` + `libgtk4-layer-shell-dev`; add `clang` + `libclang-dev` + `libssl-dev` + `libpipewire-0.3-dev`). Fedora deps refreshed by analogy (not re-verified on Fedora). |
| W4b | 2 | CI dependency-list fix | `pending` | — | — | — | yes | Add `clang`, `libclang-dev`, `libssl-dev`, `libpipewire-0.3-dev` to `.github/workflows/ci.yml`. Latent bug — passes today only because GitHub runners ship some of these. |
| W4c | 2 | Pin Rust toolchain version | `pending` | — | — | — | yes | **New 2026-05-01 from W1 findings.** Add `rust-toolchain.toml` with a pinned stable version. Upstream's `dtolnay/rust-toolchain@stable` is a moving target; with rustc 1.95.0, upstream main fails both `cargo fmt --check` and `cargo clippy -- -D warnings`. Pinning fixes the drift. |
| W5 | 2 | `dev-install-extension.sh` graceful skip on non-GNOME | `pending` | — | — | — | yes | Real check is at `scripts/dev-install-extension.sh:161-164` (not 90-93 as install.md originally claimed — fixed 2026-05-01). Exit 0 with friendly message. |
| W7 | 2 | OCR portal-response fix + rewire | `pending` | — | — | — | yes | Two-part: (1) fix `screenshot.rs:95-161` to use the XDG portal response signal instead of scraping `~/Pictures/`, (2) rewire `crates/pepperx-platform-gnome/src/context.rs:50-68` so the public `capture_supporting_context()` actually calls `screenshot_window`. The two are not currently wired together. Highest code risk in Phase 2. |
| W9 | 1 | Fix uinput-text fallback path (architectural) | `done` | `w9-uinput-text-fallback` (merged) | `2026-05-01-w9-uinput-text-fallback-design.md` | `2026-05-01-w9-uinput-text-fallback.md` | yes | **Promoted from Phase 2 to Phase 1 on 2026-05-01** when W6 blocked on missing Qt AT-SPI bridge. Architecture in `app/src/transcription.rs:474-481` is *designed* to fall back to a uinput helper when AT-SPI fails, but `ensure_runtime_supported_backend()` at `crates/pepperx-platform-gnome/src/atspi.rs:858-877` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet". W9 is now the durable fix that makes pepper-x work for any focused app on any desktop — bypasses the AT-SPI dependency W6 hit. Bigger than W6; needs its own brainstorm. |
| W8 | 3 | Platform crate split: `pepperx-platform-linux` ↔ `pepperx-platform-gnome` | `conditional` | — | — | — | maybe | Triggered only by gate criteria below. |
| W10 | 4 | KDE-native UX (tray icon + status pill + screenshot service + first-run shortcut wizard) | `pending` | — | — | — | maybe | **New 2026-05-02 from W2 brainstorm.** Comprises B + C + (d) deferred from W2 (tray icon via SNI/StatusNotifierItem, status pill via gtk4-layer-shell on KDE, KDE screenshot bridge equivalent of GNOME extension's `com.obra.PepperX.Screenshot`, first-run shortcut wizard). Trigger: review after all Phase 2 work is settled (W4b, W4c, W5, W7). May decompose into W10a/b/c/d when brainstormed. |

State legend: `pending` → `spec` → `planned` → `in-progress` → `done` (or `blocked` / `conditional`).

## Phase descriptions

**Phase 1 — Get-it-running.** Daily-driver bar above. W1 verified the build/transcription pipeline end-to-end but exposed that **insertion is broken upstream** for non-whitelisted apps. W6 was promoted to Phase 1 as the smallest unblocker — but pre-flight AT-SPI registry check on TuxedoOS revealed Kate/Konsole aren't visible to AT-SPI (Qt AT-SPI bridge not packaged), so **W6 is blocked** and **W9 is now Phase 1** as the durable fix that bypasses the AT-SPI dependency. W2 (KDE shortcut) follows W9. W6 stays in the workstream table as `blocked`; resume from its branch if/when the Qt AT-SPI bridge becomes available on TuxedoOS.

**Phase 2 — Polish & fixes.** Independent fork-local improvements; most are also future upstream candidates. Revised order on 2026-05-01 (post W6 block): W4a → W4b → W4c → W5 → W7 (W4c is new; W6 stays blocked in Phase 1; W9 graduated to Phase 1).

**Phase 3 — Architectural (conditional).** Crate restructure work that is only worth doing if local maintenance friction makes the case. Not on the critical path.

**Phase 4 — Future polish.** KDE-native UX work to bring the experience to feature-parity with the GNOME extension. Single workstream (W10) for now; may decompose into W10a/b/c/d if it gets brainstormed and split. Off the critical path; review after all Phase 2 work has settled.

## W8 decision-gate triggers

W8 is `conditional`. Self-determined — promote it to `pending` (and brainstorm it properly) if **any one** of:

- **Patch count**: 3+ KDE-specific patches accumulate on `main` that conflict awkwardly with `pepperx-platform-gnome` boundaries.
- **Maintenance friction**: a single fix requires changes in 3+ files across the platform crate, suggesting bad seams.
- **Architectural opportunity**: you're already touching the platform crate for another reason and can fold the split in cheaply.

When triggered, **stop the current workstream**, surface the trigger to the user, and run `comprehensive-review:architect-review` before writing-plans (W8 is a structural change).

## Refactor / enhancement ideas (deferred)

Considered, rejected for now, might revisit later. Each entry has a trigger condition that would promote it to a real workstream.

- **Whitelist data-table refactor** — replace the flat `match` in `crates/pepperx-platform-gnome/src/atspi.rs:315-339` with a `static` array of `(application_id, FriendlyInsertTargetClass)` pairs and look up by linear scan. Trigger: whitelist grows past ~50 entries OR an entry needs metadata beyond the category enum (e.g. per-app insertion strategy override). (Considered during W6 brainstorm; rejected as over-engineering at the current ~30-entry scale.)
- **Whitelist split into GNOME/KDE sub-functions** — `..._for_gnome_app` and `..._for_kde_app` returning `Option<FriendlyInsertTargetClass>`, composed by the public function. Trigger: ≥10 KDE-specific entries accumulate, or the flat match becomes hard to scan in a single screen. (Considered during W6 brainstorm; rejected — pure indirection for the current 6 KDE entries.)
- **Conditional helper stderr suppression** — make `fc04b8b`'s suppression toggleable via env var like `PEPPERX_HELPER_STDERR=1`. Trigger: another debugging session blocked on invisible helper logs (W1 was the first; second occurrence promotes this to a workstream — likely as W4d).

- **Startup-time AT-SPI viability check (Approach C from W9 brainstorm)** — at startup, probe the AT-SPI registry; if zero apps registered, log once (e.g. `[Pepper X] AT-SPI registry empty; using uinput-text fallback for all dictations`) and skip per-dictation AT-SPI lookups. Trigger: per-dictation AT-SPI overhead becomes measurable on TuxedoOS, OR desire for clearer one-shot "what-mode-am-I-in" startup logging.
- **GNOME cross-platform regression VM** — set up a GNOME VM (or remote test box) so changes can be smoke-tested on both KDE and GNOME. Trigger: a W9-class change feels under-tested without GNOME verification, OR a user reports a GNOME regression after changes that were only KDE-smoked.
- **Bundled sound-effect files (`bell`, `complete`)** — W9 smoke noticed `failed to play sound effect "bell": No such file or directory`. Cosmetic; pepper-x looks for sound files in a path that isn't populated by our build/install. Trigger: user complains about missing audio cues, OR we're already touching the sound-effect code path for another reason.

- **D-Bus session `.service` activation file** — ship a `com.obra.PepperX.Service.service` file under `/usr/share/dbus-1/services/` (or `~/.local/share/dbus-1/services/` for per-user) that lets `gdbus call` auto-launch pepper-x if it isn't running. Fixes the "shortcut fires while pepper-x isn't running → silent failure" UX gap properly. Trigger: a user complains about the shortcut "doing nothing" when pepper-x has crashed/exited.
- **`ToggleRecording` D-Bus method or wrapper script** — smooth the single-toggle-key UX expectation. Either a new method that introspects state and dispatches Start/Stop, or a small shipped script (`packaging/kde/pepper-x-toggle.sh`) that does the same in userspace. Trigger: more than one user asks for "one key, not two" in feedback.
- **`desktop-file-validate` as a committed CI gate** — add `tests/smoke/test_desktop_file.sh` (or a `cargo test` integration test) that runs `desktop-file-validate` against `packaging/deb/pepper-x.desktop` on every CI run. Catches Action typos, mismatched bus names, syntax drift. Trigger: someone introduces a `.desktop` regression that ships before being caught manually.

Future workstreams that surface deferred ideas append entries here rather than burying them in commit messages or PR comments.

## Non-goals (explicit)

- Cloud/remote inference. pepper-x is local-only by design.
- X11 support. Wayland-only.
- Sandboxed packaging (Flatpak / Snap). Native install only.
- Whisper or non-Parakeet ASR backends. Out of scope for this fork.
- Reviving the GNOME Shell extension on KDE. KDE Global Shortcut + autostart replaces it.
- ~~AT-SPI caret-aware insertion in KDE apps as a Phase 1 must-have. Plain uinput typing is acceptable; W6 addresses caret-aware as a Phase 2 polish.~~ **Revised 2026-05-01:** plain uinput typing is NOT acceptable because the upstream uinput-text fallback path doesn't actually work (see W9). W6 (AT-SPI whitelist) is now a Phase 1 must-have.
- Fixing OCR window context as a Phase 1 must-have. W7 addresses it as Phase 2 polish.
- Pre-emptive upstream PR filing. Vet locally first.

## Potential upstream contributions (manual decision later)

When you decide to upstream, this section lists ready-to-promote workstreams. Filing a PR is a manual event you trigger; it's not in the workstream loop.

| Workstream | Ready to upstream when | Notes |
|---|---|---|
| W4a (README) | Landed locally and tested on TuxedoOS | Doc-only; near-zero review risk. Adds `libssl-dev`, `libpipewire-0.3-dev`, `clang`, `libclang-dev`. |
| W4b (CI) | Landed locally; verify CI passes on a test branch | Infra change; verify before filing. Same packages as W4a. |
| W4c (rust-toolchain.toml) | Landed locally; CI green on the pinned version | Infra; locks rustfmt/clippy expectations against a specific Rust version. |
| W5 (script skip) | Landed locally and verified on KDE | Trivial behavior change. |
| W6 (AT-SPI) | Landed locally and verified caret-aware insert in at least Kate + Konsole | Test on KDE before filing — the whitelist is only useful if AT-SPI plumbing actually fires. **Now Phase 1 must-have for daily-driver bar.** |
| W7 (OCR fix) | Landed locally, tested on KDE *and* on a GNOME VM if possible | Touches code path used on every desktop. Highest review burden among Phase 2. |
| W9 (uinput-text fallback architectural fix) | Landed locally, tested on a non-whitelisted app on KDE *and* GNOME VM if possible | Architectural change to `ensure_runtime_supported_backend()` and the fallback path in `transcription.rs`. Coordinate with maintainer before filing — they may have a different intended design. |
| W8 | Only if triggered | Architectural; coordinate with maintainer first. |

Phase 1 work (W1, W2) is mostly local-only by nature (KDE autostart desktop file, KDE shortcut binding). The documentation portions of W1 (corrected install steps) are upstream candidates; the rest is fork-local.

## Re-orientation entry-point logic

```
roadmap state                → action
─────────────────────────────────────────────────────────────────────────────
no current_workstream         → pick next pending Wn; invoke superpowers:brainstorming
Wn = "spec"                   → invoke superpowers:writing-plans on the existing spec
Wn = "planned"                → cd to worktree; invoke superpowers:executing-plans
Wn = "in-progress, code done" → invoke superpowers:verification-before-completion
Wn = "verified, unmerged"     → invoke superpowers:finishing-a-development-branch
W8 gate triggered             → STOP. Surface to user. Do not start W8 silently.
```

See `2026-04-30-ways-of-working.md` for the full per-workstream rhythm and the situation→skill reference table.

## Links

- Ways-of-working (skill/agent reuse): `2026-04-30-ways-of-working.md`
- W1 spec: `2026-04-30-w1-build-and-first-run-design.md`
- Original upstream design: `2026-03-27-pepper-x-design.md`
- Install research notes (to be retired after W1 lands): `/pepper-x-install.md`
- CLAUDE.md (auto-loaded; points at this roadmap)

## Status log

State transitions get a one-liner here. Git history is the authoritative log; this is the human-readable summary.

- `2026-04-30` — Roadmap created. W1 spec written. W1 state: `spec`. Awaiting plan via `superpowers:writing-plans`.
- `2026-05-01` — W1 plan written and execution begun. State: `spec` → `in-progress`. Branch: `w1-build-and-first-run`.
- `2026-05-01` — W1 done with caveats. State: `in-progress` → `done (caveats)`. **Major findings:**
    - Build green after adding `libssl-dev` and `libpipewire-0.3-dev` to apt list (both missing from upstream README on every distro).
    - `cargo test --workspace` passes 116/116; `cargo fmt --check` and `cargo clippy -- -D warnings` fail on upstream main with rustc 1.95.0 (CI drift, not viability) → new W4c added to fix via `rust-toolchain.toml`.
    - Hotkey detection works only when pepper-x has `input` group access. KDE/systemd-logind retains pre-`usermod` credentials across "Log Out" — a full reboot is required for durable group-membership refresh; `sg input -c '...'` is the per-launch workaround until then.
    - **End-to-end text-insertion into KDE apps does NOT work.** Hotkey fires, audio captured, ASR transcribes correctly, cleanup runs, `[Pepper X] perf: ... insert=0.2s` is logged — but no text appears at the cursor. Root cause traced via Explore agent to `crates/pepperx-platform-gnome/src/atspi.rs:858-877`: `ensure_runtime_supported_backend()` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet". The architectural fallback in `app/src/transcription.rs:474-481` exists but isn't reached because the AT-SPI whitelist doesn't include KDE apps and the uinput-text branch is gated. Helper subprocess `pepperx-uinput-helper` confirmed never spawned (`pgrep` returns empty; pstree shows only the cleanup helper child).
    - Two new workstreams added: **W4c** (pin Rust toolchain) and **W9** (fix uinput-text fallback architecturally). **W6 promoted from Phase 2 to Phase 1** because the AT-SPI whitelist is now the unblocker for daily-driver bar; without it (or W9), nothing types into KDE apps.
- `2026-05-01` — current_workstream advanced to W6. State: `pending`. Awaits brainstorming.
- `2026-05-01` — W6 spec written and reviewed (architect-review + general-purpose fact-check). Five must-fix corrections applied: `org.kde.kmail2` → `org.kde.kmail`; existing tests at `atspi.rs:2002/2014/2026` acknowledged (W6 extends rather than adds novel coverage); smoke gate uses `pgrep` + log discriminators (not just "text appears at cursor"); regression-assertion + executable-name pinning required. State: `pending` → `spec`.
- `2026-05-01` — W6 plan written (`2026-05-01-w6-atspi-kde-whitelist.md`, 16 tasks, 909 lines) and execution begun. State: `spec` → `in-progress`. Branch: `w6-atspi-kde-whitelist`. New "Refactor / enhancement ideas (deferred)" section seeded with three entries from W6 brainstorm (whitelist data-table refactor; whitelist sub-function split; conditional helper stderr suppression).
- `2026-05-01` — **W6 blocked at pre-flight (Task 3)**. Pre-flight AT-SPI registry check via `busctl --address=$(gdbus call ... org.a11y.Bus.GetAddress) tree org.a11y.atspi.Registry`: 15 apps registered (Electron apps, Sublime Text, KDE infrastructure like ksmserver/kaccess/gmenudbusmenuproxy) — but **Kate and Konsole are NOT in the registry**. Diagnosis: TuxedoOS's `libqt6gui6:amd64` 6.9.2 ships without the Qt AT-SPI accessibility bridge. `find / -name 'libqtatspi*'` empty; `/usr/lib/x86_64-linux-gnu/qt6/plugins/` has no `accessible/` subdirectory; `apt-cache search atspi` finds no Qt-side bridge package. `QT_ACCESSIBILITY=1` is set globally but has no plugin to load. W6's whitelist patch alone cannot make Kate/Konsole appear in AT-SPI — the bridge is the actual unblocker, and it's externally unavailable. State: `in-progress` → `blocked`. **W9 promoted to Phase 1** as the durable fix that bypasses AT-SPI entirely. Casual side research thread: monitor whether KDE Neon backports / a future TuxedoOS update / source build provides the missing bridge plugin — when it does, resume W6 from its existing branch. No production code was written in W6's worktree; only roadmap state-tracking commits.
- `2026-05-01` — W9 spec written and reviewed (architect-review + general-purpose fact-check). Six must-fix corrections applied: `FriendlyInsertSelection` field count fixed (4 fields not 2, including non-empty `target_class` placeholder); change 2 expanded from 1 wrap site to 3 (find_focused_accessible at :971, inspect_focused_target_from_accessible at :972, select_friendly_insert_backend at :981); smoke gate discriminator priority reordered (text-appears primary, log secondary, pgrep liveness only); phantom stale-test deliverable removed (the cited test doesn't exist with that name); regression-protection test for all 4 backends accepted by gate added; out-of-scope expanded with cleanup-context and W9-vs-W6-behavior clarifications. State: `pending` → `spec`.
- `2026-05-02` — W9 plan written (`2026-05-01-w9-uinput-text-fallback.md`, 15 tasks, 933 lines) and execution begun. State: `spec` → `in-progress`. Branch: `w9-uinput-text-fallback`.
- `2026-05-02` — **W9 done. Daily-driver bar met on TuxedoOS.** Pre-flight grep audit found 22 `UINPUT_TEXT_BACKEND_NAME` references workspace-wide; none assert gate rejection. Code change committed (`c7ae619`): gate opened (4 backends accepted now), `wrap_atspi_failure_as_uinput_fallback` helper added at `atspi.rs:971`, applied via `.map_err` at three failure sites in `focused_friendly_target` (find_focused_accessible, inspect_focused_target_from_accessible, select_friendly_insert_backend → UnsupportedTarget). 4 new tests added (3 in atspi.rs, 1 in transcription.rs); workspace test suite 378/378 pass; clippy on `pepperx-platform-gnome` and `pepper-x-app` shows pre-existing rustc 1.95 drift only (W4c territory) — W9 introduced no new violations. Manual smoke verified end-to-end on TuxedoOS 24.04 + KDE Plasma + Wayland: dictation in **Kate** lands "Hello world from" + "Hello world, hello world" at cursor; dictation in **Konsole** also works. Helper spawn confirmed via stderr (`[Pepper X uinput] XKB layout 'us' loaded, 99 characters mapped`); cold-spawn cost 0.5s, warm-path 0.2s per dictation. AT-SPI socket warning (`Failed to connect to socket /run/user/1000/at-spi2-9VUYO3/socket`) is benign and exactly the case W9 is designed to handle. State: `in-progress` → `done`. Three new deferred-ideas entries added (startup-time AT-SPI viability check; GNOME cross-platform regression VM; bundled sound-effect files). **current_workstream advances to W2** (KDE Global Shortcut → D-Bus).
- `2026-05-02` — User squashed the local commit history to 3 workstream-shaped commits (W1, W6, W9) and pushed to `origin/main`. No content lost; this is reorganization for upstream-readability if/when we decide to PR.
- `2026-05-02` — **W4a done as a quick refresh outside the workstream framework** (per user request: "before we discuss W2, can you do a quick refresh of the README"). README rewritten beyond original W4a scope: rustup install, KDE autostart `.desktop`, broadened platform-support claim, new "Text-insertion strategy" subsection in Architecture, KDE/systemd-logind reboot-vs-logout caveat, D-Bus shortcut-binding example. Apt deps corrected (clang/libclang-dev/libssl-dev/libpipewire-0.3-dev added; cargo/libgtk4-layer-shell-dev dropped). Committed direct on main, NOT on a workstream branch — the discipline of "always through a wN branch" was set aside intentionally for this small scoped refresh.
- `2026-05-02` — W2 plan written (`2026-05-02-w2-kde-global-shortcut.md`, 12 tasks, 696 lines) and execution begun. State: `pending` → `in-progress`. Branch: `w2-kde-global-shortcut`. Six must-fix corrections applied to W2 spec from review pass (kbuildsycoca6 not update-desktop-database; --timeout 1 on gdbus calls; gdbus prerequisite documented; per-user vs system install conflict documented; errors-invisible-from-shortcut-context documented; shell-action rationale corrected from "behavior switch" to "history/telemetry attribution"). Three new deferred-ideas surfaced (D-Bus .service activation file; ToggleRecording method/script; desktop-file-validate as CI gate).
- `2026-05-02` — Created W10 (KDE-native UX) at Phase 4; added Phase 4 description; appended three new deferred-ideas entries from W2 architect-review (D-Bus .service activation file; ToggleRecording method/script; desktop-file-validate CI gate). Separate commit per architect-review's "scope creep" feedback so reviewers can revert independently of W2's main deliverables.
- `2026-05-02` — **W2 done.** Per-user `.desktop` install completed via `install -m 644 ... ~/.local/share/applications/`; `desktop-file-validate` passes (one informational hint about `Utility;AudioVideo;` having two main categories — non-blocking); `kbuildsycoca6` rebuilt KDE service cache. Manual smoke: user bound `Super+X` to "Start dictation" and `Super+Z` to "Stop dictation" (instead of the suggested Meta+V default — user's own preference). Two dictation cycles fired via the KDE shortcut, both producing text in Kate (cold-spawn `insert=0.6s total=1.9s`; warm-path `insert=0.9s total=4.8s`). Helper spawn confirmed via `[Pepper X uinput] XKB layout 'us' loaded` in stderr. AT-SPI socket warning (`/run/user/1000/at-spi2-9VUYO3/socket`) appeared exactly as W9's design predicts and was correctly handled by the universal-uinput fallback path. **Not separately verified during this smoke** (out of scope, neither blocker): coexistence with the evdev `Alt+Super` hotkey (verified during W9 smoke; the two paths are independent), `StopRecording`-while-idle behavior (user bound the action and used it during recording cycles; idle case not isolated). State: `in-progress` → `done`. **current_workstream advances to W4b** (CI dependency-list fix).
