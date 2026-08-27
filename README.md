# SPARKS Crosswalk Data

**SPARKS (Sports Property and Reference Knowledge System)** is an open registry of *sports properties* — sports, leagues, franchises, and venues — mapping each one to a stable SPARKS ID and to the identifiers used by external data providers.

Sports data is full of the same entity wearing different names and numbers in every source. Wikidata calls Fenway Park `Q49136`, the MLB StatsAPI calls it `3`, ESPN calls it `2`. SPARKS gives it one stable id, `6h0z5tye`, and records the rest.

This repository is the **source of truth**: hand-edited CSV files and their schemas. Everything else in the ecosystem is derived from it.

## Repo Topology

SPARKS is split across three repositories, with a one-way flow:

```
sparks-crosswalk-data     hand-edited CSV + schema  ← you are here
        │  push to main touching data/ or schema/
        │  → repository_dispatch "crosswalk-data-updated"
        ▼
sparks-crosswalk          generated dist/ artifacts, never hand-edited
        │
        ▼
   pages branch           Astro site serving dist/ as read-only JSON endpoints
```

- **[sparks-crosswalk-data](https://github.com/statsvine/sparks-crosswalk-data)** (this repo) — the authoritative CSV data and schema files. Edited by hand, via pull request.
- **[sparks-crosswalk](https://github.com/statsvine/sparks-crosswalk)** — build artifacts only (`dist/`), plus the Astro site on its `pages` branch. **Never edit this by hand**; it is rebuilt from this repo on every change.
- **[sparks-tools](https://github.com/statsvine/sparks-tools)** — the shared validators and build scripts that CI in both repos runs.

**Why two repos?** Separation of concerns. Keeping hand-edited source apart from generated artifacts means `git log` and `git blame` on the data stay readable rather than being buried under automated build commits, and downstream consumers read the derived artifacts rather than the raw CSV.

## Datasets

| File | Rows | Coverage |
| --- | --- | --- |
| [`data/sports.csv`](data/sports.csv) | 5 | Top-level sports |
| [`data/leagues.csv`](data/leagues.csv) | 7 | Leagues, including historical ones, with lineage |
| [`data/franchises.csv`](data/franchises.csv) | 45 | MLB franchises, active and historical, with lineage |
| [`data/venues.csv`](data/venues.csv) | 103 | Stadiums and arenas across MLB/NBA/NFL/NHL and some international sites |

Each CSV has a matching schema in [`schema/`](schema/).

### sports.csv

Baseball, basketball, football, ice hockey, soccer.

`sparks_id`, `label`, `wikidata_uri`, `iptc_media_topic_uri`, `espn_api_uri`, `reuters_sport_slug`

### leagues.csv

`sparks_id`, `sport_id`, `label`, `abbreviation`, `wikidata_uri`, `espn_api_uri`, `reuters_league_slug`, `year_founded`, `year_ceased`, `parent_league`, `superseded_by`

The two relationship fields express different things, and the test is temporal:

- **`parent_league` is concurrent containment** — the child still exists, inside the parent. The AL and NL sit under MLB.
- **`superseded_by` is sequential replacement** — the predecessor stopped existing. The 1920 APFA is superseded by the NFL.

Note the parent is *younger* than its children here (MLB 1903, AL 1901, NL 1876), because MLB was created in 1903 as an umbrella over two leagues that both continued. "Parent founded before child" is therefore not a valid invariant.

The AL and NL stopped being independent legal entities around 2000 and now function as conferences within MLB. **That distinction is deliberately not modelled.** Every provider SPARKS maps to still treats them as live leagues, and the franchise data depends on it: two franchise records exist purely to capture AL↔NL moves — the 1998 Brewers and the 2013 Astros — and both would be reduced to `mlb → mlb` if the unification were treated as a supersession.

### franchises.csv

`sparks_id`, `sport_id`, `league_id`, `label`, `location_label`, `nickname_label`, `abbreviation`, `wikidata_uri`, `status`, `year_founded`, `year_ceased`, `root_franchise`, `superseded_by`, `superseded_reason`

Franchise history is modelled explicitly rather than flattened. `status` is one of `active`, `defunct`, or `superseded`; `superseded_reason` is one of `relocation`, `realignment`, `defunct`, `contracted`, or `merger`. `root_franchise` points back to the first franchise in the lineage (and may be self-referential), so the Dodgers chain resolves as:

```
nl_bkn_1890  Brooklyn Dodgers      superseded 1890–1957  relocation
    └─ nl_lad_1958  Los Angeles Dodgers   active 1958–     root: nl_bkn_1890
```

The name is also split into parts — `label` is `location_label` + `nickname_label` — so consumers can render "Dodgers" or "Los Angeles" without string surgery.

### venues.csv

`sparks_id`, `founding_name`, `current_name`, `latitude_dd`, `longitude_dd`, `country_code`, `subdivision_code`, `city`, `wikidata_id`, `status`, `year_founded`, `year_inactive`, `year_demolished`, `mlb_id`, `espn_id`

`country_code` is ISO 3166-1 alpha-2, `subdivision_code` is ISO 3166-2. `status` is one of `active`, `inactive`, or `demolished`. Venues are named twice on purpose: `founding_name` never changes, while `current_name` tracks the sponsor of the moment — the Phoenix arena is `America West Arena` at inception and `Mortgage Matchup Center` today.

## ID Conventions

`sparks_id` is the stable primary key, always lowercase. Its shape varies by dataset:

| Dataset | Form | Example |
| --- | --- | --- |
| Sports, leagues | Readable slug | `baseball`, `mlb`, `nl` |
| Franchises | `league_city_year` | `al_nyy_1903`, `nl_bkn_1890` |
| Venues | Opaque 8-character slug | `05njz9zt`, `6h0z5tye` |

Franchise ids encode the founding year because franchises relocate and realign — `al_mil_1901` (the Brewers who became the St. Louis Browns) and `al_mil_1970` (the Brewers who came from Seattle) are genuinely different franchises that share a city.

Venue ids are deliberately opaque. Venues get renamed constantly, and any readable slug would be obsolete within a few seasons.

## External Identifiers

| Source | Field | Datasets |
| --- | --- | --- |
| [Wikidata](https://www.wikidata.org) | `wikidata_uri`, `wikidata_id` | all |
| [IPTC Media Topics](https://cv.iptc.org/newscodes/mediatopic/) | `iptc_media_topic_uri` | sports |
| ESPN core API | `espn_api_uri`, `espn_id` | sports, leagues, venues |
| [Reuters sports slugs](https://liaison.reuters.com/tools/sports-slugs) | `reuters_sport_slug`, `reuters_league_slug` | sports, leagues |
| MLB StatsAPI | `mlb_id` | venues |

## Validation

Every dataset is validated in CI. Each `.github/workflows/validate_<name>.yml` runs on push and pull request when its CSV changes, and calls the reusable `validate.yml`, which clones `sparks-tools` and runs:

```bash
python3 tools/crosswalk/validate_csv.py data/<name>.csv --schema schema/<name>.yaml --fail-fast
```

To run the same check locally, clone [sparks-tools](https://github.com/statsvine/sparks-tools) alongside this repo:

```bash
python3 ../sparks-tools/crosswalk/validate_csv.py data/venues.csv --schema schema/venues.yaml
```

### Schema format

Schemas are YAML, one entry per field, declaring:

- `type` — `string`, `integer`, `decimal`, `enum`, or `reference`
- `required` — whether the field may be empty
- `unique` — whether values must be unique across the file
- `unique_within` — narrows that uniqueness to rows sharing the listed fields
- `pattern` — a regex the value must match
- `enum` — the permitted values, for `enum` fields
- `reference_file` / `reference_column` — for `reference` fields, the cross-file foreign key to check

So `franchises.league_id` is declared a reference to `sparks_id` in `data/leagues.csv`, and CI fails on a franchise pointing at a league that doesn't exist.

`unique_within` exists because some values are only unique inside a namespace. `leagues.abbreviation` is `unique_within: [sport_id]`, so soccer's National League and baseball's National League can both be `NL` — while two baseball leagues sharing an abbreviation is still an error.

## Conventions

- CSVs are UTF-8 with LF line endings (enforced by `.gitattributes`), one header row, sorted by `sparks_id`.
- The schema is meant to be stable. Breaking changes are avoided.
- Identifiers only — no stats, standings, rosters, or other dynamic data.

### Scope: crosswalk, not registry

This repo is a **crosswalk**. It answers "which entity is this, and what does everyone else call it?" — not "tell me about this entity."

The test for whether a field belongs here is **"is it needed to establish or verify identity?"**, which is narrower than "is it an identifier" and wider than it first sounds:

- **Identifiers** obviously stay — `sparks_id` and every external id.
- **Labels stay.** You cannot review a crosswalk change or debug a mismatch against bare opaque ids. They are a matching aid, not a presentation layer.
- **Lineage stays** — `root_franchise`, `superseded_by`, `superseded_reason`, `parent_league`, `status`. Identity *over time* is the hardest part of sports crosswalking: `al_mil_1901` and `al_mil_1970` are different franchises that share a city.
- **Disambiguating years stay.** The founding year is part of a franchise id precisely because it is what tells two otherwise-identical franchises apart.
- **Descriptive metadata does not belong here.** Coordinates tell you nothing about *which* entity you have.

Richer metadata will live in a separate registry repo, following the same split the sibling PRISM project uses. See [DESIGN.md](DESIGN.md) for the full reasoning. Fields currently carried here that are registry candidates, and are expected to move:

| Field | Dataset | Why it moves |
| --- | --- | --- |
| `latitude_dd`, `longitude_dd` | venues | Pure geography |
| `country_code`, `subdivision_code`, `city` | venues | Pure geography |
| `location_label`, `nickname_label` | franchises | Derived — `label` is exactly these two joined |

They are still populated and validated for now. When the registry exists they will be marked `active: false`, which drops them from validation and from the published `dist/` artifacts without removing the data from these files.

### Labels and localization

`label` fields hold the canonical US English name — `Ice Hockey`, `Major League Baseball`, `New York Yankees`. There is deliberately no locale suffix on the field name.

Venue fields (`founding_name`, `current_name`, `city`) are *not* translations but proper nouns recorded in whatever language the thing is actually named: `Estadio Santiago Bernabéu`, `Neo Química Arena`, `Ciudad de México`. Localizing them would be wrong, so they carry no locale marker either.

If localization is ever wanted, it will arrive as a `labels.csv` sidecar keyed on `(entity_type, sparks_id, locale)` rather than as per-locale columns. That way a new language is a data addition, not a schema change — which is what the stability goal above requires.

## Repository Structure

- `data/` — source-of-truth CSV files, one per entity type
- `schema/` — the schema file for each CSV
- `.github/workflows/` — per-dataset validation, plus the dispatch that rebuilds `sparks-crosswalk`
- `DESIGN.md` — design decisions and the reasoning behind them
- `ATTRIBUTION.md` — attribution for upstream sources
- `LICENSE` — data license

## Roadmap

- Correction and cleanup of existing data
- Franchises for the NFL, NBA, and NHL — currently only MLB is covered
- Deeper venue coverage
- Franchise-to-venue tenancy links, so a franchise can be traced through the venues it has called home

## Contributing

Pull requests are welcome.

**Especially welcome:**

- Corrections to existing mappings, particularly identifiers that have drifted
- New franchises and venues, including historical ones
- Additional identifier sources — please open a discussion first, as inclusion criteria are still subjective

**Likely to be rejected:**

- **Schema changes.** Stability keeps downstream consumers working. Changes are considered only in exceptional cases.
- **Extra metadata.** This project maps identities. Stats, standings, rosters, and other dynamic data are out of scope.
- **Edits to `sparks-crosswalk`.** That repo is generated. Fix the data here and it will be rebuilt.

When in doubt, open an issue before submitting a PR.

## Attribution

See [ATTRIBUTION.md](ATTRIBUTION.md).

## License

- Data and schemas are licensed under the [Open Data Commons Attribution License (ODC-By 1.0)](https://opendatacommons.org/licenses/by/1-0/).
- Any code is licensed under MIT.
