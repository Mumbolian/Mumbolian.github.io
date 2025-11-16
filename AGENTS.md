# Repository Guidelines

## Project Structure & Module Organization
- Single-page client app; core code lives in `index.html` (prod) and `test/index.html` (active dev, hosted at https://t-man.net/test/).
- No build system or backend; all HTML, CSS, JS, and localStorage logic are in the HTML files. Assets (if added) should live beside the HTML.
- Work primarily in `test/index.html`; only copy to `index.html` with explicit approval.

## Build, Test, and Development Commands
- Serve locally from repo root: `python3 -m http.server 8000` (or `npx serve`) then open `http://localhost:8000/test/` (or `/` for prod view).
- No build step; changes are immediately reflected.
- Manual testing only; see scenarios below.

## Coding Style & Naming Conventions
- Vanilla JS (ES6+), inline CSS/JS within the HTML. Follow existing 2-space indentation and in-file patterns.
- Favor clear function names for scheduling logic (e.g., `computeSchedule`, `applyConstraints`); keep related helpers together.
- Avoid adding dependencies or external services. Persist via localStorage schema `napPlannerSettings_v6`.
- Keep comments minimal and purposeful; prefer readable code over heavy commentary.

## Testing Guidelines
- Manual validation required; key scenarios:
  - Default schedule (3 naps) renders with correct wake windows.
  - Mode toggles (2 ↔ 3 naps) preserve settings and recompute schedule.
  - Time constraints: “Must be awake by” and “Must be asleep during” adjust schedule without violating biological wake window limits.
  - Power Sleep Mode targets update schedules appropriately.
- If adding logic, include a brief checklist in PR notes describing tested paths.

## Commit & Pull Request Guidelines
- Branch discipline: work on `dev` (integration). Do **not** push to `main` directly.
- Versioning: bump `APP_VERSION` in `test/index.html` when shipping a feature; copy to `index.html` only after approval.
- Commit messages: concise imperative summary; include co-author note when generated with Claude tooling:
  ```
  git commit -m "Short description

  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude <noreply@anthropic.com>"
  ```
- PRs: describe user-visible changes, link related issues, and note manual test scenarios. Add screenshots/gifs for UI changes when feasible.

## Agent-Specific Notes
- Read `CLAUDE.md` for canonical repo policies (environment, worktrees, deployment). Honor branch protection and worktree rules; use `gh` for GitHub operations when needed.
