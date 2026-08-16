---
layout: post
title: Unlighthouse - Website Performance & SEO Audit 
subtitle: Scanning every page on a site for Lighthouse scores using a free, npm-based tool
tags: [tools, seo, cli, performance, unlighthouse]
author: Anil Thapa
---

A quick reference for auditing every page on a website (performance, SEO, accessibility, best practices) using Unlighthouse — a free, open-source, npm-based crawler built on Google Lighthouse.

## Tools Used

- **[Unlighthouse](https://unlighthouse.dev/)** (installed via [npm](https://www.npmjs.com/package/unlighthouse)) — free, open-source, no page cap. Crawls a site and runs a full Lighthouse audit on every page found.
- **Node.js 18+** — required runtime.

## Setup

No install needed — run directly with `npx`:

```bash
npx unlighthouse --site example.com
```

Or install as a dev dependency in a project:

```bash
npm install -D unlighthouse
npx unlighthouse --version
```

Unlighthouse official docs: [https://unlighthouse.dev/guide/](https://unlighthouse.dev/guide/)

## Running a Scan

```bash
npx unlighthouse --site "https://example.com"
```

This opens a live dashboard at `http://localhost:5678` showing scores for every discovered page as they're crawled.

**Flags used:**

| Flag | Purpose |
|---|---|
| `--site <url>` | Site to scan |
| `--desktop` | Audit in desktop mode (default is mobile) |
| `--debug` | Verbose output (see live crawl progress) |
| `--build-static` | Export a static HTML report instead of the live dashboard |
| `--port <number>` | Custom dashboard port (default 5678) |

**Notes:**
- No hard cap on number of pages — crawls until the whole site (or sitemap) is covered.
- Discovers pages via crawling internal links or `sitemap.xml` if one is published.
- Respects `robots.txt` by default.
- Larger sites take longer; each page runs a full Lighthouse pass, which is slower than a plain link crawl.

## CI / Non-Interactive Mode

For a one-off scan without the live dashboard (useful for reports or pipelines):

```bash
npx unlighthouse-ci --site example.com --build-static
```

This writes a static HTML report to disk instead of keeping a server running.

## Reading Results

Each page gets four scores (0–100): **Performance**, **Accessibility**, **Best Practices**, **SEO**. The dashboard lets you:
- Sort/filter by score, path, or issue type
- Drill into a single page for the full Lighthouse breakdown
- Export the whole run as CSV/HTML via `--build-static`

## Sample Results

| Metric | Count / Score |
|---|---|
| Pages scanned | 424 |
| Avg. Performance score | 78 |
| Avg. SEO score | 92 |
| Avg. Accessibility score | 85 |
| Pages below 50 (Performance) | 12 |

## Other Free Options (No Sign-Up)

| Tool | Platform | Notes | Link |
|---|---|---|---|
| **Unlighthouse** | Any (npm) | Free, unlimited, full-site Lighthouse audit with dashboard | [unlighthouse.dev](https://unlighthouse.dev/) |
| **Lighthouse CLI** | Any (npm) | Free, single-page only, no crawling | [github.com/GoogleChrome/lighthouse](https://github.com/GoogleChrome/lighthouse) |
| **PageSpeed Insights** | Web | Free, single-page only, uses Lighthouse under the hood | [pagespeed.web.dev](https://pagespeed.web.dev/) |
| **Screaming Frog SEO Spider** | Win/Mac/Linux | Free tier capped at 500 URLs; has its own SEO checks, no Lighthouse scoring | [screamingfrog.co.uk](https://www.screamingfrog.co.uk/seo-spider/) |
| **Sitespeed.io** | Any (npm) | Free, unlimited, alternative full-site performance crawler | [sitespeed.io](https://www.sitespeed.io/) |

## Caveats

- A site's [`robots.txt`](https://www.robotstxt.org/robotstxt.html) can block the crawler from certain paths — results may undercount if the site restricts crawling.
- Lighthouse scores can vary run-to-run depending on network/CPU conditions; treat single runs as indicative, not exact.
- Auditing every page runs real page loads against the target — avoid running large, unthrottled scans against sites you don't own or manage.
- JS-heavy SPAs may need extra wait time for full rendering before Lighthouse audits accurately.

## Further Reading

- Unlighthouse guide: [https://unlighthouse.dev/guide/](https://unlighthouse.dev/guide/)
- Unlighthouse config reference: [https://unlighthouse.dev/api/config](https://unlighthouse.dev/api/config)
- Lighthouse scoring docs: [https://developer.chrome.com/docs/lighthouse/overview](https://developer.chrome.com/docs/lighthouse/overview)
- robots.txt spec: [https://www.robotstxt.org/robotstxt.html](https://www.robotstxt.org/robotstxt.html)
