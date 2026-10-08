# prism-star-maps

Daily, automatically updated maps of where the stargazers of every
[prism-oncology](https://github.com/prism-oncology) repository are — one map per
repo plus an org-wide map — published on GitHub Pages.

```
GitHub (gh api graphql)  ──►  stargazers + profile location
         │
         ▼
OpenStreetMap Nominatim  ──►  lat/lon, city, country   (cached in data/geocode-cache.json)
         │
         ▼
data/stats/<repo>.json   ──►  site/<repo>.html  (Leaflet map, top countries, top places)
                              site/index.html   (org-wide map + repo cards)
```

## How it works

| Step | Where |
|---|---|
| 1. List public org repos and their stargazers (`login`, `location`) in one GraphQL query per 100 stargazers | `scripts/fetch_stargazers.py` |
| 2. Normalise free-text locations and geocode new ones with Nominatim (1 req/s, cached forever, misses cached as `null`) | `scripts/fetch_stargazers.py` |
| 3. Aggregate per repo: place → count, country → count; plus a de-duplicated org-wide map | `data/stats/*.json`, `data/summary.json` |
| 4. Render a static site and deploy it to GitHub Pages | `scripts/build_site.py`, `.github/workflows/update-maps.yml` |

The workflow runs every day at 04:17 UTC, on manual dispatch, and whenever the
scripts change. No Python dependencies — only the standard library and `gh`
(pre-installed on GitHub runners).

## Setup

1. **Create the repo** `prism-oncology/prism-star-maps` and push this folder.
2. **Create a token** (see below) and add it as the repository secret
   `STARGAZERS_TOKEN` (*Settings → Secrets and variables → Actions*).
3. **Enable Pages**: *Settings → Pages → Source: GitHub Actions*.
4. Run the workflow once from the *Actions* tab (*Update star maps → Run workflow*).

The site will be at `https://prism-oncology.github.io/prism-star-maps/`.

### Token

Since **July 2026 GitHub only lets a repository's admins and collaborators list
its stargazers** (anti-scraping change to `/repos/{owner}/{repo}/stargazers`
and the matching GraphQL connection). The workflow's default `GITHUB_TOKEN` is
scoped to this repo only, so it can't read the other repos' stargazers.

Options, best first:

- **GitHub App owned by the org** (no personal account involved) installed on
  all `prism-oncology` repos; mint a token in the workflow with
  [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
  and pass it as `GH_TOKEN`. There is no dedicated "stargazers: read"
  permission; third-party tools report needing *Contents* access — start with
  read-only and raise it only if the API returns *Not Found*.
- **Fine-grained PAT** from an org admin, resource owner `prism-oncology`,
  *All repositories*, same permissions as above (the org may need to approve it).
- **Classic PAT** from an org admin/collaborator with the `public_repo` scope.

If a repo can't be read, the run doesn't fail: that repo keeps its previous map
and shows a warning on its page.

## Privacy

GitHub restricted stargazer lists precisely because they were being scraped, so
by default **only aggregated counts are published** (e.g. "Lyon, France — 3").
No usernames are written to the repo or the site. Locations are what users
chose to put on their public profile, geocoded to city level.

Set `INCLUDE_LOGINS: "true"` in the workflow to attach `@usernames` to map
popups — think twice before doing this on a public site.

## Configuration

Environment variables of `fetch_stargazers.py` (set in the workflow):

| Variable | Default | Meaning |
|---|---|---|
| `ORG` | `prism-oncology` | Organisation to scan |
| `INCLUDE_FORKS` | `false` | Map forked repos too |
| `INCLUDE_ARCHIVED` | `true` | Map archived repos too |
| `INCLUDE_LOGINS` | `false` | Show usernames in map popups |
| `MAX_GEOCODE` | `800` | New Nominatim lookups per run (≈1.1 s each); the rest is done on the next run |

## Run locally

```bash
export GH_TOKEN=...            # or just `gh auth login`
python scripts/fetch_stargazers.py
python scripts/build_site.py
python -m http.server -d site 8000   # open http://localhost:8000
```

## Notes

- Typically 40–60 % of stargazers have a location, and most of those geocode;
  joke locations ("Earth", "localhost") are cached as misses.
- Nominatim's [usage policy](https://operations.osmfoundation.org/policies/nominatim/)
  asks for ≤1 request/s and an identifying User-Agent; both are respected, and
  the cache means a daily run usually makes only a handful of new lookups.
- Map tiles: Esri World Gray Canvas (no API key needed), © OpenStreetMap contributors.
