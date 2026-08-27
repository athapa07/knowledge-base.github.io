---
layout: post
title: Unlighthouse - Extracting Issues & Building Client Reports -- 2
subtitle: Scanning by category, pulling failures into a spreadsheet, and reporting scores without confusing your client
tags: [tools, seo, cli, performance, accessibility, unlighthouse]
author: Anil Thapa
---

Follow-up to [Part 1](https://athapa07.github.io/knowledge-base.github.io/2026-08-17-unlighthouse-website-audit/), which covered installing Unlighthouse and running a basic full-site scan. This post covers the workflow for turning a scan into something you can actually hand to a client: scanning one category at a time, extracting only the failing issues into a CSV, checking page counts and scores, and writing up the results without the report looking self-contradictory.

## Tools Used

- **[Unlighthouse](https://unlighthouse.dev/)** (installed globally or via npx) — same tool as Part 1.
- **Node.js 18+** — required runtime, also used to run the extraction scripts below.

## Scanning One Category at a Time

By default Unlighthouse scans all four categories (performance, accessibility, best practices, SEO) on every page. Restricting a scan to a single category speeds up the crawl and keeps the JSON output focused — but there's no CLI flag for it. It has to go in a config file, via `lighthouseOptions.onlyCategories`.

**`unlighthouse.config.js`:**

```js
export default {
  site: 'example.com',
  scanner: { device: 'desktop' },
  lighthouseOptions: {
    onlyCategories: ['accessibility'], // swap per category, see table below
  },
  ci: { buildStatic: true },
}
```

| Category | `onlyCategories` value |
|---|---|
| Accessibility | `['accessibility']` |
| Best Practices | `['best-practices']` |
| SEO | `['seo']` |
| Performance | `['performance']` |

**Setup notes:**
- Keep each category in its **own folder** (e.g. `~/unlighthouse-accessibility`, `~/unlighthouse-seo`) — each scan writes to a local `.unlighthouse/` folder, and running a new category from the same folder overwrites the previous scan's output.
- Run from inside that folder so the config file is auto-detected:

```bash
cd ~/unlighthouse-accessibility
unlighthouse-ci
```

- Performance scans take noticeably longer per page than the others, since Lighthouse has to actually load and time each page rather than run static checks.

## Where the Results Live

Each scan writes a `.unlighthouse/` folder containing:
- A static HTML report (from `buildStatic: true`)
- Per-route JSON reports with the full Lighthouse result for every crawled page

Filenames vary by Unlighthouse version — confirm what you've got before writing an extraction script:

```bash
find .unlighthouse -name "*.json"
```

In testing, the per-page files were named `lighthouse.json`, nested under `.unlighthouse/reports/<route>/`.

## Extracting Failures Into a Spreadsheet

A raw Lighthouse report includes every audit, passes included — clients don't need to see those. What matters is anything scored below 1 (a full pass).

**`extract-issues.js`** — save in the scan folder, run with `node extract-issues.js`:

```js
import fs from 'fs';
import path from 'path';

const REPORTS_DIR = '.unlighthouse';
const CATEGORY = 'accessibility'; // 'accessibility' | 'best-practices' | 'seo' | 'performance'
const OUTPUT_FILE = 'accessibility-issues.csv';

const issuesByAudit = {};
let filesProcessed = 0;

function walk(dir) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) walk(full);
    else if (entry.name === 'lighthouse.json') processReport(full);
  }
}

function processReport(filePath) {
  const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
  const lhResult = data.report || data.lhr || data;
  const url = lhResult.finalUrl || lhResult.requestedUrl || filePath;
  const audits = lhResult.audits || {};
  const categoryRefs = lhResult.categories?.[CATEGORY]?.auditRefs || [];

  if (categoryRefs.length === 0) {
    console.warn(`No ${CATEGORY} auditRefs found in ${filePath}`);
    return;
  }

  filesProcessed++;

  for (const ref of categoryRefs) {
    const audit = audits[ref.id];
    if (!audit || audit.score === 1 || audit.score === null) continue;

    if (!issuesByAudit[ref.id]) {
      issuesByAudit[ref.id] = {
        title: audit.title,
        description: audit.description,
        impact: audit.score === 0 ? 'Fail' : 'Partial',
        pages: [],
      };
    }
    const displayValue = audit.displayValue ? ` (${audit.displayValue})` : '';
    issuesByAudit[ref.id].pages.push(url + displayValue);
  }
}

walk(REPORTS_DIR);

const rows = [['Issue', 'Severity', 'Description', 'Affected Pages', 'Page Count']];
for (const [id, issue] of Object.entries(issuesByAudit)) {
  rows.push([
    issue.title,
    issue.impact,
    issue.description.replace(/\n/g, ' '),
    issue.pages.join(' | '),
    issue.pages.length,
  ]);
}

const csv = rows.map(r => r.map(c => `"${String(c).replace(/"/g, '""')}"`).join(',')).join('\n');
fs.writeFileSync(OUTPUT_FILE, csv);
console.log(`Processed ${filesProcessed} report files.`);
console.log(`Found ${Object.keys(issuesByAudit).length} distinct ${CATEGORY} issues.`);
```

Only two lines change per category:

| Category | `CATEGORY` | `OUTPUT_FILE` |
|---|---|---|
| Accessibility | `'accessibility'` | `accessibility-issues.csv` |
| Best Practices | `'best-practices'` | `best-practices-issues.csv` |
| SEO | `'seo'` | `seo-issues.csv` |
| Performance | `'performance'` | `performance-issues.csv` |

**`audit.displayValue`** is included next to each URL — mostly useful for performance audits (e.g. "3.2 s", "1.8 MB"), a harmless no-op for the other categories.

**Troubleshooting "Found 0 distinct issues":**

| Check | Command |
|---|---|
| Confirm report filenames match the script | `find .unlighthouse -name "*.json"` |
| Inspect one file's actual JSON shape | `cat .unlighthouse/reports/lighthouse.json \| node -e "const d=JSON.parse(require('fs').readFileSync(0,'utf-8')); console.log(Object.keys(d)); console.log(d.categories ? Object.keys(d.categories) : 'no categories key at top level')"` |
| Confirm the config's `onlyCategories` matches what you're extracting | `cat unlighthouse.config.js` |

## Checking Page Counts and Scores

**Pages scanned:**

```bash
find .unlighthouse/reports -name "lighthouse.json" | wc -l
```

Run from inside the specific category's scan folder — each has its own `.unlighthouse` directory.

**Single page's score:**

```bash
cat .unlighthouse/reports/lighthouse.json | node -e "const d=JSON.parse(require('fs').readFileSync(0,'utf-8')); const r=d.report||d.lhr||d; console.log(r.categories.seo.score * 100)"
```

Swap `seo` for `accessibility`, `performance`, or `['best-practices']` (brackets needed for the hyphenated key).

**Site-wide average + worst pages — `average-score.js`:**

```js
import fs from 'fs';
import path from 'path';

const REPORTS_DIR = '.unlighthouse';
const CATEGORY = 'seo'; // change per category

let scores = [];

function walk(dir) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) walk(full);
    else if (entry.name === 'lighthouse.json') {
      const data = JSON.parse(fs.readFileSync(full, 'utf-8'));
      const lhResult = data.report || data.lhr || data;
      const score = lhResult.categories?.[CATEGORY]?.score;
      if (score !== null && score !== undefined) {
        scores.push({ url: lhResult.finalUrl || full, score: score * 100 });
      }
    }
  }
}

walk(REPORTS_DIR);

const avg = scores.reduce((sum, s) => sum + s.score, 0) / scores.length;
console.log(`Pages scored: ${scores.length}`);
console.log(`Average ${CATEGORY} score: ${avg.toFixed(1)}`);

const lowest = [...scores].sort((a, b) => a.score - b.score).slice(0, 5);
console.log(`\nLowest scoring pages:`);
lowest.forEach(s => console.log(`  ${s.score.toFixed(0)} — ${s.url}`));
```

Performance scores vary far more across pages than SEO or accessibility — a heavy image- or PDF-loaded page can score very differently from a lightweight text page. A wide spread here is normal, not a bug.

## Why a "100" Score Can Still Show Failing Issues

The single most common point of confusion when reading these reports.

The score and the issue list answer **two different questions**:

| | Question answered |
|---|---|
| Category score | "Averaged across all checks and all pages, how well did the site do overall?" |
| Issue list | "Are there any specific things wrong, anywhere?" |

Each category is made up of ~10–15 individual audits, each carrying its own weight in a weighted average. A failing audit with low weight — say, missing meta descriptions on a handful of pages — barely moves the aggregate score, especially when averaged across dozens or hundreds of pages where most content passes everything else. It's the same reason a student can score 98% on a test and still have gotten two questions wrong.

**Reporting implication:** leading a client report with a big "100" and listing failures underneath reads as self-contradictory even though it isn't. Fix this by either (a) explicitly noting the score is a weighted average, not a pass/fail gate, or (b) not leading with the score at all — lead with the issue count instead, and put the aggregate score further down as context.

## Client Report Template

```
## [Category] Assessment

A total of [N] pages were scanned for [category] health. The assessment
identified [X] distinct issues, affecting a combined [Y] pages out of the
[N] scanned. Details for each are below, followed by the overall aggregate
score for context.

### Issues identified

**1. [Issue title]**
- Severity: Fail / Partial
- Affected pages: [count]

[Plain-language explanation of what this audit checks and why it matters.]

Affected pages:
- [url]
- [url]

### Recommendations

- [Concrete fix for issue 1]
- [Concrete fix for issue 2]

### Aggregate score (for context)

The site's overall [category] score, averaged across all [N] pages and all
technical checks, was [score]. This reflects [strong/moderate] fundamentals
site-wide — [note on why the score doesn't fully capture the affected pages].
```

For issues affecting a large number of pages, keep only the top 3–5 example URLs inline ("and 13 more — see appendix") and move the full list to an appendix or attached spreadsheet, so the main report stays skimmable in a client meeting.

## Sample Results

| Category | Pages Scanned | Distinct Issues Found | Aggregate Score |
|---|---|---|---|
| SEO | 92 | 2 (20 pages affected) | 100 |
| Accessibility | 92 | 4 | — |
| Best Practices | — | — | — |
| Performance | — | — | — |

## Quick Reference — All Commands

```bash
# Set up a category-specific scan folder
mkdir ~/unlighthouse-<category> && cd ~/unlighthouse-<category>
# (save unlighthouse.config.js with the right onlyCategories here)

# Run the scan
unlighthouse-ci

# Confirm report filenames
find .unlighthouse -name "*.json"

# Count pages scanned
find .unlighthouse/reports -name "lighthouse.json" | wc -l

# Extract failing issues to CSV
node extract-issues.js

# Check site-wide average score + worst pages
node average-score.js

# Check one page's score directly
cat .unlighthouse/reports/lighthouse.json | node -e "const d=JSON.parse(require('fs').readFileSync(0,'utf-8')); const r=d.report||d.lhr||d; console.log(r.categories.seo.score * 100)"
```
## Working File
[Downlaodable setup](https://athapa07.github.io/knowledge-base.github.io/assets/files/unlighhouse.zip)


## Caveats

- File names (`lighthouse.json` vs `report.json` vs `payload.json`) and JSON nesting can differ between Unlighthouse versions — always confirm with `find` and a manual `cat` inspection before trusting a script's output of "0 issues."
- A high aggregate score does not mean zero issues — always check the issue list, not just the score, before telling a client a category is "clean."
- Node may print an ES module warning if `package.json` doesn't declare `"type": "module"` in the scan folder — harmless, but easy to silence.

## Further Reading

- Unlighthouse config reference: [https://unlighthouse.dev/api/config](https://unlighthouse.dev/api/config)
- Lighthouse scoring methodology: [https://developer.chrome.com/docs/lighthouse/performance/performance-scoring/](https://developer.chrome.com/docs/lighthouse/performance/performance-scoring/)
- Lighthouse SEO audits reference: [https://developer.chrome.com/docs/lighthouse/seo/](https://developer.chrome.com/docs/lighthouse/seo/)