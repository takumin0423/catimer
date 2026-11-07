# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

catimer is a Tauri 2 + React 19 + TypeScript (Vite) desktop app, currently still the unmodified `create-tauri-app` template (the `greet` command and demo UI are placeholders). No test runner, linter, or formatter is configured yet.

The chosen tech stack and development process are in `docs/tech-stack.md`. The product concept and MVP scope are in `docs/concept.md`; domain terms are in `GLOSSARY.md` (use its terms, not the listed _Avoid_ synonyms, in code, issues, and UI copy); hard-to-reverse decisions are in `docs/adr/`. Read these before designing or implementing a feature.

## Who writes what

The owner is using this project to learn Rust and Tauri, so the work is split:

- **`crates/catimer-core` (domain and persistence) is written by the human.** Do not write or edit its code, even when asked to implement a feature that touches it. Instead, suggest test cases, review, explain idiomatic Rust, and give hints. If the human explicitly asks you to write a specific piece of it, confirm once before doing so.
- **Wiring and the frontend are yours.** Tauri commands, plugins, tray, popover, build, CI, and React UI can be implemented by the agent.
- Issues follow the same split: domain sub-issues are labelled `ready-for-human`, wiring/UI sub-issues `ready-for-agent`.

## Commands

Package manager is **pnpm**.

- `pnpm tauri dev` — run the desktop app (starts Vite via `beforeDevCommand`, then builds/launches the Rust side)
- `pnpm dev` — frontend only in a browser at http://localhost:1420 (Tauri APIs like `invoke` won't work here)
- `pnpm build` — type-check (`tsc`) and build the frontend into `dist/`
- `pnpm tauri build` — production app bundle (runs `pnpm build` first)
- `cargo check` / `cargo test` / `cargo clippy` — run inside `src-tauri/` for the Rust side

## Architecture

Two halves communicating over Tauri IPC:

- **Frontend** (`src/`): React app bootstrapped from `index.html` → `src/main.tsx` → `src/App.tsx`. Calls Rust via `invoke("<command>", args)` from `@tauri-apps/api/core`.
- **Backend** (`src-tauri/`): Rust crate. `main.rs` only calls `catimer_lib::run()`; all logic lives in `src/lib.rs` (the `_lib` crate name is required to avoid a bin/lib name clash on Windows, and enables mobile entry points).

Adding a backend feature typically touches several places:
1. Define a `#[tauri::command]` fn in `src-tauri/src/lib.rs` (or a module it imports).
2. Register it in `tauri::generate_handler![...]` in `run()`.
3. Call it from the frontend with `invoke`.
4. If it uses a Tauri plugin, add the crate to `Cargo.toml`, `.plugin(...)` in `run()`, the npm package to `package.json`, **and** the plugin's permission to `src-tauri/capabilities/default.json` — calls fail at runtime without the capability entry.

## Config notes

- Vite dev server is pinned to port 1420 with `strictPort` (HMR on 1421 when `TAURI_DEV_HOST` is set); this must match `build.devUrl` in `src-tauri/tauri.conf.json`.
- Vite ignores `src-tauri/**` for watching; Rust changes are picked up by `tauri dev` itself.
- Bundle identifier: `dev.takumin0423.catimer`. CSP is currently disabled (`"csp": null`).
- Rust edition 2024; release profile uses LTO, `panic = "abort"`, and strip.

## Git workflow

Every task runs on its own branch and lands through a PR. `main` is protected by a ruleset (PR required, squash merge only, no force push), so a direct push to it is rejected.

1. Start from fresh `main`: `git switch main && git pull`.
2. Branch: `git switch -c <type>/<short-description>` (e.g. `feat/timer-display`, `fix/...`, `chore/...`).
3. Do the work and commit on that branch.
4. Push and open a PR against `main` with `gh pr create`. The squash commit on `main` takes the PR title and body, so write those as the final commit message.

### Git and `gh` under the Bash sandbox

The Bash sandbox is managed by the organization, so its settings cannot be loosened from this repo (sandbox keys such as `sandbox.excludedCommands` in `.claude/settings*.json` are ignored). Don't edit sandbox or permission settings to work around the rules below.

- `origin` uses HTTPS (`https://github.com/takumin0423/catimer.git`) because SSH cannot work inside the sandbox: port 22 is blocked and `~/.ssh` is unreadable. Keep it on HTTPS.
- `git pull` / `git fetch` work inside the sandbox once `github.com` is allowed for the command.
- `git push` and every `gh` command fail inside the sandbox because the macOS Keychain that holds the credentials is unreachable. Run them with the sandbox disabled from the first attempt instead of trying sandboxed first; each run still goes through approval.
- If that approval is denied, do not look for another way around it. Give the user the exact command to run themselves with the `!` prefix.

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `takumin0423/catimer`, managed via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
