# Home-to-work commuting matrix (Mexico 2020)

How `data/Mexico_data/activities/work/MX-work-destination-likelihood.csv` is
produced, and why each decision was taken. Built by
`data/Mexico_data/source/build_work_od.py`.

This is the Mexican counterpart of the England/Wales file
`data/activities/work/EW-work-destination-likelihood.csv`, which the engine
consumes to draw a workplace municipio and a work mode for each worker.

## 1. The source

**Censo de Poblacion y Vivienda 2020, Cuestionario Ampliado (extended
questionnaire), person records.**

INEGI ran the 2020 census with two instruments. The *cuestionario basico* went
to every household and produces the complete-count tabulations (ITER). The
*cuestionario ampliado* went to a subset and asks a longer battery, including
employment and commuting. Only the extended questionnaire records where a person
works.

Downloaded per state:

```
https://www.inegi.org.mx/contenidos/programas/ccpv/2020/microdatos/Censo2020_CA_{slug}_csv.zip
```

with `slug` in `ags, bc, bcs, cam, coa, col, chs, chh, cdmx, dgo, gto, gro, hgo,
jal, mex, mich, mor, nay, nl, oax, pue, qro, qroo, slp, sin, son, tab, tam, tla,
ver, yuc, zac`. Each archive holds `Personas<NN>.CSV`, `Viviendas<NN>.CSV` and
`Migrantes<NN>.CSV`. Files are **latin-1**, comma separated, one row per person.
The same endpoint already backs `build_structure_mixture.py`.

### What the raw rows look like

`Personas01.CSV` has 91 columns. The ones this build uses:

| column | meaning | values |
|---|---|---|
| `ENT` | entidad of residence | `01`..`32` |
| `MUN` | municipio of residence | `001`..`570` |
| `CONACT` | condition of activity | see below |
| `MUN_TRAB` | **municipio of work** | `001`..`570`, `999` = unspecified |
| `ENT_PAIS_TRAB` | **entidad or country of work** | `001`..`032` = entidad, `>032` = foreign country, `999` = unspecified |
| `TIE_TRASLADO_TRAB` | commute duration band | `1`..`7`, `9` |
| `MED_TRASLADO_TRAB1..3` | commute modes (up to three) | `01`..`12`, `99` |
| `FACTOR` | expansion weight | integer |

`CONACT`:

```
10 Trabajo
13 Declares job-seeking, verification recovers that they work
14 Declares pensioner, verification recovers that they work
15 Declares student, verification recovers that they work
16 Declares housework, verification recovers that they work
18 Declares other status, verification recovers that they work
20 Had a job but did not work
30 Sought work          40 Pensioner        50 Student
60 Housework            70 Permanently unable to work
80 Does not work        99 Not specified
```

`TIE_TRASLADO_TRAB`:

```
1 up to 15 min    2 16-30 min    3 31 min to 1 h    4 >1 h to 2 h
5 more than 2 h   6 cannot be determined            7 does not commute
9 not specified
```

Code lists come from the INEGI microdata catalogue, one page per variable:
`https://www.inegi.org.mx/rnm/index.php/catalog/632/variable/F13/<VID>?name=<NAME>`
(`V362` CONACT, `V364` SITTRA, `V374` ACTIVIDADES_C, `V377` TIE_TRASLADO_TRAB,
`V602` MUN_TRAB, `V603` ENT_PAIS_TRAB, `V604` MED_TRASLADO_TRAB1).

## 2. Interpretation

### Who counts as a worker

`CONACT in {10, 13, 14, 15, 16, 18, 20}`. This matches the ITER definition of
`POCUPADA` — worked, or had a job but did not work in the reference week. The
`13`–`18` codes are people who first self-described as students, pensioners or
homemakers and were reclassified as working during verification; omitting them
would drop roughly 8,000 weighted workers in Aguascalientes alone.

### Where they work

Destination is `ENT_PAIS_TRAB` + `MUN_TRAB` combined into an `MX` + entidad +
municipio code. Coverage in Aguascalientes:

| outcome | share of workers |
|---|---|
| a Mexican municipio | 99.4% |
| a foreign country | 0.3% |
| unspecified (`999`) | 0.2% |

### Work mode

The census records commute *duration* and *mode*, not an employment arrangement,
so `Place of work indicator` is derived from `TIE_TRASLADO_TRAB`:

- `7` (does not commute) -> `From_Home`
- anything else -> `Normal`

Two checks support this reading: when `TIE_TRASLADO_TRAB = 7`, `MED_TRASLADO_TRAB1`
is blank in 100% of cases, and the work municipio equals the residence municipio
in 98% of cases.

**England's `Hybrid` category has no Mexican equivalent.** The 2020 census asks
how long the trip to work takes, not how many days a week it is made, so a
hybrid worker is indistinguishable from a daily commuter. The Mexican file
therefore carries only `From_Home` and `Normal`.

### Why municipio and not something finer

Municipio (SGU) is the finest level at which this can be built, for two
independent reasons:

1. INEGI applies disclosure control to the economic variables below municipio. In
   ITER, `PEA` and `POCUPADA` are suppressed (`*`) for **42.5% of localities**
   (82,011 of 193,160) but for **0%** of the 2,469 municipios. `POBTOT` is never
   suppressed at any level — only the economic variables are.
2. The population and household inputs (`demographics_male.csv`,
   `demographics_female.csv`, `households.csv`) are themselves keyed at
   municipio. There is no finer unit to allocate workers to.

### Why codes and not names

England joins its matrix on `LGU_origin_name`. Mexico cannot: Oaxaca contains two
distinct municipios named *San Juan Mixtepec* and two named *San Pedro Mixtepec*.
The Mexican file is keyed on `MX`-prefixed INEGI codes throughout. In the engine
a Mexican `GeographicalUnit.name` **is** the code, so a config reading
`property: "name"` resolves to `MX01001` and matches directly.

## 3. Transformation

Per state file, for every person with `CONACT` in the employed set:

```
origin      = "MX" + ENT + MUN
destination = "MX" + ENT_PAIS_TRAB + MUN_TRAB    (a Mexican municipio)
            | "OUTSIDE_MX"                        (ENT_PAIS_TRAB > 032)
            | "UNSPECIFIED"                       (either field 999 or blank)
mode        = "From_Home" if TIE_TRASLADO_TRAB == 7 else "Normal"
```

Accumulate `FACTOR` into `flows[(origin, destination, mode)]` and into
`origin_totals[origin]`, then emit one row per flow with
`Likelihood = Count / Total_Origin_Workers`.

### Output schema

```
SGU_origin_code,SGU_destination_code,Place of work indicator,Count,Total_Origin_Workers,Likelihood
MX01001,MX01001,From_Home,36146,438864,0.08236264537533268
MX01001,MX01001,Normal,381839,438864,0.8700622516314849
MX01001,MX01005,Normal,6142,438864,0.013995224032957818
```

England's `LGU_origin_name` / `LGU_destination_name` columns are dropped for the
naming reason above. `OUTSIDE_MX` and `UNSPECIFIED` play the role of England's
`888888888` (offshore) and `999999999` (outside UK) sentinels and are intended to
be dropped via `exclude_destinations`.

### National output

93,616 flows over 2,466 origins; 52,250,929 weighted workers; 11.1% `From_Home`.

## 4. Validation

### In the engine

The Aguascalientes world build assigns `workplace_location` and `work_mode` to
**635,779 people with 0 failures**, across 11 unique destination municipios — the
whole state, with out-of-boundary destinations redistributed as configured. That
635,779 is exactly the `employee` + `own_account` population from the work-status
table, so the matrix is being drawn on by precisely the people who work.

Of 5,327 workers whose drawn destination fell outside the loaded state and were
bounced back in, 3,622 were placed and 1,705 became explicit outside-workers with
no venue — the `redistribute` / `deprioritize_flag` path behaving as specified.

### Against the sources

- Likelihood sums to exactly 1.0 for every one of the 2,466 origins.
- Every non-sentinel destination code resolves to a municipio in
  `geography/hierarchy.csv`; no orphan codes.
- Weighted **total population** reconstructs ITER `POBTOT` for Aguascalientes to
  within 0.31% (1,421,198 vs 1,425,607), confirming `FACTOR` is applied correctly.
- Flows are geographically coherent. Largest Aguascalientes flows: Jesus Maria ->
  Aguascalientes 13,240 (22.5% of that municipio's workers), San Francisco de los
  Romo -> Aguascalientes 3,724 (13.8%), Pabellon de Arteaga -> San Francisco de
  los Romo 3,346 (17.4%). Aguascalientes city retains 95.2% of its own workers;
  peripheral municipios retain 58-81%.

## 5. Limitations

### Counts are not comparable to ITER; only shares are

Weighted employment from the extended questionnaire runs **14.5% below** ITER
`POCUPADA` nationally (52,250,929 vs 61,121,324). The gap is not uniform — it is
9.2% in Aguascalientes — so it cannot be corrected with a single scale factor.
Per municipio in Aguascalientes:

| municipio | extended questionnaire | ITER `POCUPADA` | diff |
|---|---|---|---|
| MX01001 | 438,864 | 476,502 | -7.9% |
| MX01003 | 20,048 | 25,597 | -21.7% |
| MX01008 | 2,981 | 3,768 | -20.9% |
| MX01011 | 27,027 | 28,926 | -6.6% |
| **state** | **629,560** | **692,983** | **-9.2%** |

This is not a weighting error — total population reconstructs to within 0.31%.
It is a measurement difference between the two instruments: the extended
questionnaire asks a longer employment battery in a different context and
classifies fewer people as employed.

**Consequence: consume `Likelihood`, not `Count`.** Worker *totals* per municipio
must come from ITER `POCUPADA` (complete enumeration, unsuppressed at municipio);
this matrix supplies only the *distribution* of those workers over destinations.
`Count` and `Total_Origin_Workers` are retained for provenance and to mirror the
England schema.

### Sampling is uneven, but favourably so

The extended questionnaire was applied completely in small municipios and sampled
in large ones. Nationally it holds **15,015,683 person records expanding to
125,515,554** — a 12.0% sample overall — but that average is misleading, because
**26.3% of all records carry `FACTOR = 1`**, i.e. one in four people in the file
was enumerated rather than sampled.

The two extremes: in Oaxaca, `FACTOR = 1` for 67.1% of records and **392 of 570
municipios are 100% enumerated** (median municipio sampling fraction 99.7%,
minimum 5.4%). Aguascalientes, being urban, is sampled at 6.7% with a median
`FACTOR` of 5.

The practical effect is favourable: flows out of small municipios — where thin
cells would otherwise be noisiest — are close to exact, while sampled municipios
are large enough that their flows are stable. No smoothing is applied.

### Three municipios have no rows

`MX04012` (Seybaplaya), `MX07125` (Honduras de la Sierra) and `MX29048` (La
Magdalena Tlaltelulco) appear in `geography/hierarchy.csv` and in ITER with real
worker counts (6,674 / 5,302 / 892) but never appear as a residence municipio in
the extended-questionnaire microdata, so they have no origin row.

They are not silently patched. A national run must either exclude them or accept
that residents of those three municipios fail workplace assignment loudly, which
is the intended behaviour — the engine works on complete data or stops.

### Intra-municipio commuting is invisible

The matrix resolves to municipio. Municipios are large in rural states, so a long
trip that stays inside one municipio is indistinguishable from a short one. This
is a property of the finest geography the data permits, not of the method.
`TIE_TRASLADO_TRAB` is available per person and can recover the duration
distribution if that matters downstream.

## 6. Not yet used

The same person records also carry:

- `MED_TRASLADO_TRAB1..3` — up to three commute modes per person (`01` walking,
  `02` bicycle, `03` metro/light rail, `04` trolleybus, `05` metrobus, `06`
  bus/combi/colectivo, `07` school transport, `08` taxi, `09` app taxi, `10`
  motorcycle, `11` car, `12` other). This is the counterpart of England's
  `mode_transport_by_oa.csv`.
- `ACTIVIDADES_C` (4-digit SCIAN) with `MUN_TRAB` and `SEXO` — sector by
  *workplace* municipio, finer than the state-level ENOE cube.
- `MUN_ASI`, `ENT_PAIS_ASI`, `TIE_TRASLADO_ESCU`, `MED_TRASLADO_ESC1..3` — the
  same home-to-school structure, usable against the 261,842-row school list.
