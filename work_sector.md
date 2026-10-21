# Work sector by workplace municipio and sex (Mexico 2020)

How `data/Mexico_data/activities/work/MX_industry_sex_municipio.csv` is produced,
and why each decision was taken. Built by
`data/Mexico_data/source/build_industry_sex.py`.

This is the Mexican counterpart of `EW_industry_sex_lad.csv`,
which the engine's `work_sector_assignment` step samples to give each worker a
sector.

## Why this file is load-bearing

The company distributor matches a person's `work_sector` against an
establishment's `industry_code`. Without a sector, that match has nothing to work
with and **no worker is assigned to any company** — the whole workplace layer is
inert. This file is the input that makes `companies.csv` usable.

## 1. The source

**Censo de Poblacion y Vivienda 2020, Cuestionario Ampliado, person records** —
the same archives as the commuting matrix, downloaded from:

```
https://www.inegi.org.mx/contenidos/programas/ccpv/2020/microdatos/Censo2020_CA_{slug}_csv.zip
```

Columns used, from `Personas<NN>.CSV` (latin-1):

| column | meaning | values |
|---|---|---|
| `CONACT` | condition of activity | employed set below |
| `ACTIVIDADES_C` | **activity of the employing business** | 4-digit SCIAN, `9999` = not specified |
| `MUN_TRAB` | municipio of work | `001`..`570`, `999` = unspecified |
| `ENT_PAIS_TRAB` | entidad or country of work | `001`..`032`, `>032` foreign, `999` unspecified |
| `SEXO` | sex | `1` = hombre, `3` = mujer |
| `FACTOR` | expansion weight | integer |

`ACTIVIDADES_C` is the establishment's activity, not the person's occupation —
`OCUPACION_C` is the separate occupation variable and is not used here. In
Aguascalientes it takes 167 distinct values, e.g. `3360` (vehicle manufacturing),
`1110` (agriculture), `4611` (food retail), `7221` (restaurants), `6112`
(education).

## 2. Interpretation

### Employed set

`CONACT in {10, 13, 14, 15, 16, 18, 20}`, matching the ITER `POCUPADA` definition
and the commuting matrix build.

### Keyed on workplace, not residence

A worker's sector is a property of the place they work. England keys on the
workplace LAD (`LGU_name` matched against `person.properties["workplace_location"]`),
and the Mexican file does the same at municipio. `workplace_location` is assigned
by the commuting step, which runs first.

**This is why the build must cover all 32 states even to model one.** Each state
archive holds *residents*. Keyed on workplace, a partial run gives a municipio only
the workers who happen to live in the states processed, and silently omits everyone
commuting in from elsewhere. Running `ags` alone produces 154 rows — 77 workplace
municipios across the country, because Aguascalientes residents work in all of them
— and `MX01001`'s own row would be missing every in-commuter from Zacatecas and
Jalisco.

The commuting matrix does **not** share this hazard: it is keyed on origin, so a
state's own file holds all of its residents' flows and a per-state run is valid.
The asymmetry is easy to miss.

### Sex

`SEXO` `1` -> `male`, `3` -> `female`, matching `person.sex`, which the engine
normalises to the lowercase strings `'male'` / `'female'`
(`may/population/population.py:163,171`). England's file uses the same two values.

The mapping was verified rather than assumed. Weighted Aguascalientes workers by
`SEXO` against ITER `POCUPADA_M` / `POCUPADA_F`:

| | vs ITER male | vs ITER female |
|---|---|---|
| `SEXO=1` (372,045) | **92.2%** | 128.6% |
| `SEXO=3` (257,515) | 63.8% | **89.0%** |

Both diagonal figures sit just under 100%, consistent with the known ~9%
extended-questionnaire shortfall. The reversed mapping is arithmetically
impossible.

### Sector vocabulary

Truncated to the **2-digit SCIAN sector**, with `31`/`32`/`33` grouped as `31-33`
and `48`/`49` as `48-49` — 20 sectors. Identical treatment on the DENUE side, so
`work_sector` and `industry_code` share one vocabulary. DENUE codes to 6 digits,
the census to 4; both truncate cleanly.

### `9999` "No especificado" is dropped

`ACTIVIDADES_C = 9999` is ~0.9% of weighted workers. It is dropped rather than
carried as a 21st sector, for a concrete reason: **no establishment carries sector
99**, so a worker assigned to it could never be matched to a company and would
fail workplace assignment for a measurement artefact rather than a modelling
reason.

Dropping renormalises the remaining sectors proportionally, since the engine
normalises each row. This follows the treatment of census "no especificado"
categories elsewhere in this world (dropped for households at 0.22%, redistributed
proportionally for demographic ages).

### Foreign and unspecified workplaces are dropped

A worker whose workplace is outside Mexico or unspecified (~0.5% combined) has no
municipio key and contributes to no row. They are excluded from the commuting
matrix by `exclude_destinations` anyway, so they never reach this step.

## 3. Transformation

For every person with `CONACT` in the employed set, a resolvable Mexican workplace
municipio, a specified sector, and a known sex:

```
key    = ("MX" + ENT_PAIS_TRAB + MUN_TRAB, "male" | "female")
sector = 2-digit SCIAN of ACTIVIDADES_C, with 31-33 and 48-49 grouped
```

Accumulate `FACTOR` into `table[key][sector]`, then emit one row per key with a
column per sector.

### Output schema

```
SGU_workplace_code,Sex,Total,11,21,22,23,31-33,43,46,48-49,51,52,53,54,55,56,61,62,71,72,81,93
MX01001,female,188990,1013,0,426,2490,28607,3645,42155,1477,1712,4018,1968,6629,59,7788,17720,14626,1443,19532,23028,10654
MX01001,male,253551,5474,367,1325,31243,58741,10159,37726,15240,2532,2852,2027,11565,22,9624,9650,6846,3240,14622,18478,11818
```

`Total` is the row sum, retained for provenance; the engine normalises. Sector
columns are named by SCIAN code, not by label — England's long human-readable
headers (`Agriculture; Forestry; Fishing`) exist because its config maps them via
`value_columns`, but the engine imposes no vocabulary and codes are unambiguous.

The sample row above shows the file working as intended: in Aguascalientes city,
construction (`23`) is 31,243 male against 2,490 female, while health (`62`) is
14,626 female against 6,846 male. Sector is strongly sex-dependent, which is the
whole reason the England file keys on sex and this one does too.

### National output

4,934 rows — exactly 2,467 municipios x 2 sexes, no municipio missing a sex.
50,291,926 weighted workers. 382 KB.

Two hierarchy municipios have no row: `MX04012` Seybaplaya and `MX07125` Honduras
de la Sierra, the two that are absent from every source. See the cross-source
table in [README.md](README.md).

## 4. Validation

- `Total` equals the sum of the 20 sector columns in **every** row; no row totals
  zero.
- Every `SGU_workplace_code` resolves to a municipio in `geography/hierarchy.csv`;
  no orphans.
- `SEXO` mapping verified against ITER `POCUPADA_M` / `POCUPADA_F`, above.

### It joins to `companies.csv`

The point of this file is that `work_sector` must match an establishment's
`industry_code`. Cross-tabulating both sides on (municipio, sector):

| | |
|---|---|
| (municipio, sector) cells with workers | 42,449 |
| of those, cells with **zero** establishments | 6,554 |
| workers in unmatched cells | 2,302,843 of 50,291,926 = **4.6%** |

**95.4% of workers land in a (municipio, sector) cell containing at least one
establishment**, which confirms the two SCIAN vocabularies reconcile — a mismatch
would have shown up as a near-total failure, not 4.6%.

The 4.6% that do not match are exactly the predicted gap, and are concentrated
where DENUE's coverage is known to be structural rather than accidental:

| sector | unmatched workers | share of that sector |
|---|---|---|
| 11 Agricultura | 1,793,316 | **32.5%** |
| 23 Construccion | 413,857 | 10.5% |
| 21 Mineria | 30,124 | 13.7% |
| 48-49 Transportes | 26,750 | 1.1% |
| 55 Corporativos | 5,439 | 24.2% |
| 46 Comercio al por menor | 5,928 | 0.1% |

Retail at 0.1% is the control: where DENUE covers an activity, coverage is close
to complete. Agriculture at 32.5% is INEGI's documented exclusion of agricultural
activities showing up end to end.

Note this counts cells with *at least one* establishment, not sufficient capacity.
Total demand here (50.3M) also overstates what actually competes for companies,
since only employees at fixed premises are eligible — 34,238,074 nationally
(65.6% of workers), against 42,927,972 of capacity.

Once capacity is taken into account rather than mere presence, the national
shortfall rises from 4.6% to **21.3%** of employees. See
[companies.md](companies.md).

### In the engine

The Aguascalientes world build assigns `work_sector` to **635,779 people with 0
failures**, and the company matcher indexes exactly **20 unique value
combinations** — the vocabulary reconciles at runtime, not just in this analysis.

## 5. Limitations

- **Counts are extended-questionnaire estimates**, running ~9-14.5% below ITER for
  the reasons documented in [work_origin_destination.md](work_origin_destination.md).
  Only the within-row distribution is consumed.
- **Thin cells in small municipios.** A municipio-by-sex row in a small workplace
  municipio may rest on few sampled workers spread over 20 sectors. Mitigated by
  the extended questionnaire being applied *completely* in small municipios
  (`FACTOR = 1` for 67.1% of Oaxaca records; 392 of 570 Oaxaca municipios fully
  enumerated), so the thinnest rows are counts rather than estimates. No smoothing
  is applied.
- **Sectors 11 and 23 have workers but almost no establishments.** This file will
  faithfully assign agriculture and construction sectors to workers that
  `companies.csv` cannot then house, because DENUE excludes agricultural
  activities and building sites are not registrable addresses. See
  [companies.md](companies.md).
- **England's config declares `workplace_sgu` as a required attribute.** Mexico has
  no such level; the Mexican `work_sector_assignment` must key on
  `workplace_location` alone.
