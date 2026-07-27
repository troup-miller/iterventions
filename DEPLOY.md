# Deploy runbook — iterventions.com

How this repo becomes a live website, explained from zero.

This is written for someone who has not used Cloudflare Pages or GitHub Actions before. Every
step is copy-paste. Every step shows what a correct result looks like, so you can tell whether
it worked without guessing.

**Shell:** commands are written for **Git Bash**, which is the shell in use on this machine.
Where PowerShell needs a different form it is given underneath, marked *PowerShell*. `git`, `gh`
and `npx` behave identically in both — only variables, `curl` and quoting differ:

| | Git Bash | PowerShell |
|---|---|---|
| set a variable | `NAME=value` | `$env:NAME = "value"` |
| read one | `$NAME` | `$env:NAME` |
| curl | `curl` | `curl.exe` (bare `curl` is an alias for something else) |

Mixing them up produces confusing errors — `bash: :NAME: command not found` means you pasted
PowerShell syntax into bash.

**Time:** about 30 minutes of actual work, plus up to 15 minutes of waiting for a TLS
certificate at the end. You can walk away during the wait.

---

## Contents

- [Part 1 — What is actually going on](#part-1--what-is-actually-going-on)
- [Part 2 — Vocabulary](#part-2--vocabulary)
- [Part 3 — The one genuinely dangerous command](#part-3--the-one-genuinely-dangerous-command)
- [What actually gets deployed](#what-actually-gets-deployed)
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

```bash
npx wrangler pages deploy .        # ← DO NOT RUN THIS FROM THIS FOLDER
```

...it will cheerfully upload all 108 MB, including your unprocessed source photos and the video,
to a public website.

**The rule:** deploy by `git push`. If you ever genuinely need a manual deploy, do it from a
fresh clone in a temporary directory — a clone physically cannot contain `__local/`:

```bash
git clone https://github.com/troup-miller/iterventions.git /tmp/itv-deploy
cd /tmp/itv-deploy
npx wrangler pages deploy . --project-name=iterventions
cd /c/users/troup/repos/iterventions
rm -rf /tmp/itv-deploy
```

Note that a manual deploy like this skips the CI prune step described below, so it will publish
`README.md`, `CLAUDE.md` and friends. Another reason to let CI do it.

---

## What actually gets deployed

Pages uploads **everything** in the directory it is given, dotfiles included. It does not read
`.gitignore`, and it does not read `.assetsignore` either — that was tried against a preview
deployment on wrangler 3.90 and every path it was supposed to exclude still returned 200,
including `.assetsignore` itself. The only thing wrangler drops on its own is `.git/`.

So `deploy.yml` prunes the checkout by hand before handing it over:

```yaml
      - name: Remove everything that is not the website
        run: |
          rm -rf .github .claude
          rm -f ./*.md .editorconfig .gitattributes .gitignore
```

The runner's checkout is a throwaway on a rented machine, so deleting from it costs nothing and
is completely deterministic — it does not depend on wrangler honouring anything.

Without this step the deploy runbook, the agent charters and the CI configuration are all
crawlable at `iterventions.com/DEPLOY.md` and friends. Nothing secret, since the repo is public,
but none of it is the website.

**If you add a new kind of non-site file to the repo, add it to that `rm` line.** Verify after
deploying:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://iterventions.pages.dev/README.md   # want 404
curl -s -o /dev/null -w '%{http_code}\n' https://iterventions.pages.dev/            # want 200
```

`_headers` and `_redirects` are deliberately **not** pruned. Pages consumes them as configuration
and never serves them.

---

## Where things stand right now

Verified 2026-07-26.

| Thing | State |
|---|---|
| Repo on GitHub | ✅ `github.com/troup-miller/iterventions` (public) |
| Branches | ✅ `main` (production, PR-only), `dev` (integration) |
| Site content | ✅ 2 project pages, 22 images, ~6 MB, preflight GO |
| `git` push/pull auth | ✅ working, account pinned to `troup-miller` |
| Node.js | ✅ v24.15.0 (`npx` 11.12.1) |
| `gh` CLI | ✅ logged in as `troup-miller` |
| Cloudflare API token | ✅ created, Pages: Edit |
| Cloudflare account ID | ✅ recorded |
| GitHub repo secrets | ✅ both set |
| Pages project | ✅ `iterventions`, production branch `main` |
| **Deployed** | ✅ **live at `iterventions.pages.dev`** |
| Custom domain | 🟡 both hostnames attached, **pending** — needs two DNS records, Phase 6a |
| Browser verification | ⬜ Phase 7, needs human eyes |

Phases 1–5 are done. Deploying from here is `git push`. What remains is two DNS records and a
look at the site in a real browser.

---

## Phase 0 — Check your tools

Confirms you have what you need before starting. Nothing here changes anything.

```bash
cd /c/users/troup/repos/iterventions
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

```bash
git status --short
```

If `__local/` ever appears in that output, **stop** and fix `.gitignore` before committing.

---

## Phase 1 — Log in to GitHub  · *done*

*You have to do this one yourself — it opens a browser. Already done as `troup-miller`; kept
here for a rebuild or a new machine.*

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

```bash
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
| 3 | Zone | DNS | **Edit** | lets Cloudflare create the DNS record when you attach a custom domain |

**Row 3 is optional, and this token does not have it.** The consequence is specific and worth
knowing before you decide: without zone access, attaching a custom domain creates the hostname
but **not** the DNS record that makes it resolve, leaving it stuck at `pending` with
`CNAME record not set` — see [Phase 6a](#6a-attach-both-hostnames). Adding that record by hand
is a one-time job, so a deploy token that cannot touch your DNS is a fair trade. Fewer
permissions is better.

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

```bash
read -rs CLOUDFLARE_API_TOKEN && export CLOUDFLARE_API_TOKEN
```

Press Enter, paste the token, press Enter again. Nothing echoes — `-s` is silent, so a blank
line is what success looks like. `read` keeps the value out of your shell history, which typing
`CLOUDFLARE_API_TOKEN=abc...` directly would not.

*PowerShell:*

```powershell
$env:CLOUDFLARE_API_TOKEN = Read-Host "Paste the Cloudflare API token"
```

Either way the variable lasts only as long as this terminal window. That is deliberate.

### 2c. Verify the token, and get your account ID for free

```bash
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

**If you see `Invalid API Token` from a token you know is good** — see the note on account-owned
tokens under [Phase 3](#phase-3--give-the-credentials-to-github). It is a red herring.

Store the account ID in a variable too. Unlike the token this is not secret, so type it inline
and **read it back before continuing** — putting the wrong value here is the single most likely
way to break the deploy:

```bash
CLOUDFLARE_ACCOUNT_ID=paste-the-32-hex-here
echo "${#CLOUDFLARE_ACCOUNT_ID} characters — must be 32"
echo "$CLOUDFLARE_ACCOUNT_ID" | grep -Eq '^[0-9a-f]{32}$' && echo "shape OK" || echo "WRONG SHAPE"
export CLOUDFLARE_ACCOUNT_ID
```

*PowerShell:*

```powershell
$env:CLOUDFLARE_ACCOUNT_ID = "paste-the-32-hex-here"
$env:CLOUDFLARE_ACCOUNT_ID.Length      # must print 32
```

---

## Phase 3 — Give the credentials to GitHub

Now hand both values to GitHub so the robot machine can use them. `gh secret set` reads from
standard input, so piping the variable means the token never appears as text in a command.

```bash
printf '%s' "$CLOUDFLARE_API_TOKEN"  | gh secret set CLOUDFLARE_API_TOKEN
printf '%s' "$CLOUDFLARE_ACCOUNT_ID" | gh secret set CLOUDFLARE_ACCOUNT_ID
gh secret list
```

`printf '%s'` rather than `echo` because `echo` appends a newline, and a secret with a trailing
newline in it produces errors that look nothing like "you have a stray newline."

*PowerShell:*

```powershell
$env:CLOUDFLARE_API_TOKEN  | gh secret set CLOUDFLARE_API_TOKEN
$env:CLOUDFLARE_ACCOUNT_ID | gh secret set CLOUDFLARE_ACCOUNT_ID
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

### The two mistakes that actually happened here

Both of these bit on the first real attempt, on 2026-07-26. Neither error message named the
actual problem, which is why they are written down.

**1. The token got pasted into both secrets.** Two consecutive paste prompts look identical, and
`CLOUDFLARE_ACCOUNT_ID` ended up holding the API token. Cloudflare then tried to route to
`/accounts/<a whole API token>/pages/projects/iterventions` and returned:

```
Could not route to /client/v4/accounts/***/pages/projects/iterventions,
perhaps your object identifier is invalid? [code: 7003]
```

**7003 means a URL path segment could not be parsed — read it as "your account ID is wrong."**
The shape check in Phase 2c exists to catch exactly this before you get here.

**2. The token was missing the Pages permission.** Hidden behind the first problem, and it would
have failed next with `Authentication error`. A token can be perfectly valid, authenticate fine,
list your account — and still be forbidden from touching Pages. Row 1 of the permissions table
in Phase 2a is the one that matters.

**And one red herring.** Cloudflare's own token-verify endpoint returns **401 Invalid API Token**
for this token:

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  https://api.cloudflare.com/client/v4/user/tokens/verify
```

That is expected and means nothing is wrong. `/user/tokens/verify` only accepts *user-owned*
tokens — the ones created under **My Profile → API Tokens**. A token created under **Manage
Account → API Tokens** is *account-owned*, and this endpoint rejects it while everything else
works. Do not chase it.

### Check the secrets before spending a deploy on them

You cannot read a secret back, but you can check its shape. This is worth 20 seconds:

```bash
gh secret list                       # both names present?
echo "${#CLOUDFLARE_ACCOUNT_ID}"     # 32
echo "${#CLOUDFLARE_API_TOKEN}"      # 40+, and NOT equal to the line above
```

If those two lengths are the same number, you have pasted the same value into both. That was
mistake 1.

To prove the credentials work end to end before deploying — this is the call the deploy itself
makes first:

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects" \
  | head -c 200
```

`"success":true` means the account ID parses and the token may touch Pages. Anything else and
the deploy will fail for the reason shown there, more clearly than wrangler will phrase it.

---

## Phase 4 — Create the Pages project

The container on Cloudflare's side. It must be named `iterventions`, because that is what
`.github/workflows/deploy.yml` passes to `--project-name`.

```bash
npx wrangler pages project create iterventions --production-branch main
```

**What good looks like:**

```
✨ Successfully created the 'iterventions' project.
```

**Confirm:**

```bash
npx wrangler pages project list
```

You should see `iterventions` listed, with production branch `main`.

**If it says the project already exists**, that is fine — skip ahead. **If it says
`--production-branch` is required**, add it as above; a project created with the wrong
production branch will build previews forever and never go live.

---

## Phase 5 — First deploy

Everything is wired. Trigger a run:

```bash
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

```bash
gh run view --log | grep -o "https://.*pages.dev"
```

Or:

```bash
npx wrangler pages deployment list --project-name=iterventions
```

Either gives you a `https://<something>.iterventions.pages.dev` address. **Open it.** The site is
live — just not yet on your domain.

**Do Phase 7's checks now, before attaching the domain.** It is much more relaxing to fix
problems on a URL nobody knows about.

**If the run failed:**

```bash
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

**Both hostnames are already attached** — done via the Pages API on 2026-07-26. They are sitting
at status **pending**, and the reason is worth understanding, because it is a trap:

```
"name": "iterventions.com",
"status": "pending",
"verification_data": { "status": "pending", "error_message": "CNAME record not set" }
```

Attaching a hostname to a Pages project and creating the DNS record that points at it are **two
different operations against two different permissions.** The deploy token has Pages access and
no zone access at all — `GET /zones?name=iterventions.com` returns zero results for it — so
Cloudflare accepted the hostname and then could not write the record.

This is the permission marked "safe to skip" in the Phase 2a table. It is safe to skip for
*deploying*; it is what makes *attaching a domain* automatic. Skipping it is a reasonable
trade — a deploy token that cannot touch DNS is a smaller blast radius — it just means the
records go in by hand, once, forever.

**So add the two records.** Dashboard → select the **iterventions.com** zone → **DNS** →
**Records** → **Add record**, twice:

| Type | Name | Target | Proxy |
|---|---|---|---|
| CNAME | `@` | `iterventions.pages.dev` | **Proxied** (orange cloud) |
| CNAME | `www` | `iterventions.pages.dev` | **Proxied** (orange cloud) |

`@` means the bare domain. A CNAME at the apex is normally illegal in DNS; Cloudflare allows it
via CNAME flattening, which is one of the genuine advantages of the domain already being here.

The orange cloud matters. Grey-clouded (DNS-only) records skip Cloudflare's edge, and the
certificate will never issue.

Within a few minutes both entries under **Workers & Pages → iterventions → Custom domains**
flip from **Pending** to **Active**, and the certificate issues. Up to 15 minutes is normal.

*(The alternative is to go the other way: Workers & Pages → iterventions → Custom domains →
Set up a custom domain. That flow creates the DNS record for you because it acts as your
dashboard session rather than as the deploy token. Either route ends in the same place.)*

**Check from the terminal:**

```bash
nslookup iterventions.com
curl -sI https://iterventions.com | grep -iE "HTTP/|server"
```

*PowerShell:* write **`curl.exe`**, not `curl` — bare `curl` there is an alias for
`Invoke-WebRequest`, which takes entirely different arguments and will just error.

**What good looks like:** `HTTP/2 200` and a `server: cloudflare` line.

### 6b. Redirect www to the bare domain

You now have two addresses serving identical content. Search engines dislike that, and this
site's `<link rel="canonical">`, `og:url` and `sitemap.xml` all already say the bare domain
`https://iterventions.com/`. So make `www` redirect to it.

**This is already handled in the repo.** There is a `_redirects` file at the root:

```
https://www.iterventions.com/*  https://iterventions.com/:splat  301
```

Pages reads `_redirects` as configuration and never serves it — confirmed, `/\_redirects`
returns 404 on the live site. Keeping the rule in the repo rather than in the dashboard means
it is reviewable in a diff and travels with the project.

It cannot take effect until `www.iterventions.com` is actually attached to the Pages project in
6a, because until then nothing is listening on that hostname.

**Verify once the domain is attached:**

```bash
curl -sI https://www.iterventions.com | grep -iE "HTTP/|location"
```

**What good looks like:** `HTTP/2 301` and `location: https://iterventions.com/`.

If `_redirects` turns out not to fire — Pages' support for absolute-URL sources has changed over
time — fall back to a dashboard Redirect Rule: zone `iterventions.com` → **Rules → Redirect
Rules → Create**, if hostname equals `www.iterventions.com`, then dynamic
`concat("https://iterventions.com", http.request.uri.path)`, status **301**.

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

```bash
git add -A
git commit -m "what changed"
git push
gh run watch
```

That is the entire loop. No build, no `node_modules`, no lockfile.

Per this repo's branching rules (see `CLAUDE.md`), real work goes on a branch and reaches `main`
through a pull request:

```bash
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

```bash
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

```bash
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
| `Could not route to /client/v4/accounts/…​ [code: 7003]` | **`CLOUDFLARE_ACCOUNT_ID` is not an account ID.** Usually the API token pasted into it by mistake | re-set it — Phase 2c, and run the shape check |
| `Authentication error [code: 10000]` | the API token lacks **Cloudflare Pages: Edit**, or is scoped to the wrong account | edit the token's permissions — Phase 2a, row 1 of the table. Editing keeps the same token value, so the secret does not need re-setting |
| `Invalid API Token` from `/user/tokens/verify` | your token is account-owned, that endpoint only accepts user-owned ones | ignore it — test with the `/pages/projects` call in Phase 3 instead |
| Deploy is green but `/README.md` serves | the prune step in `deploy.yml` is missing or incomplete | see [What actually gets deployed](#what-actually-gets-deployed) |
| `bash: :NAME: command not found` | PowerShell syntax pasted into Git Bash | `NAME=value`, not `$env:NAME = "value"` |
| `gh workflow run` says the workflow does not exist | `workflow_dispatch` is only exposed for workflows present on the **default branch** | trigger it with a push instead, or merge it to `main` first |
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
- The three `DATA PTS` figures on the home cards are the only numbers on the site not traceable
  to a photograph or a log entry.

---

## Git identity — stopping the account picker

Git Credential Manager pops an account chooser whenever it has more than one GitHub account
cached and nothing tells it which to use. Naming the account up front makes it resolve silently.

This is **already configured** on this machine. It is recorded here because `.git/config` is not
part of the repository — a fresh clone will not have it, and you will get the picker again.

```bash
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

```bash
git config --get-urlmatch credential https://github.com
git config --list --show-origin | grep credential.helper
```

The first should report `username troup-miller`. The second should print **exactly one** helper
line, `manager`. More than one means the chain calls Credential Manager repeatedly — which is
what caused the picker to appear here in the first place.

`gh` keeps its own separate login. If several accounts end up in it, `gh auth switch` chooses.

---

## Who does what

| Step | Who | Why | State |
|---|---|---|---|
| Phase 0 — tool check | either | read-only | ✅ |
| Phase 1 — `gh auth login` | **you** | opens a browser | ✅ |
| Phase 2 — Cloudflare token | **you** | dashboard only, and it is secret material | ✅ |
| Phase 3 — repo secrets | **you** | the token must not pass through a transcript | ✅ |
| Phase 4 — Pages project | either | one command | ✅ |
| Phase 5 — first deploy | either | one command | ✅ |
| Phase 6 — custom domain | **you** | dashboard, and DNS mistakes are slow to undo | ⬜ |
| Phase 7 — verification | **you** | needs a real browser and real eyes | ⬜ |
| Routine deploys | either | `git push` | — |

The pattern: anything involving a browser, a password or a secret is yours. Anything that is a
command with a predictable result can be handed to Claude.
