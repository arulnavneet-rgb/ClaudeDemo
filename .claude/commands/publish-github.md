---
description: Security-scan, push to GitHub, write README, set up Pages + CI/CD, and fill in the repo About section
argument-hint: <github-repo-url>
allowed-tools: Bash(git:*), Bash(gh:*), Bash(grep:*), Bash(find:*), Bash(ls:*), Bash(cat:*), Bash(file:*), Bash(which:*), Bash(osascript:*), Bash(python3 -m http.server:*), Bash(kill:*), Bash(mkdir:*), Read, Write, Edit, Glob, Grep, mcp__playwright__browser_resize, mcp__playwright__browser_navigate, mcp__playwright__browser_wait_for, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

Publish this project to GitHub. Target repo: **$ARGUMENTS**

If `$ARGUMENTS` is empty, stop and ask the user for the repo URL (e.g. `https://github.com/<owner>/<repo>`). Parse `<owner>` and `<repo>` from it (strip any trailing `.git` or `/`). Accept SSH form `git@github.com:<owner>/<repo>.git` too.

Read `CLAUDE.md` first and respect its hard constraints (single `index.html`, no external resources, no persistence, no UOB branding beyond what it allows). Don't change `index.html` as part of this command.

Work through the steps **in this order**. The security scan runs first on purpose: anything sensitive must be caught before it's pushed. Report progress briefly after each step.

## 0. Preflight

- `git rev-parse --is-inside-work-tree`. If this isn't a git repo, run `git init -b main`.
- `which gh` and `gh auth status`. If `gh` is missing or not logged in, carry on with the git-only steps and collect the GitHub-side steps (Pages source, About section, repo creation) into a **manual checklist** for the end. Tell the user that `brew install gh && gh auth login` lets the command do them automatically next time. Don't try to work around a missing login with tokens.
- `git status` and `git remote -v`. Note the current branch.

## 1. Security scan (blocking)

Scan **every file that would be pushed**: tracked files plus untracked files that aren't ignored (`git ls-files --cached --others --exclude-standard`). Also scan the history of commits that aren't on the remote yet (`git log -p origin/<branch>..HEAD` when the remote branch exists, otherwise `git log -p`).

Look for:
- **Secret files:** `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `id_ed25519*`, `*.keystore`, `credentials*`, `secrets*`, `*.sqlite`/`*.db`, `.npmrc`/`.pypirc` containing tokens, `*.tfstate`.
- **Secret patterns** (grep -nE, case-insensitive where it makes sense):
  - AWS: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key`
  - GitHub: `gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_[A-Za-z0-9_]{20,}`
  - Slack `xox[baprs]-`, Google `AIza[0-9A-Za-z_-]{35}`, Stripe `sk_live_`, OpenAI/Anthropic `sk-[A-Za-z0-9_-]{20,}`, `sk-ant-`
  - `-----BEGIN [A-Z ]*PRIVATE KEY-----`
  - Assignments like `(password|passwd|secret|api[_-]?key|token|auth)\s*[:=]\s*['"][^'"]{6,}`
  - Connection strings with credentials: `[a-z]+://[^/\s:]+:[^@\s]+@`
- **Personal data:** real-looking email addresses, phone numbers, NRIC/SSN-style IDs, internal hostnames or IPs. For this project, check `FORMSUBMIT_ENDPOINT` in `index.html`: if it holds a real email address rather than the `YOUR_EMAIL` placeholder, flag it, because the address will be public in the page source.
- **Junk:** `.DS_Store`, editor folders, `node_modules`, large binaries (> 5 MB).

Then:
- Make sure a `.gitignore` exists and covers `.DS_Store`, `.env*`, `*.pem`, `*.key`, `.claude/settings.local.json`, `node_modules/`, `.playwright-mcp/`. Create or extend it (don't remove existing lines).
- Present findings as a table: file:line, what matched, severity (block / review / ok), recommended action. Mask secret values (show the first 4 chars at most).
- **If anything is severity "block", stop here.** Don't commit or push. Explain how to remove it, and if it's already in a commit, explain that the secret must be rotated and history rewritten. Don't rewrite history yourself.
- For "review" items (like a public email address), ask the user whether to proceed.

## 2. README

Create or update `README.md` from the actual code (read `index.html` and `CLAUDE.md`, don't invent features). Include:
- Title and a one-paragraph description, stating that this is a demo/training app for a fictitious bank and isn't an official system.
- **Live demo** link: `https://<owner>.github.io/<repo>/` (lowercase owner).
- Features list.
- How to run (double-click `index.html`, or `python3 -m http.server` to test email notifications).
- Configuration: how to set `FORMSUBMIT_ENDPOINT`.
- Note that there's no persistence: a refresh resets the board.
- Tech constraints (vanilla HTML/CSS/JS, single file, no dependencies).
- Deployment: GitHub Actions → Pages, with a workflow status badge `https://github.com/<owner>/<repo>/actions/workflows/pages.yml/badge.svg`.

If a README already exists, keep any hand-written sections that are still accurate and update the rest.

## 3. Screenshot

Capture the board with the Playwright MCP server (configured in `.mcp.json`) and show it in the README.

- Serve the folder in the background: `python3 -m http.server 8765`. Use http rather than `file://`, which the Playwright MCP may block. Remember the process ID so you can stop it afterwards.
- `browser_resize` to 1440 × 900, `browser_navigate` to `http://localhost:8765/index.html`, then `browser_wait_for` the text "Backlog" so the cards have rendered.
- `browser_take_screenshot` with `filename: "docs/screenshot.png"` (viewport only, not `fullPage`; `scale: "css"`). Create `docs/` first if it's missing. The seed data is fictitious, so the image holds nothing sensitive, but look at it before committing to be sure.
- Stop the server (`kill <pid>`), call `browser_close`, and delete the `.playwright-mcp/` log folder the server leaves behind.
- In `README.md`, put `![IT PMO Kanban board](docs/screenshot.png)` directly under the **Live demo** line. If a screenshot line is already there, keep it and just overwrite the image.
- Stage `docs/screenshot.png` in the commit step. The Pages workflow deploys only `index.html`, so the image appears in the README on GitHub but not on the live site.

If the Playwright tools aren't available, or fail with "Chromium distribution 'chrome' is not found" or "Browser ... is not installed", tell the user to run `npx @playwright/mcp@latest install-browser chrome-for-testing` (it installs the exact build the server expects) and reconnect the server with `/mcp`. Skip this step and carry on; don't block the publish on the screenshot.

## 4. CI/CD workflow

Make sure `.github/workflows/pages.yml` exists and:
- triggers on `push` to `main` and `workflow_dispatch`
- has a **check** job that runs on every push and pull request: confirm `index.html` exists, and fail if it references external resources (`grep -nE '(src|href)=["'\'']https?://'`, excluding the FormSubmit URL inside the script) or uses `localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important`
- has a **deploy** job that `needs: check`, runs only on push to `main`, stages only `index.html` into `_site/`, and uses `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages` with `pages: write` and `id-token: write` permissions plus a `concurrency: pages` group

If the file already exists, edit it in place rather than rewriting it from scratch. Keep the comment style that's already there.

Also update the **Deployment** section of `CLAUDE.md` so the live URL and actions URL match `<owner>/<repo>`.

## 5. Commit and push

- Show `git status` and the list of files to be committed. Never `git add` anything the scan flagged.
- Stage the specific files (avoid a blind `git add -A` if untracked files you didn't review are present).
- Commit with a clear message describing what changed.
- Set the remote: if `origin` is missing, add it; if it points somewhere else, **ask before** changing it to `$ARGUMENTS`.
- If the repo doesn't exist on GitHub and `gh` is available, ask whether to create it (`gh repo create <owner>/<repo> --public --source . --remote origin`). Pages on a free account needs a public repo.
- Push with `git push -u origin main` (or the current branch). **Never force-push.** If the push is rejected because the remote has commits, stop and explain the options (pull/rebase) instead of overwriting.

## 6. GitHub Pages

With `gh`:
- Check: `gh api repos/<owner>/<repo>/pages`.
- If Pages isn't enabled: `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow`.
- If it's enabled with a different source: `gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow`.
- Watch the deploy: `gh run list --workflow pages.yml --limit 1`, then `gh run watch <id> --exit-status`. If it fails, show the failing step's log (`gh run view <id> --log-failed`) and diagnose.
- Read back the live URL from `gh api repos/<owner>/<repo>/pages --jq .html_url`.

Without `gh`: add "Settings → Pages → Source: GitHub Actions" to the manual checklist.

## 7. Repo About section

With `gh`, set the description, website and topics in one go:

```
gh repo edit <owner>/<repo> \
  --description "<one-line description from the README>" \
  --homepage "https://<owner>.github.io/<repo>/" \
  --add-topic kanban --add-topic vanilla-js --add-topic github-pages --add-topic demo
```

Keep the description under 350 characters, factual, and free of official-bank wording. Confirm with `gh repo view <owner>/<repo> --json description,homepageUrl,repositoryTopics`.

Without `gh`: add the exact description, website URL and topics to the manual checklist (repo page → ⚙ next to "About").

## 8. Final report

Finish with a short summary:
- Security scan result (what was checked and what was found or fixed)
- Commit hash pushed and branch
- README / screenshot / workflow / CLAUDE.md changes
- Pages status and live URL
- Actions run URL and result
- About section values
- Any **manual checklist** items still left for the user
