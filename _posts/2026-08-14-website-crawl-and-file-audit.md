---
layout: post
title: Website Crawl & File Audit
subtitle: Counting pages, PDFs, and images on any site with free, unlimited tools
tags: [tools, seo, cli]
author: Anil Thapa
---

A quick reference for counting pages, PDFs, and images on any public website using free, unlimited tools.

## Tools Used

- **[HTTrack](https://www.httrack.com/)** (installed via [Homebrew](https://formulae.brew.sh/formula/httrack)) — free, open-source, no page/URL cap. Mirrors an entire website locally.
- **`find` + `wc`** — standard macOS/Linux CLI tools ([GNU findutils](https://www.gnu.org/software/findutils/)) to count files by type after the crawl.

## Setup

```bash
brew install httrack
httrack --version
```

HTTrack official docs: [https://www.httrack.com/html/fcguide.html](https://www.httrack.com/html/fcguide.html)

## Running a Crawl

```bash
mkdir ~/site-crawl
cd ~/site-crawl
httrack "https://example.com" -O ~/site-crawl -%e0 -v
```

**Flags used:**

| Flag | Purpose |
|---|---|
| `-O ~/site-crawl` | Output directory |
| `-%e0` | Stay on the same domain (don't follow external links) |
| `-v` | Verbose output (see live progress) |
| `-r5` *(optional)* | Limit crawl depth to 5 levels |

**Notes:**
- No hard cap on number of pages — crawls until the whole site is mirrored.
- Respects `robots.txt` by default.
- Larger sites take longer and use more disk space.

## Counting Results

Basic counts:

```bash
find ~/site-crawl -iname "*.html" | wc -l   # HTML pages
find ~/site-crawl -iname "*.pdf" | wc -l    # PDFs
find ~/site-crawl -iname "*.doc" | wc -l    # DOC
find ~/site-crawl -iname "*.docx" | wc -l   # DOCX
```

Better combined image count (catches all common formats in one pass):

```bash
find ~/site-crawl \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.gif" -o -iname "*.svg" -o -iname "*.webp" \) | wc -l
```

Total file count (any type):

```bash
find ~/site-crawl -type f | wc -l
```

Full breakdown of every file extension present, ranked by count:

```bash
find ~/site-crawl -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
```

## Sample Results

| Type | Count |
|---|---|
| HTML pages | 424 |
| PDFs | 6 |
| JPG images | 534 |
| JPEG images | 30 |
| PNG images | 253 |
| Total images (jpg + jpeg + png + gif) | 819 |
| DOC/DOCX | 0 |

## Other Free Options (No Sign-Up)

| Tool | Platform | Notes | Link |
|---|---|---|---|
| **HTTrack** | Win/Mac/Linux | Free, no cap, mirrors full site locally | [httrack.com](https://www.httrack.com/) |
| **Xenu's Link Sleuth** | Windows | Free, no cap, link/page/image list, dated UI | [home.snafu.de/tilman/xenulink.html](https://home.snafu.de/tilman/xenulink.html) |
| **Screaming Frog SEO Spider** | Win/Mac/Linux | Free tier capped at 500 URLs per crawl; polished UI | [screamingfrog.co.uk](https://www.screamingfrog.co.uk/seo-spider/) |
| **Scrapy** | Any (Python) | Free, unlimited, requires scripting | [scrapy.org](https://scrapy.org/) |
| `sitemap.xml` check | Any | Instant, free, only works if site publishes one | Append `/sitemap.xml` to any domain |
| `site:domain.com` on Google | Any | Rough indexed-page estimate, not a real crawl | [google.com/search?q=site:example.com](https://www.google.com/search?q=site:example.com) |

## Caveats

- A site's [`robots.txt`](https://www.robotstxt.org/robotstxt.html) can block crawlers from certain paths — results may undercount if the site restricts crawling.
- Multiple image sizes/thumbnails for the same photo can inflate image counts — worth spot-checking a folder.
- Always get permission before running an aggressive crawl against a site you don't own or manage.

## Further Reading

- HTTrack manual: [https://www.httrack.com/html/fcguide.html](https://www.httrack.com/html/fcguide.html)
- `find` command reference (GNU): [https://www.gnu.org/software/findutils/manual/html_mono/find.html](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
- robots.txt spec: [https://www.robotstxt.org/robotstxt.html](https://www.robotstxt.org/robotstxt.html)
- Screaming Frog SEO Spider docs: [https://www.screamingfrog.co.uk/seo-spider/user-guide/](https://www.screamingfrog.co.uk/seo-spider/user-guide/)