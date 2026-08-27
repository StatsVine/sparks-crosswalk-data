# SPARKS Design Notes

Decisions about what this repo models and why. Written to stop settled questions being
relitigated, and to record the reasoning — including the evidence — behind choices that
look arbitrary from the outside.

Status: living document. Decisions marked **Open** are not settled.

---

## 1. Scope: crosswalk, not registry

SPARKS answers *"which entity is this, and what does everyone else call it?"* — not
*"tell me about this entity."*

**The test for a field: is it needed to establish or verify identity?** That is narrower
than "is it an identifier" and wider than it sounds:

| Keep | Why |
| --- | --- |
| Identifiers | The product |
| Labels | You cannot review a crosswalk change or debug a mismatch against bare opaque ids |
| Lineage | Identity *over time* is the hardest part of sports crosswalking |
| Disambiguating years | The founding year is in a franchise id precisely because it separates two otherwise-identical franchises |

| Move out | Why |
| --- | --- |
| Geography | Coordinates say nothing about *which* entity this is |
| Derived fields | A second source of truth that can silently drift |
| Time-varying membership | Dynamic data, explicitly out of scope |

Richer metadata belongs in a future registry repo, mirroring the PRISM split.

**Registry candidates currently carried here**, expected to move via `active: false`
(honoured by both the validator and the dist build, so a field can be retired without
deleting data):

- `venues`: `latitude_dd`, `longitude_dd`, `country_code`, `subdivision_code`, `city`
- `franchises`: `location_label`, `nickname_label` — derived; `label` is exactly these
  two joined, verified across all 45 rows

---

## 2. Repo topology

```
sparks-crosswalk-data     hand-edited CSV + schema   <- source of truth
        |  push to main touching data/ or schema/
        |  -> repository_dispatch "crosswalk-data-updated"
        v
sparks-crosswalk          generated dist/, never hand-edited
        |
        v
   pages branch           Astro site serving dist/ as read-only JSON
```

`sparks-tools` holds the shared validators and build scripts both repos run.

**Why two repos:** separation of concerns. Mixing them buried real data edits under
automated build commits, and consumers should read derived artifacts rather than raw CSV.
This mirrors PRISM, which had already made the same split.

---

## 3. Labels and localization

`label_en-US` was renamed to `label` (and `location_label`, `nickname_label`).

The suffix was applied to the three schemas that are *always* en-US and omitted from
`venues`, which actually holds Portuguese and Spanish (`Neo Química Arena`,
`Estadio Santiago Bernabéu`, `Ciudad de México`) — exactly inverted.

There is also a category error in suffixing venue names at all. **Venue names are proper
nouns, not translations.** `Estadio Santiago Bernabéu` is not the Spanish rendering of an
English name; it is the name. `Ice Hockey` genuinely does translate. Only the latter is a
localization problem, and it covers five rows.

**Decision:** no locale suffix. If localization is ever wanted it arrives as a
`labels.csv` sidecar keyed on `(entity_type, sparks_id, locale)`, so a new language is
data rather than a schema change — which is what the stability goal requires. PRISM does
not suffix either.

---

## 4. Leagues

### parent_league vs superseded_by

The two fields express different relations, and the test is temporal:

- **`parent_league` — concurrent containment.** The child still exists, inside the parent.
- **`superseded_by` — sequential replacement.** The predecessor stopped existing.

AL and NL sit under MLB (concurrent). APFA is superseded by the NFL (sequential).

Note the parent is *younger* than its children (MLB 1903, AL 1901, NL 1876) because MLB
was created as an umbrella over two leagues that both continued. **"Parent founded before
child" is not a valid invariant here.**

### The ~2000 AL/NL unification is deliberately not modelled

AL and NL stopped being independent legal entities around 2000 and now function as
conferences. SPARKS does not represent that, for two reasons:

1. **Every provider still treats them as live leagues.** MLB StatsAPI serves league ids
   103 and 104; ESPN and Reuters likewise. A crosswalk should mirror the consensus of its
   sources, not impose a more historically precise ontology than any of them use.
2. **The franchise data depends on it.** Two franchise records exist purely to capture
   AL↔NL moves — the 1998 Brewers and the 2013 Astros. Treating unification as a
   supersession reduces both to `mlb -> mlb` and destroys the information.

Alternatives were rejected: putting post-2000 franchises under MLB splits identical events
into incompatible representations (the Brewers' 1998 move reads AL→NL while the Astros'
2013 move reads "National League to Major League Baseball"); creating new franchise records
at dissolution requires inventing a lineage event for something that never happened on the
field.

### Open: a `level` enum

MLB StatsAPI exposes an umbrella, leagues, divisions, spring circuits (Cactus, Grapefruit),
defunct majors (Federal, Players, Union Association, American Association) and seven Negro
Leagues — roughly six *kinds* of row for one table. Divisions fit the recursive
`parent_league` structure and need no new file, but at that point "what kind of thing is
this row" stops being inferable and `level` becomes load-bearing.

### Open: league-level superseded_reason

`apfa-1920 -> nfl` is a **rename**, which is not in the franchise reason enum. Partial
mergers (ABA→NBA, WHA→NHL absorbed four teams each; the rest folded) are not clean
successions either. A league-level vocabulary would need something like
`rename | merger | absorption` rather than reusing the franchise one.

---

## 5. The franchise/era split

**Decision: split `franchises.csv` into a franchise table and a `franchise_eras.csv`
child table.**

### The problem

SPARKS franchise rows are era-scoped (`al_phi_1901`, `al_kca_1955`, `al_oak_1968`,
`al_ath_2025`). Providers slice franchises at different grains, so era-scoped rows cannot
host every identifier.

Hosting lineage-scoped ids on the *current* row fails outright: when the Athletics move to
Las Vegas the id must migrate from the Sacramento row to the new one. **An identifier whose
location is a function of time is not an identifier** — every relocation silently
invalidates cached mappings.

Hosting them on the *root* row works mechanically (the root never changes) but makes one
row mean two things: `al_phi_1901` would read "Philadelphia Athletics, ceased 1954" *and*
"MLB id 133", the second describing a team playing this season.

### The evidence

Verified against live APIs, using the Athletics (four eras, three cities) and the Raiders:

| Source | Test | Result |
| --- | --- | --- |
| MLB StatsAPI | Athletics 1950–2025 | `id=133` across Philadelphia, Kansas City, Oakland, Sacramento; `firstYearOfPlay: 1901` throughout |
| MLB StatsAPI | league changes | Astros `id=117` spans NL→AL; Brewers `id=158` spans Seattle→Milwaukee *and* AL→NL |
| ESPN MLB | Athletics 1950–2025 | `id=11` across all four eras |
| ESPN NFL | Raiders 1980–2024 | `id=13` across Oakland→LA→Oakland→Las Vegas |
| ESPN NFL | Washington rename | `id=28` across Redskins→Commanders |
| Pro Football Reference | — | stable across every relocation and rename 2002–2026 (verified in PRISM) |
| Wikidata | Astros, Brewers | `Q848117` and `Q848103` shared across league-change eras, but **distinct per relocation** |

**The pattern is structural, not incidental: contemporary operational providers are
lineage-scoped; historical and research sources are era-scoped.** Operational providers
answer "the franchise you follow today," so relocation must not break the id. Research
sources answer "the team that played in 1955," so the era *is* the entity. SPARKS
crosswalks both kinds, so it will permanently accumulate identifiers at both grains.

**A single provider spans both grains.** MLB StatsAPI team 133:

| Franchise-stable | Era-scoped |
| --- | --- |
| `id` (133) | `teamCode` (pha → kc1 → oak → ath) |
| `firstYearOfPlay` (1901) | `abbreviation` (PHA → KCA → OAK → ATH) |
| | `name`, `locationName`, `shortName`, `franchiseName`, `teamName`, `clubName` |

So mixed grain cannot be avoided by curating the source list.

### Franchise ids are opaque

Origin-based readable ids were considered and rejected. They are historically true but
**actively misleading**: `mil_1901` is the Orioles while the Brewers are `sea_1969`;
`was_1901` and `was_1961` are the Twins and Rangers while Washington's team is `mon_1969`.
Nine of thirty do not hint at the current team. An opaque id says nothing; an origin id
says something confidently wrong.

Era ids stay readable (`al_oak_1968`) and are immutable once written — the era genuinely
*is* league+location+year scoped.

The opacity costs less than it appears because **era rows are self-describing**: identical
opaque `franchise_id` strings cluster visually, and every other column on the row carries
meaning. The key does grouping, not identification.

### There is no stable franchise name — anywhere

Five of ten multi-era franchises change nickname (Brewers→Browns→Orioles,
Pilots→Brewers, Senators→Twins, Senators→Rangers, Expos→Nationals).

MLB agrees: under `id=110`, `teamName` and `clubName` go Brewers → Browns → Orioles. Only
`id` and `firstYearOfPlay` survive a lineage. `franchiseName` is a trap — it varies
(Philadelphia → Kansas City → Oakland → Athletics) despite its name.

**Decision:** `franchises.label` is the **current-or-final** era label — always populated,
meaningful for defunct franchises, and explicitly display-only. It is derived, so it must be
a *checked* mirror: a `sparks-tools` rule asserting `franchise.label == latest_era.label`.
Derived-and-validated is materially different from derived-and-hoped.

An abbreviation was rejected for this role: it is not unique across sports (LA is Lakers,
Angels and Rams) and reads as a join key.

### status stays explicit; years are derived

`year_founded` and `year_ceased` are `min`/`max` over eras — derived, so omitted.

**`status` is not derivable.** A folded franchise and a mid-relocation franchise are
structurally identical from the eras alone: both have a latest era with a `year_ceased` and
no successor. The Athletics between Oakland ending 2024 and Sacramento starting 2025 look
exactly like a franchise that died in 2024. Only an explicit marker distinguishes them.

The enum shrinks to `active | defunct`; `superseded` was only ever an era-to-era relation.
Note `defunct` is currently used zero times — all 30 franchises are active — so this breaks
the moment historical data lands.

### Sketch

```yaml
# franchises.csv
sparks_id:   opaque 8-char, required, unique      # ^[a-z0-9]{8}$
sport_id:    reference -> sports, required         # immutable
label:       required, unique: false               # current-or-final era label, display only
status:      enum [active, defunct], required
mlb_id:      optional                              # lineage-scoped
espn_id:     optional, unique: false               # see cross-sport collision below

# franchise_eras.csv
sparks_id:        al_oak_1968                      # readable, immutable once written
franchise_id:     reference -> franchises
league_id:        reference -> leagues             # varies: Astros NL->AL
label, location_label, nickname_label
abbreviation                                       # display shorthand, NOT an identifier
year_founded, year_ceased
superseded_reason
wikidata_uri:     unique: false                    # repeats across league-change eras
mlb_team_code, mlb_abbreviation, espn_abbreviation # era-scoped join keys
```

`root_franchise` disappears — it becomes the parent foreign key. `superseded_by` becomes
era ordering. `superseded_reason` changes meaning to "why this era ended."

---

## 6. Identifiers

### What counts as an identifier

**Two-part test: is it unique within its provider's namespace, and does it appear as a key
in real data?**

Redundancy with a provider's numeric id is irrelevant. If someone hands you a file keyed on
`OAK`, `mlb_id=133` does not help — the abbreviation *is* the key for that data. A crosswalk
maps the keys people actually hold, not the keys a provider considers canonical.

| Field | Verdict |
| --- | --- |
| `teamCode`, `abbreviation` | Identifiers — crosswalk, era grain |
| `teamName`, `locationName`, `shortName`, `franchiseName` | Display strings — registry |
| `fileCode` | **Do not track.** Reads `oak` for the 1950 Philadelphia and 1960 Kansas City eras, then flips to `ath` in 2025. A retroactively-rewritten current code: stable-looking and actively hazardous |

### SPARKS' own `abbreviation` is not an identifier

It stores **city codes, not team codes**, so it is ambiguous by construction:

```
PHI: Philadelphia Athletics, Philadelphia Phillies
STL: St. Louis Browns,       St. Louis Cardinals
BOS: Boston Red Sox,         Boston Braves
MIL: four different franchises
WAS: three different franchises
```

MLB disambiguates properly (`PHA` vs `PHI`, `SLB` vs `STL`). SPARKS' field is a deliberate
display shorthand and nothing may join on it. Provider codes sit beside it as real keys.

### Typed columns, not generic

`mlb_team_code` rather than `league_team_code`, because:

1. Every other external id in SPARKS is named for its source; a generic column would be the
   only break in the pattern.
2. **A single league issues multiple codes that diverge** — MLB's `teamCode=kc1` against
   `abbreviation=KCA` is not a case variant. A generic column forces you to discard one.
3. Self-describing values matter most in the file humans review.

Sparsity is accepted: "this entity is known to these N of M sources" *is* the data, and
`build_crosswalk_dist.py` skips blanks when building `by_field` maps, so sparse columns
produce compact artifacts. The cost is confined to eyeballing raw CSV.

### ESPN ids collide across sports

`id=11` is the Athletics in MLB, the Colts in NFL, the Pacers in NBA. So
`franchises.espn_id` **cannot be `unique: true`** — uniqueness is per sport. Since the
validator only does global uniqueness, this needs `unique: false` plus a `sparks-tools`
rule scoped on `(sport_id, espn_id)`.

`venues.espn_id` stays `unique: true` — venues come from a single ESPN namespace.

---

## 7. Open questions

**Coverage semantics.** A blank means both "this provider has no id for this entity" and
"nobody has checked." For a crosswalk, coverage *is* the product, so losing that distinction
matters — and it is what makes automated verification hard to write, since a checker cannot
tell a real gap from unresearched. Making external ids optional widened this. Options: a
sentinel value, or a coverage sidecar declaring which sources have been swept.

**No verification tooling.** PRISM runs nine `check_*.yml` workflows reconciling ids against
upstream. SPARKS has none, so identifiers here are written once and never re-checked, while
ESPN and Reuters change slugs and Wikidata items get merged. `check_wikidata.yml` is directly
portable.

**Sports have no relationship or lifecycle modelling.** Leagues get `parent_league` and
`superseded_by`; franchises get lineage. Sports get neither, so `canadian-football` or
`futsal` could not be expressed as variants. Currently an accident rather than a decision.

**League membership is many-to-many.** A team is in the AL *and* the Cactus League
simultaneously. A single `league_id` on an era cannot express that. Membership therefore
belongs in the registry as a mapping table, not as a field — worth settling before the era
schema is finalised.

**A founding-year disagreement.** MLB StatsAPI gives the Brewers franchise
`firstYearOfPlay: 1968`; SPARKS has `al_sea_1969` founded 1969. Probably expansion-awarded
versus first-played. SPARKS should state which convention it follows.

**`soccer` is an orphan** — defined as a sport, referenced by zero leagues and zero
franchises, while `venues.csv` holds at least four soccer grounds. Harmless, but carrying no
weight.

---

## 8. Roadmap implications

- **NFL franchises unblock PRISM.** Its `teams.csv` carries a `sparks_id` column sitting
  empty, explicitly waiting on SPARKS football data.
- **The Negro Leagues are the largest well-sourced gap.** MLB elevated seven to major-league
  status in 2020 and StatsAPI carries ids for all of them (426–432). Nobody has this cleanly
  crosswalked.
- **Franchises is the weakest crosswalk table and the most important one.** It carries one
  external identifier against venues' three, while team identity is the canonical crosswalk
  problem. The split plus provider codes is what fixes that.
