---
layout: post
title: Linkinator - Broken Link Audit
subtitle: Scan a whole site for broken links, filter just the 404s, and decode every status code
tags: [tools, seo, cli, linkinator]
author: Anil Thapa
---

A quick reference for finding broken links on any website (or local build) using a free, open-source command line tool, then slicing the report by status code.

> **How to use this page:** every command you need to copy is marked with **▶ RUN** above its code block. Skim for those.

## Tools Used

- **[Linkinator](https://github.com/JustinBeckwith/linkinator)** (installed via [npm](https://www.npmjs.com/package/linkinator)) — free, open-source, no URL cap. Crawls a site and checks every link, image, script, and stylesheet.
- **[jq](https://jqlang.github.io/jq/)** (installed via [Homebrew](https://formulae.brew.sh/formula/jq)) — command line JSON filter, used to pull out only the status codes you care about.

## Setup

**▶ RUN** install Linkinator (needs [Node.js](https://nodejs.org/) LTS):

```bash
node -v
npm install -g linkinator
linkinator --help
```

**▶ RUN** install jq (macOS):

```bash
brew install jq
jq --version
```

No global install? Prefix any command with `npx`, for example `npx linkinator https://example.com --recurse`.

## Running a Scan

**▶ RUN** scan the whole site and save a full JSON report:

```bash
mkdir ~/link-audit
cd ~/link-audit
linkinator https://example.com --recurse --retry --concurrency 10 --timeout 20000 --format json > report.json
```

**Flags used:**

| Flag | Purpose |
|---|---|
| `--recurse` | Follow every internal page, not just the start URL |
| `--format json` | Machine-readable report (use `csv` for Excel/Sheets) |
| `--retry` | Retry when a server answers 429 (rate limit) |
| `--concurrency 10` | Number of simultaneous requests (be gentle on the server) |
| `--timeout 20000` | Wait 20 seconds before giving up on a link |
| `--verbosity error` *(optional)* | Only print broken links in the terminal |
| `--check-fragments` *(optional)* | Also verify `#anchor` links |
| `--skip "regex"` *(optional)* | Ignore URLs matching a pattern |

**Notes:**
- Only pages on the starting domain are crawled. External links are checked once but not followed.
- Sites built with JavaScript (single-page apps) may be under-scanned, because Linkinator reads the raw HTML.
- Always get permission before scanning a site you don't own or manage.

## Choose Your View

### Option 1: Only 404s (Not Found)

**▶ RUN** list every 404 as readable JSON:

```bash
jq '.links[] | select(.status==404)' report.json
```

**▶ RUN** 404s as a CSV (url, status, parent page):

```bash
jq -r '["url","status","parent"], (.links[] | select(.status==404) | [.url,.status,.parent]) | @csv' report.json > 404-only.csv
```

**▶ RUN** count the 404s:

```bash
jq '[.links[] | select(.status==404)] | length' report.json
```

**▶ RUN** quick version straight from the terminal, no JSON needed:

```bash
linkinator https://example.com --recurse --verbosity error | grep "\[404\]"
```

### Option 2: All other broken links (everything except 404)

**▶ RUN** every broken link that is not a 404:

```bash
jq '.links[] | select(.state=="BROKEN" and .status!=404)' report.json
```

**▶ RUN** the same, saved as CSV:

```bash
jq -r '["url","status","parent"], (.links[] | select(.state=="BROKEN" and .status!=404) | [.url,.status,.parent]) | @csv' report.json > other-broken.csv
```

### Option 3: All broken links (404 and everything else)

**▶ RUN**

```bash
jq '.links[] | select(.state=="BROKEN")' report.json
```

**▶ RUN** broken links saved as CSV:

```bash
jq -r '["url","status","parent"], (.links[] | select(.state=="BROKEN") | [.url,.status,.parent]) | @csv' report.json > all-broken.csv
```

### Option 4: One specific status code

Change the number to any code from the table below.

**▶ RUN**

```bash
jq '.links[] | select(.status==500)' report.json
```

### Summary: how many of each code?

**▶ RUN** broken-link count grouped by status code:

```bash
jq '[.links[] | select(.state=="BROKEN")] | group_by(.status) | map({status: .[0].status, count: length})' report.json
```

## ⭐ Download All Broken Links to Excel

> **📥 Want a spreadsheet of every broken link? Run these two commands.** The result is a file you can open in Excel, Numbers, or Google Sheets, with one row per broken link and the page it lives on.

**▶ RUN** Step 1: scan the site and save the report (skip if you already have `report.json`):

```bash
linkinator https://example.com --recurse --retry --concurrency 10 --timeout 20000 --format json > report.json
```

**▶ RUN** Step 2: export **all broken links** (404 and every other code) to an Excel-ready file:

```bash
printf '\xEF\xBB\xBF' > broken-links.csv
jq -r '["Status","Broken URL","Found on page"], (.links[] | select(.state=="BROKEN") | [.status,.url,.parent]) | @csv' report.json >> broken-links.csv
```

The first line writes a UTF-8 marker so Excel displays special characters correctly. The second line appends the data.

**▶ RUN** Step 3: open it in Excel (macOS):

```bash
open -a "Microsoft Excel" broken-links.csv
```

On Windows, double-click `broken-links.csv`.

### Want a real `.xlsx` file instead?

**▶ RUN** one-time install:

```bash
pip3 install pandas openpyxl
```

**▶ RUN** convert the report into a sorted `.xlsx` with all broken links:

```bash
python3 -c "
import json, pandas as pd
d = json.load(open('report.json'))
rows = [{'Status': l['status'], 'Broken URL': l['url'], 'Found on page': l.get('parent','')} for l in d['links'] if l['state']=='BROKEN']
pd.DataFrame(rows).sort_values(['Status','Found on page']).to_excel('broken-links.xlsx', index=False)
print(len(rows), 'broken links saved to broken-links.xlsx')
"
```

**Tips:**
- Sort or filter the **Status** column in Excel to view only 404s, or everything else.
- The **Found on page** column tells you where to go and fix each link.
- Only the **Status 0** rows have no real HTTP code, so check those URLs for typos.

## Reading the Report

Each entry in the report looks like this:

```json
{
  "url": "https://example.com/old-page",
  "status": 404,
  "state": "BROKEN",
  "parent": "https://example.com/blog/some-post",
  "failureDetails": []
}
```

| Field | Meaning |
|---|---|
| `url` | The link that failed |
| `status` | The HTTP status code (or `0` if no response at all) |
| `state` | `OK`, `BROKEN`, or `SKIPPED` |
| `parent` | The page that contains the broken link — **this is where you fix it** |
| `failureDetails` | The underlying error, when available (JSON output only) |

## What Each Status Code Means

Linkinator treats `2xx` as OK and follows `3xx` redirects. Anything in the `4xx`/`5xx` range, or `0`, is reported as **BROKEN**.

| Code | Name | What it means | What to do |
|---|---|---|---|
| **0** | No response | The request failed before any server replied: DNS failure, refused connection, timeout, SSL error, or a malformed URL | Check the URL for typos (a missing `/` is common). Check `failureDetails` for `ENOTFOUND`, `ETIMEDOUT`, `ECONNREFUSED`, or a certificate error |
| **200** | OK | Link works | Nothing |
| **301** | Moved Permanently | Redirects to a new URL permanently | Update the link to the new URL (not broken, but worth cleaning up) |
| **302 / 307** | Temporary Redirect | Redirects for now | Usually fine; confirm the destination is right |
| **400** | Bad Request | The server couldn't understand the URL | Look for illegal characters, spaces, or a malformed query string |
| **401** | Unauthorized | Login required | Expected for members-only pages; skip them or ignore |
| **403** | Forbidden | Server refuses access, or is blocking scanners | Open it in a browser. If it loads, it's bot protection, so add the domain to `--skip` |
| **404** | Not Found | The page or file doesn't exist | Fix the link, restore the page, or add a redirect. **The most common broken link** |
| **405** | Method Not Allowed | The server rejects the type of request used | Usually a false alarm; check in a browser |
| **410** | Gone | The page was removed on purpose | Remove the link or point it somewhere current |
| **429** | Too Many Requests | You're being rate limited | Lower `--concurrency`, keep `--retry`, and rescan |
| **500** | Internal Server Error | The remote server crashed on this request | Retry later; if it persists, tell the site owner |
| **502** | Bad Gateway | A proxy or upstream server returned a bad response | Retry later |
| **503** | Service Unavailable | Server is down or overloaded | Retry later |
| **504** | Gateway Timeout | An upstream server took too long | Retry later or raise `--timeout` |
| **999** | Non-standard | LinkedIn and some other sites use this to block bots | Add to `--skip` |

**Rule of thumb:** `4xx` usually means *the link is wrong* (fix it). `5xx` usually means *the other server has a problem* (retry before acting). `0` means *nothing answered at all* (check the URL itself).

## Skipping False Positives

Some sites block scanners and return false errors. Skip them:

**▶ RUN**

```bash
linkinator https://example.com --recurse --skip "linkedin\.com|facebook\.com|instagram\.com|^mailto:|^tel:" --format json > report.json
```

## Saving Your Settings

Put this in `linkinator.config.json` so you don't retype flags:

```json
{
  "recurse": true,
  "retry": true,
  "concurrency": 10,
  "timeout": 20000,
  "format": "json",
  "skip": ["linkedin\\.com", "facebook\\.com", "instagram\\.com", "^mailto:", "^tel:"]
}
```

**▶ RUN**

```bash
linkinator https://example.com --config linkinator.config.json > report.json
```

## Scanning a Local Site

**▶ RUN** a running dev server:

```bash
linkinator http://localhost:3000 --recurse --format json > report.json
```

**▶ RUN** a static build folder (Linkinator starts a temporary server for you):

```bash
linkinator ./public --recurse --format json > report.json
```

**▶ RUN** a folder of Markdown files:

```bash
linkinator ./docs --recurse --markdown --format json > report.json
```

## Quick Cheat Sheet

| Goal | Command |
|---|---|
| Install | `npm install -g linkinator` |
| Full scan to JSON | `linkinator https://example.com --recurse --format json > report.json` |
| Only 404s | `jq '.links[] \| select(.status==404)' report.json` |
| All broken except 404 | `jq '.links[] \| select(.state=="BROKEN" and .status!=404)' report.json` |
| All broken | `jq '.links[] \| select(.state=="BROKEN")' report.json` |
| All broken to Excel | `printf '\xEF\xBB\xBF' > broken-links.csv && jq -r '["Status","Broken URL","Found on page"], (.links[] \| select(.state=="BROKEN") \| [.status,.url,.parent]) \| @csv' report.json >> broken-links.csv` |
| Count by status | `jq '[.links[] \| select(.state=="BROKEN")] \| group_by(.status) \| map({status: .[0].status, count: length})' report.json` |

## Other Free Options (No Sign-Up)

| Tool | Platform | Notes | Link |
|---|---|---|---|
| **Linkinator** | Win/Mac/Linux | Free, no cap, JSON/CSV output, easy to script | [github.com/JustinBeckwith/linkinator](https://github.com/JustinBeckwith/linkinator) |
| **Xenu's Link Sleuth** | Windows | Free, no cap, dated UI | [home.snafu.de/tilman/xenulink.html](https://home.snafu.de/tilman/xenulink.html) |
| **Screaming Frog SEO Spider** | Win/Mac/Linux | Free tier capped at 500 URLs per crawl; polished UI | [screamingfrog.co.uk](https://www.screamingfrog.co.uk/seo-spider/) |
| **HTTrack** | Win/Mac/Linux | Mirrors a site locally for file counts, not link checking | [httrack.com](https://www.httrack.com/) |

## Caveats

- Linkinator has no built-in "only show 404" flag, which is why this guide filters the JSON report with `jq`.
- CSV output is convenient for spreadsheets, but the `failureDetails` column is often blank. Use JSON when you need the underlying error.
- A `403` or `429` on an external site is often bot protection, not a genuinely broken link. Confirm in a browser before fixing.
- Large sites take a while. Start with low concurrency and raise it if the server handles it.
- Always get permission before running a scan against a site you don't own or manage.

## Further Reading

- Linkinator repository and options: [https://github.com/JustinBeckwith/linkinator](https://github.com/JustinBeckwith/linkinator)
- jq manual: [https://jqlang.github.io/jq/manual/](https://jqlang.github.io/jq/manual/)
- HTTP status codes (MDN): [https://developer.mozilla.org/en-US/docs/Web/HTTP/Status](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
