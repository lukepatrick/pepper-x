# W4b — CI Dependency-List Fix + Lightweight Hardening

Workstream W4b of the [TuxedoOS/KDE viability roadmap](2026-04-30-tuxedoos-kde-viability-roadmap.md). Small, mechanical change to `.github/workflows/ci.yml`. No production-code changes.

## Goal

Fix the latent fragility in `.github/workflows/ci.yml` discovered during W1: the apt list is missing four packages (`clang`, `libclang-dev`, `libssl-dev`, `libpipewire-0.3-dev`) that the build genuinely needs but happen to be pre-installed on GitHub's `ubuntu-latest` runners today. Also drop one dead reference (`libgtk4-layer-shell-dev`), add `--locked` to cargo invocations, add a missing `cargo build --release` step, and fold in four CI-hygiene additions surfaced by a `cicd-automation:deployment-engineer` review.

## Background — why this is Phase 2

W1 surfaced that pepper-x's `cargo build` requires `libssl-dev` and `libpipewire-0.3-dev` (transitively pulled by `ort-sys` via `ureq` → `native-tls`, and by the `pipewire` crate respectively). W4a corrected the README's apt list. W4b mirrors that correction in CI plus a couple of related improvements that fit the "while we're touching ci.yml" boundary without crossing into W11's caching/matrix territory.

The `cicd-automation:deployment-engineer` review during W4b brainstorm flagged four CI-hygiene additions worth folding in (timeout-minutes, permissions block, concurrency control, CARGO_TERM_COLOR) that are tiny YAML additions but real wins. The user approved folding them in.

## Done criteria

1. **`.github/workflows/ci.yml` updated** with the final form below (Section "Final ci.yml"):
   - Apt list: `clang`, `libclang-dev`, `libpipewire-0.3-dev`, `libssl-dev` added; `libgtk4-layer-shell-dev` removed; alphabetized for diff readability.
   - `--locked` added to `cargo clippy`, `cargo test`, and the new `cargo build`.
   - New `cargo build --locked --release` step added at the end of the job.
   - `pull_request:` trigger gains a `branches: [main]` filter for clarity.
   - Top-level `permissions: contents: read` added.
   - Top-level `concurrency:` block (group keyed on `${{ github.ref }}`, `cancel-in-progress: true`).
   - Top-level `env: CARGO_TERM_COLOR: always`.
   - Job gains `timeout-minutes: 30`.
2. **CI actually passes** on a push to `main` (or via a test branch). Verified by inspecting the GitHub Actions run page; release-build step completes; no apt-install warnings about missing packages.
3. **Roadmap updates**:
   - W4b row state advances `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populated.
   - `current_workstream:` advances to W4c on completion.
   - New W11 row added (Phase 4 — CI performance + matrix expansion) in a separate commit so reviewers can revert independently.
   - One new entry appended to "Refactor / enhancement ideas (deferred)" — defensive `libudev-dev`/`ca-certificates` listing — in the same separate commit.
4. **Branch merged to main** and worktree cleaned up.

## Approach

### Final `ci.yml` content

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

### Justification for each change (cargo-lock-traced)

- **`clang` + `libclang-dev`**: `crates/pepperx-cleanup-helper/Cargo.toml:12-13` pulls `llama-cpp-sys-4` 0.2.23 directly; that crate's `build.rs` uses bindgen, which requires `libclang.so` at build time.
- **`libssl-dev`**: `crates/pepperx-models/Cargo.toml:10` declares `ureq = "2.12"`, which transitively pulls `native-tls` → `openssl-sys` 0.9.112 (`Cargo.lock:1770`). `openssl-sys` build needs the OpenSSL headers.
- **`libpipewire-0.3-dev`**: `crates/pepperx-audio/Cargo.toml:12` declares `pipewire = "0.9.2"` (Linux-only target). Transitively pulls `libspa-sys` and `pipewire-sys`, both resolved via pkg-config.
- **`libgtk4-layer-shell-dev` removed**: zero matches in any `Cargo.toml`/`Cargo.lock`/source file. Verified during W4a brainstorm.
- **`--locked`**: forces CI to use `Cargo.lock` verbatim; fails immediately if the lockfile would be regenerated. Forces dep bumps to be intentional commits. Fact-check confirmed current lockfile is consistent (`cargo update --workspace --dry-run` returns "Locking 0 packages").
- **`cargo build --locked --release`**: closes the "test passed but release build broken" gap. Compiles the entire dep graph with `--release` semantics, exercising bindgen-driven sys crates more aggressively than `cargo test` does. Would have caught W1's `openssl-sys missing` and `libspa-sys missing` failures before they hit a developer's local environment.
- **`pull_request: branches: [main]`**: scopes the trigger; clarifies intent. Currently `pull_request:` with no filter triggers on all PR targets including any future feature branches.
- **`permissions: contents: read`**: least-privilege. This CI doesn't push, comment, or modify GitHub state — only reads source. Default is read-write; tightening costs nothing.
- **`concurrency:` with `cancel-in-progress: true`**: cancels superseded runs when pushing rapid commits to the same branch (typical PR iteration pattern). Saves CI minutes.
- **`env: CARGO_TERM_COLOR: always`**: GitHub Actions strips ANSI by default; this re-enables colored cargo output in the run logs.
- **`timeout-minutes: 30`**: default is 6 hours. Bindgen + llama-cpp-sys-4 + ort can hang if a network step times out; 30 minutes is more than 2× the longest expected build time.

### What's intentionally NOT here

Per the deployment-engineer review's recommendations against scope creep:
- **`RUSTFLAGS: -D warnings`** — would conflict with clippy's `-D warnings` and risks failing release/test builds on benign rustc warnings unrelated to clippy lints.
- **SHA-pinning of GitHub Actions** — supply-chain hardening; out of scope for a personal fork. `@v4` and `@stable` are appropriate.
- **`cargo update --locked --dry-run` as an explicit gate** — `--locked` on each cargo step covers the same concern.

## Testing strategy

### Unit tests / static checks

`desktop-file-validate`-equivalent for ci.yml: `actionlint` exists but is not currently in the apt list and would be a new tool dependency. **Not adding it for W4b** — it would be an actionlint-as-CI-gate idea worth its own future workstream if regressions surface. For W4b, the spec's manual review + the `act` (or actual GitHub) CI run is the verification.

### Manual verification

After committing and pushing the change:

1. **Watch the next CI run.** GitHub Actions web UI shows whether the run succeeds. Specifically:
   - All seven steps green (checkout → install deps → set up Rust → fmt → clippy → test → build release).
   - No "package not found" errors in the apt-install step.
   - No "lockfile would need updating" errors from `--locked`.
   - Release build completes.
   - Total wall-clock time within `timeout-minutes: 30`.
2. **Confirm `concurrency` works.** Push two commits in rapid succession to a test branch; the older run should show as "Cancelled" in the Actions UI. (Optional verification; not gating.)
3. **Confirm `permissions: contents: read` doesn't break the run.** The first run after the change is the test — if anything in CI silently relied on write permissions, it'd fail.

### What this can't easily test locally

The runner-image-specific behavior (which packages are pre-installed on `ubuntu-latest`) can't be reproduced on TuxedoOS or any other local environment. The verification requires an actual CI run.

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| New `cargo build --release` step ~2x runtime in dep graph compile (debug then release, no cache) | Medium | Acceptable cost; caching deferred to W11. Documented as known-cost so it's not re-litigated. |
| `--locked` causes CI failure when Cargo.lock drifts from Cargo.toml | Low | Fact-check confirmed lockfile is currently consistent. Future drift is a feature: forces intentional dep-bump commits. |
| `concurrency: cancel-in-progress: true` cancels in-flight runs on rapid pushes | Low | Intended behavior; saves CI minutes. Cancelled runs show as "cancelled" rather than "failed" — fine. |
| `permissions: contents: read` breaks something that needs write | Low | This CI doesn't push, comment, or modify any GitHub resource. If a future workflow needs write, it'll declare its own `permissions:` block. |
| GitHub `ubuntu-latest` rolls forward and changes pre-installed package set | Low | The four explicit additions protect against most rolls. `libudev-dev`/`ca-certificates` not added — pre-installed today; a future runner change could break this. Captured as a deferred-ideas entry. |
| Spec discrepancy between actual ci.yml content and README/install.md apt list | Low | W4a already harmonized README/install.md. W4b harmonizes CI. Both lists now match modulo runtime-only packages (ca-certificates etc. that are pre-installed on dev machines and runners). |

## Out of scope

- **Caching (`Swatinem/rust-cache` or manual)** — W11 territory.
- **Multi-version Ubuntu matrix** — W11 territory.
- **Parallel job split** — W11 territory.
- **`rust-toolchain.toml` for rustup pinning** — W4c.
- **SHA-pinning GitHub Actions** — supply-chain hardening; out of scope.
- **`libudev-dev`, `ca-certificates`** as defensive apt entries — pre-installed today; deferred-ideas candidate if a future runner change breaks the build.
- **Dropping `libgirepository1.0-dev` / moving `tesseract-ocr` to runtime-only** — agent suggested potential cleanup, but verification is out of W4b scope; deferred-ideas candidate.
- **`actionlint` as a committed CI gate** — separate future workstream if YAML regressions become a real problem.
- **Upstream PR filing** — local-only per established pattern.

## Auxiliary deliverables (folded into W4b's commits)

Per the established W2 pattern, the new-workstream + deferred-ideas additions go in a separate commit on the W4b branch so reviewers can revert independently of the main `ci.yml` change.

1. **Roadmap state transitions** — W4b row `pending` → `planned` → `in-progress` → `done`. Spec/Plan columns populate. `current_workstream:` advances to W4c.
2. **Status-log entries** — one line per transition; final entry captures CI run outcome (green / red and what was caught).
3. **New workstream W11** added to the workstream table at Phase 4 (alongside W10), in a separate commit:
   - `W11 | 4 | CI performance + matrix expansion | pending | — | — | — | maybe | New 2026-05-02 from W4b brainstorm. Comprises C deferred from W4b: cache target/ between runs (Swatinem/rust-cache), split jobs (build/test/lint) for parallelism, matrix testing on multiple Ubuntu versions. Trigger: CI runtime becomes painful, OR a runner-image change breaks the build and we want defensive pre-installation.`
4. **One new entry appended to "Refactor / enhancement ideas (deferred)"**:
   - **Defensive apt listing for runner-rollforward protection** — explicitly add `libudev-dev` and `ca-certificates` to the CI apt list (currently pre-installed on `ubuntu-latest` but a future image change could break the build silently). Same trigger as W11's runner-image-change clause.

## Hand-off to writing-plans

Terminal state of this brainstorm: invoke `superpowers:writing-plans` with this spec as input. Plan will translate the four-stage structure (worktree setup → ci.yml edit + commit → W11 + deferred-ideas commit → push and verify CI run) into ordered, individually-verifiable tasks. The CI run verification is the soft user-action checkpoint (looking at the GitHub Actions web UI); everything else is autonomous.
