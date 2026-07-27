# Deploy runbook — iterventions.com

How this repo becomes a live website, explained from zero.

This is written for someone who has not used Cloudflare Pages or GitHub Actions before. Every
step is copy-paste. Every step shows what a correct result looks like, so you can tell whether
it worked without guessing.

**Shell:** commands are written for **PowerShell** on Windows, which is what this machine opens
by default. Where PowerShell differs from other shells it is called out. `git`, `gh` and `npx`
behave identically everywhere; only variables and `curl` differ.

**Time:** about 30 minutes of actual work, plus up to 15 minutes of waiting for a TLS
certificate at the end. You can walk away during the wait.

---

## Contents

- [Part 1 — What is actually going on](#part-1--what-is-actually-going-on)
- [Part 2 — Vocabulary](#part-2--vocabulary)
- [Part 3 — The one genuinely dangerous command](#part-3--the-one-genuinely-dangerous-command)
- [Where things stand right now](#where-things-stand-right-now)
- [Phase 0 — Check your tools](#phase-0--check-your-tools)
- [Phase 1 — Log in to GitHub](#phase-1--log-in-to-github)
- [Phase 2 — Get Cloudflare credentials](#phase-2--get-cloudflare-credentials)
- [Phase 3 — Give the credentials to GitHub](#phase-3--give-the-credentials-to-github)
- [Phase 4 — Create the Pages project](#phase-4--create-the-pages-project)
- [Phase 5 — First deploy](#phase-5--first-deploy)
- [Phase 6 — Point the domain at it](#phase-6--point-the-domain-at-it)
- [Phase 7 — Verify with a browser](#phase-7--verify-with-a-browser)
- [After setup — the routine](#after-setup--the-routine)
- [Rollback](#rollback)
- [Troubleshooting](#troubleshooting)
- [Git identity — stopping the account picker](#git-identity--stopping-the-account-picker)

---

## Part 1 — What is actually going on

Once this is set up, publishing a change to the website is one command: `git push`. Everything
after that happens without you. Here is the chain, so that when something breaks you know which
link to look at.

```
  you                GitHub                  GitHub Actions            Cloudflare
  ───                ──────                  ──────────────            ──────────
  git push  ───────► main branch  ─────────► a robot machine  ───────► Pages project
                     updated                 boots up,                 receives the files,
                                             downloads the repo,       puts them on a CDN,
                                             runs wrangler             serves iterventions.com
```

Step by step:

1. **You push.** Your commits travel from your laptop to GitHub.
2. **GitHub notices.** There is a file in this repo at `.github/workflows/deploy.yml`. It says
   "when someone pushes to `main`, do the following." GitHub reads it and starts a job.
3. **A temporary Linux machine boots.** GitHub rents you one for free. It downloads a fresh copy
   of the repo — *only the files git tracks*, which is why `__local/` can never leak this way.
4. **It runs `wrangler`.** `wrangler` is Cloudflare's command-line tool. The job hands it two
   secrets (an API token and an account ID) so Cloudflare will accept the upload, then uploads
   the repo contents.
5. **Cloudflare serves it.** The files land in a "Pages project" and get copied to Cloudflare's
   servers worldwide. Your domain points at that project.

There is **no build step**. This site is one HTML file plus 22 photographs. Nothing gets
compiled, bundled or transformed. What is in the repo is exactly what gets served. That removes
an entire category of things that can go wrong.

The setup below exists almost entirely to accomplish step 4 — teaching GitHub how to prove to
Cloudflare that it is allowed to upload.

---

## Part 2 — Vocabulary

You will meet these words. None of them are complicated, but they get used as if you already
know them.

**Cloudflare Pages** — Cloudflare's static website hosting. You give it a folder of files, it
serves them fast, worldwide, over HTTPS, for free. It is not a server; there is nothing running.
It hands out files.

**Pages project** — a named container inside your Cloudflare account that holds one website.
Ours will be called `iterventions`. The name matters: it appears in `deploy.yml`, and if the two
disagree the deploy fails with "project not found."

**Deployment** — one upload of one version of the site. Every push creates a new one. Cloudflare
keeps all of them forever, which is why rollback is instant.

**Production branch** — the git branch whose deployments become the real website. Ours is `main`.

**Preview deployment** — a deployment from any *other* branch. Cloudflare gives it its own URL
(`<branch>.iterventions.pages.dev`) so you can look at a change before it is live. This is free
and automatic; you do not configure it.

**API token** — a long random string that proves to Cloudflare "whoever holds this is allowed to
do X." Like a hotel key card: not your password, works only for specific doors, and you can
cancel it without changing anything else. GitHub needs one to upload on your behalf.

**Account ID** — a public-ish identifier for your Cloudflare account. Not secret, but the API
needs it to know which account to act on.

**Repository secret** — an encrypted value stored on GitHub. Workflows can read it; humans
cannot read it back, not even you. This is where the API token lives. It is never in the repo.

**wrangler** — Cloudflare's CLI. You do not install it; `npx` downloads it on demand.

**`npx`** — a command that comes with Node.js. It runs a tool without permanently installing it.
The first run is slow because it downloads; later runs are cached.

**`gh`** — GitHub's official CLI. Already installed on this machine.

---

## Part 3 — The one genuinely dangerous command

Read this before you run anything.

The folder `__local/` in this repo is **108 MB** — the design prototype, the full-resolution
source photographs, and a 7.8 MB video. It is listed in `.gitignore`, so git ignores it
completely and it is not on GitHub.

**Automated deploys are safe.** The robot machine downloads the repo from GitHub, and `__local/`
is not on GitHub, so it cannot upload what it does not have.

**Manual deploys from this folder are not safe.** `wrangler` does **not** read `.gitignore`. If
you run this:

```powershell
npx wrangler pages deploy .        # ← DO NOT RUN THIS FROM THIS FOLDER
```

...it will cheerfully upload all 108 MB, including your unprocessed source photos and the video,
to a public website.

**The rule:** deploy by `git push`. If you ever genuinely need a manual deploy, do it from a
fresh clone in a temporary directory — a clone physically cannot contain `__local/`:

```powershell
git clone https://github.com/troup-miller/iterventions.git $env:TEMP\itv-deploy
cd $env:TEMP\itv-deploy
npx wrangler pages deploy . --project-name=iterventions
cd C:\Users\troup\Repos\iterventions
Remove-Item -Recurse -Force $env:TEMP\itv-deploy
```

---

## Where things stand right now

Verified on this machine, 2026-07-26.

| Thing | State |
|---|---|
| Repo on GitHub | ✅ `github.com/troup-miller/iterventions` |
| Branches published | ✅ `main`, `dev` |
| Site content | ✅ 2 project pages, 22 images, ~6 MB, preflight GO |
| `git` push/pull auth | ✅ working, account pinned to `troup-miller` |
| Node.js | ✅ v24.15.0 (`npx` 11.12.1) |
| `gh` CLI | ⬜ installed (2.92.0), **not logged in** |
| Cloudflare API token | ⬜ not created |
| Cloudflare account ID | ⬜ not recorded |
| GitHub repo secrets | ⬜ not set |
| Pages project | ⬜ not created |
| Custom domain | ⬜ not attached |

Phases 1–4 are one-time. After that, deploying forever after is `git push`.

---

## Phase 0 — Check your tools

Confirms you have what you need before starting. Nothing here changes anything.

```powershell
cd C:\Users\troup\Repos\iterventions
git --version
node --version
gh --version
git status
```

**What good looks like:**

```
git version 2.x.x
v24.15.0
gh version 2.92.0 ...
On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

**If `node --version` fails** — install Node.js LTS from nodejs.org, then reopen the terminal.
Nothing else here works without it.

**If `git status` shows changes** — that is fine, they just will not be deployed until committed.

One safety check worth doing any time you add new files:

```powershell
git status --short
```

If `__local/` ever appears in that output, **stop** and fix `.gitignore` before committing.

---

## Phase 1 — Log in to GitHub

*You have to do this one yourself — it opens a browser.*

The `gh` CLI login is separate from the login git already uses for push and pull. You need it so
that the next phases can store secrets and start workflows from the command line.

In this Claude Code session, put `!` in front so the output appears in the conversation:

```
! gh auth login
```

Or run it in any terminal without the `!`.

**It will ask you five questions. Answer them like this:**

| Question | Answer |
|---|---|
| What account do you want to log into? | **GitHub.com** |
| What is your preferred protocol for Git operations? | **HTTPS** |
| Authenticate Git with your GitHub credentials? | **Yes** |
| How would you like to authenticate? | **Login with a web browser** |
| Copy this one-time code: `XXXX-XXXX` | Copy it, press Enter, paste in the browser |

Your browser opens, you paste the code, you pick the account **troup-miller**, you click
Authorize. Back in the terminal it says `✓ Logged in as troup-miller`.

**Confirm:**

```powershell
gh auth status
```

**What good looks like:**

```
github.com
  ✓ Logged in to github.com account troup-miller (keyring)
  - Active account: true
  - Git operations protocol: https
  - Token scopes: 'gist', 'read:org', 'repo', 'workflow'
```

`repo` and `workflow` must both be in that scope list. `repo` lets it set secrets; `workflow`
lets it start deploys. If they are missing, run `gh auth refresh -s repo,workflow`.

---

## Phase 2 — Get Cloudflare credentials

*The token part is dashboard-only — Cloudflare will not create a token from the CLI, because
that would be a chicken-and-egg problem.*

### 2a. Create the API token

Open **https://dash.cloudflare.com/profile/api-tokens** in a browser.

1. Click the blue **Create Token** button.
2. Scroll past the templates to the bottom. Click **Get started** next to **Create Custom Token**.
3. **Token name:** `iterventions-pages-deploy`. This is only a label for you — name it something
   you will recognise in a year when you are wondering what it does.
4. Under **Permissions**, you are building three rows. Each row is three dropdowns. Click
   **+ Add more** to get another row.

| Row | First dropdown | Second dropdown | Third dropdown | Why |
|---|---|---|---|---|
| 1 | Account | Cloudflare Pages | **Edit** | lets it upload the site |
| 2 | Account | Account Settings | **Read** | lets it look up your account ID |
| 3 | Zone | DNS | **Edit** | only needed if you want automation to touch DNS later — safe to skip |

Row 3 is optional. Skip it if you are attaching the domain by hand in Phase 6, which is what
this runbook does. Fewer permissions is better.

5. Under **Account Resources**, set the dropdown to **Include** and pick your account by name.
   Do not leave it on "All accounts."
6. Under **Client IP Address Filtering**, leave it empty unless you have a static IP. Home
   internet connections change IP, and a filter would lock out your own deploys.
7. **TTL** — leave it. An expiring token means deploys silently start failing in six months.
8. Click **Continue to summary**, read it, click **Create Token**.

**The token is now on screen. This is the only time you will ever see it.** Leave this browser
tab open and go to 2b. Do not paste it into a file, a note, or a chat window — including this
one. If you lose it, no harm done: delete it and make another.

### 2b. Load the token into your terminal, without it hitting your command history

Back in PowerShell:

```powershell
$env:CLOUDFLARE_API_TOKEN = Read-Host "Paste the Cloudflare API token"
```

Press Enter, paste, press Enter again. `Read-Host` keeps the value out of your PowerShell
history file, which typing `$env:X = "abc..."` directly would not.

This variable lasts only as long as this terminal window. That is deliberate.

### 2c. Verify the token, and get your account ID for free

```powershell
npx wrangler whoami
```

First run takes 30–60 seconds while `npx` downloads wrangler. Be patient.

**What good looks like:**

```
 ⛅️ wrangler 4.x.x
-------------------
Getting User settings...
👋 You are logged in with an API Token, associated with the email your@email.com.
┌──────────────────────┬──────────────────────────────────┐
│ Account Name         │ Account ID                       │
├──────────────────────┼──────────────────────────────────┤
│ Your Account         │ 0123456789abcdef0123456789abcdef │
└──────────────────────┴──────────────────────────────────┘
```

That 32-character hex string is your **account ID**. It is not secret. Copy it.

**If instead you see `Authentication error [code: 10000]`** — the token is wrong or lacks
permissions. Go back to 2a and check the three dropdowns on row 1 say exactly
Account / Cloudflare Pages / Edit.

Store the account ID in a variable too:

```powershell
$env:CLOUDFLARE_ACCOUNT_ID = "paste-the-32-character-id-here"
```

---

## Phase 3 — Give the credentials to GitHub

Now hand both values to GitHub so the robot machine can use them. `gh secret set` reads from
standard input, so piping the variable means the token never appears as text in a command.

```powershell
$env:CLOUDFLARE_API_TOKEN  | gh secret set CLOUDFLARE_API_TOKEN
$env:CLOUDFLARE_ACCOUNT_ID | gh secret set CLOUDFLARE_ACCOUNT_ID
gh secret list
```

**What good looks like:**

```
✓ Set Actions secret CLOUDFLARE_API_TOKEN for troup-miller/iterventions
✓ Set Actions secret CLOUDFLARE_ACCOUNT_ID for troup-miller/iterventions
NAME                    UPDATED
CLOUDFLARE_ACCOUNT_ID   less than a minute ago
CLOUDFLARE_API_TOKEN    less than a minute ago
```

`gh secret list` shows names and timestamps only. Nobody, including you, can read the values
back — GitHub encrypts them one-way. If you need to change one, you overwrite it.

**Never use `gh secret set NAME --body "the-token"`.** That form puts the secret into your shell
history in plain text, where it stays.

The names must match `deploy.yml` exactly — it looks for `CLOUDFLARE_API_TOKEN` and
`CLOUDFLARE_ACCOUNT_ID`. Capitalisation counts.

---

## Phase 4 — Create the Pages project

The container on Cloudflare's side. It must be named `iterventions`, because that is what
`.github/workflows/deploy.yml` passes to `--project-name`.

```powershell
npx wrangler pages project create iterventions --production-branch main
```

**What good looks like:**

```
✨ Successfully created the 'iterventions' project.
```

**Confirm:**

```powershell
npx wrangler pages project list
```

You should see `iterventions` listed, with production branch `main`.

**If it says the project already exists**, that is fine — skip ahead. **If it says
`--production-branch` is required**, add it as above; a project created with the wrong
production branch will build previews forever and never go live.

---

## Phase 5 — First deploy

Everything is wired. Trigger a run:

```powershell
gh workflow run "Deploy to Cloudflare Pages" --ref main
gh run watch
```

`gh run watch` follows it live and exits when it finishes. It takes 30–60 seconds — there is no
build, so almost all of that is the machine booting.

**What good looks like:**

```
✓ main Deploy to Cloudflare Pages · 1234567890
Triggered via workflow_dispatch about 1 minute ago

JOBS
✓ Publish in 32s (ID 9876543210)
```

**Find the URL your site is now on:**

```powershell
gh run view --log | Select-String "pages.dev"
```

Or:

```powershell
npx wrangler pages deployment list --project-name=iterventions
```

Either gives you a `https://<something>.iterventions.pages.dev` address. **Open it.** The site is
live — just not yet on your domain.

**Do Phase 7's checks now, before attaching the domain.** It is much more relaxing to fix
problems on a URL nobody knows about.

**If the run failed:**

```powershell
gh run view --log-failed
```

That prints only the failing step. See [Troubleshooting](#troubleshooting) — the first-time
failures are almost always a token permission or a project-name mismatch.

---

## Phase 6 — Point the domain at it

*Dashboard, and there is a good reason: DNS mistakes are the slowest kind to undo.*

`iterventions.com` is already registered with Cloudflare, which makes this much easier than it
would otherwise be — no nameserver changes, no waiting for propagation.

### 6a. Attach both hostnames

1. Cloudflare dashboard → **Workers & Pages** → click **iterventions**.
2. **Custom domains** tab → **Set up a custom domain**.
3. Type `iterventions.com`. Click **Continue** → **Activate domain**.
4. Repeat for `www.iterventions.com`.

Cloudflare writes the DNS records itself and issues a TLS certificate. The status will read
**Initializing** for a few minutes, then **Active**. Up to 15 minutes is normal. Go make coffee.

**Check from the terminal:**

```powershell
nslookup iterventions.com
curl.exe -sI https://iterventions.com | Select-String "HTTP/|server"
```

In PowerShell you must write **`curl.exe`**, not `curl` — bare `curl` is an alias for a different
PowerShell command that takes different arguments.

**What good looks like:** `HTTP/2 200` and a `server: cloudflare` line.

### 6b. Redirect www to the bare domain

You now have two addresses serving identical content. Search engines dislike that, and this
site's `<link rel="canonical">`, `og:url` and `sitemap.xml` all already say the bare domain
`https://iterventions.com/`. So make `www` redirect to it.

1. Dashboard → select the **iterventions.com** zone (not the Pages project) → **Rules** →
   **Redirect Rules** → **Create rule**.
2. **Rule name:** `www to apex`
3. **If — Custom filter expression:** field `Hostname`, operator `equals`, value
   `www.iterventions.com`
4. **Then — Type:** Dynamic. **Expression:**
   ```
   concat("https://iterventions.com", http.request.uri.path)
   ```
5. **Status code:** `301`. Leave "Preserve query string" **on**.
6. **Deploy**.

**Verify:**

```powershell
curl.exe -sI https://www.iterventions.com | Select-String "HTTP/|location"
```

**What good looks like:** `HTTP/2 301` and `location: https://iterventions.com/`.

If you would rather serve `www` as the real address, that is a perfectly fine choice — but then
reverse the rule *and* update the canonical link, `og:url` and `sitemap.xml` in `index.html` in
the same change. Half-doing it sends contradictory signals to search engines.

---

## Phase 7 — Verify with a browser

The automated checks are already covered — run the `deploy-preflight` agent before any push.
This list is the part that needs human eyes.

**Routing**
- [ ] the site loads over HTTPS with no mixed-content warning
- [ ] `#home`, `#log`, `#project/gauss`, `#project/ordinance` all render
- [ ] bare `#project` lands on Gauss Sequencer
- [ ] a deep link (`/#project/ordinance`) loads the right view on a **fresh reload**, not just
      when clicked from inside the site
- [ ] clicking a sub-nav link jumps within the page and **does not** bounce to home
- [ ] a nonsense hash like `#banana` falls back to home

**The instrument**
- [ ] the scope sweep animates, and dwells visibly longer on **Failure** than on anything else
- [ ] the READOUT panel text changes as the sweep crosses each segment
- [ ] Gauss Data Notebook: both canvases draw (gate drive, velocity)
- [ ] Ordinance Data Notebook: the backlash canvas draws and the two sweeps do not overlay

**Content**
- [ ] all 22 photographs load — no broken-image icons on either project page
- [ ] nothing jumps around as images arrive
- [ ] fonts load with no flash of system sans-serif
- [ ] the MOTHBALLED badges render in grey, not green

**Print**
- [ ] `PRINT PASSPORT` on Gauss → one ink-on-white sheet, passport only
- [ ] `PRINT PASSPORT` on Ordinance → same, and **only that project's** passport

**Responsive / accessibility**
- [ ] narrow the window below 560px — the hamburger appears, the drawer opens and closes
- [ ] `Tab` through the home page — focus outlines are green, never the browser default
- [ ] turn on Windows Settings → Accessibility → Visual effects → Animation effects **off**,
      reload: the sweep freezes fully drawn, nothing fades or lifts

**Infrastructure**
- [ ] `/404` shows the themed 404 page, not Cloudflare's
- [ ] `/robots.txt` and `/sitemap.xml` both load
- [ ] an image loads directly: `/gauss/v02/gauss_v02_20260404_img_01.jpg`
- [ ] DevTools (F12) → Console → no red errors
- [ ] DevTools → Network → `index.html` says `no-cache`, images say `immutable`

---

## After setup — the routine

From here on, publishing is:

```powershell
git add -A
git commit -m "what changed"
git push
gh run watch
```

That is the entire loop. No build, no `node_modules`, no lockfile.

Per this repo's branching rules (see `CLAUDE.md`), real work goes on a branch and reaches `main`
through a pull request:

```powershell
git switch dev
git pull
git switch -c feat/image-lightbox
# ...make changes...
git add -A
git commit -m "feat: image lightbox"
git push -u origin feat/image-lightbox
gh pr create --base dev --fill
```

Every branch you push gets its own preview URL, so you can look at the change before merging:

```powershell
npx wrangler pages deployment list --project-name=iterventions
```

---

## Rollback

Cloudflare keeps every deployment forever. You have two options and they are not equivalent.

**Fast — undo the website, leave the repo alone.** Dashboard → Workers & Pages → iterventions →
**Deployments** → find the last good one → **⋯** → **Rollback**. Live in seconds.

Use this when the site is visibly broken and you want it fixed *now*. But understand what it
does not do: the bad commit is still on `main`. The next time you push anything, the broken
version ships again along with it.

**Proper — undo the repo, let CI redeploy.**

```powershell
git log --oneline -5           # find the bad commit's short hash
git revert <hash>              # makes a NEW commit that undoes it
git push
```

`git revert` does not rewrite history — it adds a commit that reverses an earlier one. Safe on a
shared branch, and it leaves an honest record of what happened.

**Rule of thumb:** roll back in the dashboard to stop the bleeding, then revert in git before you
finish for the day.

---

## Troubleshooting

| What you see | What it means | What to do |
|---|---|---|
| `Authentication error [code: 10000]` | the API token lacks **Cloudflare Pages: Edit**, or is scoped to the wrong account | recreate the token — Phase 2a, check row 1 of the permissions table |
| `Project not found` | the Pages project name and `--project-name` in `deploy.yml` disagree | `npx wrangler pages project list` and compare, exactly, including case |
| Workflow does not start at all | you pushed to a branch other than `main`, or Actions is disabled | GitHub → repo → Settings → Actions → General → allow all actions |
| Workflow fails instantly with a secrets error | Phase 3 did not take | `gh secret list` — both names must be there, spelled exactly |
| Deploy succeeds but the site looks old | your browser cached it | Ctrl+F5. `_headers` sets `index.html` to `no-cache`, so this should only happen once |
| Deploy uploads ~110 MB | you ran `wrangler` from this folder | see [Part 3](#part-3--the-one-genuinely-dangerous-command). Delete that deployment |
| Custom domain stuck on "Initializing" | certificate still issuing | wait 15 minutes; if it persists, remove the domain and re-add it |
| `curl : A parameter cannot be found that matches parameter name 'sI'` | PowerShell aliased `curl` to `Invoke-WebRequest` | write `curl.exe`, with the extension |
| Every pull request shows a red ✗ | `deploy.yml` also runs on `pull_request`, and before Phase 3 there are no secrets | expected until setup is done. After that, a red ✗ on a PR is real |
| A browser account picker appears on every push | GCM does not know which GitHub account to use | see [Git identity](#git-identity--stopping-the-account-picker) below |

---

## Known gaps carried into production

- `_headers` sets long-lived caching for `*.jpg` and `*.png` only. All 22 current images are
  `.jpg`, so nothing is broken — but add `*.webp` and `*.svg` rules before using those formats.
- No analytics, deliberately. Cloudflare Web Analytics is one script tag and needs no cookie
  banner, if it is ever wanted.
- The image lightbox is not built. It is the top open item — 22 photographs and no way to see
  them full-size. See `CLAUDE.md`.
- `main` has no branch protection rule, so "pull requests only" is currently a convention rather
  than something GitHub enforces. One command fixes that, once `gh` is logged in.

---

## Git identity — stopping the account picker

Git Credential Manager pops an account chooser whenever it has more than one GitHub account
cached and nothing tells it which to use. Naming the account up front makes it resolve silently.

This is **already configured** on this machine. It is recorded here because `.git/config` is not
part of the repository — a fresh clone will not have it, and you will get the picker again.

```powershell
# 1. this repo (lives in .git/config — does NOT survive a fresh clone)
git config --local user.name  "troup-miller"
git config --local user.email "troup.miller@gmail.com"
git config --local credential.https://github.com.username "troup-miller"
git config --local credential.https://github.com.provider  "github"

# 2. the remote URL — git then supplies a username on every request for this remote
git remote set-url origin "https://troup-miller@github.com/troup-miller/iterventions.git"

# 3. every repo on this machine (~/.gitconfig)
git config --global credential.https://github.com.username "troup-miller"
git config --global credential.https://github.com.provider  "github"
```

The account name in a URL is **not** a secret — it is the same string as in the repo path. No
token ever belongs in a URL, a config file, or this repo.

**Check what git will actually send:**

```powershell
git config --get-urlmatch credential https://github.com
git config --list --show-origin | Select-String "credential.helper"
```

The first should report `username troup-miller`. The second should print **exactly one** helper
line, `manager`. More than one means the chain calls Credential Manager repeatedly — which is
what caused the picker to appear here in the first place.

`gh` keeps its own separate login. If several accounts end up in it, `gh auth switch` chooses.

---

## Who does what

| Step | Who | Why |
|---|---|---|
| Phase 0 — tool check | either | read-only |
| Phase 1 — `gh auth login` | **you** | opens a browser |
| Phase 2 — Cloudflare token | **you** | dashboard only, and it is secret material |
| Phase 3 — repo secrets | **you** | the token must not pass through a transcript |
| Phase 4 — Pages project | either | one command |
| Phase 5 — first deploy | either | one command |
| Phase 6 — custom domain | **you** | dashboard, and DNS mistakes are slow to undo |
| Phase 7 — verification | **you** | needs a real browser and real eyes |
| Routine deploys | either | `git push` |

The pattern: anything involving a browser, a password or a secret is yours. Anything that is a
command with a predictable result can be handed to Claude.
