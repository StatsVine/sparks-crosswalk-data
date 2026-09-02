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
espn_id:     optional, unique_within: [sport_id]   # ids collide across sports

# franchise_eras.csv
sparks_id:        al_oak_1968                      # readable, immutable once written
franchise_id:     reference -> franchises
primary_league:   reference -> leagues             # varies: Astros NL->AL
label, location_label, nickname_label
abbreviation                                       # display shorthand, NOT an identifier
year_founded, year_ceased
superseded_reason
wikidata_uri:     unique: false                    # repeats across league-change eras
mlb_team_code, mlb_abbreviation, espn_abbreviation # era-scoped join keys
```

### League lives on the era, not the franchise

A franchise-level league was considered and rejected. The hierarchy has tiers with
different stability — top-level league (`mlb`), sub-league or conference (`al`, `nl`),
division — and only the middle tier is era-scoped for MLB, which makes a franchise-level
`mlb` look safely immutable.

It is not. **Ten AFL franchises became NFL franchises in 1970**; four ABA franchises
became NBA in 1976; four WHA became NHL in 1979. Those franchises genuinely changed
top-level league, so the field would have to be a derived, validated mirror of the
current-or-final era — a third such mirror to maintain, for a value that is already one
join away.

Keeping league on the era alone also expresses league supersession naturally: a franchise
moving from a superseded league to its successor is simply a new era.

`sport_id` stays on the franchise because it really is immutable — AFL to NFL is football
to football.

This leaves `franchises.csv` with exactly two non-identity fields, each separately
justified: `label` for review ergonomics, and `status` because it is not derivable.

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
`franchises.espn_id` is not globally unique — uniqueness is per sport.

**Resolved in the schema rather than worked around.** `sparks-tools` gained
`unique_within`, which narrows uniqueness to rows sharing the listed fields:

```yaml
espn_id:
  unique: true
  unique_within: [sport_id]
```

This was not a one-off for ESPN. `leagues.label` and `leagues.abbreviation` were
globally unique and already over-constrained: "Premier League" recurs across football,
rugby, darts and snooker, and "National League" is both MLB and English football's fifth
tier. Both are now `unique_within: [sport_id]`.

A field named in `unique_within` must exist in the schema — checked at schema-load time,
because a typo would silently widen uniqueness back to global.

`venues.espn_id` stays globally `unique: true` — venues come from a single ESPN namespace.

---

## 7. Open questions

**No verification tooling.** PRISM runs nine `check_*.yml` workflows reconciling ids against
upstream. SPARKS has none, so identifiers here are written once and never re-checked, while
ESPN and Reuters change slugs and Wikidata items get merged. `check_wikidata.yml` is directly
portable. Note that a checker needs the known-permanent gaps in section 8 to avoid
re-flagging them every run.

**Sports have no relationship or lifecycle modelling.** Leagues get `parent_league` and
`superseded_by`; franchises get lineage. Sports get neither, so `canadian-football` or
`futsal` could not be expressed as variants. Currently an accident rather than a decision.

**A founding-year disagreement.** MLB StatsAPI gives the Brewers franchise
`firstYearOfPlay: 1968`; SPARKS has `al_sea_1969` founded 1969. Probably expansion-awarded
versus first-played. SPARKS should state which convention it follows.

**`soccer` is an orphan** — defined as a sport, referenced by zero leagues and zero
franchises, while `venues.csv` holds at least four soccer grounds. Harmless, but carrying no
weight.

---

## 8. Resolved: coverage is deliberately not modelled

**Decision: a blank cell stays blank and means only "there is no id here." SPARKS does not
distinguish "confirmed absent" from "nobody has checked" — no sentinel values, and no
coverage sidecar.**

### There are three causes of blankness, not two

The question was framed as a two-way ambiguity. Auditing the current data found three:

| Cause | Verified example |
| --- | --- |
| **Unresearched** | `franchises.csv` carries `wikidata_uri` and nothing else. No franchise row has ever been swept for an ESPN or MLB id — the column does not exist yet |
| **Genuinely absent** | `al.espn_api_uri`, `nl.espn_api_uri`. ESPN's league namespace has no `/leagues/al`; it models only `mlb`. No amount of searching produces a value |
| **The provider draws the entity boundary elsewhere** | `apfa-1920.wikidata_uri`. Wikidata has no APFA item: "American Professional Football Association" and "APFA" are *aliases* on `Q1215884`, which SPARKS already assigns to `nfl`. Since `leagues.wikidata_uri` is `unique: true`, SPARKS structurally cannot record it |

This table is the durable part of the decision. Those blanks are **permanent and correct**,
and they look like data-entry errors to anyone who has not checked. Whoever writes the first
`check_*.yml` will otherwise either "fix" them or tune the checker until it stops
complaining.

### Why not a sentinel value

A sentinel (`-`, `n/a`) in the cell turns *the absence of an identifier* into *a value of
the identifier column*. Every layer that currently handles ids correctly would need a
special case in order to keep doing so:

- **The schema.** `espn_id` is `type: integer, pattern: ^\d{1,5}$`. A sentinel must be
  exempted per-column, per-type, permanently.
- **The dist build.** `build_crosswalk_dist.py` skips blanks when building `by_field` maps.
  A sentinel is not blank, so either the build learns about it too or `"-"` becomes a key in
  the ESPN map — pointing at every entity ESPN does not have.
- **Consumers.** A JSON reader must know that `espn_id: "-"` is not an id. That is a trap,
  served to everyone downstream whether they care about coverage or not.

### Why not a coverage sidecar either

A `data/coverage.csv` declaring which sources had been swept over which scope was designed
and rejected as speculative. **Its only real consumer is verification tooling that does not
exist** (section 7).

The cost is not the file, it is the protocol the file needs to stay honest. A sweep is a
point-in-time claim about a row set that keeps growing: add a Negro League franchise
tomorrow and a past ESPN sweep would silently annex it, turning an unresearched blank into
a confirmed absence. Keeping that sound requires recording an in-scope row count per sweep
and re-bumping it on every data edit that changes one — a standing manual obligation on a
45-row table, paid now, to serve a checker that may never be written. It also wanted two
`sparks-tools` changes before it could even be validated.

Section 6 already states the position that makes this affordable to skip: *"Sparsity is
accepted: 'this entity is known to these N of M sources' is the data."*

### What this costs

Every blank reads as unknown. That is the **honest** default — nothing claims coverage it
does not have — and at current scale (5 sports, 7 leagues, 45 franchises, 110 venues) the
picture is small enough to hold directly.

**What would reopen it:** the first `check_*.yml`. A checker reconciling SPARKS against a
live provider list cannot tell a permanent gap from an unresearched one, so it will need an
exception list. At that point the exception list *is* the coverage table, and it should be
built as data rather than buried in a workflow file. Until then, the table above is the
record.

### Consequences for the franchise/era split

**The split is unblocked and its table shape is unchanged.** This was the open question that
could still have moved it, and the answer moves nothing — and now adds nothing to build.
Because absence is never written into a cell, the provider-code columns in the section 5
sketch stay typed, stay optional, and gain no `*_checked` companions.

---

## 9. Resolved: league membership

Raised as a many-to-many problem — a team is in the AL *and* the Cactus League
simultaneously — but that conflates two things:

- **League affiliation** — exactly one per era. Identity-relevant, encoded in era ids
  (`al_oak_1968`), and providers key on it (MLB's team object carries `league: {id: 103}`).
- **Competition participation** — many. Spring circuits, postseason, and in soccer a club
  is in La Liga *and* the Champions League *and* a domestic cup at once.

The Cactus League is the second kind, so it does not threaten a singular field. **Decision:**
`franchise_eras.primary_league` stays singular, named to say so. Competition participation
is out of scope for the crosswalk.

Secondary leagues split across the boundary the same way everything else does:

- **The Cactus League as an entity** -> `leagues.csv`. It has an MLB StatsAPI id (114), a
  name and an abbreviation; external sources have identifiers for it.
- **Which franchises are in it** -> registry. Not identity-establishing, nobody joins on
  it, and it changes when teams switch spring facilities.

Soccer is what reopens this, not spring training. If soccer leagues land, "competition"
becomes a first-class concept needing somewhere to live, which forces the `level` question
at the same time. Dormant while coverage is MLB/NFL/NBA/NHL.

---

## 10. Open: extracting the schema tooling

`prism-tools/crosswalk/validate_players.py` and `sparks-tools/crosswalk/validate_csv.py`
are the same program, forked — identical function names, comments and error strings. They
have diverged in complementary directions: SPARKS gained `reference`, `enum`, `active`,
`decimal`, `unique_within`, sort checking and tests; PRISM gained core-plus-source schema
layering.

PRISM's schema vocabulary (`description`, `notes`, `pattern`, `required`, `type`, `unique`)
is a **strict subset** of SPARKS', so there is no divergence to reconcile — only a union to
take.

The only SPARKS-specific coupling is three lines hardcoding `sparks_id` for the sort check;
`build_crosswalk_dist.py` has none. Extraction is roughly: make the id field configurable,
union the feature sets, and have both repos clone it as they already clone tools.

Worth doing **before** promoting `unique_within` to PRISM by hand, or the same feature ends
up maintained in two forks — which is how these two got here.

---

## 11. Roadmap implications

- **NFL franchises unblock PRISM.** Its `teams.csv` carries a `sparks_id` column sitting
  empty, explicitly waiting on SPARKS football data.
- **The Negro Leagues are the largest well-sourced gap.** MLB elevated seven to major-league
  status in 2020 and StatsAPI carries ids for all of them (426–432). Nobody has this cleanly
  crosswalked.
- **Franchises is the weakest crosswalk table and the most important one.** It carries one
  external identifier against venues' three, while team identity is the canonical crosswalk
  problem. The split plus provider codes is what fixes that.
