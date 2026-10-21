# Leisure and hospitality venues (Mexico, DENUE April 2020)

How `data/Mexico_data/venues/leisure_activities/*.csv` is produced, and why
each decision was taken. Built by `data/Mexico_data/source/build_leisure_venues.py`
(cinemas, gyms, leisure, restaurants, bars, cafes) and
`data/Mexico_data/source/build_malls.py` (malls).

Companion to [companies.md](companies.md): same source (DENUE), same
vintage, same per-establishment transformation logic, different intent.
`companies.csv` is one workplace file grouped to 2-digit SCIAN sector so it
can be matched against the census work-sector vocabulary. These files
instead split out specific consumer-facing venue types, kept at the full
6-digit SCIAN code, because there is no census vocabulary to reconcile
against here -- the 6-digit code is just a useful subtype label within each
file (e.g. "bares, cantinas" vs "centros nocturnos, discotecas" inside
`bars.csv`).

## 1. The source: same vintage as `companies.csv`, deliberately

Same DENUE register as `companies.csv` (see [companies.md](companies.md) S1
for the by-entidad vs by-activity split and why by-activity is the only
dateable cut), and the **same April 2020 vintage**, fetched from the same
archived by-activity URLs (`BASE` in both scripts). This was a deliberate
correction: an earlier version of this build read the already-downloaded
*current* (2026) DENUE files instead, which would have mixed a 2020
workplace snapshot (`companies.csv`) with six-years-later leisure venues in
the same world build. Kept at one vintage so a world combining workplaces
and leisure destinations describes one consistent economic snapshot.

Fetching live from INEGI (rather than reading local files) surfaced a
practical wrinkle worth recording: **INEGI's server has highly variable
time-to-first-byte** -- observed delays of 20-40 minutes before a large
sector file starts responding, then a fast transfer (tens of MB in under a
minute) once it does. This is not a client-side hang; `socket.setdefaulttimeout`
plus chunked reads with progress logging (both scripts) make it visible
rather than silently stuck, and retry on genuine connection failures. Building
the full national mall list (`build_malls.py`, all 25 sector files) took
roughly two hours of wall-clock time for this reason, almost all of it spent
waiting on INEGI's server rather than transferring or processing data.

## 2. Category definitions

Six categories share one script (`build_leisure_venues.py`); malls are big
enough structurally to need their own (`build_malls.py`, S3).

| file | SCIAN codes | sector part(s) fetched |
|---|---|---|
| `cinemas.csv` | `512130` | 51 (informacion en medios masivos) |
| `gyms.csv` | `713941`-`713944` | 71 (esparcimiento) |
| `leisure.csv` | all of 71 except the gym codes | 71 |
| `restaurants.csv` | `722511`-`722514`, `722516`-`722519` | 72_1, 72_2 |
| `bars.csv` | `722411`, `722412` | 72_1, 72_2 |
| `cafes.csv` | `722515` | 72_1, 72_2 |

`build_leisure_venues.py` fetches each distinct sector part exactly once
(51, 71, 72_1, 72_2) and fans each row out to every category whose code set
it matches, rather than downloading 72_1/72_2 three times over for
restaurants, bars and cafes separately.

### Cinemas

`512130` "Exhibicion de peliculas y otros materiales audiovisuales" is the
only SCIAN code for cinema exhibition; unambiguous. 960 establishments
nationally in the April 2020 file.

### Gyms

`713941`/`713942` (clubes deportivos, private/public) and `713943`/`713944`
(centros de acondicionamiento fisico, private/public) are carved out of
sector 71 rather than left inside `leisure.csv`, because they dominate it by
volume and are a functionally distinct venue type (regular repeat visits,
exercise contact patterns) from one-off attractions. 26,533 establishments.

### Leisure

Everything else in sector 71: performing-arts companies (theatre, dance,
music), professional sports and promoters, museums and historic sites, zoos
and botanical gardens, amusement and water parks, arcades, casinos and
lottery/betting outlets, golf courses, marinas, bowling, billiards, and hobby
clubs. Deliberately kept as one broad "attractions and recreation" file rather
than split further -- SCIAN's own subsector boundaries here (performing arts
vs. spectator sports vs. gambling vs. amusement) don't map cleanly onto a
short, human list of venue types, and splitting further multiplies files
without a clear modelling reason to treat them differently. Revisit if a
specific subtype (e.g. casinos) needs isolating later. 37,598 establishments.

### Restaurants, bars, cafes

All three read both `72_1` and `72_2` -- INEGI splits sector 72 into two
arbitrary part files that do not respect any activity boundary (confirmed:
both files carry the identical code set), so both must be read and
concatenated, exactly as `companies.csv` does for the same split.

- **Restaurants**: the eight `7225xx` "restaurantes"/prepared-food codes
  (`722511`-`722514`, `722516`-`722519`). Excludes `722310`/`722320`/`722330`
  (comedor para empresas, banquetes, alimentos en unidades moviles) --
  catering and institutional food services, not a place someone visits.
  562,279 establishments.
- **Cafes**: `722515` alone ("cafeterias, fuentes de sodas, neverias,
  refresquerias y similares") is exactly the cafe/soda-fountain category and
  nothing else, so it is split out of `restaurants.csv` rather than folded
  in. 75,407 establishments.
- **Bars**: `722411` (centros nocturnos, discotecas) and `722412` (bares,
  cantinas) -- both nightlife-oriented, merged into one file since the
  distinction (dancing vs. drinking) is not one the venue schema tracks.
  30,637 establishments.

## 3. Malls: reconstructed, not coded

DENUE has no SCIAN code for "shopping mall," because a mall is a building,
not an economic activity -- the code belongs to each tenant (a bank, a shoe
shop, a cinema), not to the mall itself. Every establishment in *any* sector
instead carries two fields tagging where it sits:

- `tipoCenCom` -- the type of complex (shopping plaza, market, hospital,
  industrial park, ...)
- `nom_CenCom` -- the complex's name

`build_malls.py` therefore fetches **all 25 by-activity sector files** (the
full DENUE register, not just retail), and reconstructs a mall as the group
of tenant establishments sharing `(entidad, municipio, nom_CenCom)`.

### Which `tipoCenCom` values count as "mall"

Counted nationally across all 25 April 2020 files before deciding:

| count | tipoCenCom | kept? |
|---|---|---|
| 187,972 | MERCADO PUBLICO | no -- public/traditional market, different retail format |
| **96,804** | **CENTRO Y PLAZA COMERCIAL** | **yes** |
| 37,776 | EDIFICIO COMERCIAL Y DE SERVICIOS | no -- any generic office/services building, too broad |
| 19,473 | CENTRAL DE ABASTO | no -- wholesale supply market |
| 15,243 | HOSPITAL, CENTRO MEDICO O CLINICA | no |
| 14,584 | OTRO CONGLOMERADO | no -- see below |
| 12,811 | TIANGUIS, BAZAR O PULGA | no -- street market |
| **8,899** | **PASAJE Y ANDADOR COMERCIAL** | **yes** -- commercial arcade/walkway |
| 1,469 | CENTRAL CAMIONERA | no -- bus terminal |
| 836 | AEROPUERTO | no |
| 721 | PARQUE RECREATIVO | no -- already covered by `leisure.csv` |
| 151 | PARQUE INDUSTRIAL | no |
| 47 | CIUDAD INDUSTRIAL | no |
| 13 | ZONA INDUSTRIAL | no |
| 12 | CORREDOR INDUSTRIAL | no |
| 9 | CORREDORES TURISTICOS | no |
| 2 | AEROPUERTOS | no (label variant of AEROPUERTO) |
| 1 | ESCUELAS | no |
| **1** | **CENTRO COMERCIAL** | **yes** -- label variant, negligible n |

**`OTRO CONGLOMERADO`** ("other conglomerate") is excluded despite its size.
It is the closest April-2020 counterpart to the 2026 release's `OTROS
CONJUNTOS COMERCIALES` label (see below), which would have been included --
but that label is unambiguously commercial ("other commercial complexes")
and was negligible (4 rows nationally in the 2026 catalogue), while
"conglomerado" alone does not specify commercial use and 14,584 rows is too
large a category to fold in on an ambiguous label. Same reasoning as
excluding `EDIFICIO COMERCIAL Y DE SERVICIOS`: when a label doesn't clearly
mean "shopping mall," it is left out rather than assumed in. This is a
judgment call, not a measurement -- if it turns out `OTRO CONGLOMERADO` is
mostly shopping centers under a different name, revisit it; the two catalogs
(April 2020 vs. 2026) don't share a stable label set to check against.

105,704 tenant rows qualify nationally under the three kept categories.

### Avoiding double-counting with the six category files

A mall's cinema, gym, restaurant, bar or cafe tenant is *also* a DENUE row in
its own right, with a SCIAN code already captured by `build_leisure_venues.py`.
Left unhandled, that establishment would be counted twice: once as its own
venue row in `cinemas.csv`/`gyms.csv`/`leisure.csv`/`restaurants.csv`/
`bars.csv`/`cafes.csv`, and again folded anonymously into its mall's
`tenant_count` and `employee_count`.

Measured directly (matching each of the six category code sets against
`tipoCenCom`): of 105,704 mall-type tenant rows, **15,415 (14.6%)** carry a
code already in one of the six files.

| category | national total | inside a mall | share of category |
|---|---|---|---|
| cinemas | 960 | 130 | 13.5% |
| gyms | 26,533 | 903 | 3.4% |
| leisure | 37,598 | 1,331 | 3.5% |
| restaurants | 562,279 | 9,862 | 1.8% |
| bars | 30,637 | 652 | 2.1% |
| cafes | 75,407 | 2,537 | 3.4% |

Cinemas are disproportionately mall-anchored (13.5% of all cinemas
nationally sit inside a mall -- multiplexes are a classic mall anchor
tenant); restaurants contribute the largest raw overlap (9,862 food-court-type
units) despite being the smallest share of their own category.

`build_malls.py` excludes any tenant whose `codigo_act` is in
`EXCLUDED_CODES` (the union of all six category code sets) from a mall's
`tenant_count`/`employee_count`/centroid. `malls.csv` therefore represents
each mall's *other* tenants only -- retail shops, banks, salons, professional
services, and anything else not already tracked as its own venue type -- and
its `employee_count` adds cleanly alongside the six category files without
double-counting the same staff.

### Reconstruction

For each kept tenant row, grouped on `(cve_ent, cve_mun, nom_CenCom.upper())`:

- `latitude`/`longitude` = mean of tenant coordinates (centroid)
- `employee_count` = **sum** of tenant `employee_count` draws (same
  band-draw method as S4 below) -- an approximation of the mall's total
  on-site headcount across all its tenants, not a `per_ocu` band of its own,
  since the mall itself has none
- `industry_code` = the tenant `tipoCenCom` value occurring most often within
  the group (a mall is overwhelmingly one `tipoCenCom`, but a handful mix
  categories at the edges)
- `tenant_count` = number of tenant establishments grouped into the venue
  (carried as an extra property column, same mechanism `companies.md` S3
  describes for `industry_code`/`employee_count`)
- `name` = a synthetic `mall_<n>` index, **not** `nom_CenCom` verbatim --
  mall names are free text (accents, punctuation, abbreviations vary) and are
  not carried into the id to keep it stable; the original name is discarded
  after grouping. If traceability back to the source name matters, change
  `build_malls.py` to keep it as an extra column.

11,312 malls reconstructed from 87,639 tenant rows: 105,704 mall-type rows,
minus 15,350 excluded as already counted in the six category files (S3
above), minus a small number of matching-type rows with an empty
`nom_CenCom` (dropped since there is nothing to group them by). Note some
malls that existed in the pre-exclusion build disappeared entirely here --
a "mall" whose only DENUE-visible tenant was, say, a single restaurant is no
longer a distinct venue once that restaurant is excluded, which is correct:
it was never more than that restaurant to begin with.

### Food courts and mall-anchor tenants: `in_mall_type`/`in_mall_name`

Excluding a mall's cinema/gym/restaurant/bar/cafe tenant from `malls.csv`
(S3 above) avoids double-counting its capacity, but it also throws away the
fact that the tenant sits inside a mall at all -- a food-court taco stand
would otherwise be indistinguishable from a standalone street-front
restaurant. `build_leisure_venues.py` carries that link forward instead:
every row in all six category files gets two extra columns, `in_mall_type`
and `in_mall_name`, populated from the establishment's own `tipoCenCom`/
`nom_CenCom` whenever `tipoCenCom` is one of the same three `MALL_TYPES`
labels `build_malls.py` uses (both scripts keep this set identical, so "is
this venue inside a mall" means the same thing everywhere) -- blank
otherwise. Example, from `restaurants.csv`:

```
geo_unit,name,industry_code,...,in_mall_type,in_mall_name
MX01001,denue_36863,722511,...,CENTRO Y PLAZA COMERCIAL,CENTRO COMERCIAL PLAZA PATRIA
MX01001,denue_8624508,722517,...,CENTRO Y PLAZA COMERCIAL,PLAZA EL CAMPANARIO
```

This is informational only -- it does not change which file a row lands in,
its `employee_count`, or anything about `malls.csv`. It exists so a model
that wants food-court-style shared-space contact logic (or wants to identify
a mall's anchor cinema/gym) can group rows across the six category files by
`in_mall_name` without re-deriving DENUE's tenant-to-mall link from scratch.
Grouping by `in_mall_name` reconstructs exactly the tenant set the mall
build excluded (130 cinemas, 903 gyms, 1,331 leisure, 9,862 restaurants, 652
bars, 2,537 cafes -- the same table as S3 above), so a "food court" for a
given mall is `restaurants.csv`/`bars.csv`/`cafes.csv` rows filtered to that
`in_mall_name`, not a separate file. Same name-matching-fragmentation caveat
as `malls.csv` applies (below): a mall typed inconsistently across its
tenants groups as multiple names, not one.

### Known limitation: name-matching fragmentation

Grouping is exact-match on uppercased `nom_CenCom` within the same
municipio. DENUE mall names are free text entered independently per
establishment with no canonical registry and no dedupe -- "PLAZA GALERIAS",
"GALERIAS PLAZA", "PLAZA GALERIAS CENTRO" and a misspelled variant would
split into four separate venues instead of one. This was not corrected
(would need fuzzy matching, and a threshold choice with no ground truth to
tune it against). Consequence: mall count is inflated and per-mall
`tenant_count`/`employee_count` are undercounted, more severely for malls
whose tenants type the name inconsistently. Small malls with few tenants are
least affected; large well-known malls with many independently-run tenants
are most affected, which is the opposite of what you'd want. The largest
reconstructed malls (600+ tenants, in Chihuahua, Jalisco and Estado de
Mexico) plausibly correspond to real major shopping centers, which is a
reassuring sanity check but not a validation.

## 4. Transformation (shared logic, both scripts)

Per establishment (cinemas/gyms/leisure/restaurants/bars/cafes):

```
geo_unit       = "MX" + cve_ent + cve_mun
name           = "denue_" + id
industry_code  = codigo_act verbatim (6-digit SCIAN, NOT truncated to sector)
sizeband       = per_ocu verbatim
employee_count = uniform draw within band; Pareto tail for the open band
latitude       = latitud
longitude      = longitud
in_mall_type   = tipoCenCom, if it is one of MALL_TYPES; else blank
in_mall_name   = nom_CenCom, if in_mall_type is non-blank; else blank
```

`employee_count` uses the identical method as `companies.csv`: uniform
integer draw on the closed bands' inclusive range, truncated Pareto (shape
1.2, max 10,000) on the open top band ("251 y mas personas"), seeded
(`SEED = 20200315`, same seed as `companies.csv`). See
[companies.md](companies.md) S2 "Size band -> headcount" for the full
justification; not repeated here.

Unlike `companies.csv`, `industry_code` is **not** truncated to a 2-digit
SCIAN sector. There is no census work-sector column to reconcile against for
these venue types, so the full 6-digit code is kept as the useful subtype
label (e.g. distinguishing golf courses from casinos within `leisure.csv`).
If a future config needs the 2-digit sector, truncate `industry_code[:2]` at
load time rather than re-running the build.

## 5. Output

```
data/Mexico_data/venues/leisure_activities/
├── cinemas.csv      (960 establishments)
├── gyms.csv         (26,533 establishments)
├── leisure.csv      (37,598 establishments)
├── restaurants.csv  (562,279 establishments)
├── bars.csv         (30,637 establishments)
├── cafes.csv        (75,407 establishments)
└── malls.csv        (11,312 reconstructed venues, from 87,639 tenant rows after excluding overlap with the six category files, see S3)
```

All counts are national (all 32 entidades) -- not filtered to any single
state or to the Aguascalientes test world. `ageb`/`manzana` are dropped for
the same reason given in `companies.md` S3: no population data exists below
municipio to make use of them.

## 6. Limitations carried over from `companies.csv`

- DENUE excludes agricultural/forestry activity and lists fixed premises
  only -- irrelevant here since none of these categories are agricultural,
  but street-vendor food stalls and informal bars with no fixed address are
  absent by the same construction that excludes informal workers from
  `companies.csv`.
- The top-band Pareto tail is assumed, not measured (see `companies.md` S2).
- `employee_count` is not calibrated to any independent visitor or capacity
  survey -- it measures staffing, not visitor throughput, and should not be
  read as a venue's contact capacity without that distinction in mind.
- Same pandemic-timing caveat as `companies.md` S5: April 2020 fieldwork
  observed establishments during the first weeks of Mexico's COVID-19
  shutdown (began 23 March 2020), so a business closed by the shutdown may
  still appear (`fecha_alta` records registration, not trading status).
