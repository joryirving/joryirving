# Personal Site — Cinematic Scroll Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the lynx "link-in-bio" home page with a dark, cinematic, scroll-driven personal-brand page and add a themed full HTML resume at `/resume`, without disturbing `/metrics`, `/resume/cv.pdf`, or the deploy pipeline.

**Architecture:** Custom Hugo layouts under `links/layouts/` override the lynx theme for the home page (`index.html`) and add a resume page (`_default/resume.html`), while lynx stays installed and continues to render `/metrics`. Both custom pages are **standalone HTML documents** (their own `<html>`) that share `<head>`/script partials — deliberately NOT using a project-level `_default/baseof.html`, because that would override lynx's baseof and break `/metrics`. Styling and JS go through Hugo's asset pipeline (minify + fingerprint); Lenis is vendored locally so there is no runtime CDN dependency.

**Tech Stack:** Hugo (extended) + Go modules, vanilla JS, Lenis 1.3.23 (smooth scroll), native CSS scroll/IntersectionObserver reveals. No build-time npm for the site.

## Verification model (read before starting)

This is a static Hugo template/CSS site — there is **no unit-test framework**, and inventing fake unit tests would be worse than useless. The per-task "test cycle" is therefore **build + assert + visual**:

- **Assert:** `cd links && hugo --gc --minify` builds with no error, then `grep` the generated files under `links/public/` for expected markers. The "failing" state is simply that the marker is absent before you implement the task — confirm that, then implement, then confirm it's present.
- **Visual:** `cd links && hugo server -D` and view `http://localhost:1313/` in a browser. The final task does a full visual + behavior pass (smooth scroll, reveals, reduced-motion, mobile width) using the `verify` skill.

`links/public/` is a build artifact and is git-ignored conceptually; do **not** commit it. Commit only sources.

## Global Constraints

- Site root for all Hugo commands is `links/` (the Hugo project lives there, `baseURL = https://www.jory.dev/`).
- **No runtime third-party CDN.** Lenis is vendored at `links/assets/js/vendor/lenis.min.js`. Fonts are self-hosted or a system stack only.
- **Do not modify, move, or delete** `links/static/resume/cv.pdf`, `links/content/metrics.html`, `links/config/_default/module.toml`, or `.github/workflows/*`.
- **Do not** create `links/layouts/_default/baseof.html` (it would override lynx and break `/metrics`).
- Accessibility: every animation must respect `prefers-reduced-motion: reduce` (degrade to no motion); all content must be readable with JavaScript disabled.
- Dark theme, aligned with the existing `one_dark_pro` stats palette.
- Aesthetic & sections are fixed by the spec: Hero, About, Skills, Homelab/Projects, Contact footer (landing); full themed resume at `/resume`.
- Spec: `docs/superpowers/specs/2026-06-26-personal-site-cinematic-redesign-design.md`.
- Resume content source of truth: `links/data/resume.yaml`. Current role is **StackAdapt — DevOps Engineer, May 2025–Present**; Tempo is past (**Oct 2022 – May 2025**).
- The public HTML resume **omits the phone number** (privacy); the PDF retains it.

---

### Task 1: Toolchain, baseline build, and vendored Lenis

**Files:**
- Create: `links/assets/js/vendor/lenis.min.js` (downloaded once, then committed)
- Create/modify: `.gitignore` (ensure `links/public/` and Hugo lock/cache are ignored)

**Interfaces:**
- Consumes: nothing.
- Produces: a confirmed-working local Hugo build of the *current* site, and `assets/js/vendor/lenis.min.js` (UMD bundle exposing the global `Lenis`) for Task 3.

- [ ] **Step 1: Install the toolchain**

Hugo-extended needs Go to fetch the lynx module.

Run:
```bash
brew install hugo go
hugo version    # must say "extended"
go version
```
Expected: Hugo version string containing `+extended`, and a Go version.

- [ ] **Step 2: Establish the baseline build (this is the "before" assertion)**

Run:
```bash
cd links && hugo --gc --minify
```
Expected: build succeeds (lynx module downloads on first run), `links/public/index.html`, `links/public/metrics/index.html`, and `links/public/resume/cv.pdf` all exist. This is the untouched lynx site — confirm it works before changing anything.

- [ ] **Step 3: Ensure build artifacts are git-ignored**

Read the repo `.gitignore` (create it at repo root if absent). Ensure it contains:
```gitignore
# Hugo build output / locks / cache
links/public/
links/resources/
links/.hugo_build.lock
.hugo_build.lock
```
If `links/public/` is currently tracked, untrack it: `git rm -r --cached links/public` (do not delete the files).

- [ ] **Step 4: Vendor Lenis (download once, then it lives in git)**

Run:
```bash
mkdir -p links/assets/js/vendor
curl -fsSL https://unpkg.com/lenis@1.3.23/dist/lenis.min.js -o links/assets/js/vendor/lenis.min.js
head -c 120 links/assets/js/vendor/lenis.min.js   # sanity check it's JS, not an error page
test -s links/assets/js/vendor/lenis.min.js && echo OK
```
Expected: file is non-empty JS (downloading at author time is fine — there is no *runtime* CDN dependency once vendored).

- [ ] **Step 5: Verify the vendored bundle exposes a global**

Run:
```bash
grep -c "Lenis" links/assets/js/vendor/lenis.min.js
```
Expected: a count > 0 (the UMD bundle defines `Lenis`).

- [ ] **Step 6: Commit**

```bash
git add .gitignore links/assets/js/vendor/lenis.min.js
git commit -m "chore: vendor Lenis 1.3.23 and ignore Hugo build output"
```

---

### Task 2: Landing page structure + head partial + dark base CSS (no motion yet)

**Files:**
- Create: `links/layouts/index.html` (standalone home document)
- Create: `links/layouts/partials/site-head.html` (shared `<head>`)
- Create: `links/assets/css/site.css` (dark theme + layout + sections)

**Interfaces:**
- Consumes: nothing from other tasks. Reads `.Site.Params.author.headline` and `.Site.Title` from `links/hugo.toml`.
- Produces:
  - `partials/site-head.html` — a head partial called as `{{ partial "site-head.html" . }}`; emits the fingerprinted stylesheet link from `assets/css/site.css`, meta, and favicons. Reused by Task 4.
  - DOM contract for Task 3: animatable elements carry the attribute `data-reveal` and optional inline `style="--delay:.1s"`; sections use ids `#hero #about #skills #homelab #contact`.

- [ ] **Step 1: Confirm the marker is absent (the "before" state)**

Run:
```bash
cd links && hugo --gc 2>/dev/null; grep -l "data-reveal" public/index.html 2>/dev/null || echo "MARKER ABSENT (expected)"
```
Expected: `MARKER ABSENT (expected)` — the current lynx home has no such markup.

- [ ] **Step 2: Create the shared head partial**

Create `links/layouts/partials/site-head.html`:
```html
{{- $css := resources.Get "css/site.css" | minify | fingerprint -}}
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>{{ if .IsHome }}{{ .Site.Title }}{{ else }}{{ .Title }} · {{ .Site.Title }}{{ end }}</title>
<meta name="description" content="{{ .Site.Title }} — Site Reliability / DevOps Engineer." />
<link rel="canonical" href="{{ .Permalink }}" />
<link rel="stylesheet" href="{{ $css.RelPermalink }}" integrity="{{ $css.Data.Integrity }}" />
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png" />
<link rel="manifest" href="/site.webmanifest" />
<meta property="og:title" content="{{ .Site.Title }}" />
<meta property="og:type" content="website" />
<meta property="og:url" content="{{ .Permalink }}" />
```
(The favicon/manifest files are published by the lynx theme's static mount — they already appear in `public/`.)

- [ ] **Step 3: Create the standalone home layout with all five sections**

Create `links/layouts/index.html`. Content is fully populated (no animation classes wired to JS yet beyond the `data-reveal` attribute, which is inert without Task 3):
```html
<!doctype html>
<html lang="en-us" data-theme="dark">
<head>
  {{ partial "site-head.html" . }}
</head>
<body>
  <main>
    <!-- HERO -->
    <section id="hero" class="hero">
      <div class="hero__bg" aria-hidden="true"></div>
      <div class="hero__inner">
        <p class="eyebrow" data-reveal>Site Reliability Engineer</p>
        <h1 class="hero__title" data-reveal style="--delay:.08s">Jory&nbsp;Irving</h1>
        <p class="hero__tagline" data-reveal style="--delay:.16s">
          Father · SRE/DevOps · gym rat · car nerd · general nerd.
        </p>
        <a class="scroll-cue" href="#about" aria-label="Scroll to content" data-reveal style="--delay:.28s">
          <span></span>
        </a>
      </div>
    </section>

    <!-- ABOUT -->
    <section id="about" class="section section--about">
      <div class="wrap">
        <h2 class="section__title" data-reveal>About</h2>
        <p class="lede" data-reveal style="--delay:.06s">
          A generalist IT professional and SRE/DevOps engineer with 15+ years across technology —
          drawn to complex projects, exciting implementations, and constant improvement.
        </p>
        <p data-reveal style="--delay:.12s">
          I practice strong DevOps fundamentals, pairing technical and analytical skill with a
          sense of urgency to solve problems quickly in fast-paced environments. Quick to learn,
          and happiest when the people around me are set up to do their best work too.
        </p>
      </div>
    </section>

    <!-- SKILLS -->
    <section id="skills" class="section section--skills">
      <div class="wrap">
        <h2 class="section__title" data-reveal>Tech stack</h2>
        <ul class="skills-grid">
          {{ range $i, $s := slice "Kubernetes" "AWS" "Terraform" "ArgoCD / Flux" "Docker" "CI/CD" "GitOps" "Grafana" "Prometheus" "Linux" }}
          <li class="skill" data-reveal style="--delay:{{ mul $i 0.05 }}s">{{ $s }}</li>
          {{ end }}
        </ul>
      </div>
    </section>

    <!-- HOMELAB / PROJECTS -->
    <section id="homelab" class="section section--homelab">
      <div class="wrap">
        <h2 class="section__title" data-reveal>Homelab &amp; projects</h2>
        <div class="cards">
          <a class="card card--feature" href="https://github.com/joryirving/home-ops" target="_blank" rel="noopener noreferrer" data-reveal>
            <h3>home-ops</h3>
            <p>A GitOps-managed Kubernetes homelab — my playground for the patterns I run in production: declarative infra, automated reconciliation, and observability from the ground up.</p>
            <span class="card__cta">View on GitHub →</span>
          </a>
          <div class="card card--stats" data-reveal style="--delay:.08s">
            <h3>GitHub activity</h3>
            <img src="/metrics/heavy.svg" alt="GitHub metrics" loading="lazy" />
            <a class="card__cta" href="/metrics">See full metrics →</a>
          </div>
        </div>
      </div>
    </section>

    <!-- CONTACT -->
    <footer id="contact" class="section section--contact">
      <div class="wrap">
        <h2 class="section__title" data-reveal>Let's connect</h2>
        <nav class="links" aria-label="Contact links" data-reveal style="--delay:.06s">
          <a href="mailto:jory@jory.dev">Email</a>
          <a href="https://www.linkedin.com/in/joryirving/" target="_blank" rel="me noopener noreferrer">LinkedIn</a>
          <a href="https://www.instagram.com/lildrunkensmurf/" target="_blank" rel="me noopener noreferrer">Instagram</a>
          <a href="https://github.com/joryirving/home-ops" target="_blank" rel="me noopener noreferrer">Homelab</a>
          <a href="/metrics">Metrics</a>
          <a href="/resume">Resume</a>
        </nav>
        <p class="colophon">© {{ now.Year }} Jory Irving · Built with Hugo</p>
      </div>
    </footer>
  </main>
</body>
</html>
```

- [ ] **Step 4: Create the dark base stylesheet**

Create `links/assets/css/site.css`:
```css
/* ---- Lenis (smooth scroll) recommended base ---- */
html.lenis, html.lenis body { height: auto; }
.lenis.lenis-smooth { scroll-behavior: auto !important; }
.lenis.lenis-stopped { overflow: hidden; }

/* ---- Tokens (one_dark_pro-aligned dark theme) ---- */
:root {
  --bg: #0b0e14;
  --panel: #11151d;
  --text: #e6edf3;
  --muted: #8b94a3;
  --accent: #61afef;
  --accent-2: #c678dd;
  --maxw: 60rem;
  --ease: cubic-bezier(.2,.7,.2,1);
}
* { box-sizing: border-box; }
html { scroll-behavior: smooth; -webkit-text-size-adjust: 100%; }
body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  font: 400 clamp(1rem, .9rem + .4vw, 1.15rem)/1.65 system-ui, -apple-system, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
}
a { color: inherit; }
img { max-width: 100%; height: auto; display: block; }
.wrap { width: min(var(--maxw), 92vw); margin-inline: auto; }
.section { padding: clamp(4rem, 12vh, 9rem) 0; }
.section__title {
  font-size: clamp(1.6rem, 1.2rem + 2vw, 2.6rem);
  font-weight: 800; letter-spacing: -.02em; margin: 0 0 1.5rem;
}
.lede { font-size: clamp(1.15rem, 1rem + .8vw, 1.6rem); color: var(--text); }

/* ---- Hero ---- */
.hero { position: relative; min-height: 100svh; display: grid; place-items: center; overflow: hidden; text-align: center; }
.hero__bg {
  position: absolute; inset: -20% -20% -20% -20%;
  background:
    radial-gradient(40% 40% at 30% 30%, color-mix(in oklab, var(--accent) 40%, transparent), transparent 70%),
    radial-gradient(40% 40% at 70% 60%, color-mix(in oklab, var(--accent-2) 38%, transparent), transparent 70%);
  filter: blur(40px); opacity: .55;
  animation: drift 18s var(--ease) infinite alternate;
}
@keyframes drift { to { transform: translate3d(4%, -3%, 0) scale(1.08); } }
.hero__inner { position: relative; z-index: 1; padding: 1rem; }
.eyebrow { text-transform: uppercase; letter-spacing: .28em; font-size: .8rem; color: var(--muted); margin: 0 0 1rem; }
.hero__title {
  font-size: clamp(3rem, 2rem + 10vw, 8rem); font-weight: 900; letter-spacing: -.03em;
  line-height: .95; margin: 0;
  background: linear-gradient(120deg, var(--text), var(--accent) 60%, var(--accent-2));
  -webkit-background-clip: text; background-clip: text; color: transparent;
}
.hero__tagline { color: var(--muted); font-size: clamp(1.1rem, 1rem + .8vw, 1.6rem); margin: 1.25rem auto 0; max-width: 36ch; }
.scroll-cue { display: inline-block; margin-top: 2.5rem; width: 26px; height: 42px; border: 2px solid var(--muted); border-radius: 14px; position: relative; }
.scroll-cue span { position: absolute; left: 50%; top: 8px; width: 4px; height: 8px; margin-left: -2px; border-radius: 2px; background: var(--text); animation: cue 1.6s var(--ease) infinite; }
@keyframes cue { 0%{opacity:0;transform:translateY(0)} 30%{opacity:1} 100%{opacity:0;transform:translateY(14px)} }

/* ---- Skills ---- */
.skills-grid { list-style: none; margin: 0; padding: 0; display: grid; gap: .75rem; grid-template-columns: repeat(auto-fill, minmax(9rem, 1fr)); }
.skill { background: var(--panel); border: 1px solid color-mix(in oklab, var(--text) 10%, transparent); border-radius: .75rem; padding: 1rem 1.1rem; font-weight: 600; }

/* ---- Cards ---- */
.cards { display: grid; gap: 1.25rem; grid-template-columns: 1fr; }
@media (min-width: 48rem) { .cards { grid-template-columns: 1fr 1fr; } }
.card { background: var(--panel); border: 1px solid color-mix(in oklab, var(--text) 10%, transparent); border-radius: 1rem; padding: 1.5rem; text-decoration: none; transition: transform .3s var(--ease), border-color .3s var(--ease); }
.card:hover { transform: translateY(-4px); border-color: color-mix(in oklab, var(--accent) 50%, transparent); }
.card h3 { margin: 0 0 .5rem; font-size: 1.4rem; }
.card p { color: var(--muted); margin: 0 0 1rem; }
.card__cta { color: var(--accent); font-weight: 600; }

/* ---- Contact ---- */
.section--contact { border-top: 1px solid color-mix(in oklab, var(--text) 8%, transparent); }
.links { display: flex; flex-wrap: wrap; gap: 1rem 1.75rem; }
.links a { color: var(--text); text-decoration: none; font-weight: 600; padding-bottom: 2px; border-bottom: 2px solid transparent; transition: border-color .2s var(--ease), color .2s var(--ease); }
.links a:hover { color: var(--accent); border-color: var(--accent); }
.colophon { color: var(--muted); font-size: .85rem; margin-top: 2.5rem; }
```

- [ ] **Step 5: Build and assert the structure renders**

Run:
```bash
cd links && hugo --gc --minify
grep -o 'id="hero"\|id="about"\|id="skills"\|id="homelab"\|id="contact"' public/index.html | sort -u
grep -c "data-reveal" public/index.html
grep -o 'css/site\.[a-f0-9]*\.css' public/index.html | head -1
```
Expected: all five section ids printed, a `data-reveal` count > 0, and a fingerprinted `css/site.<hash>.css` reference.

- [ ] **Step 6: Visual check (static, no motion yet)**

Run `cd links && hugo server` and open `http://localhost:1313/`. Expected: dark page, gradient hero with large gradient name, readable About/Skills/Homelab/Contact sections, the `home-ops` card and the metrics SVG visible, footer links present. Content is fully visible (reveal is not yet wired, so nothing should be hidden). Stop the server.

- [ ] **Step 7: Commit**

```bash
git add links/layouts/index.html links/layouts/partials/site-head.html links/assets/css/site.css
git commit -m "feat: custom dark cinematic landing page (static structure)"
```

---

### Task 3: Scroll animations — Lenis smooth scroll + reveal-on-scroll + reduced-motion

**Files:**
- Create: `links/assets/js/main.js`
- Modify: `links/layouts/partials/site-head.html` (append the script includes)
- Modify: `links/assets/css/site.css` (append reveal styles)

**Interfaces:**
- Consumes: the vendored global `Lenis` from Task 1; the `data-reveal` / `--delay` DOM contract and section ids from Task 2; the `site-head.html` partial from Task 2.
- Produces: nothing consumed by later tasks (Task 4's resume page intentionally does NOT use Lenis/reveal — it's a document page).

- [ ] **Step 1: Confirm scripts are not yet wired (before state)**

Run:
```bash
cd links && hugo --gc 2>/dev/null; grep -c "lenis" public/index.html
```
Expected: `0`.

- [ ] **Step 2: Append reveal styles to the stylesheet**

Append to `links/assets/css/site.css`:
```css
/* ---- Scroll reveal ---- */
[data-reveal] {
  opacity: 0;
  transform: translateY(26px);
  transition: opacity .7s var(--ease), transform .7s var(--ease);
  transition-delay: var(--delay, 0s);
  will-change: opacity, transform;
}
[data-reveal].is-visible { opacity: 1; transform: none; }
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
  [data-reveal] { opacity: 1 !important; transform: none !important; transition: none !important; }
  .hero__bg, .scroll-cue span { animation: none !important; }
}
```

- [ ] **Step 3: Create the JS (Lenis + IntersectionObserver), guarded for reduced-motion / touch / no-JS**

Create `links/assets/js/main.js`:
```js
(() => {
  'use strict';
  const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  const isTouch = window.matchMedia('(hover: none), (pointer: coarse)').matches;

  // Smooth scroll: skip for reduced-motion, touch devices, or if Lenis failed to load.
  if (!reduceMotion && !isTouch && typeof window.Lenis === 'function') {
    // eslint-disable-next-line no-new
    new window.Lenis({ autoRaf: true, lerp: 0.1 });
  }

  // Reveal on scroll.
  const els = document.querySelectorAll('[data-reveal]');
  if (reduceMotion || !('IntersectionObserver' in window)) {
    els.forEach((el) => el.classList.add('is-visible'));
    return;
  }
  const io = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add('is-visible');
        io.unobserve(entry.target);
      }
    });
  }, { threshold: 0.15, rootMargin: '0px 0px -10% 0px' });
  els.forEach((el) => io.observe(el));
})();
```

- [ ] **Step 4: Wire the scripts into the head partial**

Append to `links/layouts/partials/site-head.html` (only emit on the home page; the resume page doesn't need motion):
```html
{{- if .IsHome -}}
{{- $lenis := resources.Get "js/vendor/lenis.min.js" | fingerprint -}}
{{- $main := resources.Get "js/main.js" | minify | fingerprint -}}
<script src="{{ $lenis.RelPermalink }}" integrity="{{ $lenis.Data.Integrity }}" defer></script>
<script src="{{ $main.RelPermalink }}" integrity="{{ $main.Data.Integrity }}" defer></script>
{{- end -}}
```
> Note: `defer` preserves order, so `Lenis` is defined before `main.js` runs. A no-JS visitor still sees everything because `is-visible` styling only *adds* visibility — but `[data-reveal]` starts at `opacity:0`. To guarantee no-JS readability, add the fallback in Step 5.

- [ ] **Step 5: Add a no-JS fallback so hidden elements still show without JavaScript**

Append to `links/assets/css/site.css`:
```css
/* No-JS fallback: if JS never runs, reveal everything. */
.no-js [data-reveal] { opacity: 1; transform: none; }
```
And in `links/layouts/index.html`, change the opening `<html ...>` tag to mark no-js by default and let JS remove it. Replace:
```html
<html lang="en-us" data-theme="dark">
```
with:
```html
<html lang="en-us" data-theme="dark" class="no-js">
<head>
<script>document.documentElement.classList.remove('no-js');</script>
```
…and remove the now-duplicate `<head>` line that previously followed the `<html>` tag (there must be exactly one `<head>`). The tiny inline script runs immediately, so JS-enabled browsers drop `no-js` before paint and the reveal animation works; JS-disabled browsers keep `no-js` and see all content.

- [ ] **Step 6: Build and assert the scripts are wired**

Run:
```bash
cd links && hugo --gc --minify
grep -o 'js/vendor/lenis\.[a-f0-9]*\.js' public/index.html | head -1
grep -o 'js/main\.[a-f0-9]*\.js' public/index.html | head -1
grep -c "is-visible" public/css/site*.css
```
Expected: a fingerprinted lenis path, a fingerprinted main.js path, and an `is-visible` count > 0 in the bundled CSS.

- [ ] **Step 7: Visual check (motion)**

`cd links && hugo server`, open `http://localhost:1313/`. Expected: smooth inertial scrolling; sections/skills fade-and-rise into view as you scroll (staggered in the skills grid); hero gradient drifts. Then in browser devtools enable "Emulate prefers-reduced-motion: reduce" and reload — content is fully visible with no motion and normal scrolling. Stop the server.

- [ ] **Step 8: Commit**

```bash
git add links/assets/js/main.js links/layouts/partials/site-head.html links/assets/css/site.css links/layouts/index.html
git commit -m "feat: Lenis smooth scroll + reveal-on-scroll with reduced-motion + no-js fallbacks"
```

---

### Task 4: Resume page — data file, themed template, PDF download

**Files:**
- Create: `links/data/resume.yaml`
- Create: `links/content/resume.md`
- Create: `links/layouts/_default/resume.html` (standalone document)
- Modify: `links/assets/css/site.css` (append resume styles)

**Interfaces:**
- Consumes: the `site-head.html` partial from Task 2 (reused for `<head>`); `--bg/--text/--accent` tokens from Task 2's CSS.
- Produces: a page at `/resume` rendered from `.Site.Data.resume`. Relies on the `cv.pdf` already at `links/static/resume/cv.pdf` (do not touch it).

- [ ] **Step 1: Confirm `/resume` is not yet an HTML page (before state)**

Run:
```bash
cd links && hugo --gc 2>/dev/null; test -f public/resume/index.html && echo "EXISTS" || echo "ABSENT (expected)"
```
Expected: `ABSENT (expected)` (only `public/resume/cv.pdf` exists today).

- [ ] **Step 2: Create the resume data file (single source of truth)**

Create `links/data/resume.yaml` (phone intentionally omitted for the public page):
```yaml
name: Jory Irving
title: SRE / DevOps Engineer
email: jory@jory.dev
pdf: /resume/cv.pdf
summary: >-
  A generalist IT professional and SRE/DevOps engineer with 15+ years across
  technology, who enjoys complex projects, exciting implementations, and constant
  improvement. I practice strong DevOps fundamentals — pairing technical and
  analytical skill with a sense of urgency to solve problems quickly in fast-paced
  settings — and I encourage the people around me to do their best work too.
experience:
  - company: StackAdapt
    role: DevOps Engineer
    start: May 2025
    end: Present
    highlights:
      - Rolled out metadata-labelling standardization across Kubernetes infrastructure, enabling dynamic owner-based service alert routing.
      - Improved Observability-stack reliability — migrated Grafana and Grafana OnCall backends from SQLite to Postgres and enabled Grafana high-availability mode.
      - Rolled out Gatus plus accompanying Prometheus alerts for endpoint monitoring and fast feedback to critical service teams.
  - company: Tempo Software
    role: DevOps / SRE
    start: October 2022
    end: May 2025
    highlights:
      - Managed multiple EKS clusters across regions, environments, and AWS accounts.
      - Managed AWS and Datadog via IaC and CI/CD pipelines; managed Kubernetes services via GitOps with ArgoCD.
      - Modularized IaC to keep code DRY; rolled out new services including database backends with automatic provisioning.
      - Reduced Datadog spend and optimized metric collection.
  - company: Blacksquare.io
    role: DevOps / SRE / Cloud Engineer
    start: September 2021
    end: October 2022
    highlights:
      - Managed multiple AWS accounts for various clients, products, and dev environments, with a mix of Terraform and CloudFormation.
      - Containerized software on Fargate for high availability and automatic recovery/scaling; migrated infrastructure to reduce costs.
      - Set up backups and PITR for databases; raised cloud security toward SOC 2 compliance.
      - Owned RACI documentation and processes; set up monitoring to ensure uptime and reduce redundancy.
  - company: ColdBore Technology
    role: DevOps / SRE / Cloud Engineer
    start: April 2021
    end: August 2021
    highlights:
      - Built new technology to gather and format data from multiple sources.
      - Implemented cost savings reducing a cloud environment's cost by 35%+; imported all infrastructure to code.
      - Implemented SSO for internal users; migrated EC2-hosted Docker to Fargate and optimized images.
  - company: Brightside by ATB
    role: DevOps / SRE
    start: May 2019
    end: April 2021
    highlights:
      - Helped build a banking app — turned long-lived "pet" environments into scalable, dynamic "cattle" and migrated all infrastructure to code.
      - Implemented zero-downtime deployments and cut deployment time by 50%+; integrated code scanning and quality analysis into builds.
      - Provided 99.99% infrastructure uptime and earned glowing security and pen-test reports.
  - company: Parkland Fuel
    role: Desktop Solution Designer
    start: February 2018
    end: September 2018
    highlights:
      - Inventoried desktop fleet for an acquisition and produced procurement recommendations.
      - Designed and ran a phased deployment schedule minimizing disruption, with a white-glove approach to affected users.
  - company: Talisman Energy
    role: Desktop Solutions — Infrastructure
    start: September 2014
    end: September 2017
    highlights:
      - Administered an MDM solution for 2000+ devices, including a tested DR failover solution.
      - Migrated all users to a new MDM during the Repsol acquisition; packaged/deployed apps via SCCM 2012 to 4000+ PCs.
skills:
  - AWS
  - CI/CD
  - GitOps
  - Flux / ArgoCD
  - Terraform
  - Kubernetes
  - Docker
  - Linux
  - Windows
  - Agile
languages:
  - English
  - Basic French (read / spoken)
education:
  - school: Southern Alberta Institute of Technology
    credential: Computer Engineering Technologist Diploma
    start: Jan 2006
    end: Dec 2007
    notes: Calculus, Networking, Peripheral Systems, POSIX Operating Systems, Technical Communications, Microprocessor Architecture.
```

- [ ] **Step 3: Create the content page that selects the resume layout**

Create `links/content/resume.md`:
```markdown
---
title: "Resume"
layout: "resume"
---
```
(Hugo resolves `layout: resume` for a regular page to `layouts/_default/resume.html`. This publishes to `/resume/`, which does NOT collide with the static `/resume/cv.pdf`.)

- [ ] **Step 4: Create the standalone resume template**

Create `links/layouts/_default/resume.html`:
```html
<!doctype html>
<html lang="en-us" data-theme="dark">
<head>
  {{ partial "site-head.html" . }}
</head>
<body class="resume">
  {{ $r := .Site.Data.resume }}
  <main class="wrap resume__main">
    <header class="resume__header">
      <a class="resume__back" href="/">← Home</a>
      <h1>{{ $r.name }}</h1>
      <p class="resume__role">{{ $r.title }}</p>
      <p class="resume__contact">
        <a href="mailto:{{ $r.email }}">{{ $r.email }}</a>
        <a class="btn" href="{{ $r.pdf }}" target="_blank" rel="noopener">Download PDF ↓</a>
      </p>
      <p class="resume__summary">{{ $r.summary }}</p>
    </header>

    <section class="resume__section">
      <h2>Experience</h2>
      {{ range $r.experience }}
      <article class="job">
        <div class="job__head">
          <h3>{{ .company }} <span class="job__role">— {{ .role }}</span></h3>
          <span class="job__dates">{{ .start }} – {{ .end }}</span>
        </div>
        <ul>
          {{ range .highlights }}<li>{{ . }}</li>{{ end }}
        </ul>
      </article>
      {{ end }}
    </section>

    <div class="resume__cols">
      <section class="resume__section">
        <h2>Skills</h2>
        <ul class="taglist">{{ range $r.skills }}<li>{{ . }}</li>{{ end }}</ul>
      </section>
      <section class="resume__section">
        <h2>Languages</h2>
        <ul class="taglist">{{ range $r.languages }}<li>{{ . }}</li>{{ end }}</ul>
      </section>
    </div>

    <section class="resume__section">
      <h2>Education</h2>
      {{ range $r.education }}
      <article class="job">
        <div class="job__head">
          <h3>{{ .school }} <span class="job__role">— {{ .credential }}</span></h3>
          <span class="job__dates">{{ .start }} – {{ .end }}</span>
        </div>
        {{ with .notes }}<p class="edu__notes">{{ . }}</p>{{ end }}
      </article>
      {{ end }}
    </section>
  </main>
</body>
</html>
```

- [ ] **Step 5: Append resume styles**

Append to `links/assets/css/site.css`:
```css
/* ---- Resume page ---- */
.resume__main { padding: clamp(2.5rem, 8vh, 5rem) 0 6rem; }
.resume__back { color: var(--muted); text-decoration: none; font-size: .9rem; }
.resume__back:hover { color: var(--accent); }
.resume__header h1 { font-size: clamp(2.2rem, 1.6rem + 3vw, 3.4rem); margin: .75rem 0 .25rem; letter-spacing: -.02em; }
.resume__role { color: var(--accent); font-weight: 700; margin: 0 0 1rem; }
.resume__contact { display: flex; flex-wrap: wrap; gap: 1rem; align-items: center; margin: 0 0 1.5rem; }
.resume__contact a { color: var(--text); }
.btn { background: linear-gradient(120deg, var(--accent), var(--accent-2)); color: #0b0e14 !important; font-weight: 700; padding: .5rem .9rem; border-radius: .6rem; text-decoration: none; }
.btn:hover { filter: brightness(1.08); }
.resume__summary { color: var(--muted); max-width: 65ch; }
.resume__section { margin-top: 2.5rem; }
.resume__section h2 { font-size: 1.3rem; border-bottom: 1px solid color-mix(in oklab, var(--text) 12%, transparent); padding-bottom: .4rem; margin-bottom: 1.25rem; }
.job { margin-bottom: 1.5rem; }
.job__head { display: flex; flex-wrap: wrap; justify-content: space-between; gap: .25rem 1rem; align-items: baseline; }
.job__head h3 { margin: 0; font-size: 1.1rem; }
.job__role { color: var(--muted); font-weight: 500; }
.job__dates { color: var(--muted); font-size: .9rem; white-space: nowrap; }
.job ul { margin: .5rem 0 0; padding-left: 1.1rem; color: var(--text); }
.job li { margin: .3rem 0; }
.edu__notes { color: var(--muted); margin: .4rem 0 0; }
.resume__cols { display: grid; gap: 2.5rem; grid-template-columns: 1fr; }
@media (min-width: 40rem) { .resume__cols { grid-template-columns: 1fr 1fr; } }
.taglist { list-style: none; margin: 0; padding: 0; display: flex; flex-wrap: wrap; gap: .5rem; }
.taglist li { background: var(--panel); border: 1px solid color-mix(in oklab, var(--text) 10%, transparent); border-radius: .5rem; padding: .35rem .7rem; font-size: .9rem; }
```

- [ ] **Step 6: Build and assert the resume renders from data, and cv.pdf is intact**

Run:
```bash
cd links && hugo --gc --minify
test -f public/resume/index.html && echo "HTML OK"
test -f public/resume/cv.pdf && echo "PDF OK"
grep -c "StackAdapt" public/resume/index.html
grep -c "Download PDF" public/resume/index.html
grep -o 'href="/resume/cv.pdf"' public/resume/index.html | head -1
```
Expected: `HTML OK`, `PDF OK`, a `StackAdapt` count > 0, a `Download PDF` count > 0, and the cv.pdf link present.

- [ ] **Step 7: Visual check**

`cd links && hugo server`, open `http://localhost:1313/resume`. Expected: themed dark resume — header with name/role, email + working "Download PDF" button (opens the PDF), summary, all experience entries (StackAdapt first as current, Tempo as Oct 2022 – May 2025), skills/languages columns, education. Click "Home" returns to `/`. Stop the server.

- [ ] **Step 8: Commit**

```bash
git add links/data/resume.yaml links/content/resume.md links/layouts/_default/resume.html links/assets/css/site.css
git commit -m "feat: themed HTML resume at /resume from structured data, PDF download retained"
```

---

### Task 5: Full-site integration verification

**Files:** none created — this task verifies the whole site and fixes any regressions found.

**Interfaces:**
- Consumes: everything from Tasks 1–4.
- Produces: a verified, deployable site.

- [ ] **Step 1: Clean production build**

Run:
```bash
cd links && rm -rf public resources && hugo --gc --minify --baseURL "https://www.jory.dev/"
```
Expected: build succeeds with no errors/warnings about missing resources.

- [ ] **Step 2: Assert untouched pages still work**

Run:
```bash
cd links
test -f public/metrics/index.html && echo "metrics OK"
grep -c "heavy.svg" public/metrics/index.html
test -f public/resume/cv.pdf && echo "cv.pdf OK"
```
Expected: `metrics OK`, a `heavy.svg` count > 0 (lynx still renders `/metrics` unchanged), `cv.pdf OK`.

- [ ] **Step 3: Assert the home and resume markers**

Run:
```bash
cd links
for id in hero about skills homelab contact; do grep -q "id=\"$id\"" public/index.html && echo "$id OK"; done
grep -q "lenis" public/index.html && echo "lenis wired"
grep -q "StackAdapt" public/resume/index.html && echo "resume OK"
```
Expected: all five `… OK` lines, `lenis wired`, `resume OK`.

- [ ] **Step 4: Behavioral + responsive verification (use the `verify` skill)**

Use the **verify** skill to drive a browser against `cd links && hugo server`:
- `/` — smooth scroll feels inertial; sections reveal on scroll; hero gradient animates.
- `/` with `prefers-reduced-motion: reduce` emulated — all content visible, no motion, normal scroll.
- `/` with JavaScript disabled — all content still visible (no `opacity:0` stuck elements).
- `/` at 375px width — hero readable, skills grid reflows, cards stack, footer links wrap.
- `/resume` — renders, "Download PDF" opens `/resume/cv.pdf`, "Home" link works.
- `/metrics` — still renders the metrics SVG via lynx.

Fix any issues found, re-running the relevant task's build assertion after each fix.

- [ ] **Step 5: Final commit (only if Step 4 required fixes)**

```bash
git add -A
git commit -m "fix: integration adjustments from full-site verification"
```

- [ ] **Step 6: Hand back for deploy**

Report results. Deploy happens automatically via `.github/workflows/hugo.yml` on push to `main` — do NOT push without the user's go-ahead (see finishing-a-development-branch).

---

## Self-Review

**Spec coverage:**
- Dark cinematic landing, 5 sections (Hero/About/Skills/Homelab/Contact) → Task 2 (structure) + Task 3 (motion). ✓
- Lenis smooth scroll, vendored, no CDN → Task 1 (vendor) + Task 3 (wire). ✓
- IntersectionObserver reveals + `prefers-reduced-motion` → Task 3. ✓
- Full HTML resume at `/resume` from structured data, PDF download retained, StackAdapt current role → Task 4. ✓
- Keep lynx for `/metrics`, don't touch cv.pdf / workflows / module.toml; no project baseof → Global Constraints + verified in Task 5. ✓
- Hugo asset pipeline (minify+fingerprint), self-hosted fonts (system stack) → Task 2/3 partials + CSS. ✓
- Accessibility / no-JS readability → Task 3 Steps 2–5. ✓

**Placeholder scan:** No "TBD/TODO/implement later" in steps. The single inline `# end date inferred … confirm` comment in `resume.yaml` is intentional, documented in the spec, and surfaced for the user at review — it is real data with a flag, not a missing value.

**Type/contract consistency:** DOM contract (`data-reveal`, `--delay`, section ids `#hero/#about/#skills/#homelab/#contact`) defined in Task 2 and consumed identically in Task 3. Data keys in `resume.yaml` (`name/title/email/pdf/summary/experience[company,role,start,end,highlights]/skills/languages/education[school,credential,start,end,notes]`) match the template field accesses in Task 4 exactly. The `site-head.html` partial signature (called with page context `.`) is consistent across Tasks 2–4. ✓
