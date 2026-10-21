# Agriculture venues (Mexico 2022)

How `data/Mexico_data/venues/primary_activities/companies_agriculture.csv` is
produced, and why each decision was taken. Built by
`data/Mexico_data/source/build_companies_agriculture.py`.

This file has no England counterpart, and no counterpart anywhere else in this
world either -- it is a second venue source for the same SCIAN sector
`companies.csv` already covers, added because that first source cannot cover
sector 11 by construction. Read [companies.md](companies.md) first; this
document assumes it throughout.

## Why this file exists

`companies.md` measures the problem directly: DENUE excludes agricultural and
forestry activities by INEGI's own catalogue description, and it shows up
end to end -- 22,227 of companies.csv's 5,487,061 national rows (0.4%) carry
`industry_code = 11`, holding 384,173 of employment capacity, against
27,568 census sector-11 workers in Aguascalientes alone. Nationally,
`work_sector.md` measures 1,793,316 sector-11 workers (32.5% of the sector) in
a (municipio, sector) cell with **zero** establishments, and `companies.md`'s
capacity accounting puts 3,285,364 agriculture employees with nowhere to be
placed -- the largest single component of the 21.3% national shortfall.

That gap is not a modelling defect to fix inside `companies.csv`; DENUE
genuinely does not enumerate farms and ranches. It is a coverage gap to fill
from a source that does: INEGI runs a separate census of agricultural
production, and this file is that census turned into the same venue shape.

## 1. The source

**Censo Agropecuario y Forestal 2022**, reference period October 2021 -
September 2022, INEGI. Pre-downloaded tabulations (not micro-data) under
`DENUE_2026/CENSUS_Agriculture/conjunto_datos/`, ~100 `ca2022_*.csv` tables
covering two universes: *agropecuaria* (crop and livestock production units)
and *forestal* (forestry production units, `for*` prefix). This build uses
the agropecuaria universe only -- see Limitations.

Unlike DENUE, this is not a register of individual establishments with
addresses. It publishes municipio-level aggregates of *unidades de
producción* (UP -- production units: farms, ranches, agricultural
holdings), organised into ~15 themed tables (`agrXX` crops, `ganXX`
livestock, `moXX` labour, `maqXX` machinery, `soc0X`
sociodemographics, `cred01` credit, `prob0X` production problems, `amb01`
environment). `indice_agro.csv` in the same folder is INEGI's own table
catalogue and lists what each `ca2022_*` file contains.

Two "Mano de obra" (labour) tables carry what this build needs:

| table | grain | columns used |
|---|---|---|
| `ca2022_mo01.csv` | municipio, real geo codes | `UP_REMUN` -- number of production units using **remunerated** (paid, non-family) labour |
| `ca2022_mo02.csv` | municipio, **name only, no geo code** | `MO_REMUN` -- total **headcount** of remunerated labour |

`ca2022_mo01.csv` carries `E03` (2-digit entidad) and `CVE_MUN` (3-digit
municipio) directly -- the same codes `hierarchy.csv` uses, just split across
two columns. `ca2022_mo02.csv` carries no equivalent code column: its first
field is `"01 AGS"` (entidad code and abbreviation run together) and its
second is the municipio *name*, blank for the state-total row.

### The positional join, and why not a name join

`companies_agriculture.md`'s sibling documents establish the rule already:
Oaxaca has two municipios named *San Juan Mixtepec* and two named *San Pedro
Mixtepec* ([README.md](README.md), [work_origin_destination.md](work_origin_destination.md)).
A join on `(entidad, municipio name)` cannot disambiguate either pair, and
`ca2022_mo02.csv` offers no other key.

The two labour tables are companion parts of the same INEGI theme release,
and were verified to share one property instead: **identical municipio order,
entidad by entidad.** Checked directly -- both files hold exactly 2,464
municipio rows, in the same order, confirmed with zero positional mismatches
across all 570 Oaxaca rows including both ambiguous name pairs (`208`/`209`
San Juan Mixtepec, `318`/`319` San Pedro Mixtepec). The join used here is
therefore `zip()` over the two files in file order, not a name lookup. This
sidesteps the Oaxaca problem entirely rather than working around it.

### Encoding

Both tables are UTF-8 (confirmed directly), unlike DENUE's Latin-1/CP1252 --
no encoding conversion is needed here.

## 2. Interpretation

### `UP_REMUN` is the establishment count; `MO_REMUN` is the capacity to place

This mirrors `companies.md`'s DENUE step exactly, one level up: DENUE gives a
`per_ocu` **band** per establishment that must become a headcount; this
census gives a **municipio total** of paid workers (`MO_REMUN`) that must be
split across a municipio total of paying production units (`UP_REMUN`). Both
steps invent a number the source does not publish at the unit level, and both
are isolated and documented for exactly that reason.

**Why `UP_REMUN`, not `UP_TOTAL`.** `UP_TOTAL` includes production units
that use only unpaid family or owner labour (`UP_NOREMUN`). Restricted to who
competes for `companies.csv`, `companies.md` already draws this line:
eligibility is `SITTRA` 1 (empleado/obrero) and 4 (patrón/empleador) --
workers at a **fixed premises with a payroll**, not the broader own-account
and unpaid-family workforce. `UP_REMUN` is the agricultural census's own
version of that same distinction: a production unit that pays labour is the
agricultural equivalent of a DENUE establishment; one that runs on family
labour alone is the agricultural equivalent of an own-account worker with no
registrable premises, and is excluded from this file for the same reason
`companies.md`'s own-account workers are excluded from `companies.csv`.

This was checked, not assumed: every one of the 2,458 municipios with
`UP_REMUN > 0` also has `MO_REMUN >= UP_REMUN` (2,458 checked, 0 violations)
-- each remunerated production unit reports at least the one paid worker its
own classification implies.

### Sector vocabulary

`industry_code = "11"`, the same SCIAN sector code sector-11 rows already
carry in `companies.csv`. No grouping is needed (unlike `31-33` or `48-49`):
sector 11 (Agricultura, cría y explotación de animales, aprovechamiento
forestal, pesca y caza) is a single code, and the Censo Agropecuario's
agropecuaria universe (crops and livestock) sits inside it without
ambiguity.

### Geography

`geo_unit` is built directly from `ca2022_mo01.csv`'s own `E03` + `CVE_MUN`
codes -- `"MX" + E03.zfill(2) + CVE_MUN.zfill(3)` -- the same construction
`build_companies.py` uses for DENUE's `cve_ent` + `cve_mun`. No name lookup is
needed for this step; the positional join above only concerns attaching
`ca2022_mo02.csv`'s labour totals to these already-coded rows.

### No coordinates

DENUE carries a real address and lat/long per establishment; this census
carries neither -- it is an aggregate table, not a directory of premises.
Each synthetic establishment is placed at its municipio's centroid
(`geography/coord_sgu.csv`) plus a seeded uniform jitter of up to ±0.05
degrees (roughly ±5 km at Mexico's latitudes), so establishments sharing a
municipio are not stacked at one point. **This is a modelled placement, not a
measured one** -- the single largest fidelity difference from `companies.csv`,
which carries DENUE's real coordinates. Nothing about the census tables
supports doing better; sub-municipio geography is not published here any more
than it is for DENUE's excluded sectors.

### Establishment size: no band exists, so one is modelled

DENUE publishes a bounded `per_ocu` band per establishment (`0 a 5 personas`
.. `251 y más personas`), and `companies.md`'s within-band uniform draw
allocates a headcount without ever inventing the *shape* of the size
distribution -- the band itself is measured. The Censo Agropecuario publishes
no equivalent: `ca2022_mo01.csv`/`mo02.csv` give only a municipio total on
each side, nothing that splits `UP_REMUN` production units into size classes.

There is, in other words, no observed distribution to draw within here --
only a total to split. The method: each production unit in a municipio draws
a Pareto(shape=1.2) weight -- the same shape and the same "firm sizes are
Zipf-like" rationale `build_companies.py` uses for DENUE's open top band --
and the municipio's `MO_REMUN` total is apportioned across those weights by
the largest-remainder method, which sums to the total exactly by
construction, with a floor of 1 worker per establishment (a `UP_REMUN` unit
is remunerated by definition).

**This is the single biggest modelled assumption in this file.** Unlike
DENUE's top band, where only the tail above 251 is invented and everything
below is a measured band, *every* establishment's size here is invented --
only the municipio total is real. An early version of this draw had no upper
bound and put 47,305 of Tlanchinol's (`MX13073`) 57,501 paid workers on one
synthetic establishment among 3,870 -- implausible for a single production
unit. The fix reuses `build_companies.py`'s own truncation point,
`EMPLOYEE_CAP = 10000`, clipping any establishment that would exceed it and
re-apportioning the overflow among the establishments still under the cap.
The shape (1.2) and the cap (10,000) are both assumptions, isolated as
`PARETO_SHAPE` and `EMPLOYEE_CAP` for sensitivity testing, exactly as
`companies.md` isolates `TOP_BAND_SHAPE` and `TOP_BAND_MAX`.

### Three edge-case municipios

`El Salto` (`MX14070`), `Guadalupe` (`MX19026`) and `Monterrey` (`MX19039`)
report a blank `UP_REMUN` cell (treated as 0, consistent with how INEGI
blanks near-zero agricultural cells in fully urban municipios) alongside a
small nonzero `MO_REMUN` (155, 9 and 5 respectively -- 169 workers combined).
Rather than discard them, each municipio gets one synthetic establishment
holding its whole total, the same treatment `distribute()` gives any
municipio with exactly one remunerated production unit. At this magnitude
(169 of 17,604,378 workers, 0.001%) the choice is immaterial; it is handled
explicitly rather than silently dropped, following this project's convention
of naming small corrections rather than absorbing them.

## 3. Transformation

Per municipio, from the positionally-joined `(mo01, mo02)` pair:

```
geo_unit        = "MX" + E03 + CVE_MUN                     (from mo01, real codes)
n_establishments = round(UP_REMUN)                          (from mo01)
total_employees   = round(MO_REMUN)                          (from mo02, positional join)
```

Then, for `n_establishments` synthetic production units:

```
sizes          = Pareto(shape=1.2)-weighted, largest-remainder split of
                 total_employees across n_establishments, floor 1, cap 10,000
name           = "agro_" + geo_unit[2:] + "_" + zero-padded index
industry_code  = "11"
sizeband       = a bucket label over the drawn size (1-5 / 6-10 / 11-30 /
                 31-50 / 51-100 / 101-250 / 251+) -- for provenance only,
                 same boundaries as DENUE's bands for comparability, but
                 derived from the draw, not sourced text like DENUE's
latitude,
longitude      = municipio centroid (coord_sgu.csv) + seeded jitter (±0.05°)
```

Draws are seeded (`SEED = 20220930`, the census reference period's end,
mirroring `companies.md`'s use of the census reference date), so the file is
reproducible.

### Output schema

Identical to `companies.csv`, so the two files can be loaded together by the
same venue loader and concatenated if a config wants one combined sector-11
pool:

```
geo_unit,name,industry_code,sizeband,employee_count,latitude,longitude
MX01001,agro_01001_000000,11,1-5,2,21.838313100901612,-102.25606553090589
MX01001,agro_01001_000001,11,6-10,8,21.853217179913273,-102.33472960424757
MX13073,agro_13073_000019,11,251+,10000,21.028384801734234,-98.60953244899848
```

### National output

2,514,432 establishments across 2,456 municipios, 17,604,378 total employee
capacity, `industry_code = 11` throughout. 6 municipios present in the census
were dropped for having no match in `geography/hierarchy.csv`; 11 hierarchy
municipios have no row in the census at all. See Validation and Limitations.

Capacity by sizeband:

| sizeband | establishments |
|---|---|
| 1-5 | 1,983,507 |
| 6-10 | 309,963 |
| 11-30 | 163,308 |
| 31-50 | 26,852 |
| 51-100 | 17,649 |
| 101-250 | 9,025 |
| 251+ | 4,128 |

## 4. Validation

- `employee_count` sums to exactly 17,604,378 across all rows -- matching
  `ca2022_mo02.csv`'s national `MO_REMUN` for the 2,458 municipios retained,
  to the last worker. The largest-remainder apportionment and the
  cap-and-redistribute step are both exact by construction; this was checked
  by direct summation, not assumed.
- Every `geo_unit` resolves to a municipio in `geography/hierarchy.csv`; no
  orphans (the 6 census-only municipios are dropped before being written, not
  passed through).
- `employee_count` is >= 1 and <= 10,000 on every row -- the floor and the cap
  both hold with no exceptions.
- `industry_code` is `"11"` on all 2,514,432 rows.
- Latitude/longitude fall inside Mexico's bounds (lat 14-33, long -118 to -86)
  on every row with coordinates.
- Positional join verified independently (see §1): 0 mismatches across 2,464
  municipio rows, including all Oaxaca name-ambiguous pairs.
- `MO_REMUN >= UP_REMUN` holds for every one of the 2,458 municipios with
  `UP_REMUN > 0` -- checked directly, 0 violations, before the 3 edge-case
  municipios are folded in separately (§2).

### Coverage gain against `companies.csv`

| | DENUE sector 11 (companies.csv) | this file |
|---|---|---|
| establishments | 22,227 | 2,514,432 |
| capacity | 384,173 | 17,604,378 |
| municipios with sector-11 capacity | 1,072 | 2,456 |

Combined, sector 11 now has capacity in **2,466 of 2,469** municipios (up
from 1,072), a gain of 1,394 municipios that had zero sector-11 capacity in
`companies.csv` alone. 10 municipios have DENUE sector-11 establishments but
no row in this file -- plausibly coastal fishing/aquaculture activity
(`companies.md`'s own example row is `Piscicultura en agua dulce`, freshwater
fish farming), which sits in SCIAN sector 11 but may fall outside the Censo
Agropecuario's *agropecuaria* (crop and livestock) universe. Not
investigated further; both files should be loaded together so DENUE's real
establishments are not displaced by this one.

## 5. Limitations

- **Every establishment size is modelled, not measured.** See §2. The
  Pareto shape (1.2) and cap (10,000) are the most defensible place to
  attack this file, exactly as `companies.md` names its own top-band
  assumptions as its weakest point.
- **No coordinates exist in the source.** Placement is municipio centroid
  plus jitter, not a real address. Any analysis sensitive to establishment
  *location* within a municipio (rather than just which municipio) should
  not treat these coordinates as informative.
- **Forestry and fishing are not covered.** The Censo Agropecuario ships a
  separate *forestal* (forestry) universe (`for*` tables, catalogued in
  `indice_forestal.csv`) with no equivalent "Mano de obra" labour table
  found among them, and does not appear to cover fishing/hunting at all,
  despite both sitting inside SCIAN sector 11. This file covers the
  *agropecuaria* (crop + livestock) universe only. Sector 11's true capacity
  is therefore still understated here, just far less severely than by
  `companies.csv` alone.
- **`MO_REMUN` and `SITTRA`-eligible workers are not the same population, and
  were not cross-checked against each other.** `companies.md` validates
  DENUE capacity against census `SITTRA` 1+4 (employee/patrón) demand at the
  person level. This file's capacity comes from the agricultural census's
  own paid-labour concept (`MO_REMUN`), a different instrument measuring a
  related but not identical population. Whether the two reconcile the way
  DENUE and `SITTRA` do in `companies.md` §2 has not been checked here, and
  would be the natural next validation step before relying on this file's
  capacity numbers as tightly as `companies.md`'s.
- **6 municipios are dropped for the same reason `companies.md` documents for
  DENUE**: they were created after the geography this build targets.
  `MX02007` San Felipe, `MX04013` Dzitbalché and `MX12082`-`MX12085` (four
  municipios split from Guerrero) appear in the 2022 agricultural census but
  not in `geography/hierarchy.csv`'s 2020 vintage -- the same municipios
  `companies.md` names as created after the 2020 census, now confirmed
  independently from a third source.
- **11 hierarchy municipios have no row in the census at all** and so get no
  agriculture venues from this file (they may still have DENUE sector-11
  establishments in `companies.csv`). Not investigated individually; most are
  plausibly dense urban municipios with no active agricultural production
  unit, consistent with the pattern already seen at Guadalajara, Monterrey,
  Guadalupe and El Salto (§2).
- **Capacity is not calibrated to `work_sector`'s demand.** As with
  `companies.md`, establishment sizes are drawn independently per municipio,
  not fitted so that (municipio, sector-11) totals reproduce a known
  employment margin. Whether this file, added to `companies.csv`, closes the
  3,285,364-worker sector-11 shortfall `companies.md` measures has not been
  tested in an engine build.
