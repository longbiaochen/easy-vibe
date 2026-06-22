# Repository Guidelines

## Project Structure & Module Organization

- `docs/`: VitePress site source (Markdown content, sidebar/nav, assets referenced by docs).
- `docs/.vitepress/theme/`: custom theme, global component registration in `index.js`, shared styles in `style.css`, layout in `Layout.vue`.
- `docs/.vitepress/theme/components/appendix/*/`: interactive Vue demos used inside appendix pages (e.g. `web-basics/`, `deployment/`).
- `assets/`: repo-level images/media (if referenced, prefer linking/copying into `docs/public/` or a doc-local folder when appropriate).
- `scripts/`, `tools/`, `update_readmes.cjs`: utility scripts for maintaining docs.

## Build, Test, and Development Commands

This repo is a VitePress (Vue 3) documentation project. Requires Node.js **>= 18**.

```bash
npm install
npm run dev      # start local docs server (hot reload)
npm run build    # production build (use as CI-style check)
npm run preview  # preview the built site locally
npm run verify   # lint and production build without rewriting tracked sitemap
npm run format   # run Prettier on the whole repo
```

## Coding Style & Naming Conventions

- Formatting: Prettier (`npm run format`). Keep diffs small and avoid reformatting unrelated files.
- Vue components: Vue 3 SFCs with `<script setup>`, PascalCase filenames (e.g. `SemanticTagsDemo.vue`).
- CSS: prefer VitePress theme variables (`var(--vp-c-*)`) and keep components responsive (`@media (max-width: 720px)` when needed).
- Docs: use clear headings and short paragraphs; components are referenced in Markdown as `<ComponentName />`.

## Testing Guidelines

There is no dedicated test framework in this repo. Use `npm run build` as the primary correctness check, and manually verify interactive components in `npm run dev`.

## Commit & Pull Request Guidelines

- Commits follow a Conventional Commits style seen in history: `feat: ...`, `fix: ...`, `docs: ...` (optionally scoped like `feat(docs): ...`).
- PRs should include: a short description, screenshots/GIFs for UI or component changes, and any relevant paths touched (e.g. `docs/zh-cn/appendix/...`, `docs/.vitepress/theme/...`).

## Configuration & Deployment Notes

- `vercel.json` is present; keep builds reproducible and avoid relying on local-only assets.
- GitHub Pages deploys from `.github/workflows/deploy.yml`. Upstream `datawhalechina/easy-vibe` deploys by default; forks must opt in with repo variable `ENABLE_PAGES_DEPLOY=true`.
- Fork Pages builds should use `BASE=/<repo-name>/` and `SITE_URL=https://<owner>.github.io/<repo-name>` unless `PAGES_BASE` or `PAGES_SITE_URL` repo variables override them.

## Ship-It Workflow

- When the user says `ship it`, preserve unrelated dirty files and only stage the intended changes.
- Run `npm run verify` before release. For UI/docs rendering changes, also run a local preview and verify unauthenticated/public rendering in the official Browser plugin; use the official Chrome plugin only when the flow depends on signed-in/default-profile state.
- Release path is direct to the fork `main`: commit, push `origin main`, wait for `Deploy VitePress site to Pages`, then verify the fork Pages URL `https://<owner>.github.io/<repo-name>/`. If no Pages run appears within 60 seconds after push, trigger the same workflow manually with `gh workflow run deploy.yml --repo <owner>/<repo-name> --ref main`.
- After the fork Pages preview is verified, summarize the diff against `upstream/main` and ask explicitly before creating a PR to `datawhalechina/easy-vibe`. Do not open the upstream PR without that approval.

## Browser Automation Constraint

- Follow the global `~/.codex/AGENTS.md` official browser/GUI policy: Browser plugin for unauthenticated local/public rendering, Chrome plugin for signed-in/default-profile browser state, and Computer Use only for native desktop boundaries.
- Keep only repo-specific verification surfaces here; do not copy the full global policy block into this runbook.

## Worktree Policy

- Follow the global `~/.codex/AGENTS.md` worktree-first rule for Codex development: new non-read-only coding or multi-file documentation tasks should start in a dedicated Codex-managed worktree.
- Use the Local checkout only for read-only investigation, final handoff/inspection, tasks that must reuse a single running app/server, or when the user explicitly asks to stay local.
- Branch names should use `codex/<repo>-<short-task>`; manual long-lived worktree directories should use `~/Projects/<repo>-<short-task>`.
- Initialize dependencies inside each worktree and keep ports, databases, device/simulator state, build outputs, and ignored local config isolated per checkout.
- Preserve existing dirty checkouts. Inspect `git status --short` before editing, and do not stash, commit, remove, or migrate user changes unless explicitly asked.
- After merge or abandonment, clean up with `git worktree remove <path>` and use `git worktree prune` only for stale metadata.
