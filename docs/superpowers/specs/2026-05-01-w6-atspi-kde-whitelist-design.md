# W6 — AT-SPI App Whitelist Additions for KDE

Workstream W6 of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). First Phase 1 workstream after W1; promoted from Phase 2 on 2026-05-01 because W1 surfaced that it's the smallest change that unblocks daily-driver use of pepper-x on KDE.

## Goal

Extend `friendly_insert_target_class_from_application_id` in `crates/pepperx-platform-gnome/src/atspi.rs:315-339` to recognize six common KDE apps so that pepper-x's caret-aware AT-SPI insertion path fires for them, rather than falling through to the broken `UINPUT_TEXT_BACKEND_NAME` gate (see W9).

## Background — why this is Phase 1

W1's execution surfaced an upstream gap that wasn't visible during the original brainstorm: pepper-x is *designed* to fall back to a uinput helper for non-AT-SPI-whitelisted apps, but `ensure_runtime_supported_backend` at `atspi.rs:858-877` rejects `UINPUT_TEXT_BACKEND_NAME` with "not implemented yet" — so when a focused KDE app isn't in the whitelist, insertion silently fails after a successful transcription. The architectural fix lives in W9; W6 is the smaller patch that gets common KDE apps onto the AT-SPI path that **does** work, meeting the daily-driver bar without touching the broader fallback architecture.

## Done criteria

1. **Whitelist extended** — the `friendly_insert_target_class_from_application_id` `match` block contains entries for all six target apps (each in both bare-executable-name and `org.kde.<app>` reverse-DNS form), assigned to the categories below.
2. **Unit tests extended** — the existing tests for `friendly_insert_target_class_from_application_id` at `atspi.rs:2002` (`classify_text_editor_targets`), `:2014` (`..._browser_textarea_targets`), and `:2026` (`..._terminal_targets`) are extended to cover the 12 new strings (6 apps × 2 forms each). Each modified test (or new sibling test) keeps at least one pre-existing assertion alive as a regression check (e.g. `org.gnome.TextEditor → TextEditor`, `firefox → BrowserTextarea`, `xterm → Terminal`). Failure messages identify which row failed. **Plus** a small new test pinning `friendly_application_id_from_executable_name("kate") == "kate"` (and one for each of the 6 KDE basenames) so a future GNOME-style remap shim can't silently break KDE.
3. **Build green** — `cargo build --release`, `cargo test --workspace`, and `cargo clippy -- -D warnings` exit 0. (`cargo fmt --check` is excluded — see W1 findings; W4c will fix that drift.)
4. **Manual smoke pass in Kate** — hold-to-record into a Kate document, dictation appears at the cursor, caret-aware insertion confirmed (text inserts at cursor position even after focus games, distinguishing AT-SPI from plain uinput typing).
5. **Manual smoke pass in Konsole** — same as Kate but in a shell prompt.
6. **Roadmap updated** — W6 row state set to `done`; `current_workstream:` advances to W2; status log captures the W6 outcomes (smoke-pass status, any AT-SPI surprises).
7. **Deferred-ideas section added to roadmap** — new section seeded with the three entries identified during brainstorming.
8. **Branch merged to fork's `main`** — `w6-atspi-kde-whitelist` worktree branch merges cleanly via `--no-ff`, then is removed.

## Target apps and categorization

| App | Executable basename | Reverse-DNS form | Category |
|-----|--------------------|--------------------|----------|
| Kate | `kate` | `org.kde.kate` | `TextEditor` |
| KWrite | `kwrite` | `org.kde.kwrite` | `TextEditor` |
| Konsole | `konsole` | `org.kde.konsole` | `Terminal` |
| Kontact | `kontact` | `org.kde.kontact` | `TextEditor` |
| KMail | `kmail` | `org.kde.kmail` | `TextEditor` |
| Falkon | `falkon` | `org.kde.falkon` | `BrowserTextarea` |

Note: `org.kde.kmail` is the canonical KMail D-Bus name on modern KDE/Plasma 6 (verified via `/usr/share/dbus-1/services/org.kde.kmail.service` on the install). The reverse-DNS entries are defensive forward-compat — at runtime, pepper-x derives `application_id` from the executable basename via `friendly_application_id_from_executable_name` (`atspi.rs:1110-1116`), and KDE app basenames already match what we're whitelisting on. The reverse-DNS variants would only fire if the discovery logic ever changed to use D-Bus app IDs.

## Approach — minimal match-arm additions

Approach 1 of the three brainstormed (data-table refactor and sub-function split rejected as over-engineering for the current scale; both captured in the new roadmap "Refactor / enhancement ideas (deferred)" section). Single edit:

- **Modify** `crates/pepperx-platform-gnome/src/atspi.rs:315-339` — extend three `match` arms (`TextEditor`, `BrowserTextarea`, `Terminal`) with the new entries. Existing entries unchanged.
- **Extend** the existing per-category tests in the `#[cfg(test)] mod accessible_insert_runtime_helpers` block (starts at `atspi.rs:1937`): `classify_text_editor_targets` (`:2002`), `classify_browser_textarea_targets` (`:2014`), `classify_terminal_targets` (`:2026`). Each gets its respective new KDE entries plus a kept-alive existing-entry regression assertion.
- **Add** a small new test in the same module pinning `friendly_application_id_from_executable_name(<kde-basename>) == <kde-basename>` for the 6 KDE apps (asserts no remap is currently needed; surfaces if a future shim breaks the assumption).
- **No change** to `friendly_application_id_from_executable_name` at `atspi.rs:1110-1116`. KDE app executable names already match what we're whitelisting on; no remapping needed (unlike GNOME's `gnome-text-editor` → `org.gnome.TextEditor` shim).

Total code change: roughly 12-15 lines across one file (production) plus ~20 lines for the test.

## Testing strategy

### Unit tests (gating, run by `cargo test`)

Three existing tests for `friendly_insert_target_class_from_application_id` already live in `atspi.rs` (in the `mod accessible_insert_runtime_helpers` test module starting at `:1937`): `classify_text_editor_targets` (`:2002`), `classify_browser_textarea_targets` (`:2014`), `classify_terminal_targets` (`:2026`). W6 **extends** these — each gets its respective new KDE entries (TextEditor: kate/kwrite/kontact/kmail and reverse-DNS forms; BrowserTextarea: falkon and reverse-DNS form; Terminal: konsole and reverse-DNS form), and each retains at least one pre-existing assertion as a regression check.

In addition, W6 adds a small new test pinning `friendly_application_id_from_executable_name` to return the KDE basenames as-is (`kate → kate`, `kwrite → kwrite`, `konsole → konsole`, `kontact → kontact`, `kmail → kmail`, `falkon → falkon`). This locks the assumption that no executable-name remap is needed for KDE apps; a future GNOME-style shim that accidentally remaps a KDE basename would surface here.

Failure messages identify the specific row in all cases.

### Manual smoke gate (run by user, post-build, pre-merge)

The gate must **discriminate AT-SPI caret-aware insert from plain uinput typing**. Both backends place text at the cursor in a typical buffer — observationally indistinguishable to a human in the success case. The discriminator is **(a) `pepperx-uinput-helper` is NOT spawned during dictation** (AT-SPI insert never spawns the helper; uinput typing always does), and **(b) pepper-x stderr logs that an AT-SPI backend was selected**, not `uinput-text`. "Text appears at cursor" alone is not the gate.

1. `cargo build --release` from the W6 worktree.
2. `sudo install -m 755 target/release/pepper-x /usr/local/bin/`. Helper binaries are unchanged from W1; do not reinstall them.
3. Restart `pepper-x` (kill any running instance; launch from a regular terminal — `input` group is durable post-reboot). Capture stderr to a file so the log can be inspected: `pepper-x 2>&1 | tee /tmp/w6-smoke.log`.
4. **Kate test**:
   - Open Kate, focus a document, position cursor mid-line if possible.
   - Hold Alt+Super, dictate "hello world", release.
   - Confirm cleaned text appears at the cursor.
   - **Discriminator (a)**: in another terminal, immediately run `pgrep pepperx-uinput-helper`. Expected: empty (no PID). If a PID is returned, dictation went through plain uinput typing — W6's whitelist patch did NOT route Kate to the AT-SPI path.
   - **Discriminator (b)**: in `/tmp/w6-smoke.log`, find the log line for this dictation cycle and confirm an AT-SPI backend was selected (something like `friendly-insert` / `atspi-editable-text`), NOT `uinput-text`.
5. **Konsole test**: same as Kate but `echo hello` at a shell prompt. Both discriminators apply identically.
6. **If either discriminator fails** (helper spawned OR log shows uinput-text path): AT-SPI visibility on KDE is the suspect. Invoke `superpowers:systematic-debugging`. Possible root causes: `qt-at-spi` not registering Qt apps (especially Plasma 6 = Qt6; needs `QT_ACCESSIBILITY=1` env or the right plugin loaded), KDE Wayland-specific AT-SPI gaps, or session-bus/`XDG_RUNTIME_DIR` discrepancies. **Do not patch around** — surface to user. Decision tree if smoke fails: (a) if AT-SPI registry doesn't see Kate/Konsole at all, the whitelist patch is not the unblocker → **W9 promotion ahead of W2**; (b) if AT-SPI sees them but selection logic still picks uinput-text, escalate as a separate diagnostic before any further coding.

Falkon's manual smoke is excluded from the gate (Q2 Option C choice). It can be tested ad-hoc when convenient; failure would inform Falkon's `BrowserTextarea` category claim but doesn't block W6.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| AT-SPI registry doesn't see KDE Qt apps even with `at-spi2-core` running | Medium | Manual smoke catches this immediately. If it fires, **stop W6** — escalate to investigate AT-SPI visibility (`qt-at-spi` package, KDE Wayland config) before declaring W6 done. May force W9 promotion. |
| `org.kde.kmail` is the wrong D-Bus name on this KDE version | Low | Doesn't matter at runtime since the executable basename `kmail` is the form actually matched. Reverse-DNS entries are defensive forward-compat. |
| Existing entry regression (a GNOME app stops being whitelisted) | Low | Existing match arms structurally unchanged; only new alternatives added. `cargo test` regression-checks via existing tests. |
| Hotkey collision (KDE pre-binds Alt+Super) | Low | Independent of W6 — already documented in `pepper-x-install.md`. Rebind in pepper-x Settings → Recording. |
| `cargo clippy -- -D warnings` fails on the new test code | Low | Use `assert_eq!` with the message arg (no `panic!`/`unimplemented!`); avoid `#[allow(dead_code)]`. Standard Rust idioms. |

## Out of scope (explicit non-goals)

- **The W9 architectural fix.** `UINPUT_TEXT_BACKEND_NAME` "not implemented yet" gate stays. W6 routes KDE apps onto the AT-SPI path that already works; W9 makes the uinput fallback work for *any* focused app. Different workstream.
- **Additional KDE apps.** Dolphin, KDevelop, Krita, KCalc, KMail-as-Akonadi, etc. Adding any here is scope creep — promotion to a future W6.x or new workstream if real use-cases surface.
- **Falkon manual smoke.** Excluded from the gate per Q2 Option C; ad-hoc testing only.
- **Touching `friendly_application_id_from_executable_name`.** No remapping is needed for KDE apps; their basenames already match.
- **Pinning the Rust toolchain.** That's W4c's job. `cargo fmt --check` and `clippy -D warnings` are expected to fail on upstream code drift (per W1 findings) — W6's Done criteria explicitly excludes the fmt gate.
- **Upstream PR filing.** W6 lands locally on `lukepatrick/pepper-x:main` only. Filing a PR to `obra/pepper-x` is a separate manual decision later, drawing from the roadmap's "Potential upstream contributions" table.

## Auxiliary deliverables (folded into W6's commits)

Cheap and roadmap-adjacent, ride along on the W6 branch:

1. **Roadmap status table updates** — W6 row state transitions: `pending` → `planned` (when plan written) → `in-progress` (when execution starts) → `done` (when merged). `current_workstream:` line advances on each transition.
2. **Status-log entries** — one line per transition, plus a final summary capturing what worked, what didn't, and any surprises (especially smoke-test outcomes).
3. **New roadmap section: "Refactor / enhancement ideas (deferred)"** — initial entries:
   - **Whitelist data-table refactor** — replace the flat `match` in `atspi.rs:315-339` with a `static` array of `(app_id, FriendlyInsertTargetClass)` pairs. Trigger: whitelist grows past ~50 entries OR an entry needs metadata beyond the category enum.
   - **Whitelist split into GNOME/KDE sub-functions** — `..._for_gnome_app` and `..._for_kde_app` returning `Option<FriendlyInsertTargetClass>`, composed by the public function. Trigger: ≥10 KDE-specific entries accumulate, or the flat match becomes hard to scan in a single screen.
   - **Conditional helper stderr suppression** — make `fc04b8b`'s suppression toggleable via `PEPPERX_HELPER_STDERR=1` env var. Trigger: another debugging session blocked on invisible helper logs (W1 was the first; second occurrence promotes this to a workstream).

Future workstreams that surface deferred ideas append entries here rather than to ad-hoc places.

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. The plan will translate the four-phase structure (worktree setup → code change + unit tests → manual smoke → wrap-up) into ordered, individually-verifiable tasks with concrete commands and stop-on-failure guidance. Manual smoke remains a hard sync point requiring user action.
