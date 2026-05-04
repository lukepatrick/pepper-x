# W5 — `dev-install-extension.sh` graceful skip on non-GNOME

## Goal

`scripts/dev-install-extension.sh` currently exits 1 with the message *"gnome-extensions is required to install the Pepper X extension"* when the `gnome-extensions` CLI isn't on PATH. That's the case on every non-GNOME desktop (KDE Plasma, Sway, headless CI, anywhere `gnome-shell` isn't installed). On those systems, the script's purpose is moot — there's no GNOME Shell to install an extension into — but the hard error makes the install path noisy and forces every non-GNOME consumer (CLAUDE.md, README.md, `pepper-x-install.md`) to document a "skip this step" workaround.

W5 changes the script's behavior on non-GNOME from **"hard error → caller must skip"** to **"clean self-skip with a one-line informational message"**. Doc references to the old "exits with error" behavior get refreshed in the same workstream.

## Scope

In scope:
1. Edit `scripts/dev-install-extension.sh`: change the `gnome-extensions` guard from `exit 1` to `exit 0` with a friendlier message.
2. Refresh five doc references in `pepper-x-install.md`, plus one each in `CLAUDE.md` and `README.md`.
3. Manual KDE smoke test on the dev box (TuxedoOS).
4. Roadmap state transition: W5 `pending → in-progress → done`; status-log entry; `current_workstream` advances to W7.

Out of scope:
- Automated bash-script test for the skip behavior (no precedent in this repo for shell-script unit tests; consistent with W2/W4b/W4c handling — manual smoke is the established pattern for script changes).
- Restructuring `check_extension` order to fail-fast on missing GNOME tooling before running `python3`. Discussed under "Risks acknowledged" below — the current ordering preserves `--check` mode semantics.
- Verifying behavior on GNOME (not testable on the dev box without a VM; relies on no-functional-change reasoning since the only edited lines are inside the `! command -v gnome-extensions` branch, which is unreachable when the CLI exists).
- GNOME extension code changes (out of W5 scope; there are no functional defects in the extension itself driving this work).

## Approach

**Approach A** (chosen). Detection method: keep the existing `command -v gnome-extensions` guard. If the CLI is missing for any reason — non-GNOME desktop, headless CI, broken GNOME install — the script's install actions are skipped. KISS, no new flag, no `XDG_CURRENT_DESKTOP` sniffing.

Approaches B (also check `XDG_CURRENT_DESKTOP` to distinguish "non-GNOME desktop" from "GNOME with broken tooling") and C (add a `--graceful-skip` opt-in flag) were considered and rejected. B introduces extra complexity for a narrow diagnostic distinction that doesn't change what the script can do; C is more conservative for upstream-PR review but adds a flag for behavior the user can already observe via the new exit-0 message.

## The change

`scripts/dev-install-extension.sh` lines 161–164:

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

The four-line block stays at the same location in the script (after `check_extension` at `:103`, after `--check` mode at `:157-159`). `check_extension` keeps running on all platforms — its file-existence + regex + AST checks are useful as a sanity gate even when install will skip, and `--check` mode (already present at `:157-159`) deliberately depends on it for pre-deployment validation.

The message wording deliberately points at the README section rather than naming the D-Bus service / specific bind mechanism, so the script doesn't drift if W2's KDE-trigger recipe evolves.

## Doc cleanup

Seven references across three files contradict the new behavior:

1. `CLAUDE.md:79` — current text: *"the script exits with an error if `gnome-extensions` isn't on PATH"*. Update to: *"the script exits cleanly with a skip message on non-GNOME (W5 fix)"*.

2. `README.md:101` — current text: *"`scripts/dev-install-extension.sh` will exit with an error if `gnome-extensions` isn't on PATH"*. Drop the warning; replace with *"is safe to run on non-GNOME — it skips cleanly with an informational message"*.

3. `pepper-x-install.md:30` — current text: *"Hard requirement on `gnome-shell` 48+. **Skip the install step entirely.**"* Soften to: *"Auto-skipped on non-GNOME after W5; running the script there is a no-op."*

4. `pepper-x-install.md:96` — the `[ ] **Skip**` checklist item. Update from *"requires `gnome-extensions` and exits with an error on KDE"* to *"on KDE, the script auto-skips after W5 — running it is harmless"*.

5. `pepper-x-install.md:204` — current text: *"exits with error if `gnome-extensions` not on PATH. Just don't run this script."* Update to: *"the script auto-skips on non-GNOME (W5)."*

6. `pepper-x-install.md:233` — current text describes the W5 design proposal. Mark as DONE with the W5 commit reference.

7. `pepper-x-install.md:283` — current text: *"Install binaries; **skip** the GNOME extension script."* Update to: *"the GNOME extension script auto-skips on non-GNOME (W5)."*

`gnome-extension/README.md:18` doesn't promise an error on non-GNOME, so it's fine as-is. Optional: add a one-line "(safe to run on non-GNOME — exits cleanly)" reassurance there, but no functional change.

## Testing

1. **Manual KDE smoke** (the dev box, TuxedoOS): run `bash scripts/dev-install-extension.sh` (no flag) and confirm:
   - Exit status is 0.
   - The two-line skip message prints to stderr.
   - No files created in `~/.local/share/gnome-shell/extensions/pepperx@obra/` (verify with `ls`).

2. **Manual KDE `--check` mode**: run `bash scripts/dev-install-extension.sh --check` and confirm:
   - Exit status is 0 (unchanged from before W5).
   - `check_extension` runs (validates extension code) and produces no errors.

3. **`scripts/verify-extension-install.sh`** (the existing test that wraps `--check`): runs cleanly. No regression possible since `--check` short-circuits at `:157-159` before reaching the W5-edited lines.

4. **GNOME path**: not testable on the dev box. Relies on no-functional-change reasoning — the only edited lines are inside the `! command -v gnome-extensions` branch, which is unreachable when the CLI is present.

## Done criteria

1. `scripts/dev-install-extension.sh:161-164` matches the "After" block in "The change" section above.
2. Manual KDE smoke (item 1 in Testing) confirms exit 0 + skip message + no files installed.
3. Manual KDE `--check` smoke (item 2) confirms unchanged behavior.
4. `scripts/verify-extension-install.sh` runs without error.
5. The seven doc references in "Doc cleanup" are refreshed.
6. Roadmap updated: W5 row → `done`; status-log entry; `current_workstream` advances to W7.
7. The status-log entry explicitly notes the W1-era doc-fix loop (line citation `:90-93` → `:161-164`) closes here, matching W4c's log style.

## Risks acknowledged

| Risk | Mitigation |
|---|---|
| `check_extension` (`:19-103`) calls `python3` and runs unconditionally. On a barebones non-GNOME box without `python3`, the script still exits 1 before reaching the new graceful-skip branch (python3 missing). | **Accepted.** `python3` is essentially always present on Linux desktops (Ubuntu/Debian/Fedora/Arch all ship it by default). Restructuring to fail-fast on missing GNOME tooling above `check_extension` would break `--check` mode's deliberate "validate extension code on any platform" semantics (its sole pre-deployment use case). |
| A genuinely-broken GNOME install (`gnome-shell` present but `gnome-extensions` CLI accidentally removed) now silently skips instead of erroring. | **Accepted.** This is the trade-off of Approach A. Vanishingly rare; if a user runs into it, the skip message is informative enough that they can debug. |
| `set -euo pipefail` (`:3`) interaction with the new `if ! command -v ...` branch. | **None.** The `if !` swallows the failure exit cleanly; bash spec-compliant. Flagged for cosmetic awareness only. |
| The script's error-message wording change might break a caller that parses script output. | **None.** Caller analysis confirmed: only `scripts/verify-extension-install.sh:19` invokes the script, and it uses `--check` mode (which doesn't reach the edited lines). No CI / packaging / test caller relies on `exit 1`. |
| Doc-update sweep misses a stale reference. | **Mitigated.** Reviewer pass already surfaced two `pepper-x-install.md` references the original brainstorm missed (`:30`, `:283`). Done criteria #5 enumerates seven specific lines so executor knows exactly what to touch. |

## Workstream shape

Single-file behavior change + 3-file doc sweep + manual smoke. Estimated commits: 2 (one for the script + docs; one for the roadmap state transition). Estimated lines changed: ~15. Comparable to W4a in scope.

This is upstream-PR-candidate yes — the change is desktop-agnostic and benefits all non-GNOME pepper-x users. The condensed `W5: dev-install-extension.sh graceful skip on non-GNOME` commit is the natural shape for an upstream cherry-pick if/when filed.
