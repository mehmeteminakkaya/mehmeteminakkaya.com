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

**Never `Read` the whole file** — the base64 font payloads are **lines 43–52** (13–33 KB *per line*) and will blow the token limit. Read around them or grep. To find them mechanically rather than trusting these numbers:
`awk '{ if (length($0) > 2000) print NR": "length($0) }' index.html`

| Lines (approx) | Content |
| :--- | :--- |
| 1–40 | `<head>` meta, OpenGraph, `application/ld+json` Person schema |
| 41–52 | `<style>` opens; embedded WOFF2 `@font-face` (MEA Manrope / Instrument Serif / DM Mono) |
| 53–453 | Design system + all component CSS, ending with responsive (900px / 620px), `@media print`, `prefers-reduced-motion` |
| 455–731 | Markup: `header.topbar`, `main` with sections `#isler` / `#hikaye` / `#lab` / `#iletisim`, `#cv-dialog`, `#toast`, `#command-palette` |
| 732–1168 | Single IIFE holding all behavior |

## Conventions that matter

- **Theming**: HSL/hex tokens on `:root`, overridden by `[data-theme="night"]`. `setTheme()` persists to `localStorage` key `mea-theme` and syncs the `theme-color` meta. Any new color must go through a `--token`, never a literal.
- **`--blue` is a *ground*, `--blue-text` is *ink* — never mix them up.** `--blue` (#3157ff) is the fill of `.project-main`, `.contact-panel` and `.button.primary`, so it must stay dark in both themes or the white text on top of it breaks. Blue *text* therefore has its own token: `--blue-text`, #3157ff by day and **#7d9bff at night**, because #3157ff on the night paper only reached 3.2–3.5:1 (AA needs 4.5). Everything that paints blue letters — `.section-kicker`, `.cert-provider`, `.cv-block h3`, `.hero-copy strong`, `.hero h1 .soft`, `.nav-links a:hover`, `.timeline-item:hover`, `.tabbar a[aria-current]` — uses `--blue-text`. Add a new blue text rule to that list, not to `--blue`.
- **Mobile navigation lives in `.tabbar`**, a fixed bottom bar shown under 900px (where `.nav-links` is hidden). It sets `aria-current` from an `IntersectionObserver` with a `-45%` root margin, and `body` gets `padding-bottom: 62px` so it never covers the footer. Both are reset in `@media print`. The `⌘K` button stays visible on mobile but swaps its label to `⌕ Ara` via `.cmd-long` / `.cmd-short`; its accessible name is completed by an `.sr-only` span rather than an `aria-label`, because an `aria-label` that omits the visible text fails `label-content-name-mismatch`.
- **The command palette is keyboard-driven**: `↑`/`↓` move `.is-active` across `.command-list li:not([hidden])`, `Enter` clicks the active row, filtering resets to the first match, and the ends wrap. The `<small>` in each row is the `↵` marker and only shows on the active row — do not put fake shortcut hints there (`T`, `!`, `01`) unless they are actually bound to keys.
- **Inverted / fixed-background regions — the biggest theming trap.** `.lab` deliberately swaps roles (`background: var(--ink); color: var(--paper)`), so it is dark by day and **light by night**. Anything inside it that hardcodes `var(--acid)` becomes unreadable at night — that is what `--lab-accent` exists for (acid by day, `#4f6112` olive at night), and it must be used for every accent inside `#lab`, including the inline `style="color:var(--lab-accent)"` in the terminal's JS output strings. Conversely `.idea-machine`, `.project-main`, `.contact-panel` and `.ticker` keep a **fixed** accent background in both themes, so their text must use literal `#11120f` / `#f2f0e9`, never `var(--ink)` / `var(--paper)` — those flip out from under the fixed background. Lighthouse only audits the default theme, so **check night mode by hand** (`data-theme="night"`) after touching colors — and check **hover and focus states too**, which no static audit reaches: `.skill-pill:hover` once inverted to `background: var(--paper)` while keeping hardcoded `#11120f` text and went dark-on-dark (1.01:1) at night. The same trap caught the first focus-ring attempt; see the focus bullet below.
- **Only ask for weights the embedded fonts actually ship.** This is the trap that made the site look "thin and blurry" and it is invisible to every contrast audit. The three faces are *not* equally equipped:
  - **MEA DM Mono — 400 and 500 only.** Any mono rule asking for 600/700 gets **synthetic bold** from the browser, which at 11–12px smears the glyphs instead of thickening them. `.prompt`, `.cv-block h3` and bare `<strong>` inside mono blocks all did this. Emphasis inside mono must come from **color**, not weight; `.console strong, .hero-note strong, .terminal-output strong { font-weight: 500 }` pins the `<strong>` default (700) back down.
  - **MEA Instrument Serif — 400 only** (plus italic 400). Never bold it.
  - **MEA Manrope — variable 400–800**, so sans text may use any weight freely.
  Verify with: `sed -n '43,52p' index.html | cut -c1-120`.
- **Microtext floor: nothing renders below ~11.5px, and the mono micro tier is weight 500.** The site is full of small uppercase mono labels (`.number`, `.cert-provider`, `.cert-foot`, `.footer-inner`, `.cv-date`, `.idea-count`, `.timeline-date`, `.section-kicker`, `.status`, `.command-hint`, `.tabbar a`, `.console`). These were 10.2–11.8px at weight 400 and read as mush in Turkish, where ı/İ/ğ/ş/ç lose their marks first. They were lifted to .72–.83rem at weight 500 on 2026-08-19. **Do not reintroduce a `.6xrem` mono label**, and remember three sizes live in *inline* `style=` attributes (the `TRT / UTC+3` span, the hero status line, the terminal highlight `<p>`) where a CSS-only sweep will miss them.
- **Body text on a saturated accent ground is weight 500, not 400.** The same measured ratio reads weaker on vivid colour than on paper, and a 400 cut visibly dissolves — this is what "the font is too thin to read" meant on the BenimHakkımda card. `.project-main p`, `.project-jam p` and `.mini-card.accent p` therefore carry `font-weight: 500` *and* a darker ink than a plain card would need. Orange `#ff6b35` is the brightest ground on the page and suffered most: its paragraph went `#4b271c` → `#2b1409` (4.61 → 6.14:1). Blue went to solid `#fff` (4.76 → 5.33:1), acid to `#3f4031` (6.90 → 8.96:1). Plain cards on paper stay at 400 — do not blanket-bump `.card p`. **A 4.6:1 pass on an accent ground is not "fine"; treat ~6:1 as the floor there.**
- **The tech-tag pills and the skills cloud are gone (2026-08-19), by explicit request — do not reintroduce them.** Every `<span class="tag">` on the project cards and the whole `.skills-cloud` / `.skill-pill` block in `#lab` were removed, along with their CSS. Nothing was lost: each card's stack is already stated in its paragraph prose, and the skills list still lives in the terminal `skills` command and in the CV dialog. The container that held the tags survives as **`.card-links`** (renamed from `.card-tags`) because the GitHub / Canlı Demo links live inside it — if you ever delete that div, you delete the project links with it.
- **Keyboard focus is one universal two-tone ring — do not give it a themed colour.** The site shipped for months with *no* `:focus-visible` rule at all (and `.cli-input { outline: none }`, so the terminal field showed nothing). A single accent colour cannot work here: focusable elements sit on paper, card, blue, acid, orange and the inverted lab ground, and every colour picked disappears on one of them — the first attempt used `var(--paper)`, which on the *fixed* blue `.project-main`/`.contact-panel` flipped dark at night, and a white ring on the acid `.contact-primary` measured 1.18:1. The rule is therefore one dark ring plus one light halo (`outline: 3px solid #11120f` + `box-shadow: 0 0 0 6px #f2f0e9`), so whatever the ground, one of the two stays visible. Literal colours, never tokens. Verified: all 24 focusable elements ≥ 4.68:1 in both themes. The terminal is the one exception — the ring goes on `.cli-input-line` via `:has()`, because a ring on the bare input overflowed the narrow row.
- **The command palette returns focus to whatever opened it.** `closePalette()` restores `paletteOpener`; without it focus fell to `<body>` and keyboard users lost their place.
- **Sound**: every interactive handler calls `playSound(freq, type, duration, gain)`. New buttons should too, for consistency.
- **Reveal animations**: elements get `class="reveal"` and an `IntersectionObserver` adds `.visible`. Note line 309 intentionally forces `.reveal { opacity: 1 }` as a no-JS/slow-optimizer fallback — the observer is progressive enhancement, so content must be readable without it.
- **Turkish locale**: use `toLocaleLowerCase('tr')` / `toLocaleUpperCase('tr')` for any user-typed string comparison (dotted/dotless İ/ı). The command palette filter already does.
- **CV printing**: `@media print` hides `body > *:not(.cv-dialog)`, so `#cv-print` → `window.print()` produces a clean PDF from the `<dialog>` markup. Print output is the only "export"; keep CV blocks `break-inside: avoid`.
- **Easter eggs**: typing `mea` anywhere and the palette's `chaos` action toggle `body.chaos`.

## Auditing contrast in the browser — two traps that produce fake results

A DOM sweep that resolves every text node's colour is the only way to check this page (Lighthouse audits one theme and never opens the dialog or the palette). Two things will silently corrupt the numbers:

1. **`body` has `transition: color .35s, background-color .35s`.** Flip `data-theme` and `getComputedStyle` returns the *mid-transition* value, so text and ground briefly read as the same colour and every node reports ~1.0:1. Inject `*{transition:none !important;animation:none !important}` and wait a tick before measuring, or you will "discover" dozens of failures that do not exist.
2. **The same transition trap ruins focus-indicator measurement.** `.button` transitions `box-shadow`, so reading the computed shadow right after `.focus()` returns the *start* of the animation — a zero-size transparent shadow — and every focus ring looks like it fails. Kill transitions here too.
3. **Do not pull a colour out of `box-shadow` with a bare number regex.** The computed value is `rgb(242, 240, 233) 0px 0px 0px 6px`; a regex that grabs the first four numbers reads the leading `0px` offset as the alpha channel and discards the shadow entirely. Match the leading colour token only. This produced a full page of phantom focus failures before it was caught.
4. **`color-mix()` computes to `color(srgb r g b / a)`, not `rgba()`.** A regex written for `rgba()` picks the wrong channels — it reads the alpha as blue and defaults alpha to 1 — which makes the 5 %-alpha `.console` background look opaque and dark. Parse the `color()` form explicitly, and composite the *whole* ancestor chain (including the element's own background) rather than stopping at the first "opaque-looking" one.

With both handled, the sweep reports **0 AA failures in each theme**, dialog and palette open, and the tightest ratio on the page is 4.61:1.

## Copy rules — the site is written in the first person, not in press-release

All user-facing copy was rewritten on 2026-08-19 because it read as machine-written. Keep it that way:

- **No aphorisms, no parallel triads, no manifesto sentences.** ("Doğru problemi seçmek, sade bir yol çizmek ve …" was exactly the pattern that had to go.) Say what the thing does, in one plain sentence.
- **No buzzword stacking**: "modern SaaS platformu", "AI destekli", "orkestrasyon", "containerize edilmiş", "amiral gemisi", "interaktif portfolyo". Name the actual library or model instead.
- **No invented metrics.** Two idea-machine entries claimed "3 saniyede" and "mikrosaniye yanıt süreli"; nothing measured them.
- **No decorative emoji in prose or terminal output.** The only emoji left are UI affordances: 🔊/🔇 on the sound button, 🔗 on the share button, and the ✦ brand mark.
- **Second person singular, not the formal plural** — "help yaz", not "help yazınız".
- **No left/right directions in copy.** The lab intro said "Soldaki terminal … Sağda da …", which is simply wrong under 900px where `.lab-grid` collapses to one column. Describe things by name, not by where they happen to sit on a wide screen.

**Stack claims must be checked against the source repo, not against an older README.** The site shipped three wrong ones for months: KobiFlow was advertised as "Gemini OCR" (it is NVIDIA NIM — `meta/llama-3.2-11b-vision-instruct` for invoices, `mistralai/mistral-nemo-12b-instruct` for text, `pytesseract` as an optional fallback), BenimHakkımda as "Node.js, Express, Gemini AI" (it is bare `node:http` with zero npm deps, on NVIDIA NIM `meta/llama-3.1-8b-instruct`), and Study Buddy as "Llama 3.3 70B" (the sketch pins Groq `llama-3.1-8b-instant`, with Wit.ai for STT and VoiceRSS for TTS, on an SH1106 OLED — not SSD1306). Mİ-GO also goes through a Cloudflare Worker to NVIDIA NIM. Only ToDoGemini genuinely uses Gemini. Verify in the sibling project directory before writing a stack into a card, a CV entry, or a terminal case.

## Content lives in more than one place — keep them in sync

- **A project** appears in three places: its `article.card` in the `.bento` grid (`#isler`), its terminal command case (`migo`, `nexus`, `studybuddy`, `benimhakkimda`, `todogemini`) *and* the `projects` listing case, and its `.cv-entry` under Projeler in `#cv-dialog`.
- **The contact email** appears ~12 times: JSON-LD, `mailto:` links, `#copy-email`, `#idea-discuss` mailto template, the terminal `contact`/`email` case, and the CV sheet. The last two commits exist solely because one of these was missed — grep the full old address before declaring an email change done. As of 2026-08-17 the working tree still holds one uncommitted fix (JSON-LD `"email"`, line 25) and **production still serves the old `aktaha@gmail.com` there**; live output is otherwise byte-identical to HEAD.
- **The terminal** (`#terminal-cli-input` keydown switch) is a plain `switch` on the lowercased command. Adding a command means adding the `case` *and* the `help` output listing *and* the command list in `README.md`. `benimhakkimda` and `todogemini` had working cases but were missing from `help` for months — the drift goes both ways. The opposite drift also existed: the terminal's **boot transcript** showed `$ whoami` and `$ cat current_focus.txt` as if they had been typed, but neither had a `case`, so typing them answered "no such command". Both are real commands as of 2026-08-19; anything the boot transcript displays must be executable.
- **A command-palette row's label must match the section it jumps to.** The row read "Seçilmiş işlere git" for months after the heading became "Projeler". Rename both together.
- **The idea machine** reads from the `ideas` array; the `FİKİR nn / NN` counter derives from `ideas.length`, so only the array needs editing. **But the panel is a fixed-height box showing variable-length text — measure all 16, never just the one on screen.** `.idea-machine` is `align-content: space-between`, which leaves *zero* gap once content fills the box: at 1440px, 15 of the 16 ideas had the headline touching the buttons, and the panel swung 430→554px per idea, reflowing the whole lab grid on every click. Fixed 2026-08-19 with a real `gap: 28px`, a type scale the panel can actually hold (`clamp(2.4rem,3.8vw,3.5rem)/1.02` — the old unconstrained `4.5vw` grew with the *viewport* while the panel stayed ~460px wide), and a mobile `min-height` raised 400→420px so the tallest idea no longer resizes the box. If you add a longer idea, re-measure: cycle all entries and check the gap and the panel height stay constant.

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
- **Typography legibility pass (2026-08-19).** The complaint was "some text is so thin it can't be read", and it was *not* a contrast problem — every one of these nodes already passed AA. Three separate causes: 27 distinct styles rendered below 13px (smallest 10.2px), the whole mono micro tier sat at weight 400, and 11 nodes were faux-bolded because DM Mono has no 600/700. Fixed by raising the floor to 11.5px, moving the micro tier to weight 500, pinning mono `<strong>` to 500, dropping `.tag`'s stacked `opacity: .75`, darkening day `--muted` #686961 → #5a5b54 (5.31 → 6.56:1), and cutting the grain overlay from `opacity: .28` to `.16` (it sits at `z-index: 20` *over* the text with `mix-blend-mode: multiply`, so it was muddying every small glyph). **AA numbers alone will not catch this class of bug** — check rendered size and real available weight too.
- **Idea-machine crowding (2026-08-19).** See the idea-machine bullet under content-sync: no grid `gap` plus a viewport-scaled type ramp meant the headline collided with the buttons on 15 of 16 ideas and the panel height jumped on every click. Now a constant 430px (420px mobile) with a ≥66px gap at every width.
- **UI sweep (2026-08-19).** Tag pills and the skills cloud removed on request; `.card-tags` renamed `.card-links`. Found and fixed alongside: no `:focus-visible` anywhere, palette focus not restored on close, a stale palette label, and left/right wording that breaks on mobile. Checked clean at 1600 → 360px (real device emulation, no horizontal scroll, no overlap, no text overflow), heading order, landmarks, external-link `rel`, button accessible names, reduced-motion, print, and 16 terminal commands.
- **Cloudflare fossil removed** — the injected `/cdn-cgi/.../email-decode.min.js` tag 404'd and logged a MIME error.
- **Favicon** — inline SVG data URI (dark rounded square + blue dot, matching the `MEA / 2026` brand mark). Stops the `/favicon.ico` 404.
- **`og:image`** — `og.jpg` (1200×630, 176 KB, downscaled from the 1731×909 `og.png` master, which stays in the repo as the source) plus width/height/type/alt and `twitter:image`.
- **`robots.txt` + `sitemap.xml`** added; `sitemap.xml` carries a hand-maintained `<lastmod>`, so bump it when content changes materially.

- **Anti-spoofing DNS** (2026-08-17, verified on `ns1.vercel-dns.com` and public resolvers): apex TXT `v=spf1 -all` and `_dmarc` TXT `v=DMARC1; p=reject; sp=reject; rua=mailto:mehmeteminakkaya12@gmail.com`. The domain sends no mail — contact runs through Gmail — so the null SPF is correct. **If mail on `@mehmeteminakkaya.com` is ever set up, both records must change first**, or every message will be rejected.

Still open:

- **`Mehmet Emin Akkaya CV.pdf` has two masters and the workspace copy is the live one.** The authoritative file is `../Mehmet Emin Akkaya CV.pdf` in the `PROJELERİM` root; the repo copy is a snapshot of it, refreshed on 2026-08-19 (383 KB → 386 KB). When the CV changes, copy it in again rather than editing the repo copy. Note that **nothing in `index.html` links to this PDF** — the CV visitors see is the in-page `#cv-dialog`, and `#cv-print` prints *that*, not the PDF. So the two can drift apart silently; if the PDF is ever linked from the page, the dialog markup has to be brought in line with it first.
- The CV PDF is deliberately **left unlinked** (decision, 2026-08-19): the in-page dialog is the CV visitors see, the PDF is just the copy kept at a known URL. Do not add a download link without first reconciling the two.
