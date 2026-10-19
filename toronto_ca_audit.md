# toronto.ca website audit

**Date:** 2026-10-17
**Site:** https://www.toronto.ca (City of Toronto)
**Method:** an outside-in audit using only public information. Sources: DNS and WHOIS, TLS and HTTP headers, robots.txt, the full sitemap (5,835 URLs), page source, a real-browser pass over 6 representative pages (network, cookies, DOM, axe-core 4.10.2 accessibility scan, keyboard pass, 390 px mobile layout), a polite sample crawl of 80 sitemap URLs stratified by section, and Lighthouse 12.6.0 lab tests (mobile and desktop) on 3 pages plus 3 peer cities.

**Status key**
- **Found:** confirmed from public data.
- **Issue:** a problem worth fixing.
- **Needs access:** can only be answered with City accounts (registrar, CMS, analytics, Search Console, hosting) or by asking staff.
- **Partial:** checked on a sample, not the whole site.

Supporting file: `toronto_ca_content_inventory.csv` (every sitemap URL with section, template guess, depth and last-modified date; sampled rows also have status, redirect target, title, meta-description length, H1, word count and PDF links).

---

## Executive summary

The site is a mature, custom WordPress build behind Amazon CloudFront. It is fast at the server and generally accessible. The main risks are old front-end code, weak script controls on the 311 page, and privacy disclosure.

**Top 10 findings, ranked by risk**

| # | Finding | Area | Severity |
|---|---|---|---|
| 1 | The 311 page loads Microsoft's chat widget from a third-party CDN pinned to `latest`, plus React 16 from unpkg.com, with no Subresource Integrity. The Content Security Policy is report-only and allows `'unsafe-inline' 'unsafe-eval' https:`, so it blocks nothing. A compromised CDN file would run with full access to the page. | Security | High |
| 2 | Site search (find.toronto.ca, SearchBlox) runs **jQuery 2.2.4**, which has published XSS vulnerabilities (CVE-2019-11358, CVE-2020-11022, CVE-2020-11023). The main site uses Bootstrap 3.4.1 (end of life since 2019; CVE-2024-6484 affects 3.x), jQuery 3.6.0, jQuery Migrate, and IE8 polyfills. | Security / tech debt | High |
| 3 | No cookie notice or consent. An Oracle Infinity analytics cookie (`ORA_FPC`, 400 days, not `Secure`) is set on the first page view. reCAPTCHA and Google Translate load from Google on every page. The privacy statement was last updated 2019-12-20 and names none of these vendors. | Privacy | High |
| 4 | DMARC is `p=none`, so mail spoofing `@toronto.ca` is reported but not blocked. A city address is a prime phishing target. | Email security | High |
| 5 | Mobile performance is weak on key pages. Homepage: Lighthouse 67, LCP 7.6 s. 311 page: 41, LCP 8.1 s, about 4 MB and 730 ms blocking time. Server response is excellent (about 10 ms), so the cost is all front-end. | Performance | Medium |
| 6 | The 311 page adds a second viewport tag with `maximum-scale=1`, which blocks pinch-zoom. That fails WCAG 1.4.4, and AODA requires WCAG 2.0 AA. | Accessibility | Medium |
| 7 | The homepage has an empty `<h1>` and no `<main>` landmark. | Accessibility / SEO | Medium |
| 8 | The sitemap is incomplete and dirty. It omits news releases and all PDFs, and about 5% of sampled entries redirect or are `noindex`. robots.txt blocks Bingbot from `/news/*`, which hides news releases from Bing, DuckDuckGo and Copilot. | SEO | Medium |
| 9 | Meta descriptions on landing pages are auto-generated lists of child links. 78% of sampled pages exceed 160 characters, and 7.5% have none. Structured data is only `WebSite`, with no `GovernmentOrganization`, `BreadcrumbList` or `Event`. | SEO | Low |
| 10 | WordPress core is **6.8.3**, while the City's own open-data site already runs **7.1**. Confirm the 6.8 branch still gets security fixes. | Tech debt | Medium |

**What's done well:** HTTPS everywhere with HSTS, TLS 1.2 and 1.3 only (1.0 and 1.1 refused), no mixed content, a CAA record limiting who can issue certificates, strict SPF (`-all`), wp-login, xmlrpc and the REST API closed to the public, real 404s (no soft 404s), alt attributes on 762 of 762 sampled images, a skip link, labelled form fields, a visible keyboard focus ring, no axe colour-contrast failures, no horizontal scroll at phone width, and CloudFront cache hits with about 10 ms server response. Against peer cities on mobile, Toronto's homepage performance (67) beats Ottawa (62), Vancouver (56) and Montréal (46).

---

## 1. Access and ownership

| Item | Status | Finding |
|---|---|---|
| Domain registrar, expiry, auto-renew | Found / Needs access | Registrar **Webnames.ca Inc.** (IANA 456). Registrant: The City of Toronto (Webnames privacy proxy). Created 2002-04-19, **expires 2034-04-18**, status `clientTransferProhibited` (transfer lock on, good). Auto-renew setting and login: needs access. |
| DNS provider and records | Found | NS: **easyDNS** (dns1–4 across .com/.net/.org/.info). SOA primary `ns3.toronto.ca`, contact `isg.toronto.ca`. Records below. **DNSSEC not enabled** (no DS record). |
| Hosting account and plan | Needs access | www → **Amazon CloudFront** (`d12dj7ytm0t2bf.cloudfront.net`, Toronto POP YTO53). Apex `toronto.ca` → 206.130.170.36 (City IP range), which 301-redirects to www. Origin servers, specs, bandwidth and backups aren't visible from outside. |
| CMS logins, users and roles | Needs access | WordPress. The login page is not public (`/wp-login.php` returns 404) and the REST API index redirects to the homepage, so the admin is likely on a separate internal host. User accounts were deliberately not enumerated. |
| FTP/SSH/cPanel/database | Needs access | Nothing exposed publicly. |
| Third-party accounts | Found (identification only) | Oracle Infinity (analytics), Google reCAPTCHA v3, Google Translate, Medallia (feedback surveys), Microsoft Bot Framework (311 chat), SearchBlox (site search), Upaknee and Mailchimp (e-updates), DocuSign and Atlassian (verified via DNS TXT), Google Search Console (verified via DNS TXT), FrontDesk Suite (Toronto Public Health booking), ThoughtExchange (engagement), Entrust (TLS), easyDNS, AWS CloudFront, Akamai (secure.toronto.ca). |
| Builder, maintainer, contracts, licences | Needs access | Custom theme `cot` and plugin `cot-custom` suggest an in-house team. Response header `x-custom-cot-info: cot_20250211_1210` looks like a build/release tag (2025-02-11). Licences to confirm: Font Awesome 5.15.4 (Free or Pro?), Roboto web font (OFL), stock images. |

**Public DNS records**

| Type | Value |
|---|---|
| A (apex) | 206.130.170.36 |
| CNAME www | d12dj7ytm0t2bf.cloudfront.net |
| CNAME secure | secure.toronto.ca.edgekey.net (Akamai) |
| CNAME open | d3lsotq79zoipg.cloudfront.net |
| CNAME map | map.toronto-dns.ca → 206.130.170.31 |
| MX | 10 smtp.toronto.ca, 20 smtp-c.toronto.ca (self-hosted mail gateways) |
| TXT SPF | `v=spf1 ip4:206.130.167.77 ip4:204.187.67.88 a:stoic-1.inet.toronto.ca a:stoic-2.inet.toronto.ca -all` |
| TXT DMARC | `v=DMARC1; p=none; fo=1; rua=mailto:dmarc_agg@vali.email; ruf=mailto:dmarc@toronto.ca` (Valimail receives reports) |
| TXT other | google-site-verification, 2 × docusign, atlassian-domain-verification, 2 unlabelled base64 tokens |
| CAA | `issue "entrust.net"`, `issue "sectigo.com"` |
| DKIM | Not found on common selectors (`selector1/2`, `google`, `default`, `k1`, `s1/s2`). The real selector is unknown, so this is not proof DKIM is missing. Confirm with the mail team. |

## 2. Technology stack

| Item | Status | Finding |
|---|---|---|
| CMS and version | Found | **WordPress 6.8.3** (RSS `<generator>` and `?ver=6.8.3` on assets). open.toronto.ca runs WordPress 7.1 with a CKAN data backend. |
| Theme | Found | Custom theme **`cot`**, built in-house, not off the shelf. Many assets are versioned 2015–2017 (`navigation.js?ver=20151215`, `search.js?ver=20170712`), so the theme's core dates from the 2017 relaunch. |
| Front-end libraries | Issue | jQuery **3.6.0** (current 3.7.1 / 4.x), jQuery Migrate 3.4.0 (a dev shim left in production), jQuery UI Autocomplete 1.13.2, **Bootstrap 3.4.1** (EOL), html5shiv and respond.js (IE8 polyfills, dead code), js-cookie, the City's **COTUI 2.4.3** web-component library, and Font Awesome 5.15.4 (CSS on all pages, plus the full 442 KB JS build on the 311 page). The 311 page adds React 16.14.0 from unpkg.com and Bot Framework WebChat `latest`. |
| Plugins | Partial | Only `cot-custom` and `google-sitemap-generator` 4.1.21 (Arne Brachhold, still maintained) are visible from outside. Other custom bundles: `/resources/wp-feedback/` (page-feedback widget), `/resources/311_widgets/`, `/resources/feps_311/` (311 app), `/resources/fepe_311_issues_lookup/`. The full plugin list needs WP admin access. |
| Backend, server, database | Partial | PHP/WordPress (so almost certainly MySQL/MariaDB). The origin web server is hidden by CloudFront. |
| CDN, caching, SSL | Found | CloudFront with `s-maxage≈24 h`, `max-age=300`, HTTP/2 and HTTP/3 (`alt-svc h3`). Certificate: **Entrust OV TLS Issuing ECC CA 2**, CN `www.toronto.ca`, valid 2026-03-11 to **2027-03-11**. secure.toronto.ca: Entrust, expires **2027-02-03**. |
| Email delivery | Found | Inbound: on-site `smtp.toronto.ca`. Outbound: the SPF-listed City IPs. Newsletters: **Upaknee** (`track.upaknee.com`) and **Mailchimp** (`toronto.us10.list-manage.com`), so two email platforms are in use. |
| Staging/dev | Needs access | None publicly visible (good). |
| Backups | Needs access | Ask: frequency, storage location, and the date of the last test restore. |

**Other City properties linked from the site:** find.toronto.ca (SearchBlox search, CloudFront), secure.toronto.ca (Akamai, application platform, 403 at the root), app.toronto.ca, open.toronto.ca (open data), map.toronto.ca (its root redirects to an error page), `to311kb-web0.ecrm.inter.prod-toronto.ca` (311 knowledge base. The hostname exposes internal "eCRM" naming), and the legacy `/legdocs/` archive.

## 3. Content and structure

| Item | Status | Finding |
|---|---|---|
| Full URL list | Found (sitemap) | **5,835 pages** across 7 sitemap files (all WordPress *pages*). Not in the sitemap: news releases (`/news/…`, a separate post type), PDFs, the `/legdocs/` archive, and subdomain apps. Search Console data needs access. |
| Pages by template (URL depth heuristic) | Found | Home 1 · section landings 7 · topic landings 41 · sub-topic landings 414 · content pages 5,214 · utility (`/home/`) 154 · account 4. |
| By section | Found | city-government 2,022 · community-people 1,526 · services-payments 1,319 · explore-enjoy 573 · business-economy 234 · home (utility) 154. |
| Navigation | Found | Main menu with 5 sections (Services & Payments, Community & People, Business & Economy, Explore & Enjoy, City Government), loaded by background request from `/globalnav/header/` and fetched **twice per page**. An "I want to…" panel and topic quick links sit on the homepage. Footer: Jobs, Media Room, Contact Us, Translate, Social media, Copyright, Privacy, Accessibility. Hierarchy is deep: 1,745 pages (30%) sit 6–9 levels down. |
| Orphan pages | Needs access | Needs a full crawl compared with the sitemap, or Search Console. Not done here, because a full crawl of a government site should be agreed with its owners first. |
| Per-page purpose, owner, decision | Partial | Columns are ready in the CSV. Traffic and owner need analytics and CMS access. |
| Last updated (sitemap `lastmod`) | Found | 2026: 3,100 · 2025: 855 · 2024: 601 · 2023: 293 · **2022 or earlier: 986 (17%)** · **untouched since 2017: 217**. Over half "changed" in 2026, which may reflect bulk template saves, so `lastmod` may not show real edits. |
| Duplicate or thin content | Partial | In the 80-page sample: **11 pages (14%) under 150 words**, for example individual ward profiles and short "customer service standard" pages. One duplicate title (`Nuit Blanche – City of Toronto` on 3 URLs that redirect to the same page). Median 461 words, max 7,309. |
| Media inventory | Partial | 138 PDF links on 27 of 80 sampled pages (34% of pages link PDFs), which projects to roughly 10,000 PDF links site-wide. Uploads live in `/wp-content/uploads/YYYY/MM/`. Banner JPEGs are 200–450 KB, with no WebP/AVIF. One homepage image is a PNG named `.jpg.png`. A full media inventory needs file-system or CMS access. |
| News archive | Found | `/news/` is live. RSS returns the latest 10 items. Not in the sitemap. Archive size needs CMS access. |
| Legal pages | Issue | Privacy (`/home/privacy/`, **last modified 2019-12-20**, 743 words, mentions cookies generically, names no vendors), Copyright (2026), Social Media Terms of Use (2022), 311 Terms of Use (2026), Statement of Commitment to Accessibility (2022), Digital Accessibility Standard (2026-04-15). **No cookie notice.** |

## 4. Functionality and integrations

| Item | Status | Finding |
|---|---|---|
| Forms | Found | Every page has the site search (GET to `find.toronto.ca/webui/search.html`) and a page-feedback form (`wp_feedback_form`, radio buttons, protected by **reCAPTCHA v3**). The 311 page embeds 3 reCAPTCHA instances. Service forms live on secure.toronto.ca. Where submissions go needs access. |
| Search, maps, calculators | Found | SearchBlox search with autocomplete. 256 map-related URLs (Road Restrictions Map and others). 19 calculator pages (property tax, Municipal Land Transfer Tax). The 311 page uses turf.js (geospatial). |
| Logins / accounts | Found | `/account/` (City of Toronto login with login help, a childcare account redirect, logout). Elections financial-filing login. The identity provider isn't visible without logging in. |
| Payments / booking | Found | 1,342 URLs match "pay/payment" (mostly the Services & Payments section). Toronto Public Health booking is on **FrontDesk Suite** (reservation.frontdesksuite.ca). 116 booking/registration URLs. The payment processor is on secure.toronto.ca and needs access. |
| Data feeds / APIs | Found | open.toronto.ca (CKAN open-data portal). The 311 knowledge-base API is on an internal-named host. |
| Third-party embeds | Found | Microsoft Bot Framework WebChat (311 "Chat Assistant"), Google Translate widget (all pages), Medallia surveys, social links (Facebook, X, Instagram, YouTube, LinkedIn). |
| Tracking scripts | Found | **Oracle Infinity** (Webtrends successor; account `97j62divdr`), with `WT.*` parameters on links and redirects. **No** Google Analytics, Tag Manager, or Meta Pixel. Search Console is verified. |
| Email signups | Found | `/home/social-media/e-updates/` (20 related pages), sending through Upaknee and Mailchimp, with an unsubscribe page. |
| Custom code to rebuild | Found | The `cot` theme, `cot-custom` plugin, COTUI component library, global-nav service (`/globalnav/*`), wp-feedback widget, the 311 app bundle (`feps_311` main 6.0.0, 528 KB), 311 widgets, and the issue-lookup app. |

## 5. Search and traffic data

| Item | Status | Finding |
|---|---|---|
| Top pages, sources, devices, conversions | Needs access | Lives in Oracle Infinity. Not public. |
| Keywords, rankings, indexing errors, penalties | Needs access | Search Console is set up (DNS verification token present). Export Performance, Pages and Manual Actions reports. |
| Backlinks | Needs access | Needs Search Console Links or a paid tool (Ahrefs/Semrush). |
| Redirects | Found / Issue | Apex and http → `https://www.toronto.ca/` (301, 2 hops from `http://toronto.ca`). Legacy short URLs work but are messy: `/311/` → `/?page_id=72&WT.rd_id=…` (an ugly intermediate URL with a tracking parameter); `/parks/` → `/explore-enjoy/parks-recreation/?WT.rd_id=/parks/`. `/search/` → find.toronto.ca. Old `/wps/portal/…` URLs (the pre-2017 portal) return 404. |
| Canonicals | Found | Self-referencing canonicals on all sampled 200 pages. The redirected samples point to their targets correctly. |
| robots.txt | Issue | Blocks `/wp-admin/` and about 85 legacy PDF paths, and declares the sitemap. **Blocks Bingbot from `/news/*`.** If that wasn't deliberate, it's a major visibility loss. |
| Sitemap | Issue | Generated daily (Google XML Sitemaps 4.1.21). Pages only, no news or PDFs. In the sample, 4 of 80 entries redirect (one goes off-site to a booking system and is `noindex`). Projected: about 290 bad entries site-wide. |

## 6. On-page SEO

Sample: 80 pages stratified by section, plus 6 pages inspected in depth.

| Item | Status | Finding |
|---|---|---|
| Title tags | Found | All present, following the pattern `Page – City of Toronto`. 16 of 80 exceed 60 characters. The homepage title is just "City of Toronto" (15 characters). |
| Meta descriptions | Issue | 6 of 80 missing (Contact Us and Council among them). 62 of 80 over 160 characters, because landing pages auto-fill with a list of child links ("Building & Construction Grants, Incentives & Rebates Licences & Permits…"). |
| Headings | Issue | Inner pages: exactly one H1 on 80 of 80. **The homepage H1 is empty.** Heading-level skips (H2 → H4 "Related Information" sidebar) on Contact, Council and 311. On the 311 page, H2s appear before the H1. |
| URL patterns | Found | Clean, lowercase, hyphenated, hierarchical (`/section/topic/sub-topic/page/`). Some are very long (up to 9 levels). No query strings or uppercase in the sitemap. |
| Structured data | Issue | Only `WebSite` JSON-LD. Missing: `GovernmentOrganization`, `BreadcrumbList`, `Event` (festivals), `FAQPage`, `SpecialAnnouncement`. |
| Image alt text | Found | 762 of 762 sampled images have an alt attribute (decorative ones use `alt=""`). |
| Internal linking | Found | About 93–119 internal links per page, mostly global navigation. Open Graph and Twitter cards are present. No `hreflang`: translation runs through the Google Translate widget, so translated content isn't indexable. |
| Google Business Profile / directories | Needs access | Check the City Hall and civic centre GBP listings and the Wikipedia/Wikidata links. |

## 7. Performance

Lighthouse 12.6.0 lab tests (simulated mobile = slow 4G with CPU slowdown; desktop preset), run from Toronto.

| Page | Device | Perf | A11y | Best pr. | SEO | LCP | FCP | TBT | CLS | Weight |
|---|---|---|---|---|---|---|---|---|---|---|
| Home | Mobile | **67** | 96 | 75 | 92 | **7.6 s** | 2.3 s | 90 ms | **0.147** | 2.0 MB |
| Home | Desktop | 95 | 100 | 78 | 92 | 1.5 s | 0.4 s | 0 ms | 0.05 | 2.0 MB |
| 311 | Mobile | **41** | 90 | 79 | 85 | **8.1 s** | 5.2 s | **730 ms** | 0.068 | **3.9 MB** |
| 311 | Desktop | 91 | 93 | 78 | 85 | 1.5 s | 0.9 s | 110 ms | 0.04 | 3.9 MB |
| Services & Payments | Mobile | 98 | 96 | 79 | 92 | 2.2 s | 1.4 s | 20 ms | 0 | 1.1 MB |
| Services & Payments | Desktop | 99 | 100 | 78 | 92 | 0.9 s | 0.4 s | 0 ms | 0.002 | 1.3 MB |

Field data (real-user Core Web Vitals from Chrome UX Report): **not collected.** The free PageSpeed Insights quota was exhausted on the day of testing (HTTP 429). Re-run with an API key or check Search Console's Core Web Vitals report. INP can only come from field data.

| Item | Status | Finding |
|---|---|---|
| Page weight, requests | Issue | Home about 2 MB / ~86 requests. 311 about 4 MB / ~114 requests. |
| Largest assets | Issue | reCAPTCHA JS 347 KB (largest file on the homepage; 272 KB unused), the 311 app bundle 528 KB, Font Awesome full JS 442 KB, turf.js 144 KB, Bot Framework WebChat (most of the 1.9 MB of unused JS on 311), banner JPEGs 200–450 KB. |
| Render-blocking | Issue | 4–10 blocking CSS/JS files per page (cot.css, wp-feedback CSS, Font Awesome CSS, js-cookie, jQuery stack). Estimated savings 410–600 ms. |
| Unused CSS/JS | Issue | Home: 272 KB JS / 66 KB CSS unused. 311: **1.9 MB JS** / 120 KB CSS unused. |
| Images | Issue | No WebP/AVIF (savings of about 530 KB on home), **0 lazy-loaded images** (368 KB of off-screen carousel images load up front), oversized images on every page. |
| Caching | Issue | 54–77 resources with short browser cache lifetimes (about 400 KB–2.8 MB re-downloaded on repeat visits). |
| Duplicate requests | Issue | `/globalnav/header/` and `/globalnav/footer/` are each fetched twice per page view. |
| TTFB | Found | Excellent: about 10–40 ms on a CloudFront hit, about 500 ms uncached median in the crawl. |
| Peer comparison (mobile home) | Found | **Toronto 67** · Ottawa 62 (LCP 10.0 s) · Vancouver 56 (LCP 12.5 s) · Montréal 46 (TBT 1.2 s). Ottawa leads best practices (96 vs Toronto 75). |

## 8. Accessibility and compatibility

| Item | Status | Finding |
|---|---|---|
| Automated scan (axe-core 4.10.2) | Found | Services & Payments and Community & People: **0 violations**. Home: no `main` landmark, duplicate `nav` landmark, 5 content regions outside landmarks. Contact and Council: heading order. **311: `meta-viewport` (critical, zoom disabled)**, heading order, content outside landmarks. Lighthouse adds tap targets under 24 px (footer links) and an accessible-name mismatch on the "Open Chat Assistant" button. |
| Keyboard pass | Found | First 8 Tab stops on each page are logical (skip link → search → text-size A+/A- → I want to… → Contact 311 → menu) and all show a visible focus ring. A full keyboard walkthrough of the 311 app and forms was not done. |
| Contrast, focus, labels | Found | No contrast failures from axe. Focus visible. All form fields labelled. |
| Screen-reader basics | Partial | Skip link present, `lang="en-CA"`, alt text good. Issues: empty homepage H1, missing `main` on home, unlabelled duplicate `nav`. No manual screen-reader test (VoiceOver/NVDA) yet. |
| Mobile | Partial | At 390 px: no horizontal scroll on home, services or contact. 4–9 tap targets under 24 px per page. Tested in Chrome mobile emulation, not on real devices. |
| Cross-browser | Needs access | Only Chrome tested. Safari, Firefox and Edge passes still to do. |
| Legal standard | Found | Ontario's AODA requires WCAG 2.0 AA for public-sector websites. The City publishes a Digital Accessibility Standard (updated 2026-04-15). |

## 9. Security and compliance

| Item | Status | Finding |
|---|---|---|
| HTTPS / mixed content | Found | All hosts redirect to HTTPS. **0 mixed-content requests** on 6 pages. HSTS `max-age=31536000; includeSubDomains` (not preloaded). TLS 1.2/1.3 only. |
| Outdated software | Issue | find.toronto.ca: **jQuery 2.2.4** (known XSS CVEs) and a 2017-era copy of the theme. Main site: Bootstrap 3.4.1 (EOL; CVE-2024-6484), jQuery 3.6.0 plus Migrate, WordPress 6.8.3 (behind the 7.x line), React 16 on 311. Server-side plugin versions need access. |
| Third-party script integrity | Issue | **No Subresource Integrity** on any script. `cdn.botframework.com/botframework-webchat/latest/webchat.js` is unpinned, so whatever Microsoft publishes runs on the 311 page. React comes from unpkg.com. |
| Security headers | Issue | Present: HSTS, `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `X-Permitted-Cross-Domain-Policies: none`. **CSP is report-only** and allows `'unsafe-inline' 'unsafe-eval' blob: https:`. Missing: `Referrer-Policy`, `Permissions-Policy`. `X-XSS-Protection` is obsolete (harmless). No `/.well-known/security.txt` (404), so there's no published channel for researchers to report vulnerabilities. |
| Admin hygiene (passwords, 2FA, unused accounts) | Needs access | The admin isn't publicly reachable (good). User list, 2FA and SSO need access. |
| Form validation | Partial | Feedback and 311 forms use reCAPTCHA v3. Server-side validation needs testing with permission; no injection testing was done. |
| Cookie consent / privacy law | Issue | No consent banner. `ORA_FPC` (Oracle analytics, 400-day persistent, not `Secure`) is set on the first visit, and `ORA_COOK_STORE` goes into localStorage. reCAPTCHA (on every page) and Google Translate send visitor data to Google. The privacy statement (2019) doesn't name Oracle, Google or Medallia. **Applicable law:** as an Ontario municipality, the City falls under **MFIPPA** (Ontario's municipal privacy law), not PIPEDA. CASL applies only to commercial electronic messages, so the e-updates are likely exempt. GDPR is relevant only if the City targets EU residents. Have the City's privacy office confirm. |
| Information leakage | Low | `x-custom-cot-info` build header, WordPress version in RSS and asset URLs, and the internal hostname `…ecrm.inter.prod-toronto.ca` called from the browser. |
| Personal data collected | Partial | Visible from outside: analytics IDs and click tracking (Oracle), feedback-form answers, reCAPTCHA signals (Google), 311 service requests (name, address, contact; via the 311 app), account logins, e-update emails (Upaknee/Mailchimp, which may store data outside Canada). Storage locations need access. |

## 10. Design and user experience

| Item | Status | Finding |
|---|---|---|
| Brand assets | Partial | Logo `98ba-toronto-logo.png` (PNG, not SVG), Roboto web fonts (missing `font-display`, so text is invisible while fonts load), Font Awesome icons, the COTUI design system 2.4.3. Brand guidelines need access. |
| Visual inconsistencies / outdated patterns | Issue | Bootstrap 3 grid, a homepage image carousel (low engagement and an accessibility cost), and page-level A+/A- text-size buttons (browsers do this natively). Search (find.toronto.ca) uses an older 2017 copy of the theme, so the look changes when users search. |
| Key user journeys | Partial | The main journeys are probably: find a service → pay or apply (secure.toronto.ca), report an issue (311), find contact info, check events. The heaviest and slowest page on mobile is the 311 hub, the city's main service entry point. Drop-off data needs Oracle Infinity access. |
| Heatmaps / recordings | Found | No Clarity or Hotjar. Medallia is linked (it can do surveys and possibly session replay). Ask whether replay is enabled. |
| Competitors | Found | See the Lighthouse peers in section 7. Ottawa scores higher on best practices (96). Montréal is accessibility 100 but slow (TBT 1.2 s). Toronto is the fastest of the four on mobile. |
| User feedback | Partial | The "Page Feedback" widget on every page collects found-it / satisfaction / issues. That dataset would be the best UX evidence, and it needs access. |

## 11. Business and project context

None of this can be determined from outside. Questions to put to City stakeholders:

1. What are the site's goals and success metrics (task completion, 311 call deflection, online-payment share, satisfaction from the feedback widget)?
2. Who are the target audiences? Inferred: residents, businesses, visitors, newcomers ("Moving to Toronto"), and the media.
3. What do stakeholders like, dislike, and consider must-keep (for example the COTUI design system and the 311 hub)?
4. How often is content updated, and by whom? The site has about 5,800 pages across many divisions. Is publishing decentralized?
5. What is the budget, timeline, and maintenance model (in-house vs vendor)?
6. Launch constraints: municipal elections (advance voting is under way now, October 2026), budget season, winter operations, and Nuit Blanche and other events.

## 12. Deliverables

| Deliverable | Status | Where |
|---|---|---|
| Content inventory spreadsheet | Started | `toronto_ca_content_inventory.csv`: 5,835 URLs with section, template guess, depth and lastmod; the 80 sampled rows have status, redirect, title, description length, H1, words and PDFs. Traffic, owner and decision columns are empty, waiting on analytics and CMS access. |
| Redirect map | Started | Known redirects are listed in section 5, and the 4 redirecting sitemap URLs are flagged in the CSV (`redirects_to`). A full old→new map needs the CMS redirect table (or the plugin's export) plus Search Console's top URLs. |
| Tech and integrations inventory | Done | Sections 2 and 4. |
| Baseline metrics snapshot | Partial | Lab performance and accessibility baseline in sections 7–8, dated 2026-10-17. Traffic, rankings and conversions need Oracle Infinity and Search Console exports *before* any launch. |
| Risk list | Done | Below. |

**Risk list**

| Risk | Type | Impact |
|---|---|---|
| Unpinned third-party chat script with no SRI on the 311 page, and a non-enforcing CSP | Unsupported / security | Supply-chain compromise of the busiest service page |
| jQuery 2.2.4 on site search; Bootstrap 3 EOL; WP 6.8 behind 7.x | Unsupported software | Known CVEs; harder upgrades later |
| DMARC `p=none` | Security | `@toronto.ca` can be spoofed in phishing |
| Privacy statement from 2019, no cookie notice, analytics cookie on first visit | Compliance | MFIPPA / IPC complaint risk; trust |
| No access yet to registrar, CMS, Oracle Infinity, Search Console, hosting, backups | Missing access | Blocks sections 1, 5, 11 and half of 12 |
| Google Translate widget instead of real translations | Data you can't migrate | No indexable multilingual content; Google may retire the free widget |
| Sitemap misses news and PDFs; Bingbot blocked from news | SEO | Lost discoverability |
| About 10k PDF links and the `/legdocs/` archive | Migration | Large and hard to inventory; many PDFs are likely not accessible |
| Two email platforms (Upaknee + Mailchimp) | Data | Subscriber lists split; consent records need to be merged |
| Custom code (theme, COTUI, globalnav, 311 app, wp-feedback) | Rebuild | Needs source access and an owner |

---

## Not covered by this audit

- **Not checked without the City's permission or accounts:** registrar, CMS and hosting logins, analytics, Search Console, backups, the user list, server-side plugin versions, contracts and licences.
- **Deliberately not done:** user enumeration, vulnerability or injection testing, and a full-site crawl. These need the City's permission.
- **Sample-based:** findings marked "Partial" come from 6 pages in depth and 80 in the crawl, not all 5,835. Projections ("about 290 bad sitemap entries", "about 10k PDF links") are rough extrapolations.
- **Not collected:** real-user Core Web Vitals (PageSpeed API quota exhausted); Safari, Firefox and Edge testing; real-device testing; a manual screen-reader pass.
