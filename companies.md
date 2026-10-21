# Company venues (Mexico 2020)

How `data/Mexico_data/venues/primary_activities/companies.csv` is produced, and
why each decision was taken. Built by
`data/Mexico_data/source/build_companies.py`.

This is the Mexican counterpart of `data/venues/primary_activities/companies.csv`
for England/Wales, which the engine loads as the `company` venue type.

## 1. The source

**DENUE — Directorio Estadistico Nacional de Unidades Economicas.**

DENUE is INEGI's register of economic establishments: a census of business
*premises*, not of workers. Every establishment carries an address, geographic
keys down to block level, coordinates, a SCIAN activity code, and an employment
size band.

### Vintage, and why it dictates the download path

The build uses the **April 2020** release — the one closest to the census
reference date of 15 March 2020, two weeks apart. `fecha_alta` in these files
stops at `2020-04`, so no post-census establishment can leak in.

Getting a dated release is not straightforward, and the obvious path is a trap.
DENUE is published in two cuts:

| cut | path | vintages available |
|---|---|---|
| **by entidad** — one file per state | `/masiva/denue/denue_<NN>_csv.zip` | **current release only** |
| **by activity** — one national file per SCIAN sector | `/masiva/denue/<YYYY>_<MM>/denue_00_<sector>_<MMYY>_csv.zip` | **every release back to 2015** |

The by-entidad path is the one that surfaces in search results and documentation,
and it silently serves whatever the current release happens to be — at time of
writing, 20/05/2026. There is no way to pin it to a date. Only the by-activity
cut is archived, so **this build iterates 25 sectors rather than 32 states**.

The 20 archived vintages are `2015`, `01/2016`, `10/2016`, `03/2017`, `11/2017`,
`03/2018`, `11/2018`, `04/2019`, `11/2019`, **`04/2020`**, `11/2020`, `05/2021`,
`11/2021`, `05/2022`, `11/2022`, `11/2023`, `05/2024`, `11/2024`, `05/2025`,
`05/2026`.

These paths are not documented or guessable. They were recovered from the
bulk-download application's own JSON API:

```
POST https://www.inegi.org.mx/app/api/descarga/descarga/descargamasiva/lista/obtenerarchivos
Content-Type: application/json
{"tinfo":"6","ag":"00","prog":"0","cc":"0","subtema":"0","anio":"0","formato":"0",
 "datosAbiertos":"0","textoBuscar":"<base64 of empty string>","ingles":"0",
 "tipoInfo":"OTROS","titulo":"<base64 of 'Otros|DENUE|Actividad economica|'>"}
```

which returns 484 entries, each with a `pathLogico` and an `anioInfo` vintage
stamp. Prefix `pathLogico` with `https://www.inegi.org.mx/contenidos` and suffix
`_csv.zip` to get a download URL. The `titulo` and `textoBuscar` fields are
base64-encoded and the request fails opaquely ("Ocurrio un error, ver los
detalles") if any field is missing.

### The files

25 national files, one per SCIAN sector, ~420 MB total. Large sectors are split
by INEGI, either by SCIAN branch (`comercio al por menor` into `46111`,
`46112-46311`, `46321-46531`, `46591-46911`) or by an arbitrary part number
(`72_1`/`72_2`, `81_1`/`81_2`). Those splits do not match the sector grouping
used downstream, so files are concatenated and regrouped.

Each archive holds `conjunto_de_datos/denue_inegi_<sector>_.csv` plus a data
dictionary. Files are **latin-1**, comma separated, quoted, one row per
establishment. The 2020 files have **41 columns**; the current release has 42,
having since gained `clee`. Every column this build reads exists in both.

### What the raw rows look like

42 columns. A single row, reformatted:

```
id          11845140
nom_estab   ABONO DE LOMBRIZ EL AGUILA
codigo_act  112515
nombre_act  Piscicultura en agua dulce
per_ocu     0 a 5 personas
cve_ent     01          entidad     Aguascalientes
cve_mun     009         municipio   Tepezala
cve_loc     0003        localidad   El Aguila
ageb        0034        manzana     001
latitud     22.19551252 longitud    -102.25728565
tipoUniEco  Fijo        fecha_alta  2025-05
```

The columns this build uses are `id`, `codigo_act`, `per_ocu`, `cve_ent`,
`cve_mun`, `latitud`, `longitud`.

`per_ocu` bands, with Aguascalientes counts (71,871 establishments):

| band | establishments |
|---|---|
| `0 a 5 personas` | 60,510 |
| `6 a 10 personas` | 5,959 |
| `11 a 30 personas` | 3,683 |
| `31 a 50 personas` | 708 |
| `51 a 100 personas` | 492 |
| `101 a 250 personas` | 312 |
| `251 y mas personas` | 207 |

Geographic completeness in Aguascalientes is 100% on every one of `latitud`,
`longitud`, `cve_mun`, `cve_loc`, `ageb`, `manzana` — no gaps to handle.

## 2. Interpretation

### DENUE counts premises; the census counts people

This distinction drives everything below. DENUE is a directory of **fixed
establishments**. It does not contain street vendors, domestic workers,
agricultural labourers, building-site crews, or unregistered micro-businesses,
because those have no registrable premises. It therefore cannot host, and is not
meant to host, the whole Mexican workforce.

Measured against the census for Aguascalientes:

| | workers |
|---|---|
| ITER `POCUPADA` (all workers) | 692,983 |
| DENUE capacity if every establishment sat at its band **minimum** | 267,286 |
| DENUE capacity if every establishment sat at its band **maximum** | 738,730 |

Even at the band maximum, statewide capacity only just exceeds the workforce, and
in 6 of 11 municipios it falls *below* it (as low as 0.49x). At any realistic
within-band draw, capacity is roughly 72% of the workforce.

That gap is not an error in DENUE. It is informality, and the census can name it.
`SITTRA` (position in occupation, `V364`) splits Aguascalientes' workers:

| code | meaning | weighted | share |
|---|---|---|---|
| 1 | Empleada(o) u obrera(o) | 447,764 | 71.1% |
| 5 | Trabajador(a) por cuenta propia | 105,859 | 16.8% |
| 4 | Patron(a) o empleador(a) | 24,959 | 4.0% |
| 3 | Ayudante con pago | 20,891 | 3.3% |
| 2 | Jornalera(o) o peon(a) | 18,210 | 2.9% |
| 6 | Trabajador(a) sin pago | 11,877 | 1.9% |

Employees plus employers (`SITTRA` 1 and 4) total **472,723**, which sits
comfortably inside DENUE's 267k–739k envelope. The apparently missing 28% of
capacity is own-account workers, day labourers and unpaid family workers who
genuinely do not occupy a fixed establishment.

**Consequence: `companies.csv` is a faithful list of premises, and eligibility is
a config concern, not a data concern.** The venue file is not filtered or
inflated to match the workforce. Restricting who competes for these places
(`SITTRA` 1 and 4) belongs in the company distributor's filters.

### The gap is sector-specific

Rolling both sides to SCIAN sector for Aguascalientes:

| sector | census workers | DENUE establishments |
|---|---|---|
| 11 Agricultura | 27,568 | **49** |
| 23 Construccion | 48,537 | **491** |
| 48-49 Transportes | 22,128 | **505** |
| 31-33 Manufacturas | 137,423 | 6,428 |
| 46 Comercio al por menor | 106,671 | 26,925 |
| 72 Alojamiento y alimentos | 45,471 | 11,301 |
| 81 Otros servicios | 54,087 | 11,596 |

Agriculture, construction and transport are structurally under-covered: a farm
labourer works a field, a building crew works a site, a driver works a road —
none of which is a DENUE address. Retail, manufacturing and food services are
well covered. Any workplace-contact result for sectors 11, 23 and 48-49 rests on
premises that mostly do not exist in the register.

This is by design rather than by omission. INEGI's own catalogue entry for DENUE
states that the directory covers active economic units **excluding agricultural
and forestry activities** and certain transport and political categories
(`https://www.inegi.org.mx/rnm/index.php/catalog/587`). The measured sector-11
gap is that exclusion showing up in the data, not a defect in this build.

### Sector vocabulary

The person's assigned sector and the establishment's `industry_code` must share
one vocabulary or the distributor matches nothing. DENUE codes activity to
6-digit SCIAN (`codigo_act`); the census codes it to 4-digit SCIAN
(`ACTIVIDADES_C`). Both are truncated to the **2-digit SCIAN sector**.

Two sectors span several 2-digit prefixes and are grouped back together:

- `31`, `32`, `33` -> `31-33` (Industrias manufactureras)
- `48`, `49` -> `48-49` (Transportes, correos y almacenamiento)

This yields 20 sectors. Cross-checked on Aguascalientes: DENUE covers every
sector present in the census; the census additionally has `99` (No especificado,
5,824 workers) which has no DENUE counterpart by construction.

The engine imposes no sector vocabulary — `industry_code` values are free
strings, matched by the distributor's `matching_rules`. England's A–Q letters are
its own convention, not an engine requirement, so SCIAN codes are used directly.

### Size band -> headcount

DENUE publishes a band, never a headcount, and the engine does not interpret
`sizeband` — it reads `employee_count` as the capacity column. So each
establishment must be given a concrete headcount.

England's file was inspected as a reference and **deliberately not imitated**.
Its explosion is lumpy and undocumented: of 267,427 companies in band `0-4`,
240,270 sit at exactly 4 and only 7,188 at 1; band `5-9` piles 42,053 of 53,711
at exactly 5. Mass accumulates at band edges in a way no stated method explains.

The method used here instead:

- **Closed bands** (`0 a 5` through `101 a 250`): draw an integer uniformly on
  the band's inclusive range. `0 a 5` is drawn on [1, 5] — an establishment with
  zero people is not a workplace.
- **Open top band** (`251 y mas`): draw from a Pareto tail with shape 1.2,
  truncated at 10,000.

The top band is the consequential choice. Clipping it at the band edge would
erase every large plant, and Aguascalientes has real sites in the thousands.
Large workplaces matter disproportionately for contact modelling, so the tail is
modelled rather than truncated. Firm-size distributions are Zipf-like, hence a
Pareto shape near 1; 1.2 keeps the tail heavy without producing implausible
single sites, and 10,000 bounds it.

Both the shape and the truncation point are assumptions, not measurements. They
are the most defensible place to attack this build, and they are isolated as
`TOP_BAND_SHAPE` and `TOP_BAND_MAX` for sensitivity testing. Only 207 of 71,871
Aguascalientes establishments (0.3%) are affected, but they carry a large share
of employment.

Draws are seeded (`SEED = 20200315`, the census reference date), so the file is
reproducible.

## 3. Transformation

Per establishment:

```
geo_unit       = "MX" + cve_ent + cve_mun
name           = "denue_" + id
industry_code  = 2-digit SCIAN sector of codigo_act, with 31-33 and 48-49 grouped
sizeband       = per_ocu verbatim
employee_count = uniform draw within band; Pareto tail for the open band
latitude       = latitud
longitude      = longitud
```

### Output schema

```
geo_unit,name,industry_code,sizeband,employee_count,latitude,longitude
MX01009,denue_11845140,11,0 a 5 personas,1,22.19551252,-102.25728565
MX01001,denue_6862345,11,11 a 30 personas,13,21.90699188,-102.30980729
```

`geo_unit` is a municipio code and must match `GeographicalUnit.name` exactly.
`name` retains the DENUE `id` so any establishment can be traced back to the
register — England's synthetic `comp_0`, `comp_1` names have no such referent.
`sizeband` is carried for provenance only; the engine ignores it. `industry_code`
and `employee_count` become `venue.properties` entries, since the venue loader
reserves only `name`, `geo_unit`, `latitude`, `longitude` and level names, and
turns every other column into a property.

`ageb` and `manzana` are dropped: they are 100% populated and would support a
sub-municipio level (818 AGEBs vs 11 municipios in Aguascalientes), but no
population or household data exists below municipio, and the census suppresses
the economic variables there anyway. They remain available in the raw register if
a finer geography ever becomes viable.

### National output

5,487,061 establishments across all 32 entidades, 2,465 municipios, 20 sectors.
Total capacity 42,927,972. No malformed `geo_unit`; every one resolves to the
2020 hierarchy.

Checked against the census nationally:

| | workers |
|---|---|
| ITER `POCUPADA` (all workers) | 61,121,324 |
| implied employees at fixed premises (`SITTRA` 1+4, at the Aguascalientes share of 75.1%) | ~45,902,114 |
| DENUE capacity as built | 42,927,972 |

Capacity is **0.94x** the implied employee workforce, and 70.2% of all workers.
That ratio is the intended outcome: the file is sized to employees at premises,
not to the whole workforce, and sits slightly under even that because agriculture
and construction are excluded from the register.

Measured directly against the work-status table rather than the 75.1% estimate,
national employee demand is 34,238,074 against 42,927,972 of capacity — **0.80x**.
The aggregate has room; the shortfall below is entirely a distribution problem.

Per-band draws behave as specified:

| band | establishments | min | max | mean |
|---|---|---|---|---|
| `0 a 5 personas` | 4,760,453 | 1 | 5 | 3.0 |
| `6 a 10 personas` | 359,456 | 6 | 10 | 8.0 |
| `11 a 30 personas` | 247,912 | 11 | 30 | 20.5 |
| `31 a 50 personas` | 49,600 | 31 | 50 | 40.5 |
| `51 a 100 personas` | 32,434 | 51 | 100 | 75.6 |
| `101 a 250 personas` | 21,682 | 101 | 250 | 176.0 |
| `251 y mas personas` | 15,524 | 251 | 9,992 | 799.1 |

### Aggregate capacity is sufficient; its distribution is not

Capacity at 0.80x of demand is a national total, and totals hide the problem.
Establishments must match a worker in **both** municipio and sector, and on that
join the register runs out:

| | employees |
|---|---|
| national employee demand | 32,974,876 |
| with no establishment in their municipio *and* sector | **7,038,981 = 21.3%** |

By sector:

| sector | employees with nowhere to go |
|---|---|
| 11 Agricultura | 3,285,364 |
| 23 Construccion | 1,803,339 |
| 48-49 Transportes | 623,645 |
| 31-33 Manufacturas | 375,965 |
| 81 Otros servicios | 188,647 |
| 54 Servicios profesionales | 188,449 |

Agriculture and construction are 72% of it, which is the register's documented
exclusions and the nature of the work — a field and a building site are not
addresses.

**Aguascalientes understates this badly.** The same calculation for the test state
gives 12.8%, against 21.3% nationally, because Aguascalientes is urban and
industrial while agriculture dominates the national gap. Any workplace result
generalised from the test world inherits that optimism.

### What the vintage change cost and bought

Against the previously-built 2026 release: **-10.6%** establishments (6,138,075 ->
5,487,061), six years of business formation removed. Capacity barely moved
(43,340,954 -> 42,927,972, -1.0%) because the 2020 file carries *more*
establishments in the open top band (15,524 vs 13,198) — the loss is concentrated
in micro-establishments, which contribute little capacity each.

The gain is correctness: zero orphan municipios (was nine), and an establishment
geography contemporaneous with the population rather than six years ahead of it.

## 4. Validation: the Aguascalientes world build

`create_world.py --config configs/Mexico_2020/config.yaml`, exit 0.

```
company venues loaded    61,324   (5,425,737 filtered out as out-of-boundary)
sector index built       20 unique value combinations
eligible (employees)    477,060
allocated               417,295 (87.5%)
unplaced                 59,765   (58,060 + 1,705 redistributed -> outside)
```

The matcher building exactly **20** value combinations confirms the SCIAN
vocabulary reconciles inside the engine, not merely on paper: `work_sector` and
`industry_code` index against each other with no unmatched values.

The 12.5% unplaced is not a defect in the configs. Predicting it independently
from the source tables — comparing employee demand against capacity per
(municipio, sector) — gives **12.8%**, against the build's **12.5%**, and the
shortfall lands in the same sectors:

| sector | predicted shortfall |
|---|---|
| 23 Construccion | 23,207 |
| 11 Agricultura | 20,674 |
| 81 Otros servicios | 5,040 |
| 31-33 Manufacturas | 3,687 |

Nationally the same calculation gives 21.3% (above), so a full-country build
should be expected to leave roughly a fifth of employees without an
establishment. That is the register's coverage, faithfully represented.

## 5. Limitations

### Four municipios have no establishments

Every `geo_unit` in the file resolves to a municipio in the 2020 hierarchy — the
April 2020 vintage contains no post-census municipios, so there are no orphans.
The error runs the other way: four hierarchy municipios get zero establishments,
because DENUE's April 2020 catalogue still codes them under their parent.

| municipio | ITER `POBTOT` | ITER `POCUPADA` |
|---|---|---|
| `MX02006` San Quintin (BC) | 117,568 | 58,585 |
| `MX04012` Seybaplaya (Campeche) | 15,297 | 6,674 |
| `MX07125` Honduras de la Sierra (Chiapas) | 11,650 | 5,302 |
| `MX17036` Hueyapan (Morelos) | 7,855 | 3,774 |

`MX02006` San Quintin matters: it is a live destination in the commuting matrix
with 58,585 resident workers and nothing to host them. See the cross-source table
in [README.md](README.md).

This replaced a worse problem. The 2026 vintage had the opposite defect — nine
municipios created *after* the census (`MX24059` Villa de Pozos, `MX25019`/
`MX25020` in Sinaloa, `MX04013` Dzitbalche, `MX02007` San Felipe, `MX12082`-
`MX12085` in Guerrero) carrying 12,260 establishments that
`filter_by_geography: true` discarded **silently**, with nothing warning that a
venue's geo_unit went unmatched.

- **Capacity is not calibrated to employment.** Establishment headcounts are
  drawn from bands independently; they are not fitted so that municipio totals
  reproduce a known employment margin. Censos Economicos 2019 publishes total
  personnel by sector and municipio and would permit such a calibration.
- **The top-band tail is assumed, not measured.** See above.
- **Sectors 11, 23 and 48-49 are structurally under-covered.** See above.
- **DENUE excludes government activity unevenly.** Sector 93 (actividades
  gubernamentales) appears with 571 establishments against 28,152 census workers
  in Aguascalientes.

### April 2020 is close to the census, but the pandemic sits between them

The census reference date is 15 March 2020; the DENUE release is April 2020. Two
weeks apart, which is as close as the archive allows — but Mexico's first
COVID-19 restrictions (*Jornada Nacional de Sana Distancia*) began 23 March 2020,
between the two. April 2020 fieldwork therefore observed establishments during
the first weeks of the shutdown.

`fecha_alta` records when an establishment entered the register, not whether it
was trading, so a business closed by the shutdown may still appear. The
alternative vintages are worse: `11/2019` predates the census by four months and
misses the shutdown entirely, and `11/2020` postdates it by eight months and
absorbs the first wave of closures. April 2020 is the closest available and the
choice is deliberate, but "closest" is not "contemporaneous".
