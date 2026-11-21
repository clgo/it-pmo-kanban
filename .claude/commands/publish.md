---
description: Security-scan, push to GitHub, set up Pages + CI/CD, and fill in the repo About section
argument-hint: <owner/repo | github URL> [optional description]
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# /publish — ship this project to GitHub + GitHub Pages

The user supplies the GitHub repo in `$ARGUMENTS` (e.g. `owner/repo` or `https://github.com/owner/repo`), optionally followed by a short description for the About section.

If `$ARGUMENTS` is empty, check `git remote get-url origin`; if there is none either, ask the user for the repo and stop. Never guess an owner/repo.

Normalise to `OWNER/REPO`. Pages URL is `https://OWNER.github.io/REPO/` (use `https://OWNER.github.io/` if the repo is named `OWNER.github.io`). Prefer confirming the real URL from the API in step 5.

Work through the steps **in order**. Stop and report if any step fails; don't skip ahead. The security scan runs FIRST because pushing is irreversible once content is public.

## 0. Preflight

- `gh auth status` — if not logged in, tell the user to run `gh auth login` and stop. Do not ask for tokens.
- `git status --short` and `git branch --show-current`. Default branch should be `main`. If there are uncommitted changes, show them and ask before committing; commit only with the user's OK (follow the attribution rules in the system reminder).
- Check the repo exists: `gh repo view OWNER/REPO`. If it doesn't, ask whether to create it (`gh repo create OWNER/REPO --public --source=. --remote=origin`). Pages on free plans needs a public repo — mention this if they choose private.

## 1. Security scan (gate — must pass before any push)

Goal: nothing sensitive is committed, and nothing sensitive is sent to the internet by the app or by CI.

Run all of these and read the results; don't just check exit codes.

1. **Secrets in the working tree and history**
   - `git ls-files` — flag `.env*`, `*.pem`, `*.key`, `id_rsa*`, `credentials*`, `.claude/settings.local.json`, `*.p12`, `*.sqlite`.
   - `grep -rniE "(api[_-]?key|secret|token|passwd|password|bearer|authorization|private[_-]?key|AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{20,}|github_pat_|sk-[A-Za-z0-9]{20,}|xox[bp]-)" --exclude-dir=.git .`
   - History: `git log -p --all | grep -nE` with the same patterns (or `gitleaks detect` / `trufflehog` if installed — use them if present).
2. **Personal data**: grep for email addresses, phone numbers, and absolute local paths (`/Users/`) in tracked files and in `git log --format='%ae %an'`. Author emails in commit history are public once pushed — warn the user if a private/personal address appears and offer `noreply` (`ID+USER@users.noreply.github.com`) before pushing.
3. **Outbound network calls from the app** (project rule in CLAUDE.md): 
   `grep -nE "localStorage|sessionStorage|indexedDB|document.cookie|https?://|fetch\(|XMLHttpRequest|sendBeacon|WebSocket|<script[^>]+src|<link[^>]+href=\"http|@import|!important" index.html`
   - The only allowed external URL is the FormSubmit endpoint in the `CONFIG` block. Anything else (CDN, fonts, analytics, trackers, images) is a failure.
   - Confirm what is sent to FormSubmit: only the new task fields the user typed. No state dump, no user agent/device fingerprinting, no hidden fields. Confirm the failure path is try/catch so it cannot break the board.
   - Note: the email address in `CONFIG` ships to every visitor in page source. That is inherent to FormSubmit's design and acceptable only if the user knowingly accepts it; **report it explicitly** and suggest FormSubmit's obfuscated random-token alias instead of the raw address.
4. **Unsafe DOM patterns**: every user string reaching `innerHTML` must pass `escapeHtml()`. Grep `innerHTML|insertAdjacentHTML|document.write|eval\(|new Function|setTimeout\(\s*['"]` and review each hit.
5. **CI/CD security** (`.github/workflows/*.yml`):
   - `permissions:` is least-privilege (no `write-all`, no unneeded `contents: write`).
   - No `pull_request_target` with checkout of PR code; no `${{ github.event.* }}` interpolated into `run:` blocks (script injection).
   - Third-party actions are pinned (official `actions/*` by major version is OK; others by commit SHA).
   - No secrets echoed; Pages stages only `index.html` (not `.git`, `.claude`, `CLAUDE.md`).
6. **Syntax check**: extract the `<script>` body to the scratchpad dir and `node --check` it.

Output a short table: check → PASS / WARN / FAIL with file:line evidence. **If any FAIL, stop and fix or ask.** WARNs need an explicit user acknowledgement before continuing.

## 2. CI/CD via GitHub Actions

Ensure these workflows exist (create or update; keep existing `pages.yml` unless it's wrong):

- `.github/workflows/pages.yml` — deploy to Pages on push to `main` + `workflow_dispatch`. Uses `actions/checkout`, `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages`; permissions `contents: read, pages: write, id-token: write`; stages only `index.html` into `_site/`.
- `.github/workflows/ci.yml` — on `pull_request` and push, with `permissions: contents: read`:
  - extract `<script>` and run `node --check`
  - run the "no persistence / no external requests" grep from CLAUDE.md and fail if anything other than the FormSubmit URL matches
  - secret scan (e.g. `gitleaks/gitleaks-action` pinned, or the grep patterns above)
- Make `pages.yml` deploy only after CI passes (e.g. `workflow_run` on CI success, or a `needs:` job) if practical; otherwise keep them independent and say so.

Validate YAML locally (`python3 -c "import yaml,sys;yaml.safe_load(open(sys.argv[1]))" file`) before committing.

## 3. Push to GitHub

- Ensure `origin` points at `https://github.com/OWNER/REPO.git` (`git remote add` or `set-url`; show the user if you're changing an existing remote).
- Commit any new/changed files (workflows, command file) with a clear message.
- `git push -u origin main`. Never force-push. If rejected, report and ask.

## 4. Create / edit the GitHub Pages site

- Enable Pages with the Actions source (idempotent):
  ```
  gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow \
    || gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow
  ```
  A 409/"already exists" on POST is fine — fall through to PUT.
- If Pages cannot be enabled (private repo on free plan, no permission), report it and continue with the remaining steps that can succeed.

## 5. Deploy and verify

- Trigger and watch: `gh workflow run pages.yml` (if it didn't already start from the push), then `gh run watch` on the latest run (`gh run list --workflow=pages.yml --limit 1`).
- Get the live URL: `gh api repos/OWNER/REPO/pages --jq .html_url`.
- Verify: `curl -sI <url>` returns 200, and `curl -s <url> | grep -c "IT PMO"` is > 0. Report failures with the run's logs (`gh run view --log-failed`).
- Re-run the outbound check on the **deployed** HTML (curl it and grep for `https?://`) to confirm the live page matches what was scanned.

## 6. Repo About section (description, website, topics)

Set after the Pages URL is known so the link is correct:

```
gh repo edit OWNER/REPO \
  --description "<one line>" \
  --homepage "<pages url>" \
  --add-topic kanban --add-topic vanilla-js --add-topic single-file --add-topic demo
```

- Description: use the one the user gave in `$ARGUMENTS`; otherwise derive from CLAUDE.md, e.g. "Single-file Kanban board for an IT PMO — vanilla HTML/CSS/JS, no build, no persistence (demo/training tool)". Keep it under 350 chars, no personal data.
- Check the existing About first (`gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`) and only overwrite fields that are empty or that the user agreed to replace — show what will change.
- Disable unused features only if the user asks.

## 7. Final report

Short summary with:
- Security scan table (and any acknowledged WARNs)
- Repo URL, Pages URL (verified status code), About section values
- Workflow names and the latest run status
- Anything skipped or failed, and the exact next step for the user

Don't claim success for a step you didn't verify.
