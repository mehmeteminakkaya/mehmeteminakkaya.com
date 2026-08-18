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
- **`--blue` is a *ground*, `--blue-text` is *ink* — never mix them up.** `--blue` (#3157ff) is the fill of `.project-main`, `.contact-panel` and `.button.primary`, so it must stay dark in both themes or the white text on top of it breaks. Blue *text* therefore has its own token: `--blue-text`, #3157ff by day and **#7d9bff at night**, because #3157ff on the night paper only reached 3.2–3.5:1 (AA needs 4.5). Everything that paints blue letters — `.section-kicker`, `.cert-provider`, `.cv-block h3`, `.hero-copy strong`, `.hero h1 .soft`, `.nav-links a:hover`, `.timeline-item:hover`, `.tabbar a[aria-current]` — uses `--blue-text`. Add a new blue text rule to that list, not to `--blue`.
- **Mobile navigation lives in `.tabbar`**, a fixed bottom bar shown under 900px (where `.nav-links` is hidden). It sets `aria-current` from an `IntersectionObserver` with a `-45%` root margin, and `body` gets `padding-bottom: 62px` so it never covers the footer. Both are reset in `@media print`. The `⌘K` button stays visible on mobile but swaps its label to `⌕ Ara` via `.cmd-long` / `.cmd-short`; its accessible name is completed by an `.sr-only` span rather than an `aria-label`, because an `aria-label` that omits the visible text fails `label-content-name-mismatch`.
- **The command palette is keyboard-driven**: `↑`/`↓` move `.is-active` across `.command-list li:not([hidden])`, `Enter` clicks the active row, filtering resets to the first match, and the ends wrap. The `<small>` in each row is the `↵` marker and only shows on the active row — do not put fake shortcut hints there (`T`, `!`, `01`) unless they are actually bound to keys.
- **Inverted / fixed-background regions — the biggest theming trap.** `.lab` deliberately swaps roles (`background: var(--ink); color: var(--paper)`), so it is dark by day and **light by night**. Anything inside it that hardcodes `var(--acid)` becomes unreadable at night — that is what `--lab-accent` exists for (acid by day, `#4f6112` olive at night), and it must be used for every accent inside `#lab`, including the inline `style="color:var(--lab-accent)"` in the terminal's JS output strings. Conversely `.idea-machine`, `.project-main`, `.contact-panel` and `.ticker` keep a **fixed** accent background in both themes, so their text must use literal `#11120f` / `#f2f0e9`, never `var(--ink)` / `var(--paper)` — those flip out from under the fixed background. Lighthouse only audits the default theme, so **check night mode by hand** (`data-theme="night"`) after touching colors — and check **hover states too**: `.skill-pill:hover` inverts to `background: var(--paper)`, so its text has to be `var(--ink)`; it was hardcoded `#11120f` and went dark-on-dark (1.01:1) at night, which no static audit catches.
- **Sound**: every interactive handler calls `playSound(freq, type, duration, gain)`. New buttons should too, for consistency.
- **Reveal animations**: elements get `class="reveal"` and an `IntersectionObserver` adds `.visible`. Note line 309 intentionally forces `.reveal { opacity: 1 }` as a no-JS/slow-optimizer fallback — the observer is progressive enhancement, so content must be readable without it.
- **Turkish locale**: use `toLocaleLowerCase('tr')` / `toLocaleUpperCase('tr')` for any user-typed string comparison (dotted/dotless İ/ı). The command palette filter already does.
- **CV printing**: `@media print` hides `body > *:not(.cv-dialog)`, so `#cv-print` → `window.print()` produces a clean PDF from the `<dialog>` markup. Print output is the only "export"; keep CV blocks `break-inside: avoid`.
- **Easter eggs**: typing `mea` anywhere and the palette's `chaos` action toggle `body.chaos`.

## Copy rules — the site is written in the first person, not in press-release

All user-facing copy was rewritten on 2026-08-19 because it read as machine-written. Keep it that way:

- **No aphorisms, no parallel triads, no manifesto sentences.** ("Doğru problemi seçmek, sade bir yol çizmek ve …" was exactly the pattern that had to go.) Say what the thing does, in one plain sentence.
- **No buzzword stacking**: "modern SaaS platformu", "AI destekli", "orkestrasyon", "containerize edilmiş", "amiral gemisi", "interaktif portfolyo". Name the actual library or model instead.
- **No invented metrics.** Two idea-machine entries claimed "3 saniyede" and "mikrosaniye yanıt süreli"; nothing measured them.
- **No decorative emoji in prose or terminal output.** The only emoji left are UI affordances: 🔊/🔇 on the sound button, 🔗 on the share button, and the ✦ brand mark.
- **Second person singular, not the formal plural** — "help yaz", not "help yazınız".

**Stack claims must be checked against the source repo, not against an older README.** The site shipped three wrong ones for months: KobiFlow was advertised as "Gemini OCR" (it is NVIDIA NIM — `meta/llama-3.2-11b-vision-instruct` for invoices, `mistralai/mistral-nemo-12b-instruct` for text, `pytesseract` as an optional fallback), BenimHakkımda as "Node.js, Express, Gemini AI" (it is bare `node:http` with zero npm deps, on NVIDIA NIM `meta/llama-3.1-8b-instruct`), and Study Buddy as "Llama 3.3 70B" (the sketch pins Groq `llama-3.1-8b-instant`, with Wit.ai for STT and VoiceRSS for TTS, on an SH1106 OLED — not SSD1306). Mİ-GO also goes through a Cloudflare Worker to NVIDIA NIM. Only ToDoGemini genuinely uses Gemini. Verify in the sibling project directory before writing a stack into a card, a CV entry, or a terminal case.

## Content lives in more than one place — keep them in sync

- **A project** appears in three places: its `article.card` in the `.bento` grid (`#isler`), its terminal command case (`migo`, `nexus`, `studybuddy`, `benimhakkimda`, `todogemini`) *and* the `projects` listing case, and its `.cv-entry` under Projeler in `#cv-dialog`.
- **The contact email** appears ~12 times: JSON-LD, `mailto:` links, `#copy-email`, `#idea-discuss` mailto template, the terminal `contact`/`email` case, and the CV sheet. The last two commits exist solely because one of these was missed — grep the full old address before declaring an email change done. As of 2026-08-17 the working tree still holds one uncommitted fix (JSON-LD `"email"`, line 25) and **production still serves the old `aktaha@gmail.com` there**; live output is otherwise byte-identical to HEAD.
- **The terminal** (`#terminal-cli-input` keydown switch) is a plain `switch` on the lowercased command. Adding a command means adding the `case` *and* the `help` output listing *and* the command list in `README.md`. `benimhakkimda` and `todogemini` had working cases but were missing from `help` for months — the drift goes both ways.
- **The idea machine** reads from the `ideas` array; the `FİKİR nn / NN` counter derives from `ideas.length`, so only the array needs editing.

## Deployment

Vercel project `mehmeteminakkaya-com` (`.vercel/project.json` — present locally but **gitignored** since 2026-08-19, so it will not exist in a fresh clone; re-link with `vercel link`), configured by `vercel.json`: `cleanUrls`, no trailing slash, and `X-Content-Type-Options` / `X-Frame-Options` / `X-XSS-Protection` headers on all routes.

DNS moved from Cloudflare to **Vercel DNS** (`ns1/ns2.vercel-dns.com`) on 2026-08-16; registrar is Turkticaret.net, domain expires 2027-06-23. Apex and `www` both A-record straight to Vercel (`216.198.79.65` / `64.29.17.1`), each with its own auto-renewing Let's Encrypt cert. Cloudflare is no longer in the path at all — treat any Cloudflare-shaped code in this repo as a fossil.

Gitignored leftovers from earlier hosting experiments — not sources, do not edit or wire up: `dist/` (a Cloudflare Workers static-assets worker plus a copy of the site) and `.openai/hosting.json`.

## Known rough edges

Audited live 2026-08-17, then fixed the same day. Performance was already good and untouched — LCP 470 ms, CLS 0.00, TTFB ~195 ms, 175 KB brotli, no render-blocking requests. Lighthouse mobile now **100 / 100 / 100 / 100 in both themes**, console clean.

Fixed (don't regress these):

- **Accent-card contrast.** `.card .number` inherited `--muted` (#686961), landing at 1.04:1 on blue and 1.95:1 on orange — those `01 · MOBİL ÜRÜN` labels were invisible. Now each accent card overrides `.number` (and `.project-main p` / `.tag`) with a ground-appropriate tone.
- **Night-mode Lab section.** Acid green on the night theme's light lab background ran 1.04–1.06:1 across `.section-kicker`, `.prompt`, `.terminal-output.highlight`, the cursor and the terminal's injected links; `#idea-button` was white-on-acid (1.06:1) and `#idea-discuss` was dark-on-dark (1.01:1). Fixed via `--lab-accent` plus literal colors on the fixed-background idea machine. See the theming bullet above.
- **Night-mode blue** (2026-08-19). `--blue` was doing double duty as ground and ink; on the night paper the blue text ran 3.23–3.50:1. Split into `--blue` / `--blue-text` — see the theming bullet. Same pass fixed `.skill-pill:hover` (1.01:1 dark-on-dark at night) and lifted the terminal placeholder from 55% to 70% paper (4.01 → 6.4:1). A full DOM sweep of every text node now reports **zero** AA failures in both themes, dialogs and command palette included.
- **Cloudflare fossil removed** — the injected `/cdn-cgi/.../email-decode.min.js` tag 404'd and logged a MIME error.
- **Favicon** — inline SVG data URI (dark rounded square + blue dot, matching the `MEA / 2026` brand mark). Stops the `/favicon.ico` 404.
- **`og:image`** — `og.jpg` (1200×630, 176 KB, downscaled from the 1731×909 `og.png` master, which stays in the repo as the source) plus width/height/type/alt and `twitter:image`.
- **`robots.txt` + `sitemap.xml`** added; `sitemap.xml` carries a hand-maintained `<lastmod>`, so bump it when content changes materially.

- **Anti-spoofing DNS** (2026-08-17, verified on `ns1.vercel-dns.com` and public resolvers): apex TXT `v=spf1 -all` and `_dmarc` TXT `v=DMARC1; p=reject; sp=reject; rua=mailto:mehmeteminakkaya12@gmail.com`. The domain sends no mail — contact runs through Gmail — so the null SPF is correct. **If mail on `@mehmeteminakkaya.com` is ever set up, both records must change first**, or every message will be rejected.

Still open:

- **`Mehmet Emin Akkaya CV.pdf` has two masters and the workspace copy is the live one.** The authoritative file is `../Mehmet Emin Akkaya CV.pdf` in the `PROJELERİM` root; the repo copy is a snapshot of it, refreshed on 2026-08-19 (383 KB → 386 KB). When the CV changes, copy it in again rather than editing the repo copy. Note that **nothing in `index.html` links to this PDF** — the CV visitors see is the in-page `#cv-dialog`, and `#cv-print` prints *that*, not the PDF. So the two can drift apart silently; if the PDF is ever linked from the page, the dialog markup has to be brought in line with it first.
- The CV PDF is deliberately **left unlinked** (decision, 2026-08-19): the in-page dialog is the CV visitors see, the PDF is just the copy kept at a known URL. Do not add a download link without first reconciling the two.
