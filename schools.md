# Schools (Mexico 2020)

How `data/Mexico_data/venues/primary_activities/MX_schools.csv` and
`MX_universities.csv` are produced, and why each decision was taken. Built by
`data/Mexico_data/source/build_schools.py`.

The school list is the venue file: where schools are, how big they are, and who
staffs them. Who attends is [activity_status.md](activity_status.md); where they
attend is [school_origin_destination.md](school_origin_destination.md).

## 1. The source, and what it actually is

**SEP's Catálogo de Centros de Trabajo**, scraped from the SIGED map viewer at
`siged.sep.gob.mx/SIGED/mapas.html`. 261,841 rows.

The provenance matters more than usual, and the file advertises it: the first
column is `placemark_name`, a **KML** concept. This is a map export, not a
statistical release.

| column | meaning |
|---|---|
| `placemark_name` | the CCT (clave de centro de trabajo) — the school's key |
| `NOMBRE DE CENTRO DE TRABAJO` | the school's name |
| `TIPO EDUCATIVO` / `NIVEL` / `SERVICIO` | educational level, three depths |
| `CLAVE TURNO` / `NOMBRE TURNO` | the shift |
| `CLAVE ENTIDAD` / `CLAVE MUNICIPIO` | geography |
| `LATITUD` / `LONGITUD` | coordinates |
| `ALUMNOS TOTAL` / `HOMBRES` / `MUJERES` | enrolment |
| `DOCENTES TOTAL`, `PERSONAL TOTAL` | teachers, all staff |
| `AULAS EXISTENTES` / `EN USO` | classrooms |

### The vintage is wrong, and nothing in the file says so

**A map viewer serves the current catalogue.** It cannot be pinned to a date,
and there is no cycle field to pin it with. This is precisely the trap
[companies.md](companies.md) documents for DENUE's bulk path — the convenient
route silently returns today's data under a 2020 population, with no error.
`build_companies.py` iterates 25 sector archives rather than 32 state files for
exactly this reason. The school scrape took the convenient route.

Two independent lines of evidence date it, and they agree:

**Structural.** The file contains municipios that did not exist in 2020:

| municipio | created | rows |
|---|---|---|
| `MX02007` San Felipe, Baja California | 2021 | 38 |
| `MX04013` Dzitbalché, Campeche | 2021 | 27 |

No 2020-vintage file can contain a municipio created in 2021. (Contrast the five
municipios in [README.md](README.md)'s cross-source table: San Quintín and
Seybaplaya were created in 2020, in time for the Marco Geoestadístico 2020, and
so *are* in `hierarchy.csv`. San Felipe and Dzitbalché came later. The school
file is strictly newer than the geography.)

**Quantitative.** Primaria enrolment, administrative count on both sides, so no
sampling confound:

| | alumnos | vs this file |
|---|---|---|
| this file | 13,058,243 | — |
| SEP, ciclo 2022-23 (published) | 13,345,969 | **-2.2%** |
| SEP, ciclo 2019-20 (implied, -3.8% to 2022-23) | ~13,873,149 | **-5.9%** |

Enrolment has fallen monotonically since 2019-20 (pandemic dropout, private
school exodus, demographic decline), and this file sits *below even 2022-23*, so
it is later still — consistent with a 2025/26 catalogue. It is short roughly
**815,000 primaria students** against the census year.

A national comparison against the census does **not** settle this and should not
be quoted as if it did: census 2020 attendees (`ASISTEN=1`, all ages) are
33,025,464 against the file's 34,486,314, which is *+4.42%* — the wrong sign.
That comparison sets an administrative complete count against a sampled long
form and measures the instruments, not the year.

### Why the file is used anyway

Enrolment is consumed **as-is**, neither rescaled to 2020 nor discarded.
Rescaling would be an imputation, and this world runs on the data it has or
stops. The consequence is a seat shortfall of roughly the vintage gap, which is
predicted from the source and measured against the build rather than hidden —
the same treatment DENUE's coverage gap gets, where a predicted 12.8% unplaced
matched an observed 12.5% and so demonstrated a data property rather than a
config bug.

**Student counts never come from this file.** They come from the census. This is
the standing rule — counts from ITER, shares from the ampliado — applied to a
third source. The school list supplies geography and relative size only.

## 2. Interpretation

### The row is not a school, and not a school × turno

| key | distinct |
|---|---|
| `placemark_name` | 253,133 |
| + `NOMBRE TURNO` | 257,006 |
| + `NIVEL` | 259,948 |
| + `SERVICIO` | 259,950 |

Against 261,841 rows. So a row is roughly a **CCT × turno × nivel** — a
programme offering, not a building. Where a CCT repeats with the same turno it is
usually one premises hosting two *levels* (e.g. `21SDI0033Q` runs inicial and
preescolar at one address), not two shifts.

### Turno: the shift structure is real, but SEP only labels it for básica

```
MATUTINO     207,199    VESPERTINO    17,700    DISCONTINUO  12,353
(blank)       22,882    CONTINUO       1,432    NOCTURNO        275
```

`CLAVE TURNO` adds nothing — it is blank wherever `NOMBRE TURNO` is. And the
blanks are not random:

| tipo educativo | rows | turno labelled |
|---|---|---|
| BÁSICA | 230,432 | 100% |
| MEDIA SUPERIOR | 20,541 | **0%** |
| SUPERIOR | 10,868 | ~0% (8,521 DISCONTINUO, 2,341 blank) |

Turno is unlabelled for **every media superior row** — the level 15-17 year olds
attend, which is exactly the population that works while studying.

**But the shifts are still in the data, encoded as duplicate rows.** Media
superior has 1,743 CCTs with two rows and 74 with three, differing only in
counts — same coordinates, same servicio, same subnivel. Those extra rows are
shifts. The evidence is that they behave statistically like básica's *labelled*
matutino/vespertino pairs:

| paired CCTs | labelled MAT+VESP (básica) | unlabelled 2-row (media superior) |
|---|---|---|
| n | 2,233 | 1,730 |
| larger shift, median alumnos | 562 | **564** |
| smaller shift, median | 306 | 372 |
| smaller/larger ratio | 0.63 | 0.76 |
| combined, median | 862 | 942 |

The larger-shift medians agree to within two students. And multi-row media
superior CCTs have a median of **921** students against **76** for single-row
ones: a second shift is what a school opens when one cannot absorb local demand,
so only large urban schools run them. They hold **39.1%** of media superior
students.

The rows are therefore **kept separate**, one venue each. Two shifts share a
building but never meet, so they are distinct contact groups — which is what a
venue is for.

### Turno does not predict working while studying

This was the leading hypothesis and it does not survive. The census records
`ASISTEN` but **never which shift a student attends**, so the person-level cross
is impossible; the test has to be ecological — do municipios with more shift
capacity show more working students?

| 15-17 students, by municipio (>=500 students, has media superior) | median work-and-study rate |
|---|---|
| municipios **with** multi-shift capacity (518) | 7.10% |
| municipios **without** any (632) | 7.47% |

Pearson r between multi-shift capacity share and work-and-study rate: **+0.051**.
There is a weak gradient across quintiles (6.9% -> 8.6%) but it runs against the
between-group comparison and is consistent with urbanity, not scheduling.

Two further arithmetic checks kill it independently:

- **NOCTURNO is too small by a factor of thirty.** 275 rows, 16,274 students,
  against 498,986 working students nationally. The shift explicitly built for
  working students cannot host 3% of them.
- **There is no variance to predict with.** 85% of básica students are
  `MATUTINO`.

The mechanism is real but **universal**: the ordinary Mexican school day is a
half-day shift, so *every* student — morning, afternoon or single-shift — has
half a day free. Turno decides **when** a student could work, not **whether**.
It therefore belongs in the model as contact timing, and is kept as a venue
property; it is **not** a selector for `student_worker`.

### Venue types: school and university

`TIPO EDUCATIVO` splits BÁSICA (230,432) / MEDIA SUPERIOR (20,541) / SUPERIOR
(10,868). England separates `school` from `university`, and Mexico is mapped onto
that split:

```
school      BÁSICA + MEDIA SUPERIOR    250,973 venues
university  SUPERIOR                    10,868 venues
```

Media superior is upper secondary, ages 15-17 — England's sixth form, which sits
inside its `school` venue. Superior is licenciatura and posgrado: an 18+
population, attendance 22.0% at 18-29 against 71.7% at 15-17, and a commuting
pattern of its own. `NIVEL` is kept as a property, so a distributor can cut
finer without the venue split having pre-empted it.

### `PERSONAL TOTAL` is partly a missing-data sentinel

| column | zeros | of 261,841 |
|---|---|---|
| `ALUMNOS TOTAL` | 965 | 0.4% |
| `DOCENTES TOTAL` | 3,052 | 1.2% |
| `PERSONAL TOTAL` | **32,750** | **12.5%** |

Of those 32,750 zeros, **30,630 report teachers** — and a school with teachers
does not have zero staff. The zero is missing data wearing a count's clothes.
Those cells are emitted as **null**, not as 0: a null is absent, a 0 would be
believed. `DOCENTES TOTAL` is the reliable staff column.

This matters for placing sector-61 (servicios educativos) workers in schools
rather than DENUE establishments. National sector-61 demand is 2,790,616 workers
against 2,133,121 teachers in the file — but that comparison spans the vintage
gap and sector 61 includes staff no school employs, so it is not yet a
reconciliation. Not built; see Limitations.

## 3. Transformation

### Geography: schools are re-keyed onto the 2020 framework

Schools must key to the geography the **population** data uses, which is INEGI's
Marco Geoestadístico 2020, because that is where the students are counted. Where
SEP's filing and the 2020 boundaries genuinely disagree, the coordinates settle
it by point-in-polygon against `geography/boundaries/mexico_sgu.geojson`. This is
not a fallback: it imputes nothing, it reads the answer off the geometry.

Every row was tested, not just the suspicious ones:

| verdict | rows | share |
|---|---|---|
| coordinates agree with the claimed municipio | 258,222 | 98.62% |
| disagree | 3,527 | 1.35% |
| coordinates in no municipio | 81 | 0.03% |
| coordinates outside Mexico | 11 | 0.00% |

The disagreements are not one phenomenon, and are not treated as one:

**Case A — the claimed municipio does not exist in 2020 (72 rows).** The code is
unusable; only the coordinates can place the school. Every one resolves, and the
answers are historically correct:

| claimed | resolves to |
|---|---|
| `MX02007` San Felipe | `MX02002` Mexicali (35), `MX02001` Ensenada (3) |
| `MX04013` Dzitbalché | `MX04001` Calkiní (26), `MX04005` (1) |
| `MX01035/053/064` | `MX01001/01002/01010` |

San Felipe was carved out of Mexicali and Dzitbalché out of Calkiní, so the
geometry recovers the parent it was carved from. The `MX01xxx` three are a
different bug: they carry *Jalisco* municipio names and codes (Encarnación de
Díaz, Lagos de Moreno, Ojuelos de Jalisco) under entidad `01`, but Aguascalientes
has only 11 municipios and their CCT prefixes and coordinates are Aguascalientes.
The names were wrong; the coordinates were right.

**Case B — documented boundary divergence (127 rows).** Both are unanimous about
their destination:

| claimed | rows moving | to | why |
|---|---|---|---|
| `MX07080` Siltepec | 67 of 280 | `MX07125` Honduras de la Sierra | split off in 2019; SEP's catalogue never adopted it |
| `MX23004` Othón P. Blanco (QRoo) | 60 of 523 | `MX04010` Calakmul (Campeche) | the Campeche/Quintana Roo border dispute — INEGI and the state education authority disagree about the same ground |

**Case C — everything else (3,328 rows, 1.27%): the SEP code is kept.** These are
geocoding error, not geography, and the shape of the data says so. Of 1,602
distinct claimed->destination pairs, **965 are a single row**; only 28 have 10 or
more. A boundary change moves every school in the affected area together and
agrees on where. These scatter — up to four destinations from one claimed
municipio — and cluster in dense metros (`MX21041/21114/21119/21140` shuffling
around Puebla city, `MX01005` Jesús María -> `MX01001` Aguascalientes), where a
school sits metres from a municipio line and a slightly-off geocode lands across
it. No boundary changed in any of them.

Trusting coordinates blanketly would relocate 3,328 schools on the strength of
bad geocoding. The coordinates are demonstrably fallible: 11 rows are outside
Mexico entirely (`lon=0`, `lon=-963.543`, `lat=44.0`). Those 11 have their
coordinates blanked and keep their claimed municipio — the school exists, its
location is simply unknown.

### Column names are normalised, and this is not cosmetic

`venue_manager` reads properties with `getattr(row, col)` over `itertuples()`
(`may/geography/venue_manager.py:307`). Pandas renames any column that is not a
valid Python identifier to `_N`, so a column called `ALUMNOS TOTAL` resolves to
`None` and is **silently dropped** — no error, just a missing property. Every
header is emitted as identifier-safe snake_case.

This is the same family as the null-skipping filter foot-gun in
[README.md](README.md): the engine does the wrong thing quietly.

### School names

SEP stores names in caps and inconsistently accented (`BENITO JUAREZ`, but
`NIÑOS HEROES` with the tilde). They are normalised to Spanish title case:
whitespace collapsed (604 names have double spaces), Spanish connectors kept
lowercase inside the name (`Miguel Hidalgo y Costilla`), roman numerals preserved
(`Plantel IV`), institutional acronyms preserved (`CONALEP`, `CENDI`), and the
first *letter* capitalised rather than the first character — `str.capitalize()`
would turn `"LIC. BENITO JUAREZ"` into `"lic. benito juarez"` by uppercasing the
quote. **Accents are left as they arrive**; they cannot be restored from the
source and guessing them would be invention.

The name is a **property**, not the venue's identity: 84,318 distinct names cover
250,973 venues, and 3,180 schools are called `BENITO JUAREZ`. `name` is the CCT,
which is unique and joins back to SEP.

### Two derived columns

**`shifts_at_site`** — how many cohorts share this premises at this level, this
one included. This is what the duplicate-row finding is worth: `>1` means the
venue shares a building with another cohort it never meets. Media superior has
3,486 rows at two shifts and 222 at three; básica's 4,470 two-shift rows recover
the 2,233 labelled matutino/vespertino pairs exactly.

**Which** shift a row is stays unrecorded, and that is deliberate. The file does
not say, and it cannot be recovered. Size would guess it: on básica's labelled
pairs the larger cohort is the morning one **96.9%** of the time (2,164 of
2,233). That accuracy is not usable, for three reasons:

- it is measured on the only population that *has* labels, and would be applied
  to the one that has none — whose shift economics may differ, an afternoon
  bachillerato serving working teenagers being exactly where "bigger is morning"
  should fail;
- the 74 three-row sites have no ground truth at all. Básica has **zero** CCTs
  with three labelled turnos, and their counts (`93/0/5`, `246/10/48`) are not
  shift-shaped — a cohort of 0 is not a shift;
- nothing consumes turno. It does not predict working while studying
  (r = +0.051), so a label would buy nothing today and would have to be unpicked
  later.

If a timeline eventually needs morning/afternoon, that decision gets made
explicitly with the 96.9% on the table, not smuggled in here.

**`edad_min` / `edad_max`** — the ages the level teaches, derived from `NIVEL`.
England reads this off its register (`StatutoryLowAge` / `StatutoryHighAge`);
SEP publishes no such column.

```
INICIAL 0-2   PREESCOLAR 3-5   PRIMARIA 6-11   SECUNDARIA 12-14
MEDIA SUPERIOR 15-17   LICENCIATURA 18-50   POSGRADO 22-60
```

The school levels are the SEP normative ladder: tight, non-overlapping ranges,
assuming **start-of-school-year age with grade-for-age progression** (no
repetition). Every school-age person then maps to exactly one level, so a
student's level is not an artefact of capacity and draw order at a shared
boundary — capacity contention stays *within* a level (ages 6-11 still share
primaria, the mixing that is wanted) with no coupling across levels.

University `edad_min` is measured (nobody attends licenciatura below 18, nobody
posgrado below 22). The `edad_max` is **a deliberate plausibility cap, not a
measurement**: higher education is genuinely not age-locked and the census finds
real adult learners (1.8% attendance at 30-49), but the student rate is drawn from
an open 65+ activity band applied flat to age 99, which manufactures nonagenarian
students the census never measured at that age. Licenciatura is capped at 50 and
posgrado at 60 as an operator choice; a student above the cap for every level they
qualify for is left unplaced and reported, which knowingly discards the handful of
genuinely-measured older students rather than hiding them.

**The perfect-progression assumption is a simplification, and it contradicts the
census attainment tails.** `NIVACAD` puts some 15-year-olds still in secundaria
and some 18-19s still in bachillerato — real repeaters and late-completers. Tight
ranges push those up a level: an 18-year-old still in bachillerato becomes a
university student. This is deliberate, for tractability, and stated here rather
than smoothed over — an honest, visible assumption over a silent correction.

**The mapping is normative, not measured**, because the census records attainment
and never the level attended. Two rungs of the ladder *can* be checked, though —
attendance at a level, cross-cut by `NIVACAD` — and the census is sharp about
both:

| attendees at | first age with anyone | at that age |
|---|---|---|
| licenciatura level (`NIVACAD` 11) | **18** (0 at 17) | 94,700 |
| posgrado level (`NIVACAD` 12/13/14) | **22** (0 below) | 6,974 |

Nobody in the country attends a posgrado below 22. Those two floors are measured.
The rest are not.

**Bounds are not distributions, and posgrado skews young because of it.** A
licenciatura runs 4.5-5 years, so the real mass of posgrado is 23-24 and up —
only 7.2% of posgrado students are under 24. But `edad_min` only sets a floor: a
22-year-old is eligible for licenciatura *and* posgrado, and the distributor then
chooses by capacity, which nationally puts ~8.7% of 22-year-old students in
posgrado against a true **1.7%**.

Raising the floor to 24 would hide this, at the cost of deleting the 22-23 year
olds the census actually counted — fitting one parameter to compensate for a
different deficiency. The floor stays at the measured 22 and the skew is recorded
here instead. Posgrado is 465,006 places nationally (1.3% of all capacity), so
the absolute error is small; what is wrong is *which* ages fill them.

### Output schema

```
geo_unit,name,latitude,longitude,cct,school_name,tipo_educativo,nivel,servicio,
subnivel,turno,shifts_at_site,edad_min,edad_max,sostenimiento,alumnos_total,
alumnos_hombres,alumnos_mujeres,docentes_total,personal_total,aulas_existentes,
clave_localidad
```

```
MX21155,21EPR1544H,18.074722,-98.243056,21EPR1544H,Benito Juarez,BÁSICA,PRIMARIA,
PRIMARIA GENERAL,,MATUTINO,1,6,12,PÚBLICO,9,4,5,1,1,2,0003
```

## 4. Validation

```
MX_schools.csv        250,973 venues, 2,469 municipios, capacity 29,131,841
MX_universities.csv    10,868 venues,   835 municipios, capacity  5,354,473
```

**Every one of the 2,469 municipios has a school.** DENUE cannot say that — five
municipios have no establishment at all, and San Quintín has 58,585 workers and
nowhere to put them. Schools are the better-covered venue file by some distance.

Row disposition partitions the input exactly, which is the check that no row is
counted twice or lost:

```
agrees                                      258,222
case C, SEP code kept                         3,328
case B remapped                                 127
coords unresolvable, SEP code kept               92
case A remapped                                  72
TOTAL                                       261,841

repairs (not a partition):
coords outside Mexico, blanked                   11
PERSONAL TOTAL zero-with-teachers -> null    30,630
```

`agrees` reproduces the standalone point-in-polygon audit
(`remap_schools_geo.py`) to the row, which is the check that the build and the
audit read the geometry the same way. The 92 unresolvable are 81 whose
coordinates fall in no municipio (mostly coastal precision) plus the 11 outside
Mexico; they keep their SEP code, because a coordinate that resolves to nothing
is not evidence against one.

Capacity by level, against the census population that must fit in it:

| | file capacity | census students | |
|---|---|---|---|
| básica (3-14) | 23,750,037 | 22,215,715 | +6.9% |
| media superior (15-17) | 5,381,804 | 4,645,545 | +15.8% |
| superior (18+) | 5,354,473 | 6,162,745 | -13.1% |

These are **not clean reconciliations** — the age bands are proxies, since the
census records attainment and not the level currently attended (below), and many
media superior students are 18+. They are recorded as orders of magnitude, not
as validation.

### The world build

`python3 create_world.py --config configs/Mexico_2020/config.yaml`, exit 0.
Aguascalientes (`MX01`), 1,425,607 people. **These figures carry the
Aguascalientes bias described in [README.md](README.md) and do not generalise.**

```
school venues loaded         2,152      university venues loaded    150
school       327,110 eligible -> 325,732 placed (99.6%)
university    80,076 eligible ->  64,110 placed (80.1%)
```

The unplaced rate is predicted from the source before the build, which is what
makes it evidence rather than a puzzle:

| | demand | capacity | predicted unplaced | observed |
|---|---|---|---|---|
| school (3-17) | 331,407 | 352,319 (0.94x) | ~0% | **0.4%** |
| university (18+) | 79,426 | 69,029 (1.15x) | 13.1% | **19.9%** |

School lands where predicted. **University does not, and the gap is the check
working.** The naive 13.1% ignores that capacity is not fungible across levels:
posgrado's 5,878 places carry `edad_min: 22` and are unreachable to most
students, so the binding constraint is licenciatura's 63,151 alone. Re-predicted
on that basis: **(80,076 - 63,151) / 80,076 = 21.1%**, against 19.9% observed.
The 1.2pp residual is the ~890 posgrado places that students aged 22+ did take.

So Aguascalientes' 19.9% unplaced is a property of its data — 63,151 licenciatura
places for 80,076 adult students, 1.27x oversubscribed — and not a config bug.
This is the same argument DENUE's coverage gap gets in [companies.md](companies.md).

## 5. Limitations

**The vintage.** Enrolment is a 2025/26 catalogue under a 2020 population,
~5.9% short on primaria. Used as-is by decision; the shortfall surfaces as
unplaced students and is measured, not patched.

**The census does not record the level a student attends.** `NIVACAD` is
*attainment* — "the highest level approved", universe "residents aged three or
older" — not current enrolment. Nothing in the person record supplies it
(`ASISTEN`, `NIVACAD`, `ESCOLARI`, `ESCOACUM`, `NOMCAR_C` are the whole set). So
a student cannot be matched to a school *level* from the census directly; level
must be inferred from age, with attainment as a consistency check. This is why
the capacity table above is a proxy and not a reconciliation, and it is the
single biggest obstacle to a per-level capacity check of the kind that exposed
the work layer's 21.3% informality gap.

**Schools are workplaces, and this is not built.** `DOCENTES TOTAL` gives exact
teacher counts — not DENUE's size bands — and sector-61 workers belong in schools
rather than in DENUE establishments, placed before `company_distributor` runs, as
England does with `specific_workplace_classrooms_distributor.yaml`. The data is
here; the chain is not.

**`AULAS EXISTENTES` is unused.** Classrooms would be the natural child venue
(England's `school_classrooms.yaml`), and would give a contact group finer than a
whole shift.

**`HORTRA` is unused.** Working students report a median 15 hours a week against
45 for working non-students. Fifteen hours in a shop is not the exposure that 45
is, and the census carries the hours to weight it.

**The 11 schools with unusable coordinates** keep their municipio but have no
location, so any distance-based distributor will not place them.

**Allocation within a municipio is random, and cannot be otherwise.** England
picks a student's school by distance — `consider_by: count`, `criteria: closest`
over the 10 nearest — and that works because its people sit at Output Area level,
roughly 300 residents, so two children in one LAD genuinely have different
nearest schools.

Mexico cannot do this, and the obstacle is not that municipios are large. It is
that **a person has no sub-municipio location at all**: population and households
exist only at municipio (see [README.md](README.md)), so every resident of a
municipio would measure distance from the same centroid. "Closest" would then
funnel all ~950,000 people of Aguascalientes municipio into whichever school sits
nearest that centroid — deterministically, and far worse than random. The
schools' coordinates are precise and real, but there is no person-side coordinate
to measure them against.

So allocation is `consider_by: geo_unit` — any school in the student's
`school_location` municipio, at random, bounded by capacity — which is what
`company_distributor` already does, for the same reason. Capacity is what shapes
the result: a big school takes more students because it has more places, not
because anyone lives near it.

The consequence is that **intra-municipio spatial structure is absent**. A child
in a remote locality of a large rural municipio may be allocated to a school in
the cabecera. `CLAVE LOCALIDAD` is carried on every school row and ITER does
publish localities, so a finer key exists on the venue side — but INEGI suppresses
the population variables for 42.5% of localities, and the engine's Mexican
geography is two-level, so there is no locality for a person to live in. It is
the same disclosure control that makes municipio the floor everywhere else.
