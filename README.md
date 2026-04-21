# common-crawl-ranks-cache

Daily-polled mirror of [Common Crawl](https://commoncrawl.org) host- and domain-level web-graph ranks, Top-1M per release, produced by [`wangmm001/feedcache`](https://github.com/wangmm001/feedcache).

Common Crawl publishes a new web-graph roughly every quarter; this repo's daily cron therefore commits new files ~4 times a year. On all other days the run short-circuits without a commit.

## Layout

```
data/
├── host/
│   ├── YYYY-MM-DD_<release-id>.csv.gz     # e.g. 2026-04-21_cc-main-2026-jan-feb-mar.csv.gz
│   ├── current.csv.gz                     # pointer to the newest snapshot
│   └── current.release.txt                # plain-text: release id of current.csv.gz
├── domain/
│   └── …same shape as host/
└── graphinfo.json                         # verbatim copy of upstream release index, refreshed daily
```

## Format

Gzipped CSV, header-first, comma-separated. Schemas differ slightly per level:

**host/** — 5 columns:
```
rank,harmonicc_val,pr_pos,pr_val,host
1,3.7549092E7,5,0.004897432273872421,www.facebook.com
…
```

**domain/** — 6 columns (the upstream file carries an extra `n_hosts` column):
```
rank,harmonicc_val,pr_pos,pr_val,domain,n_hosts
1,3.053703E7,3,0.009072779578696339,facebook.com,3356
…
```

| Column | Meaning |
|---|---|
| `rank` | Position in upstream `#harmonicc_pos`, 1 through 1,000,000. |
| `harmonicc_val` | Harmonic Centrality score (scientific notation). |
| `pr_pos` | PageRank position for the same entity. |
| `pr_val` | PageRank score. |
| `host` / `domain` | Forward-order host/domain; upstream stores reverse-domain form, this mirror un-reverses it. |
| `n_hosts` | *(domain only)* Number of distinct hosts upstream aggregated into this registrable domain. |

Rows sorted by `rank` ascending.

## Consume

```bash
# Latest top-100 hosts by harmonic centrality
curl -sL https://raw.githubusercontent.com/wangmm001/common-crawl-ranks-cache/main/data/host/current.csv.gz \
  | zcat | head -101

# Which Common Crawl release is "current"?
curl -sL https://raw.githubusercontent.com/wangmm001/common-crawl-ranks-cache/main/data/host/current.release.txt
```

## Known simplifications

- Only the Top-1M entries by Harmonic Centrality are kept; full upstream ranks (~288M hosts, ~134M domains per release) are not mirrored. See the feedcache design doc for rationale.
- The `domain` granularity here is Common Crawl's own reverse-domain-merge, not Public Suffix List eTLD+1. These usually agree but disagree around user-content public suffixes such as `github.io`.
- No historical backfill: the repo starts with whatever release is current at the first cron run.

## License

Scaffolding: MIT. Data: subject to [Common Crawl's Terms of Use](https://commoncrawl.org/terms-of-use).

## How it works

Daily GitHub Actions cron (04:30 UTC) calls `wangmm001/feedcache`'s reusable workflow with `source: common-crawl-ranks`. The step `pip install`s feedcache at `main`, runs `feedcache common-crawl-ranks data/`, and commits any new files. Deterministic gzip + `git diff --cached --quiet` means no commit on unchanged days.
