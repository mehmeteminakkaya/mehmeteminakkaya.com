# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Bu dosyayı güncellerken sıfırdan yazma.** Keep this file alive by editing it in place — patch the specific lines that changed as the site evolves. Never regenerate it from a blank page or re-run a full `/init`; the gotchas below were expensive to discover and are easy to lose in a rewrite. Same rule for `README.md`.

## What this is

Mehmet Emin Akkaya's personal site / portfolio: **one self-contained `index.html`** (~990 lines, ~273 KB) with all CSS, JS, and fonts inlined. No build step, no `package.json`, no dependencies, no tests, no linter. All user-facing copy is Turkish.

Deliberate constraint: **the page makes zero external network requests.** Fonts are base64 WOFF2 `@font-face` blocks, the noise overlay and the favicon are inline SVG data URIs, and all sound is synthesized with the Web Audio API (no MP3/WAV). Do not introduce CDN links, Google Fonts, or third-party scripts. The only separate files are `og.jpg` (social preview, fetched by scrapers not browsers), `robots.txt`, and `sitemap.xml`.

## Commands

```bash
npx serve .                 # or: python -m http.server 3000
git push origin main        # this IS the deploy — see below
vercel dns ls mehmeteminakkaya.com   # CLI installed & logged in as mehmeteminakkaya
```

There is nothing to build, lint, or test. "Verifying a change" means opening the page and exercising the affected control.

**Deploys are automatic.** The Vercel project is GitHub-connected, so every push to `main` ships straight to production — there is no staging step and no `vercel --prod` needed. Note that deploys triggered from a local agent can carry `gitDirty: 1`, i.e. uncommitted working-tree files reach production without being versioned; anything that must survive the *next* clean deploy has to be committed.

## Reading index.html

**Never `Read` the whole file** — lines 34–43 are the base64 font payloads (15–33 KB *per line*) and will blow the token limit. Read around them or grep. Structure:

| Lines (approx) | Content |
| :--- | :--- |
| 1–31 | `<head>` meta, OpenGraph, `application/ld+json` Person schema |
| 32–43 | `<style>` opens; embedded WOFF2 `@font-face` (MEA Manrope / Instrument Serif / DM Mono) |
| 44–360 | Design system + all component CSS, ending with responsive (900px / 620px), `@media print`, `prefers-reduced-motion` |
| 362–625 | Markup: `header.topbar`, `main` with sections `#isler` / `#hikaye` / `#lab` / `#iletisim`, `#cv-dialog`, `#toast`, `#command-palette` |
| 627–988 | Single IIFE holding all behavior |

## Conventions that matter

- **Theming**: HSL/hex tokens on `:root`, overridden by `[data-theme="night"]`. `setTheme()` persists to `localStorage` key `mea-theme` and syncs the `theme-color` meta. Any new color must go through a `--token`, never a literal.
- **Inverted / fixed-background regions — the biggest theming trap.** `.lab` deliberately swaps roles (`background: var(--ink); color: var(--paper)`), so it is dark by day and **light by night**. Anything inside it that hardcodes `var(--acid)` becomes unreadable at night — that is what `--lab-accent` exists for (acid by day, `#4f6112` olive at night), and it must be used for every accent inside `#lab`, including the inline `style="color:var(--lab-accent)"` in the terminal's JS output strings. Conversely `.idea-machine`, `.project-main`, `.contact-panel` and `.ticker` keep a **fixed** accent background in both themes, so their text must use literal `#11120f` / `#f2f0e9`, never `var(--ink)` / `var(--paper)` — those flip out from under the fixed background. Lighthouse only audits the default theme, so **check night mode by hand** (`data-theme="night"`) after touching colors.
- **Sound**: every interactive handler calls `playSound(freq, type, duration, gain)`. New buttons should too, for consistency.
- **Reveal animations**: elements get `class="reveal"` and an `IntersectionObserver` adds `.visible`. Note line 309 intentionally forces `.reveal { opacity: 1 }` as a no-JS/slow-optimizer fallback — the observer is progressive enhancement, so content must be readable without it.
- **Turkish locale**: use `toLocaleLowerCase('tr')` / `toLocaleUpperCase('tr')` for any user-typed string comparison (dotted/dotless İ/ı). The command palette filter already does.
- **CV printing**: `@media print` hides `body > *:not(.cv-dialog)`, so `#cv-print` → `window.print()` produces a clean PDF from the `<dialog>` markup. Print output is the only "export"; keep CV blocks `break-inside: avoid`.
- **Easter eggs**: typing `mea` anywhere and the palette's `chaos` action toggle `body.chaos`.

## Content lives in more than one place — keep them in sync

- **A project** appears in three places: its `article.card` in the `.bento` grid (`#isler`), its terminal command case (`migo`, `nexus`, `studybuddy`, `benimhakkimda`, `todogemini`) *and* the `projects` listing case, and its `.cv-entry` under Projeler in `#cv-dialog`.
- **The contact email** appears ~12 times: JSON-LD, `mailto:` links, `#copy-email`, `#idea-discuss` mailto template, the terminal `contact`/`email` case, and the CV sheet. The last two commits exist solely because one of these was missed — grep the full old address before declaring an email change done. As of 2026-08-17 the working tree still holds one uncommitted fix (JSON-LD `"email"`, line 25) and **production still serves the old `aktaha@gmail.com` there**; live output is otherwise byte-identical to HEAD.
- **The terminal** (`#terminal-cli-input` keydown switch) is a plain `switch` on the lowercased command. Adding a command means adding the `case` *and* the `help` output listing.
- **The idea machine** reads from the `ideas` array; the `FİKİR nn / NN` counter derives from `ideas.length`, so only the array needs editing.

## Deployment

Vercel project `mehmeteminakkaya-com` (`.vercel/project.json`), configured by `vercel.json`: `cleanUrls`, no trailing slash, and `X-Content-Type-Options` / `X-Frame-Options` / `X-XSS-Protection` headers on all routes.

DNS moved from Cloudflare to **Vercel DNS** (`ns1/ns2.vercel-dns.com`) on 2026-08-16; registrar is Turkticaret.net, domain expires 2027-06-23. Apex and `www` both A-record straight to Vercel (`216.198.79.65` / `64.29.17.1`), each with its own auto-renewing Let's Encrypt cert. Cloudflare is no longer in the path at all — treat any Cloudflare-shaped code in this repo as a fossil.

Gitignored leftovers from earlier hosting experiments — not sources, do not edit or wire up: `dist/` (a Cloudflare Workers static-assets worker plus a copy of the site) and `.openai/hosting.json`.

## Known rough edges

Audited live 2026-08-17, then fixed the same day. Performance was already good and untouched — LCP 470 ms, CLS 0.00, TTFB ~195 ms, 175 KB brotli, no render-blocking requests. Lighthouse mobile now **100 / 100 / 100 / 100 in both themes**, console clean.

Fixed (don't regress these):

- **Accent-card contrast.** `.card .number` inherited `--muted` (#686961), landing at 1.04:1 on blue and 1.95:1 on orange — those `01 · MOBİL ÜRÜN` labels were invisible. Now each accent card overrides `.number` (and `.project-main p` / `.tag`) with a ground-appropriate tone.
- **Night-mode Lab section.** Acid green on the night theme's light lab background ran 1.04–1.06:1 across `.section-kicker`, `.prompt`, `.terminal-output.highlight`, the cursor and the terminal's injected links; `#idea-button` was white-on-acid (1.06:1) and `#idea-discuss` was dark-on-dark (1.01:1). Fixed via `--lab-accent` plus literal colors on the fixed-background idea machine. See the theming bullet above.
- **Cloudflare fossil removed** — the injected `/cdn-cgi/.../email-decode.min.js` tag 404'd and logged a MIME error.
- **Favicon** — inline SVG data URI (dark rounded square + blue dot, matching the `MEA / 2026` brand mark). Stops the `/favicon.ico` 404.
- **`og:image`** — `og.jpg` (1200×630, 176 KB, downscaled from the 1731×909 `og.png` master, which stays in the repo as the source) plus width/height/type/alt and `twitter:image`.
- **`robots.txt` + `sitemap.xml`** added; `sitemap.xml` carries a hand-maintained `<lastmod>`, so bump it when content changes materially.

- **Anti-spoofing DNS** (2026-08-17, verified on `ns1.vercel-dns.com` and public resolvers): apex TXT `v=spf1 -all` and `_dmarc` TXT `v=DMARC1; p=reject; sp=reject; rua=mailto:mehmeteminakkaya12@gmail.com`. The domain sends no mail — contact runs through Gmail — so the null SPF is correct. **If mail on `@mehmeteminakkaya.com` is ever set up, both records must change first**, or every message will be rejected.

Still open:

- `Mehmet Emin Akkaya CV.pdf` is deleted in the working tree while still tracked in HEAD (and still live, 200, 383 KB); nothing in `index.html` links to it (the CV is the in-page dialog). The `.gitignore` addition of `.vercel` is likewise staged-but-uncommitted in the working tree.
