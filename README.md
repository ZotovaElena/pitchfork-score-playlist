# Pitchfork review toolkit

Scripts for turning Pitchfork's review history into filtered lists, charts, and
Spotify playlists. Python 3.10+.

```bash
pip install pandas python-dotenv spotipy tqdm pylast
```

## Environment setup (API keys and client IDs)

`fork_fetch.py` and `fork_wayback.py` need no credentials. The Last.fm genre
lookup and the Spotify playlist builder do. Both read their credentials from a
`.env` file in the project root (loaded with `python-dotenv`).

```dotenv
# Last.fm — used by fork_prepare.py / genre_annotator.py
LASTFM_API_KEY=
LASTFM_API_SECRET=
LASTFM_USERNAME=          # optional
LASTFM_PASSWORD=          # optional

# Spotify — used by fork_playlist.py
SPOTIFY_CLIENT_ID=
SPOTIFY_CLIENT_SECRET=
SPOTIFY_REDIRECT_URI=     # only needed for --push
```

| Variable | Required for | Where to get it |
|---|---|---|
| `LASTFM_API_KEY`, `LASTFM_API_SECRET` | `fork_prepare.py` genre-fill (skip with `--no-genre`) | Create an API account at <https://www.last.fm/api/account/create> |
| `LASTFM_USERNAME`, `LASTFM_PASSWORD` | Nothing for read-only tag lookups; optional | Your Last.fm login (the password is MD5-hashed before use) |
| `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET` | `fork_playlist.py` matching/preview | Create an app at <https://developer.spotify.com/dashboard> |
| `SPOTIFY_REDIRECT_URI` | `fork_playlist.py --push` | Set it in your Spotify app's settings, and use the exact same string in `.env` |

Notes:

- The Spotify redirect URI must match one registered in the app dashboard
  character for character (e.g. `http://127.0.0.1:8888/callback`).
- The names are the project's own `SPOTIFY_*` ones, not spotipy's default
  `SPOTIPY_*`; the code reads them explicitly.
- `--push` stores the OAuth login token in `.spotify_token_cache` (gitignored).
  Delete it to force a fresh login.

## Pipeline

```
fork_fetch.py  ──▶  fork_prepare.py  ──▶  fork_filter.py  ──▶  fork_playlist.py
(download + clean)  (dedupe + genre-fill) (select a subset)   (build a Spotify playlist)
```

```bash
python fork_fetch.py                          # -> data/fork_data_YYYY-MM-DD.tsv
python fork_prepare.py                        # -> data/fork_data_prepared_YYYY-MM-DD.tsv
python fork_filter.py --min-score 9 --new-releases --format playlist -o 9plus.txt
python fork_playlist.py filtered/fork_selection_....tsv --push --playlist-name "Pitchfork 9+"
```

Each stage reads the latest dated file the previous stage produced, so you
normally just re-run them in order. `fork_wayback.py` is a separate, optional
pipeline for recovering pre-1999 reviews (see below); `notebooks/01_eda_pitchfork.ipynb`
is for charts, not filtering.

## 1. fork_fetch.py — download and clean

```bash
python fork_fetch.py          # download + clean -> data/fork_data_YYYY-MM-DD.tsv
python fork_fetch.py --keep-raw   # also keep the downloaded raw JSON
python fork_fetch.py --offline    # re-clean without re-downloading
```

Source: [The Fork](https://github.com/olievans123/The-Fork) (MIT), which scraped
Pitchfork and enriched it with MusicBrainz and Wikidata country/language data.
Underlying reviews are Pitchfork's; this is for personal/offline analysis.

Re-run this any time to pull newly published reviews — it writes a fresh dated
file rather than overwriting the previous one.

## 2. fork_prepare.py — dedupe and fill in genres

```bash
python fork_prepare.py                  # dedupe + genre-fill -> data/fork_data_prepared_YYYY-MM-DD.tsv
python fork_prepare.py --no-genre       # dedupe + format only, skip Last.fm
python fork_prepare.py --limit-genre 50 # only annotate the first N missing-genre rows
python fork_prepare.py --dry-run        # report counts, write nothing, no API calls
```

Reads the latest raw `data/fork_data_*.tsv` (never modifies it) and:

1. **dedupe** — one row per album (highest score, then earliest review)
2. **genre** — fills missing `genre`/`genres` via Last.fm (`genre_annotator.py`),
   falling back to artist-level tags, then buckets Last.fm's free-text tags onto
   Pitchfork's own 9 genre names
3. **format** — consistent column order/dtypes, sorted by `review_date`/`artist`;
   any row still without a genre is tagged `Untagged`

Genre lookups are cached in `lastfm_genre_cache.json`, so re-runs only hit the
Last.fm API for rows not already resolved (needs `LASTFM_API_KEY` /
`LASTFM_API_SECRET` in `.env`).

## 3. fork_filter.py — filter and export a selection

Every run saves the full filtered selection to
`filtered/fork_selection_<filters>.tsv` for downstream use (playlist building,
further analysis), in addition to whatever `--format`/`-o` you asked for.

```bash
# The 9.0+ club — albums that scored 9 or higher when they came out
python fork_filter.py --min-score 9 --new-releases --dedupe --format playlist -o 9plus.txt

# Every 10.0, reissue re-reviews included
python fork_filter.py --min-score 10

# Best New Music in the 2020s, electronic only
python fork_filter.py --bnm --genre Electronic --review-year 2020-2029

# Three highest-scoring new albums of each year — evenly spread, unlike a score cut
python fork_filter.py --new-releases --top-per-year 3 --format playlist -o top3.txt

# High scores from Japan / Brazil / Nigeria
python fork_filter.py --min-score 8.5 --country JP --sort score

# Follow one critic's taste
python fork_filter.py --reviewer "Jenn Pelly" --min-score 8

# Off the beaten track: strong score, no genre tag, never badged Best New Music
python fork_filter.py --min-score 8.5 --no-genre --no-bnm

# Spanish-language albums, 8.0+
python fork_filter.py --min-score 8 --language spa --sort score
```

### Exploring before you filter

```bash
python fork_filter.py --list genres           # what genre names exist
python fork_filter.py --list countries        # country codes, by count
python fork_filter.py --list reviewers
python fork_filter.py --list languages

# --list respects the other filters, so you can drill in
python fork_filter.py --min-score 9 --list countries
```

For charts and distributions (score by year, genre breakdowns, BNM trends), use
`notebooks/01_eda_pitchfork.ipynb` — `fork_filter.py` has no `--stats` flag.

### All the options

**Data** — `--data path.tsv` (default: the latest `data/fork_data_*.tsv`)

**Score** — `--min-score 9`, `--max-score 6.5`

**Dates** — `--review-year` (when Pitchfork reviewed it), `--release-year` (when the
album came out). Both accept `2015`, `2010-2019` or `1999,2003,2011`.

**Review type** — `--new-releases`, `--reissues`, `--bnm`, `--no-bnm`, `--bnr`

**Metadata** — `--genre` (repeatable), `--exclude-genre` (repeatable), `--no-genre`,
`--country` (repeatable), `--language` (repeatable), `--artist`, `--reviewer`,
`--search` (matches artist, album, the blurb and the reviewer)

**Shaping** — `--dedupe` (one row per album, keeping the highest score),
`--top-per-year N`, `--top-per-release-year N`, `--limit`, `--sort`
(`score`, `score-asc`, `date`, `date-desc`, `release`, `artist`, `random` — with
`--seed` for a repeatable shuffle)

**Output** — `--format table|playlist|csv|tsv|markdown`, `-o file`, `-q` (quiet)

Filters combine with AND, so `--genre Rap --genre Jazz` means "rap or jazz", while
`--genre Jazz --country US` means both must hold.

## 4. fork_playlist.py — turn a selection into a Spotify playlist

One track per album: the album's single most-popular track on Spotify.

```bash
# match + preview only — prints/saves the track list, touches nothing on your
# Spotify account (catalog search only needs app credentials, no login)
python fork_playlist.py filtered/fork_selection_min9_max9.9_new.tsv

# same matching (cached — no repeat API calls), then actually create and fill
# the playlist (needs a one-time browser login, playlist-modify scope)
python fork_playlist.py filtered/fork_selection_....tsv --push --playlist-name "Pitchfork 9+"

# add to an existing playlist instead of creating a new one
python fork_playlist.py filtered/fork_selection_....tsv --push --playlist-id 37i9dQZF1...

# only push albums whose title matched exactly (skip the "fuzzy" tier)
python fork_playlist.py filtered/fork_selection_....tsv --push --playlist-name "..." --exact-only
```

Two caveats:

- Spotify's API never exposes real play counts. The closest thing available is
  each track's "popularity" (0–100, Spotify's own relative ranking) — that's what
  "most reproduced" means here, not literal streams.
- Matching Pitchfork album titles to Spotify's catalog is fuzzy (reissue
  suffixes, messy titles) and won't be perfect. Unmatched or low-confidence
  albums are reported in the output TSV, never silently dropped.

Matches are cached in `spotify_match_cache.json`. Output defaults to
`playlists/<selection>_spotify.tsv`.

Requires `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET` in `.env` for catalog
search (preview mode), plus `SPOTIFY_REDIRECT_URI` for `--push`. On a headless
server, `--push` prints a Spotify login URL to open in a browser on your own
machine; the redirect page itself will fail to load (expected) — copy the URL
you land on back into the terminal prompt. The redirect URI must match one
registered in your Spotify app's dashboard exactly.

## 5. fork_wayback.py — recover the missing 1996–1998 reviews

Pitchfork's own site no longer hosts 1996–1998 reviews: no sitemap, no
per-review URL, none of the machine-readable metadata the fetcher relies on.
What survives is Internet Archive captures of hand-written monthly review
index pages, in whatever layout the site used that month — so this is a
pipeline with a measurable hit rate, run one stage at a time, not a one-shot
scraper:

```bash
python fork_wayback.py discover        # ask Wayback's CDX index what exists
python fork_wayback.py fetch           # download those captures into a cache
python fork_wayback.py inspect <n>     # look at one capture, design a rule
python fork_wayback.py parse           # run every extractor, score the result
python fork_wayback.py validate        # measure against known-good data
python fork_wayback.py merge           # fold accepted rows into fork_data_*.tsv
python fork_wayback.py selftest        # exercise the extractors on synthetic fixtures
```

Stages are resumable and cache to disk, so `parse` can be re-run freely without
re-downloading. The extractors are hypotheses, not a finished parser — `inspect`
and `validate` make their hit rate visible so you can add or fix one (about ten
lines each). Be polite to the Internet Archive: the default delay is 2s between
requests and everything is cached; don't lower it much.

## Fields in the prepared dataset

| Field | Notes |
|---|---|
| `id` | stable identifier from the source dataset |
| `artist`, `album` | HTML entities decoded; compilations are "Various Artists" |
| `score` | 0.0–10.0 |
| `review_date`, `review_year` | when Pitchfork published the review |
| `release_year` | when the album came out |
| `original_year` | set only on reissues of older albums |
| `genres`, `genre` | list, plus the first one for convenience (Last.fm-filled where Pitchfork had none, see `fork_prepare.py`) |
| `reviewer` | critic's name |
| `bnm`, `bnr` | Best New Music / Best New Reissue |
| `is_new_release`, `is_reissue` | derived, see below |
| `country`, `language` | artist origin, from MusicBrainz and Wikidata |
| `mbid` | MusicBrainz identifier, where matched |
| `blurb` | Pitchfork's one-line summary |
| `pitchfork_url`, `image` | link to the review, link to the cover art |
| `_key` | dedupe key used by `fork_prepare.py` |

## Things to know before trusting a count

**Coverage starts in January 1999** in the base dataset. Pitchfork's 1996–1998
reviews aren't included unless you've run `fork_wayback.py merge`, and those
years are exactly when it handed out 9.x scores most freely. Any "from the
beginning" list built without the Wayback merge silently starts in 1999.

**`is_new_release` is a heuristic**, not a Pitchfork field. It means: not flagged
Best New Reissue, no `original_year`, and the release year is within a year of the
review date. It gets the overwhelming majority right, but archival releases dated
to the year they were *issued* can slip through — a live box set of 1971 recordings
released in 1999 counts as a 1999 new release.

**One album can have several reviews** in the raw dataset — reissues get
re-reviewed, sometimes with a different score (*Homogenic* was 9.9 in 1997 and
10.0 in 2017). `fork_prepare.py` already dedupes to one row per album (highest
score, then earliest review); use `--dedupe` in `fork_filter.py` too if you're
filtering the raw, unprepared TSV directly.

**Genre coverage isn't uniform.** Pitchfork's own `genre` field is missing on a
meaningful slice of reviews; `fork_prepare.py` backfills most of these from
Last.fm, but rows Last.fm has no tags for either end up as `Untagged`.
`--no-genre` in `fork_filter.py` selects exactly that leftover set — a usable
"weird stuff" filter, but its size depends on how much genre-filling you've run.

**Scores drifted.** Pitchfork was far more generous early on: the mid-2000s saw
many more new albums at 9.0+ per year than recent years typically do. A fixed
score threshold therefore gives you a list dominated by one era.
`--top-per-year` is the fix if you want even coverage.

## Refreshing

```bash
python fork_fetch.py     # pull newly published reviews -> new dated data/fork_data_*.tsv
python fork_prepare.py   # re-dedupe + fill genres -> new dated data/fork_data_prepared_*.tsv
```

`fork_filter.py` and `fork_playlist.py` always pick up the latest dated file
automatically unless you pass `--data`/a selection path explicitly.
