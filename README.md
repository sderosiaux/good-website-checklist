# Guidelines to create a strong website

Here you'll find out all the things I could think of and find out, to create a "good" website.

From security, to performance, social sharing, analytics etc. I'm trying to not forget anything.
This is not about which framework to use, but about everything that makes a "good" website in general: secured, performant, social compliant, SEO compliant, offline ready, and more.

This list is growing over time. Last full rewrite: **July 2026** — the 2017 edition was full of HPKP, AMP, Google+ and polyfill.io. See [Care about deleting ?](#care-about-deleting-) for everything that got cut and why.

## You know more ?

Don't hesitate to PR! Let's try to be concise: other resources on the web go further in details for each topic, let's keep them one-liner here with a sample code when necessary.

![Be strong](vd.gif)

# Summary

- [Care about security ?](#care-about-security-)
- [Care about social ?](#care-about-social-)
- [Care about SEO ?](#care-about-seo-)
- [Care about AI assistants ?](#care-about-ai-assistants-)
- [Care about metadata ?](#care-about-metadata-)
- [Care about icons ?](#care-about-icons-)
- [Care about accessibility (a11y) ?](#care-about-accessibility-a11y-)
- [Care about privacy ?](#care-about-privacy-)
- [Care about legal ?](#care-about-legal-)
- [Care about style ?](#care-about-style-)
- [Care about browser support ?](#care-about-browser-support-)
- [Care about performance ?](#care-about-performance-)
- [Care about mobile ?](#care-about-mobile-)
- [Care about offline ?](#care-about-offline-)
- [Care about analytics ?](#care-about-analytics-)
- [Care about bugs ?](#care-about-bugs-)
- [Care about ops ?](#care-about-ops-)
- [Care about misc ?](#care-about-misc-)
- [Care about deleting ?](#care-about-deleting-)
- [More tips](#more-tips)

## Care about security ?

  - Serve everything over TLS 1.3 and automate issuance with ACME. Certificate lifetimes are collapsing (CA/B Forum ballot SC-081v3): max 200 days since 2026-03-15, 100 days in March 2027, **47 days in March 2029**. Manual renewal is already dead.
    - Let's Encrypt `shortlived` ACME profile = 6-day certs, GA since Jan 2026 https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability
    - Test with https://www.ssllabs.com/ssltest/ or self-hosted https://testssl.sh/
  - **Strict-Transport-Security**: `max-age=63072000; includeSubDomains; preload`, then submit on https://hstspreload.org. One-way door: removal propagates over weeks and it covers *every* subdomain, including internal ones.
  - Pin who may issue certs for you with a DNS CAA record
```dns
example.com. CAA 0 issue "letsencrypt.org"
example.com. CAA 0 iodef "mailto:security@example.com"
```
  - CSP and Trusted Types are the second line. The first is still context-aware output encoding: let the framework escape (`{}` in React/Vue/Svelte, `{{ }}` in Jinja/Twig), never concatenate user input into HTML/URL/JS/CSS contexts, and run anything you must render as HTML through [DOMPurify](https://github.com/cure53/DOMPurify) — or better, `setHTML()` where it's available. https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
  - **Content-Security-Policy**: CSP Level 3, nonce + `'strict-dynamic'`. Host allowlists are broken — one JSONP endpoint or open redirect on any allowed host bypasses the whole policy.
```http
Content-Security-Policy:
  script-src 'nonce-{RANDOM_PER_RESPONSE}' 'strict-dynamic' 'unsafe-inline' https:;
  object-src 'none'; base-uri 'none'; frame-ancestors 'none'; form-action 'self';
  require-trusted-types-for 'script'; trusted-types default;
  report-to csp
Reporting-Endpoints: csp="https://example.com/_/csp-reports"
```
  - `'unsafe-inline' https:` are *ignored* by CSP3 browsers, they only exist as a fallback for ancient ones
  - The nonce is a fresh CSPRNG value per response. Never reuse it, never render it where an injection could read it back.
  - `base-uri 'none'` blocks `<base>` injection that reroutes every relative script URL. Everybody forgets this one.
  - `form-action 'self'` blocks an injected `<form action="https://evil">` from exfiltrating a filled-in form. It doesn't fall back to `default-src`, so it must be listed explicitly.
  - Send CSP as a **header**, not a `<meta>`: `frame-ancestors`, `sandbox` and reporting don't work in meta, and an injection placed before the tag wins.
  - **Content-Security-Policy-Report-Only**: deploy this first, then grade the policy on https://csp-evaluator.withgoogle.com/. Full guide: https://web.dev/articles/strict-csp
  - **Trusted Types** kill DOM XSS at the sink (`innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `DOMParser.parseFromString`). Baseline since Feb 2026. Ship it in a second `-Report-Only` header before enforcing.
  - Route violations to a collector with `Reporting-Endpoints`, see [Care about bugs ?](#care-about-bugs-). Keep `report-uri` alongside for a while for Safari.
  - **X-Content-Type-Options**: `nosniff`. Still mandatory, one line, no downside.
  - **Clickjacking**: `frame-ancestors 'none'` in CSP is the real control. `X-Frame-Options: DENY` is legacy-only backup and is ignored when `frame-ancestors` is present.
  - **Referrer-Policy**: browsers already default to `strict-origin-when-cross-origin`. Only set it to go *stricter*: `no-referrer` or `same-origin`.
  - **Permissions-Policy**: deny what you don't use. `unload=()` removes *one* bfcache blocker by making `unload` handlers un-registrable — it doesn't guarantee eligibility, see [Care about performance ?](#care-about-performance-).
```http
Permissions-Policy: geolocation=(), camera=(), microphone=(), payment=(), usb=(), unload=()
```
  - **Cross-origin isolation** (needed for `SharedArrayBuffer`, `performance.measureUserAgentSpecificMemory()`, and against XS-Leaks)
    - classic: `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` (or `credentialless` when you embed CDN assets you can't add CORP to)
    - Chromium-only for now: `Document-Isolation-Policy: isolate-and-credentialless` — per-document, keeps OAuth/payment popups and third-party iframes working, no COEP needed on embeds https://developer.chrome.com/blog/document-isolation-policy
    - protect your own assets from being embedded/read cross-site: `Cross-Origin-Resource-Policy: same-origin`
    - even without full isolation, `COOP: same-origin-allow-popups` is a cheap win against tabnabbing
  - **Access-Control-Allow-Origin** (CORS): never `*` on anything authenticated, and never blindly reflect the `Origin` header — validate against an allowlist. https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
  - **ORB** (Opaque Response Blocking) replaced CORB in browsers: cross-origin `no-cors` fetches of HTML/JSON/XML now fail closed with a *network error* instead of an empty response. Check your error paths.
  - **Timing-Allow-Origin** does the opposite of what most checklists claim: it *grants* cross-origin access to `PerformanceResourceTiming` details. Add `Timing-Allow-Origin: *` on **non-personalized** static assets so your RUM works, and leave it off user-specific responses.
  - **SRI** (Subresource Integrity) on every third-party script/style, and enforce it globally so nobody forgets
```html
<script src="https://cdn.example/lib.js" integrity="sha384-…" crossorigin="anonymous"></script>
```
```http
Reporting-Endpoints: ip-endpoint="https://example.com/_/integrity-reports"
Integrity-Policy-Report-Only: blocked-destinations=(script style), endpoints=(ip-endpoint)
Integrity-Policy: blocked-destinations=(script style), endpoints=(ip-endpoint)
```
  - SRI limit: it's useless against CDNs serving *mutable* URLs (`/latest/`, tag managers, ad/analytics loaders). Pin a version or self-host.
  - **Cookies**: `Set-Cookie: __Host-session=…; Secure; HttpOnly; SameSite=Lax; Path=/`
    - `__Host-` is browser-enforced (Secure + `Path=/` + **no** `Domain`) → blocks cookie injection from a compromised sibling subdomain
    - `SameSite=Lax` is the default but set it explicitly; `Strict` for admin/banking flows; cross-site embeds need `Partitioned` (CHIPS)
    - CSRF: SameSite is defense-in-depth, not *the* defense. Keep a per-session anti-CSRF token (or an Origin-header check) on every state-changing request.
    - `Clear-Site-Data: "cache", "cookies", "storage"` on logout
  - Cheapest CSRF backstop, zero state: reject state-changing requests unless `Sec-Fetch-Site` is `same-origin`/`none` and `Sec-Fetch-Mode` isn't `no-cors`. Supported everywhere, no token plumbing. https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site
  - Keep session tokens out of `localStorage` (any XSS reads it): access token in memory + refresh token in a `__Host-`/`HttpOnly` cookie. A JWT is a token format, not an authentication design and not a session store.
  - **Passkeys over passwords** (WebAuthn discoverable credentials). Use conditional UI so passkeys show up in the browser autofill dropdown — completion ~94% vs 60-86% for cross-device flows.
```html
<input name="username" autocomplete="username webauthn">
```
  - Feature-detect with `PublicKeyCredential.isConditionalMediationAvailable()`, and prompt for passkey creation right *after* a successful password login, never behind a separate button.
  - **Bots/abuse**: [Cloudflare Turnstile](https://www.cloudflare.com/products/turnstile/) is the sane default (free, no cookies, invisible, no consent headache). reCAPTCHA now needs a GCP billing account and only 10k free assessments/month. Pair either with server-side rate limiting per IP *and* per account — a CAPTCHA is not a rate limiter.
  - **Supply chain**, where sites actually get owned in 2026 (Shai-Hulud npm worms)
    - cooldown on new versions, so a compromised release expires before it reaches your lockfile — mind the units, they differ: pnpm 11 `minimumReleaseAge` (minutes, defaults to `1440`), Yarn 4.15 `npmMinimalAgeGate` (minutes or `"1d"`, defaults to `1d`), npm 11.10 `--min-release-age` (**days**, still opt-in). pnpm silently falls back to the last old-enough version, npm and Yarn fail instead.
    - `npm ci` only, lockfile committed, `ignore-scripts=true` with a short allowlist for packages that genuinely need postinstall
    - publish with **trusted publishing (OIDC)** instead of long-lived tokens → Sigstore provenance attestations for free https://docs.npmjs.com/trusted-publishers/
    - scan with OSV-Scanner in CI https://google.github.io/osv-scanner/, Dependabot/Renovate for fix PRs, and https://socket.dev for *behavioral* malware detection — a freshly backdoored package has no CVE, so `npm audit` sees nothing
    - use a scoped/private registry to block dependency confusion, and audit `Object.prototype` sinks for prototype pollution
  - **Secrets**: [gitleaks](https://github.com/gitleaks/gitleaks) pre-commit + [trufflehog](https://github.com/trufflesecurity/trufflehog) (verified findings only) in CI + GitHub push protection org-wide as a backstop.
  - **Subdomain takeover**: delete the DNS record *before* deprovisioning the service, never after. Sweep CNAMEs weekly for provider 404 fingerprints, and watch Certificate Transparency logs for certs you didn't ask for. https://cheatsheetseries.owasp.org/cheatsheets/Subdomain_Takeover_Prevention_Cheat_Sheet.html
  - If the domain sends mail: SPF + DKIM + DMARC aligned, and publish MTA-STS (`enforce` after 2+ weeks in `testing`). Google/Yahoo/Microsoft reject non-compliant bulk mail with a permanent 550. Aim for `p=reject`.
  - Define a security policy in `/.well-known/security.txt` ([RFC 9116](https://www.rfc-editor.org/rfc/rfc9116.html)) — `Contact` and `Expires` are **required** or the file is invalid
```txt
Contact: mailto:security@example.com
Expires: 2027-01-01T00:00:00.000Z
Policy: https://example.com/security-policy
Preferred-Languages: en, fr
Canonical: https://example.com/.well-known/security.txt
```
  - `/.well-known/change-password` → 302 to your real password-change page. Password managers and Safari/Chrome deep-link to it.
  - Handle standard addresses: security@, abuse@, postmaster@, hostmaster@, webmaster@
  - Remove the server signature from response headers (`Server: Apache`, nginx etc.) and same for `X-Powered-By`
  - Protect against DDoS with a WAF + rate limiting at the edge ([Cloudflare](https://www.cloudflare.com/), Fastly, AWS Shield) rather than in app code alone
  - Read the actual threat models: https://cheatsheetseries.owasp.org/ (CSP, CSRF, XSS Prevention, Node.js, JWT) and https://owasp.org/Top10/
  - Check the **production** response, since CDNs and proxies rewrite headers
    - https://developer.mozilla.org/en-US/observatory — Mozilla Observatory moved here, deepest free header scan
    - https://securityheaders.com still works for one-off scans, but it's Snyk-owned now and its API shut down in April 2026 — don't wire it into CI
    - https://internet.nl for TLS/DNSSEC/IPv6/mail standards, https://webbkoll.5july.net for third-party tracking

## Care about social ?

  - One image rules everything: `og:image` 1200×630 (1.91:1), under 1 MB, absolute HTTPS URL, no redirect. X/LinkedIn/Slack/Discord/Bluesky/Telegram/WhatsApp all read it.
  - The 2026 minimal set is Open Graph + one X tag. Everything else is fallback.
```html
<meta property="og:type" content="website"> <!-- "article" for posts -->
<meta property="og:title" content="{{ TITLE }}">
<meta property="og:description" content="{{ DESCRIPTION }}">
<meta property="og:url" content="{{ CANONICAL_URL }}">
<meta property="og:site_name" content="{{ SITE }}">
<meta property="og:image" content="https://.../og.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="{{ WHAT THE IMAGE SHOWS }}">
<meta name="twitter:card" content="summary_large_image">
```
  - `twitter:card` is the only X-specific tag still worth shipping: X falls back to `og:*` for title/description/image, but *not* for the card type — omit it and you get the small thumbnail.
  - `og:image:width`/`height` aren't cosmetic: without them Discord may render a small thumbnail instead of a large embed.
  - `og:type=article` unlocks `article:published_time` / `article:author`, which LinkedIn and Discord surface as attribution. Slack ignores `og:type` entirely.
  - Discord colors the left border of the embed with `<meta name="theme-color">`. Nobody else does.
  - WhatsApp: hard cap 600 KB on the image, min 100px, cropped to a square in threads. Telegram: needs `og:title` minimum, image ≥200×200, no cache purge tool (append `?v=2` to force a refetch).
  - Bluesky reads Open Graph, no `twitter:*`, image ≤1 MB, always rendered landscape so square images get cropped.
  - Mastodon 4.3+ author attribution (also requires allow-listing the domain in Preferences → Public Profile → Verification)
```html
<meta name="fediverse:creator" content="@you@instance.social">
```
  - Slack prefers oEmbed over meta tags if you expose it. Worth it if you publish embeddable content.
```html
<link rel="alternate" type="application/json+oembed" href="https://example.com/oembed?url=...&format=json">
```
  - Generate OG images dynamically instead of hand-designing them: [@vercel/og](https://vercel.com/docs/og-image-generation) (Satori → SVG → resvg → PNG), `workers-og` on Cloudflare, or plain `satori` + `@resvg/resvg-js` at build time. Satori is flexbox-only, no CSS grid.
  - Validators that still work
    - [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/) — no login, forces a cache purge
    - [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/) — needs a FB developer account, "Scrape Again" is still the only FB cache purge
    - [opengraph.xyz](https://www.opengraph.xyz/) — no-login multi-platform preview
    - X: no validator anymore, post the link from a test account
  - Don't add share buttons. Third-party JS + trackers + CSP headaches for a link you can hand-write: `https://x.com/intent/post?url=…`, `https://bsky.app/intent/compose?text=…`, `mailto:`.

## Care about SEO ?

  - Serve HTTPS, one canonical host, stable URLs. A URL you change is a URL you lose in both search *and* AI training sets.
  - `robots.txt`: Google only supports `user-agent`, `allow`, `disallow`, `sitemap` ([RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html)). `crawl-delay`, `noindex`, `nofollow` are **ignored** there — `noindex` goes in a meta or an `X-Robots-Tag` header.
```
User-agent: *
Disallow: /admin/
Allow: /
Sitemap: https://example.com/sitemap_index.xml
```
  - `sitemap.xml`: 50k URLs / 50 MB max per file, else a sitemap index. Google **ignores `<priority>` and `<changefreq>`**, and uses `<lastmod>` only if verifiably accurate — a build-time `now()` on every URL gets the whole signal discarded. https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap
  - Image/video/news sitemap extensions still exist and are still read. Skip them unless media *is* the product.
  - Push new URLs with [IndexNow](https://www.indexnow.org/documentation) — Bing, Yandex, Naver, Seznam, Yep. Google does not support it and never shipped it. Still matters: Bing's index feeds ChatGPT Search and Copilot.
```bash
curl "https://api.indexnow.org/indexnow?url=https://example.com/page&key=<32-128-char-key>"
# the key file must be readable at https://example.com/<key>.txt
```
  - Canonical on every page, self-referencing, absolute URL. It's a signal, not a directive — Google can override it.
```html
<link rel="canonical" href="https://example.com/article" />
```
  - `hreflang` must be reciprocal and include `x-default`. A missing return link voids the whole cluster.
```html
<link rel="alternate" hreflang="en-us" href="https://example.com/us/" />
<link rel="alternate" hreflang="fr"    href="https://example.com/fr/" />
<link rel="alternate" hreflang="x-default" href="https://example.com/" />
```
  - JSON-LD, not Microdata or RDFa — Google parses all three but recommends JSON-LD, and its docs, tooling and examples all assume it. It must match the visible text. Types Google still renders in 2026: `Article`, `Breadcrumb`, `Product`, `Review snippet`, `Organization`, `LocalBusiness`, `Event`, `JobPosting`, `Recipe`, `Video`, `SoftwareApplication`, `ProfilePage`, `DiscussionForumPosting`, `QAPage`, `Dataset`, `Course`, `Movie`, `Speakable`. Full list: https://developers.google.com/search/docs/appearance/structured-data/search-gallery
```html
<script type="application/ld+json">
{"@context":"https://schema.org","@type":"Article","headline":"…",
 "datePublished":"2026-07-30","dateModified":"2026-07-30",
 "author":{"@type":"Person","name":"…","url":"https://example.com/about"},
 "publisher":{"@type":"Organization","name":"…"},
 "mainEntityOfPage":"https://example.com/article"}
</script>
```
  - Validate both ways: [validator.schema.org](https://validator.schema.org/) for spec conformance, [Rich Results Test](https://search.google.com/test/rich-results) for what Google will actually render. Markup that doesn't match the page is a manual-action trigger.
  - Register the site in [Search Console](https://search.google.com/search-console) **and** [Bing Webmaster Tools](https://www.bing.com/webmasters) — Bing is the upstream index for several AI assistants, and it's free reach most people skip.
  - Core Web Vitals are a real but weak ranking input, a tiebreaker rather than a lever — thresholds and tooling in [Care about performance ?](#care-about-performance-).
  - Google renders JS, but on a delayed second pass, and most AI crawlers **don't render at all**. SSR/SSG or prerender anything you want cited. https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics
  - Internal linking is now the primary discovery path. Every page reachable in ≤3 clicks from home, with real `<a href>` — not `onclick` routers.
  - Pagination: link page N to N±1 with plain `<a href>`, give each page a **self-referencing canonical** (never canonical-to-page-1), don't `noindex` deep pages you want crawled.
  - Status codes: `301` for permanent moves (one hop, no chains), `410` for deliberately gone (dropped faster than `404`), `404` for unknown, never `200` on an error page.
  - Mobile-first indexing has been 100% complete since 5 July 2024. A page that doesn't work on mobile Googlebot doesn't exist.
  - E-E-A-T is the operative quality frame. Concrete artifacts: a named author with a real bio page, `dateModified`, cited primary sources.
  - Crawl budget only matters above ~10k URLs. Otherwise ignore it.
  - Keep URLs short and meaningful, and handle page errors (404, etc.) with real links inside instead of dumping the default page.

## Care about AI assistants ?

  - Control access per bot in `robots.txt`. Current tokens verified against vendor docs ([OpenAI](https://developers.openai.com/api/docs/bots), [Anthropic](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler), [Google](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers), [Perplexity](https://docs.perplexity.ai/guides/bots)); living list at [darkvisitors.com/agents](https://darkvisitors.com/agents).
```
# Training crawlers — no traffic back, no citations
User-agent: GPTBot
User-agent: ClaudeBot
User-agent: CCBot
User-agent: Bytespider
User-agent: meta-externalagent
User-agent: Amazonbot
Disallow: /

# Training/grounding opt-out tokens (not real crawlers, no effect on Search)
User-agent: Google-Extended
User-agent: Applebot-Extended
Disallow: /

# Retrieval + user-triggered fetchers — ALLOW these if you want citations
User-agent: OAI-SearchBot
User-agent: ChatGPT-User
User-agent: Claude-SearchBot
User-agent: Claude-User
User-agent: PerplexityBot
User-agent: Perplexity-User
Allow: /
```
  - Know what each token gates: `Google-Extended` covers **Gemini training and grounding only — it does NOT remove you from AI Overviews**. AI Overviews are Search; the controls are `nosnippet` / `data-nosnippet` / `max-snippet` / `noindex`, which also kill your normal snippet. https://developers.google.com/search/docs/appearance/ai-features
```html
<span data-nosnippet>internal note, never quoted</span>
<meta name="robots" content="max-snippet:-1, max-image-preview:large" />
```
  - Enforcement lives at the edge, not in a text file: [Cloudflare AI Crawl Control](https://developers.cloudflare.com/ai-crawl-control/) classifies bots as Search / Agent / Training and blocks per category. If you *want* AI referrals, check you haven't been opted out by a default.
  - `llms.txt` — be honest: **no AI provider has confirmed reading it**, Google said publicly it doesn't support it, and log studies show near-zero crawler requests. Its one real use is developer tooling (Cursor, MCP clients) pointed at your docs on purpose. Ship it if it costs nothing, expect no citation lift. https://llmstxt.org/
  - Cloudflare's `Content-Signal` in robots.txt is a declarative rights reservation. No crawler is known to honor it yet — treat it as a legal statement, not enforcement. https://contentsignals.org/
```
# search: index & link · ai-input: use in AI answers · ai-train: train models
Content-Signal: search=yes, ai-train=no
User-agent: *
Allow: /
```
  - Standards to watch, not to deploy: the IETF [AIPREF WG](https://datatracker.ietf.org/wg/aipref/about/) (`draft-ietf-aipref-vocab`, `draft-ietf-aipref-attach`) — still Internet-Drafts, no RFC.
  - Also watch **Web Bot Auth** ([draft-meunier-web-bot-auth-architecture](https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth-architecture/)): HTTP Message Signatures instead of user-agent strings, so agents prove identity cryptographically instead of being guessed at by IP. Cloudflare ships it today, no RFC yet.
  - What makes a page quotable: one `<h1>`, semantic `<article>`/`<section>`, the answer in the first 2-3 sentences under each heading, real `<table>` markup instead of CSS grids, visible `datePublished`/`dateModified`, a named author linked to a bio, and numbers in text rather than baked into images.
  - Don't hide the answer behind JS, tabs, accordions or a cookie wall. Retrieval bots fetch raw HTML and take what's in the DOM on the first response.
  - Track AI referrals in analytics with a custom channel group ordered *above* Referral, and expect 35-70% of AI visits to land in Direct anyway (assistants often strip the referrer).
```
^.*(chatgpt\.com|chat\.openai\.com|perplexity\.ai|claude\.ai|
gemini\.google\.com|copilot\.microsoft\.com|deepseek\.com|grok\.com).*$
```
  - Watch the bots in your server logs, and verify identity by IP — vendors publish JSON ranges ([openai.com/searchbot.json](https://openai.com/searchbot.json), [perplexity.ai/perplexitybot.json](https://www.perplexity.ai/perplexitybot.json), [Cloudflare verified bots](https://developers.cloudflare.com/bots/concepts/bot/verified-bots/)). Anthropic publishes none, fall back to reverse DNS.
```bash
awk '{print $1, $12}' access.log | grep -Ei 'GPTBot|OAI-SearchBot|ChatGPT-User|ClaudeBot|Claude-User|PerplexityBot|Bytespider|meta-externalagent|CCBot' | sort | uniq -c | sort -rn
```

## Care about metadata ?

  - The only three genuinely required elements, in this order, in the first 1024 bytes (reference: [htmlhead.dev](https://htmlhead.dev/))
```html
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>...</title>
```
  - Never `user-scalable=no` or `maximum-scale=1`: WCAG requires 200% zoom, browsers increasingly ignore it anyway, and audits still flag it.
  - `viewport-fit=cover` for notch/Dynamic Island edge-to-edge layouts, paired with `env(safe-area-inset-*)` padding. `interactive-widget=resizes-content` only if the virtual keyboard must resize your layout.
  - Declare color schemes in HTML, not CSS — the browser paints faster
```html
<meta name="color-scheme" content="light dark">
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#111111" media="(prefers-color-scheme: dark)">
```
  - Language on the element, not in a meta: `<html lang="en" dir="ltr">`
  - Description for search and previews: `<meta name="description" content="...">`
  - RSS/Atom still matters (readers, Mastodon bots, LLM crawlers)
```html
<link rel="alternate" type="application/rss+xml" href="/rss.xml" />
```
  - The sitemap belongs in `robots.txt` (`Sitemap: https://…`), not in a `<link>`.
  - Avoid Google's translation bar when you know it's not needed
```html
<meta name="google" content="notranslate">
```

## Care about icons ?

  - Six files, four lines. That's the whole 2026 answer ([evilmartians, "How to Favicon"](https://evilmartians.com/chronicles/how-to-favicon-in-2021-six-files-that-fit-most-needs))
```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png"><!-- 180x180, no alpha -->
<link rel="manifest" href="/manifest.webmanifest">
```
  - Files: `favicon.ico` (32×32), `icon.svg` (embed `@media (prefers-color-scheme: dark)` *inside* the SVG), `apple-touch-icon.png` (180×180), plus `icon-192.png` / `icon-512.png` / `icon-mask.png` (512×512 maskable, safe zone = center 80%) referenced from the manifest only.
  - `sizes="32x32"` on the .ico is deliberate: `sizes="any"` makes Chrome download both the ICO and the SVG.
  - Generate with https://realfavicongenerator.net/ — it dropped Windows Metro and Touch Bar icons in Oct 2024, so its output is finally lean.

## Care about accessibility (a11y) ?

  - Build against **WCAG 2.2 AA** ([spec](https://www.w3.org/TR/WCAG22/) · [filterable quickref](https://www.w3.org/WAI/WCAG22/quickref/)). Nine criteria are new vs 2.1, and `4.1.1 Parsing` was removed.
    - **2.4.11 Focus Not Obscured (AA)**: sticky headers, cookie banners and chat widgets must not cover the focused element
    - **2.5.7 Dragging Movements (AA)**: every drag interaction needs a single-pointer alternative (tap-to-select → tap-to-place)
    - **2.5.8 Target Size Minimum (AA)**: 24×24 CSS px, or enough spacing between targets
    - **3.2.6 Consistent Help (A)**: help link/chat in the same relative order on every page
    - **3.3.7 Redundant Entry (A)**: don't re-ask for data already given in the same process
    - **3.3.8 Accessible Authentication (AA)**: allow paste into OTP/password fields, support passkeys, no puzzle CAPTCHA as the only path
  - WCAG 3.0 is not a target: [March 2026 Working Draft](https://www.w3.org/TR/wcag-3.0/), Bronze/Silver/Gold model, Candidate Rec not before ~2027. Track it, don't ship against it.
  - Legal deadlines already in force, not advice
    - **European Accessibility Act** applies since **28 June 2025** to consumer e-commerce, banking, telecom, transport booking, ebooks, AV media — including non-EU companies selling to EU consumers ([Directive 2019/882](https://eur-lex.europa.eu/eli/dir/2019/882/oj)). Microenterprise exemption: <10 staff **and** <€2M turnover.
    - **EN 301 549** is the harmonised standard; v3.2.1 maps to WCAG 2.1 AA, the next revision (2.2 AA) is in progress.
    - **ADA Title II (US)**: WCAG 2.1 AA. The DOJ's interim final rule of 20 Apr 2026 (91 Fed. Reg. 20902) pushed the deadlines to 26 Apr 2027 (population ≥50k) and 26 Apr 2028 (smaller entities) — only those dates moved, the nondiscrimination duty applies today. https://www.federalregister.gov/documents/2026/04/20/2026-07663
    - Publish an accessibility statement (conformance level, known gaps, contact, date) with the [W3C generator](https://www.w3.org/WAI/planning/statements/)
  - Automate what can be automated, and know the ceiling: [axe-core](https://github.com/dequelabs/axe-core) is the engine under Lighthouse and most scanners, and Deque's own figure is **57% of issues found automatically**. A 100 Lighthouse a11y score only proves the automatable subset passed.
  - Fail the build on violations with [`@axe-core/playwright`](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright) or [vitest-axe](https://github.com/chaance/vitest-axe)
```js
const results = await new AxeBuilder({ page }).withTags(['wcag2a','wcag2aa','wcag21a','wcag21aa','wcag22aa']).analyze();
expect(results.violations).toEqual([]);
```
  - Lint at author time with [eslint-plugin-jsx-a11y](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y) (the `jsx-eslint` org — the old `evcohen` URL points at a stale fork)
  - Crawl whole sites: [Pa11y](https://github.com/pa11y/pa11y) (`pa11y-ci` + sitemap) or [IBM Equal Access](https://github.com/IBMa/equal-access). Per-component: the [Storybook a11y addon](https://storybook.js.org/docs/writing-tests/accessibility-testing).
  - Manual spot checks: [axe DevTools](https://www.deque.com/axe/devtools/) · [WAVE](https://wave.webaim.org/) · [ARC Toolkit](https://www.tpgi.com/arc-platform/arc-toolkit/) · [Accessibility Insights](https://accessibilityinsights.io/)
  - Manual passes no tool replaces
    - unplug the mouse and Tab through every flow: visible focus everywhere, logical order, no traps, modals return focus to the trigger
    - screen readers weighted by real usage ([WebAIM Survey #10](https://webaim.org/projects/screenreadersurvey10/)): JAWS + Chrome (40.5% primary), NVDA + Chrome/Firefox (37.7% primary, free), VoiceOver (9.7% desktop but 70.6% on mobile)
    - zoom to 400% and a 320px-wide viewport: no horizontal scroll, no lost content (1.4.10 Reflow)
    - Windows High Contrast via `@media (forced-colors: active)` — verify icons and borders survive
  - Reality check: **95.9% of the top 1M home pages have detectable WCAG failures**, 56 errors/page on average, and it's getting worse as pages get more ARIA-heavy — [WebAIM Million](https://webaim.org/projects/million/). Low contrast (83.9%), missing alt (53.1%), missing form labels (51%) and empty links (46.3%) are most of it.
  - Semantic HTML first — [first rule of ARIA](https://www.w3.org/TR/using-aria/#rule1): no ARIA is better than bad ARIA. `<button>` over `<div onclick>`.
  - Copy patterns, don't invent them: [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) for tabs, combobox, disclosure, dialog.
  - Focus ring for keyboard users only, never `outline: none`
```css
:focus-visible { outline: 3px solid Highlight; outline-offset: 2px; }
```
  - Skip link as the first focusable element: `<a href="#main">Skip to content</a>` + `<main id="main" tabindex="-1">`
  - [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/dialog) `.showModal()` gives focus trap + Esc + backdrop for free; `popover` for non-modal overlays; [`inert`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inert) on background content for custom ones.
  - Announce async changes with `<div aria-live="polite" aria-atomic="true">` present in the DOM *before* you write into it.
  - Every `<video>` needs `<track kind="captions" srclang="en" src="/captions.vtt" default>` (1.2.2 AA) plus an audio description or transcript (1.2.3/1.2.5). Auto-generated captions don't meet the bar — edit them.
  - Every input gets a `<label for>` and an [`autocomplete` token](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/autocomplete) (`email`, `given-name`, `street-address`, `one-time-code`) — that covers 1.3.5, 3.3.7 and 3.3.8 at once.
  - Check contrast with https://webaim.org/resources/contrastchecker/ (4.5:1 text, 3:1 large text and UI). APCA was removed from the WCAG 3 draft in 2023 and the replacement algorithm is still TBD, so pass WCAG 2 ratios first.
  - Learn: [web.dev Learn Accessibility](https://web.dev/learn/accessibility) · [W3C WAI Tutorials](https://www.w3.org/WAI/tutorials/) · [A11y Project checklist](https://www.a11yproject.com/checklist/) · the [UK Home Office dos/don'ts posters](https://github.com/UKHomeOffice/posters/tree/master/accessibility) are still the best one-page design summary

## Care about privacy ?

  - Detect **Global Privacy Control**, a legally binding opt-out in 12+ US states (CA, CO, CT, DE, MD, MN, MT, NE, NH, NJ, OR, TX). Header `Sec-GPC: 1`, or in JS:
```js
if (navigator.globalPrivacyControl) { /* don't sell/share, suppress ad tags */ }
```
  - Declare compliance at `/.well-known/gpc.json` → `{"gpc": true, "lastUpdate": "2026-07-30"}`. Spec: https://globalprivacycontrol.org/
  - Enforcement is real: the CPPA + Colorado and Connecticut AGs ran a joint GPC sweep in Sept 2025, and opt-out mechanism failures cost Sephora $1.2M (2022), Tractor Supply $1.35M (CPPA, Sept 2025) and Disney $2.75M (California AG, Feb 2026).
  - Load **nothing non-essential before consent** — no analytics, no pixel, no font, no embed. Regulators test at network level, not visually: the CNIL fined SHEIN €150M (and Google €325M) in Sept 2025 for cookies dropped on arrival, before any interaction. https://www.cnil.fr/en/cookies-placed-without-consent-shein-fined-150-million-euros-cnil
  - "Reject all" on the **first layer**, same visual weight as "Accept all", no cookie walls, withdrawal as easy as consent. [EDPB Guidelines 03/2022 on deceptive design patterns](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-032022-deceptive-design-patterns-social-media_en).
  - Self-host fonts: `LG München I, 20 Jan 2022, 3 O 17493/20` awarded €100 damages for leaking a visitor IP to Google Fonts. How-to in [Care about style ?](#care-about-style-).
  - Kill third-party embeds, or defer them behind a click: `youtube-nocookie.com/embed/ID`, `player.vimeo.com/video/ID?dnt=1`, [lite-youtube-embed](https://github.com/paulirish/lite-youtube-embed).
  - Opting out of ad-tech APIs is now archaeology: Chrome 144 deprecated them (Jan 2026) and Chrome 150 removes the JS surface, the `Sec-Browsing-Topics` header *and* the `browsing-topics`/`interest-cohort` Permissions-Policy features themselves. Keep the header as belt-and-braces for pre-150 builds, delete it once your traffic has rolled over.
```http
Permissions-Policy: browsing-topics=(), interest-cohort=(), join-ad-interest-group=()
```
  - Cookie hardening is in [Care about security ?](#care-about-security-). The privacy-specific part: `Partitioned` (CHIPS) on anything cross-site, so an embed can't build a profile across the sites embedding it.
```http
Set-Cookie: __Host-embed=…; Path=/; Secure; SameSite=None; Partitioned
```
  - Only ship a cookie banner if you actually set non-essential cookies. The real fix is removing the trackers so no banner is needed. If you need one, use a CMP that logs proof of consent — [Klaro](https://klaro.org), [cookieconsent](https://cookieconsent.orestbida.com), Didomi, Axeptio.
  - Third-party cookies are **not** going away in Chrome: Google reversed the deprecation (April 2025), then retired most of Privacy Sandbox (Oct 2025 — Topics, Protected Audience, Attribution Reporting, IP Protection). Safari/Firefox/Brave still block them, so build first-party by default.
  - Audit what actually fires on your own pages: https://webbkoll.dataskydd.net · https://themarkup.org/blacklight

## Care about legal ?

  - Privacy policy + cookie table (names, purpose, lifetime, recipients), reachable in ≤2 clicks from every page.
  - The parts everyone forgets until the first request lands: a **retention schedule** per data category with an actual deletion job (not a policy sentence), a working route for **access/erasure/portability requests** (GDPR Art. 15-20, one month to answer), a **register of processors** with a signed DPA each (Art. 28), and a **breach runbook** — 72 hours to notify the supervisory authority (Art. 33).
  - Footer link "Do Not Sell or Share My Personal Information" (or the "Your Privacy Choices" icon from https://privacyrights.info) — no login required, and it must actually work.
  - 🇩🇪 **Impressum**: the legal basis is now **§ 5 DDG**, not § 5 TMG (repealed 14 May 2024). Generator: https://www.e-recht24.de/impressum-generator.html
  - **DSA**: even micro/small sites hosting user content owe a published point of contact (Art. 11-12), an EU legal representative if non-EU (Art. 13), ToS transparency (Art. 14), and notice-and-action + statement of reasons (Art. 16-17). Transparency reports are exempt under 50 employees / €10M.
  - **EU AI Act Art. 50** applies **2 Aug 2026**: tell users they're talking to an AI, machine-readably mark synthetic content, label deepfakes. Fines up to €15M / 3%. https://artificialintelligenceact.eu/article/50/
  - **Accessibility statement** (EAA, in force 28 June 2025): conformance status vs EN 301 549 / WCAG 2.1 AA, known barriers, feedback contact. Generator: https://www.w3.org/WAI/planning/statements/
  - Age assurance if you host adult content or target minors: UK Online Safety Act (Ofcom, up to 10% turnover), 15+ US states, EU DSA proceedings. Self-declaration is not enough.
  - Watch but don't implement yet: the **EU Digital Omnibus** (proposed 19 Nov 2025) would move cookie rules into GDPR Arts. 88a/88b with a narrow audience-measurement exemption and machine-readable browser signals. Still in trilogue, nothing in force.
  - None of this is legal advice, and most of it is jurisdiction-dependent. Get a lawyer before betting the company on a bullet point in a README.

## Care about style ?

  - Use a modern reset, not `normalize.css`: [Josh Comeau's](https://www.joshwcomeau.com/css/custom-css-reset/) · [Andy Bell's](https://piccalil.li/blog/a-more-modern-css-reset/) · [open-props/normalize](https://open-props.style/)
  - Pad against the notch and the home indicator, with the `viewport-fit=cover` meta from [Care about metadata ?](#care-about-metadata-)
```css
.bar { padding-block-end: max(1rem, env(safe-area-inset-bottom)); }
```
  - Nesting, no Sass needed: `.card { & > h2 { margin: 0 } }`
  - Container queries beat media queries for components: `.grid { container-type: inline-size }` then `@container (width > 40ch) { .card { display: grid } }`, with `cqi`/`cqb`/`cqw` units
  - `:has()` for parent/sibling selection: `.form:has(:invalid) .submit { opacity: .5 }`
  - Cascade layers kill specificity wars: `@layer reset, base, components, utilities;`
  - `@property` makes custom properties typed and animatable
```css
@property --angle { syntax: "<angle>"; inherits: false; initial-value: 0deg; }
```
  - `light-dark()` replaces duplicated `prefers-color-scheme` blocks (requires `color-scheme`)
```css
:root { color-scheme: light dark; }
body { background: light-dark(#fff, #111); }
```
  - `oklch()` for perceptually uniform, wide-gamut color: `color: oklch(62% 0.19 258)`
  - Fluid type without breakpoints: `font-size: clamp(1rem, 0.9rem + 0.5vw, 1.35rem)`
  - `subgrid` aligns card internals across a grid: `grid-template-rows: subgrid`
  - `text-wrap: balance` on headings (`pretty` is still Firefox-less, so progressive enhancement)
  - `scrollbar-gutter: stable both-edges` removes the layout jump on overflow, and `scrollbar-width`/`scrollbar-color` style it — this is what replaced the old "don't use custom scrollbar plugins" advice
  - `field-sizing: content` gives an auto-growing textarea with zero JS
  - `@starting-style` + `transition-behavior: allow-discrete` animate `display:none` → visible without JS timing hacks
  - `popover` attribute replaces the tooltip/menu library: `<button popovertarget="m">` / `<div id="m" popover>` (modals: `<dialog>`, see [a11y](#care-about-accessibility-a11y-))
  - Anchor positioning and scroll-driven animations are still **Baseline limited** — gate them behind `@supports (animation-timeline: view())` with a sane static fallback
  - `accent-color: <brand>` on form controls, cheap brand win that degrades silently
  - Always respect motion preferences
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; transition-duration: .01ms !important; scroll-behavior: auto !important; }
}
```
  - Fonts: self-host via https://fontsource.org/, one variable font instead of four statics, `font-display: swap`, and kill the swap-induced CLS with `size-adjust`/`ascent-override` in `@font-face`. Or just ship `font-family: system-ui, sans-serif`.
  - Avoid FOUC/FOIT: https://css-tricks.com/fout-foit-foft/

## Care about browser support ?

  - Stop guessing. Use **Baseline** (W3C WebDX CG): *Widely available* = interoperable for ≥30 months, *Newly* = in all core engines today, *Limited* = not yet. https://web.dev/baseline
  - Query any feature: https://webstatus.dev/ (`curl https://api.webstatus.dev/v1/features/container-queries`). caniuse and MDN both show Baseline badges now.
  - Feature data as a dependency: `npm i web-features` https://github.com/web-platform-dx/web-features
  - Browserslist speaks Baseline natively, no plugin needed
```json
"browserslist": ["baseline widely available on 2026-01-01"]
```
  - Pin the date: "widely available" is a rolling window, so an unpinned query makes builds non-reproducible.
  - Enforce it in lint, not in code review
    - CSS: `css/use-baseline` in [`@eslint/css`](https://github.com/eslint/css), or [`stylelint-plugin-use-baseline`](https://github.com/ryo-manba/stylelint-plugin-use-baseline)
    - HTML: [`@html-eslint/use-baseline`](https://html-eslint.org/docs/rules/use-baseline)
    - JS: [`eslint-plugin-baseline-js`](https://github.com/3ru/eslint-plugin-baseline-js)
```js
// eslint.config.js
rules: { "css/use-baseline": ["warn", { available: "widely" }] }
```
  - Runtime detection stays native: `@supports (anchor-name: --x)`, `@supports selector(:has(*))`, `if ("startViewTransition" in document)`.
  - `<script type="module">` alone. `nomodule` is dead weight — the browsers it feeds (IE11, Edge ≤18, Safari ≤10) are all EOL. It was never a syntax gate either: Chrome 61-79, Firefox 60-73 and Safari 10.1-13 load modules but choke on optional chaining. That's what your Baseline target and the transpiler are for.

## Care about performance ?

  - Core Web Vitals are the only 3 that count, measured at p75 over 28 days of **field** data: **LCP ≤ 2.5s**, **INP ≤ 200ms**, **CLS ≤ 0.1** (poor above 4s / 500ms / 0.25). INP replaced FID on 2024-03-12. https://web.dev/articles/vitals
  - INP is the metric most sites fail. Order of attack: everything in the "poor" bucket, then INP, then LCP, then CLS.
  - Read your CrUX data (28-day lag) at https://pagespeed.web.dev, and collect your own RUM with `web-vitals` v5 — the attribution build tells you *which element or script* caused it
```js
import {onLCP, onINP, onCLS} from 'web-vitals/attribution';
onINP(m => navigator.sendBeacon('/rum', JSON.stringify(m)));
```
  - SPAs: `soft-navigation` + `interaction-contentful-paint` entries let you slice LCP/INP/CLS per route. Diagnostic, not a vital. https://developer.chrome.com/docs/web-platform/soft-navigations
  - Lab and debug tools: Chrome DevTools Performance panel, https://www.webpagetest.org, https://www.debugbear.com, https://speedcurve.com, https://treo.sh
  - Define a performance budget and enforce it in CI, not in a doc: [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) assertions + [size-limit](https://github.com/ai/size-limit) on the JS bundle
  - Use HTTP/3 (QUIC) at the edge, advertised via `Alt-Svc` **and** a DNS `HTTPS` (SVCB) record so the first connection is already QUIC
  - Compress with Brotli `-q 11` precompressed for static assets, `zstd` for dynamic responses, gzip as fallback, and always `Vary: Accept-Encoding`
  - Immutable caching for hashed assets: `Cache-Control: public, max-age=31536000, immutable`
  - HTML is never immutable: `Cache-Control: no-cache` (revalidate, don't skip the cache), or `max-age=0, s-maxage=60, stale-while-revalidate=600` at the CDN. Don't reach for `no-store`, it throws away more than you think.
  - `103 Early Hints` fills origin think-time — 2 to 4 critical resources only, more hurts
```http
HTTP/1.1 103 Early Hints
Link: </app.css>; rel=preload; as=style
```
  - Resource hints in order of cost: `preconnect` for an origin you *will* hit (max 2-3), `preload` for late-discovered critical assets, `modulepreload` for ESM. `dns-prefetch` is now mostly redundant.
```html
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
<link rel="preload" href="/hero.avif" as="image" fetchpriority="high">
<link rel="modulepreload" href="/app.js">
```
  - Speculation Rules replace `rel=prerender` and JS hacks like instant.page — real prerender, URL patterns, eagerness levels, silently ignored where unsupported
```html
<script type="speculationrules">
{"prerender":[{"where":{"and":[{"href_matches":"/*"},
  {"not":{"or":[{"href_matches":"/logout"},{"href_matches":"/cart*"},{"href_matches":"/account*"}]}}]},
  "eagerness":"moderate"}],
 "prefetch":[{"urls":["/checkout"],"eagerness":"immediate"}]}
</script>
```
  - Ship AVIF with a WebP/JPEG fallback. JPEG XL is Safari-only, so serve it through `<picture>` or CDN negotiation, don't bet on it.
  - The LCP image: server-rendered, `fetchpriority="high"`, **never** `loading="lazy"`, one per page
```html
<img src="hero.avif" width="1200" height="630" fetchpriority="high" decoding="async" alt="…"
     srcset="hero-600.avif 600w, hero-1200.avif 1200w" sizes="(max-width: 700px) 100vw, 1200px">
```
  - Everything below the fold: `loading="lazy" decoding="async"`
  - Always set `width`/`height` (or `aspect-ratio`) on images, iframes, ads and embeds — that's where most CLS comes from. Reserve space for late-injected banners with `min-height`.
  - Optimize with `sharp`/`squoosh` in the build, https://jakearchibald.github.io/svgomg/ for SVG, or an image CDN doing format negotiation for you. Read https://images.guide/.
  - Break long tasks so the browser can paint. `scheduler.yield()` has no Safari support yet, so keep the fallback.
```js
const yieldToMain = () => globalThis.scheduler?.yield?.() ?? new Promise(r => setTimeout(r, 0));
let t = performance.now();
for (const item of items) { render(item); if (performance.now() - t > 50) { await yieldToMain(); t = performance.now(); } }
```
  - Update the UI *first*, defer the rest: paint the visible change in the event handler, then `await yieldToMain()` before analytics and logging. `scheduler.postTask(() => sendAnalytics(), {priority:'background'})` for non-urgent work, `requestIdleCallback` for true idle work.
  - Skip rendering work for offscreen sections with `content-visibility: auto; contain-intrinsic-size: auto 500px;` — the intrinsic size is mandatory, otherwise you trade CLS for INP
  - Third-party scripts are usually the INP culprit. Audit with DevTools → Performance → Long animation frames, move tag managers to a worker with https://partytown.qwik.dev, and `defer` everything non-critical.
  - Ship less JS: dynamic `import()` on interaction, islands / partial hydration / RSC instead of full-page hydration, render at the edge
  - Keep the back button instant (bfcache): no `unload` handlers — an empty one still disqualifies you, use `pagehide`/`visibilitychange` + `sendBeacon`. Audit it in the field:
```js
new PerformanceObserver(l => l.getEntries().forEach(e => console.log(e.notRestoredReasons)))
  .observe({type: 'navigation', buffered: true});
```
  - Cross-document View Transitions for perceived speed, ignored where unsupported
```css
@media (prefers-reduced-motion: no-preference) { @view-transition { navigation: auto; } }
```
  - Animate with CSS `transform`/`opacity` or `requestAnimationFrame`, never `setInterval`. Batch DOM reads and writes to avoid layout thrashing.
  - Inline the critical CSS in `<head>`, load the rest non-blocking, split vendor/app chunks, and analyze what you actually ship with https://sonda.dev or https://rsdoctor.rs
  - To do something before unloading the page, use `navigator.sendBeacon()` so you don't block the close
  - Google's own agent-readable playbooks (INP causes, long tasks, image priority): https://github.com/GoogleChrome/modern-web-guidance
  - Deeper per-item checklist: [Front-End-Performance-Checklist](https://github.com/thedaviddias/Front-End-Performance-Checklist)

## Care about mobile ?

  - Ship a `manifest.webmanifest` and cut the Apple meta soup down to the two that still do something (see below) — Safari reads the manifest now, and only falls back to the deprecated metas when it can't load it
```json
{
  "id": "/?source=pwa",
  "name": "Example", "short_name": "EXX",
  "start_url": "/", "scope": "/",
  "display": "standalone",
  "display_override": ["standalone", "minimal-ui", "browser"],
  "theme_color": "#111111", "background_color": "#ffffff",
  "description": "...",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icon-mask.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "screenshots": [{ "src": "/shot-wide.png", "sizes": "1280x720", "type": "image/png", "form_factor": "wide" }]
}
```
  - `id` is what lets you change `start_url` later without the browser treating it as a brand-new app. Set it once, never touch it.
  - `description` + `screenshots` upgrade the Android install prompt from a grey infobar to an app-store-style dialog. Cheapest install-rate win available.
  - Optional but real: `shortcuts` (max 4), `share_target` (be a share destination), `protocol_handlers`, `file_handlers`.
  - `<meta name="mobile-web-app-capable" content="yes">` replaces `apple-mobile-web-app-capable`. Caveat: removing the Apple one breaks `apple-touch-startup-image` splash screens on iOS, so ship both if you use them. `apple-mobile-web-app-status-bar-style` has no manifest equivalent — keep it for `black-translucent`.
  - iOS PWA reality: web push works since 16.4 but **only** after Add to Home Screen via Safari, `navigator.setAppBadge()` needs notification permission, every push must call `showNotification()`, and there's no `beforeinstallprompt` — so design an explicit install coach-mark. Detect install with `window.matchMedia('(display-mode: standalone)').matches`.
  - Audit the manifest with Lighthouse and https://www.pwabuilder.com/ (flags missing `id`/`screenshots`)

## Care about offline ?

  - Service worker + [Workbox](https://developer.chrome.com/docs/workbox), in dependency-patch mode these days. Active fork if you want momentum: [Serwist](https://serwist.pages.dev/).
  - The offline cookbook (the old `developers.google.com/web/fundamentals` URL redirects here): https://web.dev/articles/offline-cookbook
  - Precache pitfalls, in order of how often they bite
    - version your caches with a build hash and delete everything else in `activate`
    - `skipWaiting()` unconditionally = users get half-old/half-new chunks mid-session. Prompt instead:
```js
reg.addEventListener("updatefound", () => {           // reg.waiting is still null here
  const sw = reg.installing;
  sw.addEventListener("statechange", () => {
    if (sw.state === "installed" && navigator.serviceWorker.controller)  // not the first install
      showToast("New version", () => sw.postMessage({ type: "SKIP_WAITING" }));
  });
});
```
    - never precache `index.html` with a long-lived cache-first strategy → "stale SW hell". HTML is network-first.
    - set `Cache-Control: max-age=0` on `sw.js` itself
  - Cache API for responses, IndexedDB for data. [`idb`](https://github.com/jakearchibald/idb) (~1.2 kB, typed) or `idb-keyval` for pure key/value — localForage is ~7 kB of legacy fallbacks you no longer need.
  - Quota: `navigator.storage.estimate()`, and ask for durability with `await navigator.storage.persist()` (iOS still evicts unused PWA data).
  - Offline fallback page: catch `event.request.mode === "navigate"` failures and serve a precached `/offline.html`.
  - Background Sync is Chromium-only and Periodic Background Sync is Chromium + installed PWA only. Keep a replay queue in IndexedDB and flush on next load — don't design a feature around them.

## Care about analytics ?

  - Pick a first-party, cookieless tool: [Plausible](https://plausible.io) (AGPL, self-hostable) · [Umami](https://umami.is) (MIT) · [Matomo](https://matomo.org) · [Fathom](https://usefathom.com) · [Simple Analytics](https://www.simpleanalytics.com) · [Pirsch](https://pirsch.io) · [GoatCounter](https://www.goatcounter.com) · [Cloudflare Web Analytics](https://www.cloudflare.com/web-analytics/) (free, includes CWV)
  - Zero-JS option: parse your access logs with [GoAccess](https://goaccess.io) — `goaccess access.log -o report.html --log-format=COMBINED --anonymize-ip`
  - Proxy the analytics script from your own domain (`/js/script.js` → vendor): 25-45% of traffic blocks known analytics hostnames.
  - **Cookieless ≠ consent-exempt.** France's CNIL retired its approved-tool list on 1 Jan 2026 and replaced it with vendor self-assessment against a published grid. The exemption criteria are cumulative: own-site measurement only, anonymous aggregates, no third-party transfer, no cross-site tracking, cookie ≤13 months, data ≤25 months. https://www.cnil.fr/fr/cookies-solutions-pour-les-outils-de-mesure-daudience
  - Outside FR/LU, assume consent is required — the UK ICO's storage-and-access guidance (April 2026) confirms analytics needs prior consent under PECR.
  - GA4 is *conditionally* usable in the EU: transfers are covered by the EU-US Data Privacy Framework, upheld by the General Court in **Latombe (T-553/23, 3 Sep 2025)** — but an appeal is already pending before the CJEU (**C-703/25 P**), so treat the DPF as revocable and check the vendor is listed at https://www.dataprivacyframework.gov/list
  - GA4 also needs a signed DPA, the shortest retention, Google Signals off, and no PII in URLs. It has **never** qualified for the CNIL exemption.
  - **Consent Mode v2** is mandatory for EEA/UK/CH Google Ads and GA4 since March 2024 — 4 signals, all `denied` by default
```js
gtag('consent','default',{ad_storage:'denied',analytics_storage:'denied',
  ad_user_data:'denied',ad_personalization:'denied'});  // global default, relax via consent update
```
  - Don't write `region:['EEA',…]` — `region` takes ISO 3166-2 codes, there is no EEA shortcut, and an invalid entry silently applies to nobody. Default to denied everywhere instead.
  - Server-side tagging (GTM SS in an EU region) helps data minimization. It does **not** remove the consent requirement.
  - Verify site ownership without a tracker: `<meta name="google-site-verification" content="…">` or a DNS TXT record, in [Search Console](https://search.google.com/search-console) and [Bing Webmaster](https://www.bing.com/webmasters).
  - Google Tag Manager and the Meta Pixel are consent-gated third-party trackers in the EU, and the Meta Pixel carries additional health/sensitive-data litigation risk in the US.

## Care about bugs ?

  - [Sentry](https://sentry.io/) is still the default for frontend errors + perf + replay. Self-hosted lighter options: GlitchTip (Sentry-SDK compatible, no session replay, no source-map processing), Bugsink. PostHog if you want errors next to replays and flags.
  - Upload source maps in CI or your stack traces are noise: https://docs.sentry.io/platforms/javascript/sourcemaps/
  - Catch what SDKs miss: `window.addEventListener("error", …)`, `"unhandledrejection"`, and `reportError(e)` inside `catch` blocks so handled errors still reach `window.onerror`.
  - Session replay is the highest-risk data type you will ever collect: mask by default (`maskAllText`, `blockAllMedia`), allowlist rather than blocklist, sample low, and declare it in the privacy policy.
  - The Reporting API gets you CSP violations, deprecations, interventions and crash reports with no JS at all. `Reporting-Endpoints` replaces the deprecated `Report-To`, and both must be HTTP headers (no `<meta>`).
```http
Reporting-Endpoints: default="https://x.example/reports", csp="https://x.example/csp"
Content-Security-Policy-Report-Only: default-src 'self'; report-to csp
```
  - OpenTelemetry in the browser is **still experimental** and mostly unspecified — new home at https://github.com/open-telemetry/opentelemetry-browser. Pin versions or use a vendor distro, don't bet a migration on it yet.
  - Define error budgets before alerting ("JS error rate ≤ 0.5% of sessions, p75 INP ≤ 200 ms") and alert on budget burn, not on every exception.

## Care about ops ?

  - Uptime and synthetics: [Better Stack](https://betterstack.com/uptime) · [Checkly](https://www.checklyhq.com/) (Playwright-as-monitor) · self-hosted [Uptime Kuma](https://github.com/louislam/uptime-kuma)
  - Status page: statuspage.io is now the expensive incumbent with no built-in monitoring. [Instatus](https://instatus.com/) (free tier) · [OpenStatus](https://www.openstatus.dev/) (open source, self-hostable, Terraform provider) · Better Stack (monitoring + status bundled).
  - CI gates worth wiring, one per section above: [perf budget](#care-about-performance-), [a11y](#care-about-accessibility-a11y-), HTML validation, link check, bundle size. A checklist nobody runs is a blog post.
  - Link rot: [lychee](https://github.com/lycheeverse/lychee) (Rust, fast) via `lycheeverse/lychee-action@v2` on a cron, with `--cache --max-cache-age 1d` and a `.lycheeignore`.
  - 404s: a real page with search + top links (not a redirect to `/`), the correct 404 status, and monitor your top 404 paths — they're broken inbound links you can 301.
  - A backup you have never restored is not a backup: schedule a real restore drill, keep one copy off-provider, and check it covers the DB *and* user uploads. Rehearse the rollback too — knowing how to revert a bad deploy in one command beats any amount of pre-deploy checking.
  - Monitor the things that take the whole site down and aren't in your repo: domain expiry, TLS cert expiry, DNS record drift and registrar lock. Most uptime monitors do all four.
  - Feature flags to decouple deploy from release: [OpenFeature](https://openfeature.dev/) (vendor-neutral SDK spec), self-hosted Unleash/Flagsmith, or PostHog if you already have it.

## Care about misc ?

  - Progressive enhancement is still the cheapest resilience: server-render, make forms work with a plain `<form method="post">`, treat JS as an upgrade.
  - i18n basics: `<html lang="fr" dir="ltr">` on every page, `hreflang` for translations, and `Intl.*` (`DateTimeFormat`, `NumberFormat`, `RelativeTimeFormat`) instead of a date library.
  - Validate your pages: https://validator.w3.org/nu/?doc=https%3A%2F%2Fexample.com, and in CI use `html-validate` or `vnu-jar`.
  - Sustainability: measure at https://www.websitecarbon.com/, host green ([Green Web Foundation](https://www.thegreenwebfoundation.org/) + CO2.js), follow the [W3C Web Sustainability Guidelines](https://w3c.github.io/sustainableweb-wsg/).
  - Give users a way to send feedback: a plain form, Canny, Featurebase, or GitHub Discussions for OSS.

## Care about deleting ?

Everything below was in the 2017 edition of this list, or is still being copy-pasted from old boilerplate. Delete it.

  - ~~`Public-Key-Pins` / HPKP~~ — removed from Chrome 72 (Jan 2019) and Firefox 78 (2020, off by default since 72), never shipped by Safari or Edge, and it bricked sites permanently. Certificate Transparency + CAA replaced it.
  - ~~`X-XSS-Protection: 1; report=…`~~ — the XSS Auditor was removed from every browser and introduced vulnerabilities of its own. If you keep the header, keep it as `X-XSS-Protection: 0`.
  - ~~`X-Download-Options`~~ (IE8), ~~`X-Permitted-Cross-Domain-Policies`~~ (Flash/Reader), ~~`X-Webkit-CSP`~~, ~~`X-Content-Security-Policy`~~ — all dead.
  - ~~`X-UA-Compatible: IE=edge,chrome=1`~~ — IE11 is EOL since June 2022, and `chrome=1` was Chrome Frame (dead 2014).
  - ~~`rel="noopener"` on `target="_blank"`~~ — implicit since Safari 12.1 (2018), Firefox 79 (2020) and Chrome 88 (Jan 2021). `noreferrer` is **not** implied: keep it when you actually want to strip the `Referer`, drop it when the destination needs the attribution. `window.open()` still needs `'noopener'` in its features string.
  - ~~`Timing-Allow-Origin` "for privacy"~~ — it *grants* timing access, it doesn't restrict it.
  - ~~Smyte~~ — acquired by Twitter in 2018, service shut off overnight.
  - ~~`npm audit` as your supply-chain strategy~~ — keep it as a free local check, but it reports presence not reachability, and a malicious package has no CVE so it sees nothing.
  - ~~DNT (`DNT: 1`) and `/.well-known/dnt-policy.txt`~~ — spec discontinued, Safari dropped it in 2019, Firefox removed the toggle in v135 (Feb 2025). GPC replaced it.
  - ~~FLoC / Topics API / Protected Audience / Attribution Reporting~~ — retired by Google in Oct 2025. Don't build on them.
  - ~~IAB TCF as a compliance shortcut~~ — Brussels Market Court, 14 May 2025: TC Strings are personal data, IAB Europe is a joint controller, legitimate interest rejected. TCF doesn't launder consent.
  - ~~`analytics.js` / `ga()` / Universal Analytics~~ — stopped processing July 2023, properties deleted July 2024. GA4 uses `gtag.js`.
  - ~~Piwik~~ → renamed Matomo in 2018. ~~Gaug.es~~ → dead product.
  - ~~Google Plus, everything~~ — killed in 2019, the share docs redirect to a shutdown notice.
  - ~~`fb:admins` / `fb:app_id`~~ — only ever needed for Facebook Insights on a domain you own, irrelevant for previews.
  - ~~`twitter:title` / `twitter:description` / `twitter:image` / `twitter:creator`~~ — X falls back to `og:*`, so duplicating them just guarantees drift. ~~`twitter:domain`~~ was never a real tag. ~~cards-dev.twitter.com/validator~~ is dead.
  - ~~FB/Twitter/G+ share button widgets~~ — one of the worst INP offenders in the original list.
  - ~~AMP / `<link rel="amphtml">`~~ — no ranking benefit since the June 2021 Page Experience update, badge removed, Top Stories open to all pages. 301 your AMP URLs to canonical.
  - ~~`<meta http-equiv="Content-Type">`~~ (redundant with `<meta charset>`), ~~`Content-Language`~~ and ~~`<meta name="language">`~~ (non-standard, `<html lang>` is the only one read).
  - ~~`<meta name="keywords">`~~ — ignored by Google since 2009. ~~`news_keywords`~~, ~~`syndication-source`~~, ~~`original-source`~~ — all deprecated by Google, never replaced.
  - ~~`rel="prev"` / `rel="next"`~~ — Google confirmed in 2019 it hadn't used them for years.
  - ~~`<meta name="robots" content="index,follow">`~~ — that's the default, the tag does nothing. Only emit meta robots when *deviating*.
  - ~~`rel="pingback"` + XML-RPC~~ (dead protocol and a DDoS amplification vector), ~~RSD `rel="EditURI"`~~ and ~~`wlwmanifest`~~ (Windows Live Writer died in 2017), ~~`msapplication-task`~~, ~~`msapplication-TileColor`~~, ~~`msapplication-TileImage`~~ (Metro tiles are gone).
  - ~~`rel="shortlink"`~~, ~~`rel="image_src"`~~, ~~`rel="fluid-icon"`~~, ~~`rel="mask-icon"`~~, ~~`rel="shortcut icon"`~~ (the `shortcut` token is meaningless), ~~XFN `rel="profile"`~~ — no consumer left. Use `rel="me"` for identity verification.
  - ~~Dublin Core `DC.*` metas~~ — no search engine or platform consumes them on the web.
  - ~~The 9 `apple-touch-icon` sizes + `favicon-16/96/192` PNGs~~ — one 180×180 apple-touch-icon plus an SVG covers everything since iOS 8.
  - ~~`humans.txt`~~ — still charming, but nothing consumes it. Keep it as an easter egg, drop the `<link rel="author">`.
  - ~~FAQPage and HowTo rich results~~ — HowTo killed in 2023, FAQ rich results stopped appearing in 2026. The schema types stay valid, they just render nothing.
  - ~~polyfill.io / `cdn.polyfill.io` / `qa.polyfill.io`~~ — **supply-chain incident**. The domain was sold in Feb 2024 and from June 2024 injected malware into 100k+ sites (Sansec disclosure, June 2024; Namecheap suspended the domain days later). `cdn.polyfill.io` is now NXDOMAIN and the apex serves a 521 — the service is dead. Mirrors exist but treat them as a migration ramp, not a fix: remove the tag.
  - ~~Modernizr~~, ~~html5shiv~~, ~~conditional comments~~, ~~`github/fetch`~~, ~~`es6-promise`~~, ~~Respond.js~~, ~~`web-animations-js`~~ (archived), ~~`<script nomodule>`~~ — use `@supports` and native features instead. `core-js` only if your Browserslist target is genuinely old.
  - ~~sw-precache / sw-toolbox~~ — deprecated by Google, last release ~8 years ago. Migrate to Workbox: https://developer.chrome.com/docs/workbox/migration/migrate-from-sw
  - ~~AppCache~~ — removed from every browser.
  - ~~Quota API~~ → `navigator.storage.estimate()`. ~~`Report-To` header~~ → `Reporting-Endpoints`. ~~`rel="subresource"`~~ → never shipped. ~~`rel="prerender"`~~ → Speculation Rules.
  - ~~Prepack~~ (archived), ~~Guetzli~~ (archived, and AVIF/WebP made JPEG encoders irrelevant), ~~fastdom~~ (every framework batches DOM writes now), ~~instant.page~~ (Speculation Rules do it natively, with real prerender).
  - ~~`<link rel="stylesheet" href="//fonts.googleapis.com/…">`~~ — self-host, see the German Google Fonts ruling.
  - ~~"Use HTTP/2"~~ → HTTP/3. ~~"Gzip your resources"~~ → Brotli/zstd. ~~Varnish for static assets~~ → that's what a CDN is for.
  - ~~"Talk to a UI and UX designer"~~ and ~~"use clear and proper wording"~~ — not artifacts, they don't belong in a checklist.
  - Dead URLs that were in this list: `web.dev/measure` → https://pagespeed.web.dev · `testmysite.io`, `webbloatscore.com`, `google.github.io/tracing-framework` → gone · `html5rocks.com` CSP tutorial → https://web.dev/articles/csp · `securityheaders.io` → `.com` · `hstspreload.appspot.com` → https://hstspreload.org · `leaverou.github.io/contrast-ratio` → 404 · `addyosmani/a11y` → archived in 2018 · `evcohen/eslint-plugin-jsx-a11y` → https://github.com/jsx-eslint/eslint-plugin-jsx-a11y · `a11yproject.com/checklist.html` → https://www.a11yproject.com/checklist/ · `cookielaw.org` → a OneTrust product page.

# More tips

- https://developers.google.com/speed/docs/insights/rules
- https://htmlhead.dev/ — every element that can go in a `<head>`, with a DEPRECATED list
- https://github.com/GoogleChrome/modern-web-guidance — Google's agent-readable "don't do the 2017 thing" guides
