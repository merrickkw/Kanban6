---
description: Security-scan, then push to GitHub, update README + repo About, and deploy GitHub Pages via Actions
argument-hint: "[owner/repo or repo URL] [branch]"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(gitleaks:*), Read, Write, Edit, Glob, Grep
---

# Publish to GitHub

Publish this project to GitHub: push code, maintain the README, set the repo "About", and deploy GitHub Pages through a GitHub Actions workflow. **A security scan must pass before anything leaves this machine.**

Arguments: `$ARGUMENTS`

## 0. Gather repo details

1. Parse `$ARGUMENTS` for a repo (`owner/repo`, `https://github.com/owner/repo(.git)`, or `git@github.com:owner/repo.git`) and an optional branch (default: current branch, else `main`).
2. If no repo was given, check `git remote get-url origin`. If an origin exists, ask the user to confirm it is the target; if none exists, ask the user for the repo details. Do not guess.
3. Verify tooling: `gh auth status` must succeed. If not, stop and tell the user to run `gh auth login`.
4. Check whether the repo exists: `gh repo view <owner/repo> --json name,visibility,description,homepageUrl,repositoryTopics`. If it does not exist, ask the user whether to create it and with what visibility (public/private) before running `gh repo create`. Note: GitHub Pages on a private repo needs a paid plan.

## 1. Security scan (blocking gate)

Run every check below against **all files that would be pushed** (`git ls-files` plus untracked, non-ignored files from `git status --porcelain`), and against **git history** for secrets (`git log -p --all`). Report findings as a table: severity, file:line, finding, recommended action.

1. **Secret scanning**
   - If `gitleaks` is installed: `gitleaks detect --source . --redact --verbose` (covers history) and `gitleaks protect --staged --redact`.
   - Otherwise (and in addition), Grep for: AWS keys (`AKIA[0-9A-Z]{16}`), GitHub tokens (`gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_`), Slack tokens (`xox[baprs]-`), Google API keys (`AIza[0-9A-Za-z_-]{35}`), private keys (`-----BEGIN .*PRIVATE KEY-----`), JWTs (`eyJ[A-Za-z0-9_-]+\.eyJ`), and generic `(api[_-]?key|secret|token|password|passwd|bearer)\s*[:=]\s*['"][^'"]{8,}`.
2. **Sensitive files**: flag `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.sqlite`/`*.db`, credential/config dumps, `.claude/settings.local.json`, and anything over 5 MB. Ensure such files are in `.gitignore` (create/update it; always ignore `.claude/settings.local.json`).
3. **Personal data / email exposure**: Grep for email addresses and phone numbers. Per CLAUDE.md, the notification email must appear **only** inside `FORMSUBMIT_ENDPOINT` in `index.html`; any other occurrence is a failure. Warn the user explicitly that publishing the repo and Pages site makes that FormSubmit email address publicly visible, and get confirmation before continuing.
4. **Project constraints (CLAUDE.md)**: confirm no real UOB logo/trademark imagery or imitation of official systems, no external scripts/CDNs/web fonts, no `localStorage`/`sessionStorage`/`indexedDB`/`document.cookie`, and the only network call is the FormSubmit endpoint (Grep for `fetch(`, `XMLHttpRequest`, `http://`, `https://` and review each hit).
5. **Client-side code hygiene** in `index.html`: user strings go through `escapeHtml()` before HTML insertion; flag `eval(`, `new Function(`, `document.write(`, and unescaped `innerHTML` with user input.
6. **Workflow safety** (for `.github/workflows/*`): least-privilege `permissions`, no secrets echoed, no `pull_request_target` with checkout of PR code.

**Gate rule:** If any HIGH/CRITICAL finding exists (secrets, keys, sensitive files, or a secret already in history), **STOP**. Do not push, do not edit the remote repo, do not deploy. Explain how to fix it (for history secrets: rotate the credential first, then rewrite history) and wait for the user. MEDIUM/LOW findings: show them and ask the user whether to proceed.

## 2. Create / update README.md

Read the existing `README.md`, `CLAUDE.md`, and `index.html`, then write a README that preserves any user-written content still accurate and covers:
- Title and one-paragraph summary (demo/training Kanban board for a fictitious bank's IT PMO; not affiliated with any real bank).
- **Live demo** link: `https://<owner>.github.io/<repo>/` (or the custom domain if one is configured).
- Features (columns, add/move/delete tasks, filters, drag & drop, overdue highlighting, email notification via FormSubmit).
- How to run locally (open `index.html` in a browser; no build/server).
- Note that there is no persistence: refresh resets to seeded demo data.
- Tech constraints (vanilla HTML/CSS/JS, single file, no dependencies).
- Deployment (GitHub Pages via the Actions workflow in `.github/workflows/pages.yml`).
- Disclaimer (fictitious data, demo use only).

Do not put the notification email address in the README.

## 3. Create / update the GitHub Pages workflow

Create or update `.github/workflows/pages.yml`. If one exists, read it and modify only what is needed. Target content:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [<branch>]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Prepare site
        run: |
          mkdir -p _site
          cp index.html _site/
          touch _site/.nojekyll
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site
      - id: deployment
        uses: actions/deploy-pages@v4
```

Only `index.html` is published to the site (keeps `CLAUDE.md`, `.claude/`, and other repo files off Pages). If the project later gains more site assets, add them to the "Prepare site" step.

## 4. Re-scan, commit, and push

1. Re-run the secret and sensitive-file checks from step 1 on the final set of changed files (README, workflow, `.gitignore`).
2. Show `git status` and `git diff --stat`, then stage specific files (no blind `git add -A` if untracked files were flagged).
3. Commit with a clear message ending with the attribution trailer configured for this session.
4. If no `origin`, `git remote add origin https://github.com/<owner>/<repo>.git`; if `origin` points elsewhere, ask before changing it.
5. `git push -u origin <branch>`. Never force-push; if the push is rejected, stop and ask the user (offer `git pull --rebase`).

## 5. Enable Pages and update the repo "About"

1. Enable Pages with Actions as the source:
   - `gh api repos/<owner>/<repo>/pages --method POST -f build_type=workflow`
   - If it already exists (HTTP 409), use `--method PUT -f build_type=workflow` instead.
2. Update the About panel (show the user the proposed values first if the repo already has a description or topics they set):
   ```
   gh repo edit <owner>/<repo> \
     --description "Single-file vanilla JS Kanban board demo for an IT PMO (fictitious bank, training use)" \
     --homepage "https://<owner>.github.io/<repo>/" \
     --add-topic kanban --add-topic vanilla-javascript --add-topic html-css-js \
     --add-topic project-management --add-topic github-pages --add-topic demo
   ```

## 6. Verify and report

1. Find the triggered run: `gh run list --workflow pages.yml --limit 1`, then `gh run watch <id> --exit-status` (run in background if long). If it fails, show `gh run view <id> --log-failed` and diagnose.
2. Confirm the site URL: `gh api repos/<owner>/<repo>/pages --jq .html_url`.
3. Final summary for the user:
   - Security scan result (findings and what was done about each)
   - Commit SHA and branch pushed
   - README: created or updated
   - About: description / homepage / topics set
   - Workflow: created or updated, run status
   - Live site URL (first deployment can take a minute or two to go live)
