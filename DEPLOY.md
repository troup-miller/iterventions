# Deploy runbook — iterventions.com

From an uncommitted working directory to a live site on `iterventions.com`.

Static site, Cloudflare Pages, **no build step**. The domain is already on Cloudflare, so DNS is the
easy part. The whole path is roughly 20 minutes, most of it waiting for a certificate.

---

## State of play

Verified on this machine, 2026-07-26:

| Thing | State |
|---|---|
| `git` repo | branch `main` and `dev`, both at the same commit |
| git remote | `origin` → `github.com/troup-miller/iterventions`, `main` and `dev` published |
| GitHub CLI | `gh` 2.92.0 installed, **not logged in** |
| `wrangler` | not installed — `npx` fetches it on demand (node v24 present) |
| Cloudflare account ID | not yet recorded |
| Cloudflare API token | not yet created |
| GitHub repo secrets | not set |
| Pages project | not created |
| Site content | 2 project pages, 22 images, ~6.3 MB tracked, preflight **GO** |

Everything below assumes that starting point. Phases 1–4 are one-time setup; after that, deploying is
`git push`.

---

## ⚠ Read this before running wrangler locally

`__local/` is **108 MB** of design prototype and full-resolution source photographs, including a
7.8 MB video. It is gitignored, so CI never sees it — the GitHub Actions runner checks out only
tracked files and the workflow is safe.

**`wrangler` does not read `.gitignore`.** Running `npx wrangler pages deploy .` from this directory
would upload `__local/` to production. Do not do it. Deploy through CI (`git push`), and if you ever
need a manual deploy, do it from a fresh `git clone` of the repo in a temp directory — that clone
physically cannot contain `__local/`.

---

## Git identity — pinning the account

Git Credential Manager shows an account picker whenever it has more than one GitHub
account cached and nothing tells it which one to use. Naming the account up front makes it
resolve silently. Three places do it, and they are cheap enough to do all three:

```bash
# 1. this repo (.git/config — does NOT survive a fresh clone, so re-run it after cloning)
git config --local user.name  "troup-miller"
git config --local user.email "troup.miller@gmail.com"
git config --local credential.https://github.com.username "troup-miller"
git config --local credential.https://github.com.provider  "github"

# 2. the remote URL — git then hands GCM a username on every request for this remote
git remote set-url origin "https://troup-miller@github.com/troup-miller/iterventions.git"

# 3. every repo on the machine (~/.gitconfig)
git config --global credential.https://github.com.username "troup-miller"
git config --global credential.https://github.com.provider  "github"
```

The account name in the URL is **not** a secret — it is the same string that appears in the
repo path. No token ever goes in a URL, a config file, or this repo.

Verify what git will actually send:

```bash
git config --get-urlmatch credential https://github.com   # expect username=troup-miller
git config --list --show-origin | grep credential.helper  # expect exactly one: manager
```

More than one `credential.helper` in that last output means the chain is invoking GCM twice.
One is correct.

For the CLI, `gh auth login` stores its own credential separately from GCM; if several
accounts end up in it, `gh auth switch` picks between them.

---

## Phase 0 — Commit  · *done*

40 files, ~6 MB, published to `origin/main`. The safety check stays worth running before any
commit that touches new paths:

```bash
git check-ignore -v __local/   # must print a match
git status --short             # __local/ must never appear here
```

If `__local/` appears in `git status` at any point, **stop** and fix `.gitignore` before continuing.

---

## Phase 1 — GitHub  · *human, interactive*

The repo exists at `github.com/troup-miller/iterventions` and pushes over HTTPS through GCM.
The `gh` CLI is a separate login, still needed for Phase 3 (repo secrets) and Phase 5:

```
! gh auth login
```

Choose: GitHub.com → HTTPS → authenticate via browser → account **troup-miller**. Then:

```bash
gh auth status                      # confirm the right account
```

---

## Phase 2 — Cloudflare credentials  · *human, dashboard*

Two values are needed. Neither should ever be pasted into a chat, a commit, or a file in this repo.

### Account ID

Cloudflare dashboard → **Workers & Pages** → right-hand sidebar → **Account ID**. Copy it.

### API token

**My Profile → API Tokens → Create Token → Create Custom Token.**

| Scope | Resource | Permission |
|---|---|---|
| Account | Cloudflare Pages | **Edit** |
| Account | Account Settings | **Read** |
| Zone | DNS — `iterventions.com` | **Edit** *(only if you want CI to manage the custom domain; skip it if you attach the domain by hand in Phase 5)* |

Under *Account Resources*, restrict to the one account. Leave IP filtering empty unless you have a
static IP.

**The token is shown exactly once.** Copy it straight into Phase 3 — do not park it in a text file.

---

## Phase 3 — Repo secrets

```bash
gh secret set CLOUDFLARE_API_TOKEN     # paste the token at the prompt, then Enter
gh secret set CLOUDFLARE_ACCOUNT_ID    # paste the account ID
gh secret list                         # both should be listed
```

`gh secret set` reads from stdin, so the value never enters your shell history. Do not use
`gh secret set NAME --body "..."` — that does.

---

## Phase 4 — Pages project

The project name must match `--project-name=iterventions` in `.github/workflows/deploy.yml`.

```bash
npx wrangler pages project create iterventions --production-branch main
```

This prompts for browser auth on first run. Alternatively create it in the dashboard:
**Workers & Pages → Create → Pages → Upload assets**, name it `iterventions`, and drag `index.html`
once to initialise it.

---

## Phase 5 — First deploy

The workflow triggers on push to `main`. If Phase 1 already pushed, just re-run it:

```bash
gh workflow run "Deploy to Cloudflare Pages"
gh run watch                         # live status
gh run view --log-failed             # if it fails
```

A successful run prints a `*.pages.dev` URL. Open it and work through Phase 7 **before** attaching the
custom domain — it is much easier to fix things on a URL nobody has yet.

Branch behaviour: `main` → production; every other branch → a preview at
`<branch>.iterventions.pages.dev`. Use preview branches for design review.

---

## Phase 6 — Custom domain

Pages project → **Custom domains → Set up a custom domain** → `iterventions.com`. Repeat for
`www.iterventions.com`.

Because the zone is already on Cloudflare, the CNAME is written automatically and the certificate
issues in a few minutes.

Then pick **one** canonical host and redirect the other. The site's `<link rel="canonical">`,
`og:url` and `sitemap.xml` all currently say **apex** (`https://iterventions.com/`), so redirect
`www` → apex:

Rules → **Redirect Rules** → Create:
- If — hostname equals `www.iterventions.com`
- Then — Dynamic redirect, `concat("https://iterventions.com", http.request.uri.path)`, status **301**

If you would rather serve `www`, that is fine — but then update the canonical, `og:url` and
`sitemap.xml` in the same change, or you ship a split-brain SEO signal.

---

## Phase 7 — Verify

Static checks are already automated — run the `deploy-preflight` agent before any push. This list is
the part that needs a human with a browser.

**Routing**
- [ ] `https://iterventions.com` serves over HTTPS, no mixed-content warning
- [ ] `#home`, `#log`, `#project/gauss`, `#project/ordinance` all render
- [ ] bare `#project` lands on Gauss Sequencer
- [ ] a deep link (`iterventions.com/#project/ordinance`) reloads onto the right view
- [ ] clicking a sub-nav link jumps in-page and **does not** bounce to home
- [ ] an unknown hash falls back to home

**The instrument**
- [ ] the scope sweep animates and dwells visibly longer on **Failure** than on any other step
- [ ] the READOUT panel text changes as the sweep crosses each segment
- [ ] Gauss Data Notebook: both canvases draw (gate drive, velocity)
- [ ] Ordinance Data Notebook: the backlash canvas draws, and the two sweeps visibly do not overlay

**Content**
- [ ] all 22 photographs load — no broken-image icons on either project page
- [ ] no layout shift as images arrive (dimensions are declared, so this should hold)
- [ ] Google Fonts load with no flash of system sans
- [ ] the MOTHBALLED badges render in greys on PROJECT #002 and #003

**Print**
- [ ] `PRINT PASSPORT` on Gauss → one ink-on-white sheet, passport only, nothing else
- [ ] `PRINT PASSPORT` on Ordinance → same, and **only that project's** passport

**Responsive / a11y**
- [ ] under 560px the hamburger appears and the drawer opens and closes
- [ ] `Tab` through the home page — focus rings are green, never the browser default
- [ ] with OS "reduce motion" on: sweep freezes fully drawn, no page fade, no card lift

**Infrastructure**
- [ ] `/404` renders the themed 404
- [ ] `/robots.txt` and `/sitemap.xml` resolve
- [ ] a committed image resolves directly, e.g. `/gauss/v02/gauss_v02_20260404_img_01.jpg`
- [ ] DevTools → Console shows no errors
- [ ] DevTools → Network: `index.html` is `no-cache`, images are `immutable`

---

## Routine deploys, after setup

```bash
git add -A && git commit -m "..." && git push
gh run watch
```

That is the whole loop. No build, no lockfile, no `node_modules`.

---

## Rollback

Cloudflare Pages keeps every deployment.

- **Fastest** — Pages project → Deployments → pick the last good one → **Rollback**. Instant, no git.
- **Proper** — `git revert <sha> && git push`, which redeploys through CI and leaves history honest.

Prefer revert for anything that will outlive the afternoon; a dashboard rollback is invisible to the
repo, and the next push will happily re-deploy the broken commit.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Authentication error [code: 10000]` | token lacks **Pages: Edit**, or is scoped to the wrong account | recreate the token per Phase 2 |
| `Project not found` | name mismatch | `--project-name` in `deploy.yml` must equal the Pages project name exactly |
| Workflow never triggers | pushed to a non-`main` branch, or Actions disabled on a new private repo | check repo → Settings → Actions |
| Deploy succeeds, site is stale | browser cache | `_headers` sets `index.html` to `no-cache`; hard-reload once |
| Deploy uploads ~110 MB | you ran `wrangler` locally from the repo root | see the warning at the top — deploy from CI or a fresh clone |
| Custom domain stuck "Initializing" | certificate still issuing | wait ~15 min; if it persists, remove and re-add the domain |

---

## Known gaps carried into production

- `_headers` sets immutable caching for `*.jpg` and `*.png` only. All 22 current images are `.jpg`, so
  there is no live gap — but add `*.webp` and `*.svg` rules before using those formats.
- No analytics, deliberately. Cloudflare Web Analytics is one script tag in `<head>` and needs no
  cookie banner, if it is ever wanted.
- The image lightbox is not built. It is now the top open item, since there are 22 photographs to look
  at. See `CLAUDE.md`.

---

## Who does what

| Step | Owner | Why |
|---|---|---|
| Phase 0 — commit | either | Claude can stage and commit on request |
| Phase 1 — `gh auth login` | **human** | interactive browser auth |
| Phase 2 — Cloudflare token + account ID | **human** | dashboard only; secret material |
| Phase 3 — repo secrets | **human** | the token must not pass through a transcript |
| Phase 4 — Pages project | either | `npx wrangler` prompts for auth on first run |
| Phase 5 — first deploy | either | `gh workflow run`, `gh run watch` |
| Phase 6 — custom domain | **human** | dashboard only |
| Phase 7 — verification | **human** | needs a real browser |
| Routine deploys | either | `git push` |
