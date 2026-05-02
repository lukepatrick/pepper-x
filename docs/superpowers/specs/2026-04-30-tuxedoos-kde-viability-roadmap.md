# TuxedoOS/KDE Viability Roadmap

> **READ FIRST** every session. Single source of truth for "what's next" on this fork.

## Status

```
current_workstream:   W6 — AT-SPI app whitelist additions for KDE
phase:                1  (promoted from Phase 2 — see status log 2026-05-01)
state:                pending  (W6 awaits brainstorming; W1 done with caveats)
branch:               (none yet — created when W6 spec is written)
worktree:             (none yet)
last_updated:         2026-05-01
```

## Goal

Make `obra/pepper-x` viable on **TuxedoOS 24.04 + KDE Plasma + Wayland**. Daily-driver bar:

- `cargo build --release` succeeds.
- Hold-to-record types into Kate, KWrite, Konsole, Falkon/Firefox, Kontact. **Note (revised 2026-05-01 from W1 findings):** the plain-uinput fallback path is broken upstream for non-whitelisted apps (`ensure_runtime_supported_backend()` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet"). To meet the bar on KDE, we need either (a) **W6** AT-SPI whitelist additions so KDE apps take the AT-SPI path that *does* work, or (b) an architectural fix to the uinput-text fallback. W6 is the smaller change and now Phase 1.
- KDE Global Shortcut bound to the D-Bus service `com.obra.PepperX.Service` so dictation can be triggered without the GNOME tray icon.
- Pepper X autostarts on KDE login so the D-Bus service is available when the shortcut fires.

All work merges to `lukepatrick/pepper-x:main`. **No upstream PRs filed from this workflow** — upstreaming is a separate manual decision deferred until local vetting is complete (see "Potential upstream contributions" below).

## Workstreams

| ID | Phase | Title | State | Branch | Spec | Plan | Upstream candidate | Notes |
|----|-------|-------|-------|--------|------|------|---|---|
| W1 | 1 | Build + first-run on TuxedoOS | `done (caveats)` | `w1-build-and-first-run` (merged) | `2026-04-30-w1-build-and-first-run-design.md` | `2026-04-30-w1-build-and-first-run.md` | partial | Build, ASR, cleanup pipeline all verified end-to-end on TuxedoOS/KDE. Hotkey detection works (with `input` group; reboot required for it to take effect after `usermod`). **End-to-end text-insertion into KDE apps does NOT work** — uinput-helper fallback is gated off upstream. See W1 status-log entry 2026-05-01 and `pepper-x-install.md` "W1 execution findings" section. Apt-deps additions: `libssl-dev`, `libpipewire-0.3-dev` (also missing from upstream README). |
| W6 | 1 | AT-SPI app whitelist additions for KDE | `pending` | — | — | — | yes | **Promoted from Phase 2 to Phase 1 on 2026-05-01.** Without this, no text appears in Kate/Konsole/etc. — the AT-SPI path is the only insertion path that actually works upstream right now. Add Kate/KWrite/Konsole/Kontact/Falkon to `friendly_insert_target_class_from_application_id` at `crates/pepperx-platform-gnome/src/atspi.rs:315-339`. Smallest code change that meets the daily-driver bar. |
| W2 | 1 | KDE Global Shortcut → D-Bus | `pending` | — | — | — | no | Needs W1 + W6 (D-Bus service registers OK; insertion path needs to actually work for the shortcut to be useful). KDE-specific glue; not relevant to GNOME users. |
| W4a | 2 | README dependency-list fix | `pending` | — | — | — | yes | Drop `cargo` (apt) and `libgtk4-layer-shell-dev`; add `clang`, `libclang-dev`, **`libssl-dev`, `libpipewire-0.3-dev`** (last two added 2026-05-01 from W1 findings); note Ubuntu 24.04. Doc-only. |
| W4b | 2 | CI dependency-list fix | `pending` | — | — | — | yes | Add `clang`, `libclang-dev`, `libssl-dev`, `libpipewire-0.3-dev` to `.github/workflows/ci.yml`. Latent bug — passes today only because GitHub runners ship some of these. |
| W4c | 2 | Pin Rust toolchain version | `pending` | — | — | — | yes | **New 2026-05-01 from W1 findings.** Add `rust-toolchain.toml` with a pinned stable version. Upstream's `dtolnay/rust-toolchain@stable` is a moving target; with rustc 1.95.0, upstream main fails both `cargo fmt --check` and `cargo clippy -- -D warnings`. Pinning fixes the drift. |
| W5 | 2 | `dev-install-extension.sh` graceful skip on non-GNOME | `pending` | — | — | — | yes | Real check is at `scripts/dev-install-extension.sh:161-164` (not 90-93 as install.md originally claimed — fixed 2026-05-01). Exit 0 with friendly message. |
| W7 | 2 | OCR portal-response fix + rewire | `pending` | — | — | — | yes | Two-part: (1) fix `screenshot.rs:95-161` to use the XDG portal response signal instead of scraping `~/Pictures/`, (2) rewire `crates/pepperx-platform-gnome/src/context.rs:50-68` so the public `capture_supporting_context()` actually calls `screenshot_window`. The two are not currently wired together. Highest code risk in Phase 2. |
| W9 | 2 | Fix uinput-text fallback path (architectural) | `pending` | — | — | — | yes | **New 2026-05-01 from W1 findings.** Architecture in `app/src/transcription.rs:474-481` is *designed* to fall back to a uinput helper when AT-SPI fails, but `ensure_runtime_supported_backend()` at `crates/pepperx-platform-gnome/src/atspi.rs:858-877` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet". The headline KDE-viability issue. W6 is a sufficient workaround for whitelisted KDE apps; W9 is the durable fix that makes pepper-x work for *any* focused app on any desktop. Bigger than W6; needs its own brainstorm. |
| W8 | 3 | Platform crate split: `pepperx-platform-linux` ↔ `pepperx-platform-gnome` | `conditional` | — | — | — | maybe | Triggered only by gate criteria below. |

State legend: `pending` → `spec` → `planned` → `in-progress` → `done` (or `blocked` / `conditional`).

## Phase descriptions

**Phase 1 — Get-it-running.** Daily-driver bar above. W1 verified the build/transcription pipeline end-to-end but exposed that **insertion is broken upstream** for non-whitelisted apps. W6 was promoted from Phase 2 to Phase 1 because it's the smallest change that unblocks daily-driver use. W2 (KDE shortcut) follows W6.

**Phase 2 — Polish & fixes.** Independent fork-local improvements; most are also future upstream candidates. Revised order on 2026-05-01: W4a → W4b → W4c → W5 → W7 → W9 (W9 last because it's the deepest architectural change and may need its own multi-iteration cycle; W4c is new; W6 graduated to Phase 1).

**Phase 3 — Architectural (conditional).** Crate restructure work that is only worth doing if local maintenance friction makes the case. Not on the critical path.

## W8 decision-gate triggers

W8 is `conditional`. Self-determined — promote it to `pending` (and brainstorm it properly) if **any one** of:

- **Patch count**: 3+ KDE-specific patches accumulate on `main` that conflict awkwardly with `pepperx-platform-gnome` boundaries.
- **Maintenance friction**: a single fix requires changes in 3+ files across the platform crate, suggesting bad seams.
- **Architectural opportunity**: you're already touching the platform crate for another reason and can fold the split in cheaply.

When triggered, **stop the current workstream**, surface the trigger to the user, and run `comprehensive-review:architect-review` before writing-plans (W8 is a structural change).

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
