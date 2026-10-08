# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A two-script pipeline that maps where the stargazers of every public `prism-oncology` repo are located, and publishes one Leaflet map per repo plus an org-wide map to GitHub Pages. It runs daily in CI (`.github/workflows/update-maps.yml`, 04:17 UTC, on manual dispatch, and on pushes to `main` that touch `scripts/**` or the workflow).

## Commands

Uses the standard library only (Python 3.12 in CI). It has no dependencies, tests, linter or build tool.

```bash
export GH_TOKEN=...                      # or `gh auth login`; must be admin/collaborator on the org's repos
python scripts/fetch_stargazers.py       # GitHub + Nominatim -> data/
python scripts/build_site.py             # data/ -> site/ (no network)
python -m http.server -d site 8000
```

`build_site.py` needs `data/summary.json` and `data/stats/_all.json`, and only `fetch_stargazers.py` creates them. To work on the site without a token, write those JSON files by hand in the schema that `fetch_stargazers.py` produces.

Fetch is configured through env vars: `ORG`, `INCLUDE_FORKS`, `INCLUDE_ARCHIVED`, `INCLUDE_LOGINS`, `MAX_GEOCODE`. The README lists their defaults.

## Architecture

**Stage 1: `scripts/fetch_stargazers.py`**
- Calls GitHub through the `gh` CLI (`gh api graphql --paginate --jq ...`), not through an HTTP library. `gh_graphql_paginate` retries transient failures. It raises `PermissionError` on NOT_FOUND/FORBIDDEN/403, and the caller then marks that repo with an `error` and skips it instead of failing the run.
- Locations are normalised by `normalise_location`: emoji are stripped, whitespace is collapsed, and the text is lowercased. The normalised string is the key into `data/geocode-cache.json`.
- Geocoding goes through Nominatim at 1.1 s per request with an identifying User-Agent, which the OSM usage policy requires; do not remove either. Cache entries never expire, and misses are cached as `null` so they are not retried. Network errors are *not* cached, so they are retried on the next run. Each run does at most `MAX_GEOCODE` new lookups.
- `aggregate()` collapses users into `places` (keyed by rounded lat/lon) and `countries`. The org-wide `_all.json` uses the union of logins across repos, so a user who starred several repos is counted once.
- Outputs: `data/stats/<repo>.json`, `data/stats/_all.json` and `data/summary.json`. When a repo can't be read, its previous stats are kept and `stale_since` is set. Stats files for repos that no longer exist are deleted.

**Stage 2: `scripts/build_site.py`**
- Deletes and rebuilds `site/` (gitignored) from `data/`. The HTML, CSS (`CSS`) and map JS (`MAP_JS`, filled in with `%` formatting, so a literal `%` in that JS must be written `%%`) are inline Python strings. Leaflet loads from unpkg and the tiles come from CARTO, with the light or dark style chosen from `prefers-color-scheme`.
- Every value interpolated into the HTML goes through `esc()`. Place data is embedded as JSON with `</` escaped, and popup text is escaped again in JS.

**CI**: the workflow commits the refreshed `data/` back to `main` as `github-actions[bot]`, then builds the site and deploys it with `actions/deploy-pages`. Committed data changes don't re-trigger the workflow, because the push trigger only matches `scripts/**` and the workflow file.

## Constraints

- **Token**: since July 2026 GitHub lets only a repo's admins and collaborators list its stargazers. The default `GITHUB_TOKEN` can't do it, so CI uses the `STARGAZERS_TOKEN` secret (a GitHub App token or an org admin's PAT; see the README).
- **Privacy**: by default only aggregated counts are written to disk and to the site. Usernames are stored only when `INCLUDE_LOGINS=true`. Keep login data out of `data/` and `site/` unless that flag is set.
