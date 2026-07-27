# iterventions.com

A chronicle of engineering by repeated mistakes.

Static site. One file, no build step, no dependencies. Hosted on Cloudflare Pages.

```
index.html          the entire site — tokens, markup, behaviour
404.html            themed not-found page
_headers            Cloudflare Pages cache + security headers
robots.txt          points at sitemap.xml
sitemap.xml         one entry; the hash routes are not separately indexable
favicon.svg         phosphor trace on charcoal
gauss/              PROJECT #001 images, v00–v04 — committed, served by Pages
ordinance/          PROJECT #002 images, v00–v06
CLAUDE.md           standing rules for anyone (human or model) touching this repo
.claude/agents/     specialised agent charters
.github/workflows/  deploy on push to main
__local/            design handoff + prototypes — gitignored, never deployed
```

## Running it

Open `index.html` in a browser. That's it — there is nothing to install and nothing to build.

For the hash routes to behave exactly as they do in production, serve it over HTTP:

```bash
python -m http.server 8000     # then visit http://localhost:8000
```

## Routes

| Hash | View |
|---|---|
| `#home` | Manifesto, the live iteration-loop scope, site conventions, project index |
| `#project/gauss` | PROJECT #001 — Gauss Sequencer, eight sections, ACTIVE at v04 |
| `#project/ordinance` | PROJECT #002 — Residential Ordinance Platform, eight sections, MOTHBALLED at v06 |
| `#log` | Bench Log, reverse chronological, both projects, 16 entries |

Bare `#project` is a legacy alias for `#project/gauss`. Anything else falls back to `#home`.
`#sec-*` fragments are in-page anchors on the project pages, not routes.

## Branching

`main` is production, by pull request only. `dev` is the integration branch. Feature branches are
cut from `dev`, named `kind/short-slug`, and merged back with a merge commit. Details in
[`CLAUDE.md`](CLAUDE.md).

## Deploying

Push to `main`. `.github/workflows/deploy.yml` runs `wrangler pages deploy .` against the
`iterventions` Pages project. Every other branch gets a preview deployment at
`<branch>.iterventions.pages.dev`.

Two repo secrets are required — `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`. **Neither is
set yet, so nothing deploys yet.**

**[`DEPLOY.md`](DEPLOY.md) is the runbook.** It is written from zero: what Pages actually is, what
a token is for, every command copy-paste with its expected output, custom domain and redirect,
a browser verification checklist, rollback, and a troubleshooting table. Start there.

One warning worth repeating here: `wrangler` does not read `.gitignore`, so never run
`npx wrangler pages deploy .` from this directory — it would upload the 108 MB `__local/` folder.
Deploy through CI, or from a fresh clone.

## Adding content

Read `CLAUDE.md` first. The short version:

- **A bench log entry** → prepend an `.entry` to `.log__list`, newest first.
- **A new version** of a project → add a `<details class="ver">` to the Evolution Timeline, update the
  Prototype Passport to the new current version, update the badge and the card stats, and revise
  Current Understanding — that last one is the one everybody forgets.
- **A photograph** → resize to ~1600 px long edge, commit it to
  `{project}/{version}/{project}_{version}_{yyyymmdd}_img_{NN}.{ext}`, and add an `<img>` with
  descriptive alt, `loading="lazy"` and its true intrinsic `width`/`height`.
- **A new project** → duplicate a `<main data-route="project/…">` block, namespace its section ids,
  and add the slug to `PROJECTS` and `TITLES` in the script.

There are agent charters in `.claude/agents/` for each of these.

## Project status

`ACTIVE`, `MOTHBALLED`, or `FINISHED`. The footer legend states all three. `FINISHED` is in the
legend as a joke and has never been used on anything, which is the point.

## Constraints

No CSS framework. No JS framework. No npm. No image CDN. No build step. Every colour, size and
duration lives in the `:root` block at the top of `index.html` — nothing outside it. Failures are
orange, never red. Dark is canonical.

Nothing here is finished on purpose.
