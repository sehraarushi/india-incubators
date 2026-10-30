# India's Top 20 Startup Incubators

This repository contains the Part 2 deliverables: a Python 3 scraper that collects the **homepage** and the **About Us page** of each of the Top 20 Indian startup incubators identified in Part 1, in two forms:

|Source|What was saved|Frequency|
|-|-|-|
|**Current (live)**|The live version of each page, as of the run date|Once|
|**Archived (Wayback Machine)**|The **earliest available capture in each calendar month**|Monthly, **January 2022 to September 2026** (57 months)|

\---

## 1\. Repository contents

|File|Description|
|-|-|
|`scrape\_incubators.py`|Annotated Python 3 scraper (the main code)|
|`check\_results.py`|Produces the per-incubator coverage summary from the manifest|
|`incubators.csv`|Input list: the Top 20 incubators and their website URLs|
|`requirements.txt`|Python libraries needed (`requests`, `beautifulsoup4`)|
|`output\_part1.zip`, `output\_part2.zip`, `output\_part3.zip`|**The scraped HTML files.** Unzip all three into the same folder to get `live/` and `archive/`|
|`manifest.csv`|One row per incubator, page and month attempted: capture timestamp, status, file size, SHA-256 hash, source URL|
|`coverage\_summary.csv`|How many months were captured for each incubator|
|Part 1 files|Methodology, Top 20 ranking (PDF), 48-incubator dataset and spreadsheet|
|`Incubatorhtml.ipynb`|Jupyter notebook used to launch the run (a run log, not the code itself)|

After unzipping the three zip files, the folder structure is:

```
live/<incubator>/home\_<YYYYMMDD>.html
live/<incubator>/about\_<YYYYMMDD>.html
archive/<incubator>/<YYYY-MM>/home\_<timestamp>.html
archive/<incubator>/<YYYY-MM>/about\_<timestamp>.html
```

`<timestamp>` is the Internet Archive capture time in `YYYYMMDDhhmmss` format.

\---

## 2\. Scraping frequency

For each incubator, the script saved the **earliest available archived capture of the homepage and of the About Us page in every calendar month**, from September 2026 back to January 2022 (57 months), plus the current live version of both pages. The exact capture timestamp of every file is recorded in `manifest.csv` (column `snapshot\_timestamp`).

If the Internet Archive holds no capture for a page in a given month, that month is recorded as `no\_capture`. Months are never filled with a neighbouring month's capture, so every file's date is exactly the date stated.

\---

## 3\. How the scraper works

1. **Find captures.** For each website, the script queries the Internet Archive's CDX index with `collapse=timestamp:6`, which keeps one capture per `YYYYMM`. Results are time-ordered, so the one kept is the earliest in that month. Only successful (HTTP 200) HTML captures are used. The index treats `http`/`https`, `www`/non-`www` and trailing slashes as the same page.
2. **Download the original HTML.** Each capture is requested with the Wayback `id\_` URL flag, which returns the page as originally served, without the Archive's toolbar or rewritten links. Bytes are saved untouched, so original encoding is preserved.
3. **Find the About page.** Website menus change over time, so the script reads the links in each archived homepage and scores them: link text such as "About Us" or "Who we are" ranks above a URL that merely contains `/about`; links such as "About the program", blog, news, careers and contact are excluded. It then fetches that URL's capture from the **same month**. If a month's homepage has no capture, it falls back to the last About URL found, then to common paths (`/about-us`, `/about`, `/who-we-are`, and so on).
4. **Fetch the live version** using the same About-link discovery.
5. **Log everything** to `manifest.csv`. The run is resumable: stopped runs can be restarted and already-saved pages are skipped.

Requests are throttled (1.5 seconds by default) and retried with back-off when the Archive rate-limits.

\---

## 4\. How to run it

Requires Python 3.9 or newer.

```bash
pip install -r requirements.txt
python scrape\_incubators.py --input incubators.csv --out output
python check\_results.py output
```

Options:

|Flag|Default|Meaning|
|-|-|-|
|`--input`|`incubators.csv`|CSV with columns `name,website`|
|`--out`|`output`|Output folder|
|`--start` / `--end`|`2022-01` / `2026-09`|Month range, inclusive|
|`--delay`|`1.5`|Seconds between requests|
|`--timeout`|`60`|Seconds to wait per request|
|`--only`|all|Comma-separated incubator slugs, e.g. `t-hub,c-camp`|
|`--skip-live` / `--skip-archive`|off|Run only one half|
|`-v`|off|Debug logging|

In a Jupyter notebook, run commands with `!{sys.executable} scrape\_incubators.py ...` after `import sys`.

\---

## 5\. Manifest columns

`incubator, slug, page\_type (home | about), source (live | archive), month, requested\_url, snapshot\_timestamp, fetched\_url, status, http\_code, bytes, sha256, file, note, scraped\_at\_utc`

`status` values:

* `ok`: saved
* `no\_capture`: the Internet Archive has no capture of that page in that month (a gap in the archive, not an error)
* `fetch\_failed`: a capture exists but the download failed after 4 retries

The `sha256` column shows which months have identical page content.

\---

## 6\. Results of this run

* Run date: \[30 October 2026]
* Incubators scraped: 20 (listed in `incubators.csv`)
* Period covered: January 2022 to September 2026 (57 months)
* HTML files saved: \[1022]
* Per-incubator coverage: see `coverage\_summary.csv`
* URLs corrected from the original list: \[none]

\---

## 7\. Limitations

* **Archive gaps.** Coverage depends on how often the Internet Archive visited each site. Smaller or newer sites, and About pages in particular, have more `no\_capture` months.
* **Saved files are HTML source only.** Stylesheets, scripts and images are not saved, so opening a file directly from disk can look blank or unstyled. The text content of the page is inside the file.
* **JavaScript-built sites.** Sites that load their content with JavaScript (React, Vue and similar) may contain little text in the saved HTML, in both live and archived versions. This is a limit of the source, not of the run.
* **The About date can differ from the homepage date** in the same month, since each is the earliest capture of its own URL.
* **Domain changes.** An incubator that moved to a new domain is tracked only under the URL in `incubators.csv`.
* **Live-site certificates.** Some institutional sites have broken SSL certificates. For those, the script retries without certificate verification.
* **Re-runs** skip successful pages but log `no\_capture` months again, so `manifest.csv` can hold duplicate `no\_capture` rows. To remove them, keep the last row per `(slug, page\_type, month)`; `check\_results.py` already does this.
* **Responsible use.** The scraper identifies itself with a User-Agent and throttles its requests. Check each site's terms before redistributing the scraped content.

