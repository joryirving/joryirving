# Personal Site — Cinematic Scroll Redesign

**Date:** 2026-06-26
**Status:** Approved (design)
**Repo:** `joryirving/joryirving` → deploys to `https://www.jory.dev/` via GitHub Pages

## Goal

Modernize the personal site at `jory.dev` from a plain "link-in-bio" page into a
dark, cinematic, scroll-driven personal brand / portfolio — the Apple / Linear /
Vercel product-site feel. Keep the existing deploy pipeline, `/metrics` page, and
the downloadable resume PDF intact. Add a full HTML resume at `/resume`.

## Current State (baseline)

- **Hugo** static site, deployed to GitHub Pages by `.github/workflows/hugo.yml`
  (Hugo extended `0.128.0`, `--minify`).
- Theme: **`github.com/jpanther/lynx` v1.4.0**, imported as a Hugo module
  (`links/config/_default/module.toml`, `links/go.mod`). Tailwind-based.
- Home page (`/`): lynx-rendered centered link list — name, "SRE 💻", and link
  buttons (Email, Instagram, LinkedIn, Metrics, Resume, Homelab). Config-driven via
  `links/hugo.toml` `[params.author]`.
- `/metrics`: `links/content/metrics.html` — embeds `static/metrics/heavy.svg`
  (regenerated daily by `.github/workflows/metrics.yaml`).
- `/resume/cv.pdf`: static PDF at `links/static/resume/cv.pdf`.
- Stats SVGs (`stats/*.svg`) generated daily by `.github/workflows/stats.yml` and
  surfaced in the repo `README.md` (GitHub profile), not the site.

## Decisions (from brainstorming)

| Question | Decision |
|---|---|
| Page purpose | Personal brand / portfolio |
| Landing sections | Hero, About/story, Skills/tech stack, Homelab/projects (+ contact/links footer) |
| Aesthetic | Dark & cinematic |
| Motion tech | Native CSS scroll-driven animations + IntersectionObserver, **plus Lenis** smooth-scroll (vendored locally) |
| `/resume` | **Full HTML resume**, themed; keep `cv.pdf` as a download |
| Architecture | **Option A** — custom Hugo layouts, keep lynx only for `/metrics` |

## Architecture — Option A

Build custom Hugo layouts in `links/layouts/` that override the lynx theme for the
home page and add a resume page. Keep lynx as the module so `/metrics` continues to
render unchanged. No change to the deploy workflow.

- `links/layouts/index.html` — fully custom cinematic landing page (overrides lynx home).
- `links/layouts/resume/single.html` (or a dedicated layout) — full HTML resume page,
  served at `/resume`.
- `links/content/resume.md` (or `resume/_index.md`) — front matter that selects the
  resume layout; resume content comes from a data file (below).
- `links/data/resume.yaml` — structured resume content (single source of truth for
  the HTML resume). Keeps content separate from presentation and easy to update.
- `links/assets/css/site.css` — custom stylesheet, processed + fingerprinted +
  minified by Hugo's asset pipeline.
- `links/assets/js/main.js` — Lenis init + IntersectionObserver reveal helper.
- `links/assets/js/lenis.min.js` (or vendored under `assets/vendor/`) — **vendored
  locally**, no runtime CDN dependency.

Rejected alternatives: (B) remove lynx entirely — extra work to rebuild `/metrics`
for no gain; (C) abandon Hugo for hand-written HTML — loses the Pages pipeline and
asset handling.

## Landing Page (`/`) — sections

Single-page vertical scroll, dark & cinematic. Lenis smooth-scroll; reveals driven by
CSS scroll-driven animations + IntersectionObserver. Order:

1. **Hero** — full-viewport. "Jory Irving" large bold display type; "SRE 💻" + a
   tagline; subtle animated background (gradient mesh or grid); scroll cue. Parallax /
   fade on scroll.
2. **About / story** — short narrative from the resume intro (15+ years, generalist
   IT + SRE/DevOps, strong DevOps fundamentals, quick to learn). Text reveals on scroll.
3. **Skills / tech stack** — animated, staggered grid of tools: AWS, Kubernetes,
   Terraform, ArgoCD / Flux, Docker, CI/CD, GitOps, Linux. May optionally fold in
   `static/.../top-langs.svg` if available to the site.
4. **Homelab / projects** — feature `home-ops` / k8s as the centerpiece, link to
   GitHub, surface `stats.svg` / metrics imagery. Cards animate in.
5. **Contact / links footer** — existing links restyled to match: Email
   (`jory@jory.dev`), LinkedIn, Instagram, GitHub/Homelab, Metrics (`/metrics`),
   Resume (`/resume`).

## Resume Page (`/resume`) — full HTML resume

Styled, responsive, dark-themed HTML resume served at `/resume`, built from
`links/data/resume.yaml`. Includes a **"Download PDF"** button → `/resume/cv.pdf`
(unchanged). Sections, from the existing PDF content:

- **Summary** — generalist IT + SRE/DevOps, 15+ years, DevOps fundamentals.
- **Experience** — StackAdapt (current), Tempo Software, Blacksquare.io,
  ColdBore Technology, Brightside by ATB, Parkland Fuel, Talisman Energy (titles,
  dates, bullet highlights). StackAdapt and Tempo dates per the "current role"
  resolution below; earlier roles as in the PDF.
- **Skills** — AWS, CI/CD, GitOps, Flux/ArgoCD, Terraform, Kubernetes, Docker,
  Linux, Windows, Agile, Time Management.
- **Languages** — English; Basic French (read/spoken).
- **Education** — SAIT, Computer Engineering Technologist Diploma (2006–2007).

### Current role — resolved

The PDF was out of date (listed Tempo as present). Corrected:

- **StackAdapt — DevOps Engineer, May 2025 – Present.** Highlights:
  - Rolled out metadata-labelling standardization across Kubernetes infrastructure,
    enabling dynamic owner-based service alert routing.
  - Improved Observability-stack reliability: migrated Grafana and Grafana OnCall
    backends from SQLite to Postgres and enabled Grafana high-availability mode.
  - Rolled out Gatus plus accompanying Prometheus alerts for endpoint monitoring and
    fast feedback to critical service teams.
- **Tempo Software — DevOps/SRE, October 2022 – May 2025.** No longer the current role.

Optional enrichment: if a role reads thin in the rendered resume, pull a couple more
specifics from Jira/Slack via MCP, *targeted* at that role — not a broad data sweep.

## Tech details

- **Lenis** vendored locally under `links/assets/`; initialized in `main.js`.
  Smooth-scroll disabled / softened on touch devices where appropriate.
- **CSS** via Hugo asset pipeline (`resources.Get` → fingerprint → minify).
- **Accessibility:** respect `prefers-reduced-motion` — animations degrade to
  instant/no-motion; content fully readable without JS.
- **Fonts:** bold display font for headings, self-hosted or system stack —
  **no runtime CDN** (consistent with vendoring Lenis).
- **Responsive:** works down to mobile; layout reflows, motion tuned for small screens.
- **Dark theme** aligns with the existing `one_dark_pro` stats cards.

## Out of scope / untouched

- `/metrics` page and its `metrics.yaml` automation.
- `/resume/cv.pdf` (kept as the downloadable export).
- `.github/workflows/hugo.yml` deploy pipeline (no changes expected).
- `stats.yml` / `update-stats.yml` automation and repo `README.md` (GitHub profile).

## Success criteria

- `/` renders the dark cinematic scroll page with all five sections and working
  Lenis smooth-scroll + reveal animations; degrades gracefully with reduced-motion
  and without JS.
- `/resume` renders a themed HTML resume from `resume.yaml` with a working PDF
  download link; current role is correct (or a visible TODO placeholder pending input).
- `/metrics` and `/resume/cv.pdf` still work unchanged.
- Site builds with the existing `hugo.yml` workflow and deploys to `jory.dev`.
- No runtime third-party CDN dependencies.
