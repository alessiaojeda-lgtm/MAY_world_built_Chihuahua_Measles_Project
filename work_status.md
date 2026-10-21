# Work status by municipio, sex and age (Mexico 2020)

> **SUPERSEDED by [activity_status.md](activity_status.md).** The world no longer
> builds this file, and `work_status` is no longer an attribute: the chain reads
> `activity_status`, which crosses this split with school attendance. Read on for
> the informality argument — it is unchanged and still load-bearing — but not for
> the table or the numbers.
>
> Two figures here do not reproduce and are corrected in `activity_status.md`:
> the median hours for working non-students (40h here, **45h** measured with
> `HORTRA = 999` excluded) and for adults (45h here, **48h**). The 12-17
> works-and-studies count of 498,986 includes `SITTRA = 9` workers, which the
> built table drops, giving **498,350**.

How `data/Mexico_data/activities/work/MX_work_status_municipio.csv` is produced,
and why each decision was taken. Built by
`data/Mexico_data/source/build_work_status.py`.

This file has no England counterpart. It exists because Mexico's informality
makes England's rule — give every 18-64 adult a workplace — produce a broken
world.

## Why this file exists

England's `workplace_assignment` assigns a workplace to all working-age adults,
and its own comment concedes the point: they are the "potential workforce", and
"the actual 'work' activity should be assigned separately based on employment
rates". England never does that separately, and gets away with it because its
company file is synthesised to match its workforce.

Mexico's is not. DENUE lists registrable establishments, so it structurally
cannot host street traders, farm labourers, building crews or unpaid family
workers. Applying England's rule to Aguascalientes:

| eligibility rule | demand | vs capacity (580,057) |
|---|---|---|
| all 18-64 | 863,243 | **1.49x** |
| all workers | 692,983 | 1.19x |
| employees at fixed premises | 472,723 | **0.81x** |

At 1.49x, a third of adults would bounce off full establishments and be dropped
by `when_full: exclude` into `fallback: skip` — with allocation order, not any
real characteristic, deciding who works. That is precisely the silent fallback
this world is built to refuse. This file supplies the characteristic instead.

## 1. The source

**Censo 2020, Cuestionario Ampliado, person records** — the same archives as the
commuting matrix and the sector table.

| column | meaning | values |
|---|---|---|
| `CONACT` | condition of activity | employed set is `{10,13,14,15,16,18,20}` |
| `SITTRA` | **position in occupation** | `1`..`6`, `9` = not specified |
| `SEXO` | sex | `1` = hombre, `3` = mujer |
| `EDAD` | age | `CONACT` is only asked of 12+ |
| `FACTOR` | expansion weight | integer |

`SITTRA` (`V364`) is the lever. Nationally, by weighted workers:

| code | meaning | workers | share |
|---|---|---|---|
| 1 | Empleada(o) u obrera(o) | 32,597,623 | **62.4%** |
| 5 | Trabajador(a) por cuenta propia | 11,145,640 | **21.3%** |
| 2 | Jornalera(o) o peon(a) | 2,571,510 | 4.9% |
| 3 | Ayudante con pago | 2,451,083 | 4.7% |
| 6 | Trabajador(a) sin pago | 1,812,196 | 3.5% |
| 4 | Patron(a) o empleador(a) | 1,640,451 | 3.1% |
| 9 | No especificado | 32,426 | 0.1% |

One in five Mexican workers is *cuenta propia* — self-employed with no employees.
That single category is most of the reason DENUE cannot host the workforce.

## 2. Interpretation

### Three states

```
employee    SITTRA 1 or 4  — works at someone's fixed premises
own_account SITTRA 2,3,5,6 — works, but not at a registrable establishment
not_working CONACT not in the employed set
```

**Why `4` (patron) counts as an employee.** An employer works at their own
establishment, and DENUE's `per_ocu` bands are explicitly "incluyendo al dueno" —
the register already counts the owner in the headcount, so excluding them would
leave a place unfilled.

**Why `2`, `3`, `5`, `6` do not.** A jornalero works a field, an ayudante and a
cuenta propia typically work a street pitch or a client's premises, and a
trabajador sin pago is family labour. None has a registrable address, which is
exactly why DENUE's capacity falls short of the workforce and not short of
employees.

This is not a modelling convenience. INEGI's own catalogue states DENUE excludes
agricultural and forestry activities (`rnm/index.php/catalog/587`), and the
`SITTRA` split is the census independently drawing the same line.

### `SITTRA = 9` is dropped

0.06% of weighted workers nationally (32,426). They fall into no status and are
omitted from `Total`, which renormalises them proportionally across the three
states. At this magnitude the choice is immaterial; carrying a fourth "unknown"
status would propagate a measurement artefact into every downstream filter.

### Keyed on residence, and on age

Work status is a property of the person, assigned before they are given a
workplace, so it is keyed on **residence** municipio. Unlike the sector table, a
per-state run is therefore valid.

Age bands are narrow where labour-force participation moves fast and wide where
it is flat:

```
12-14   15-17   18-29   30-49   50-64   65+
```

**No 18-64 window.** Mexico's labour force does not fit England's. Nationally, by
weighted workers:

| band | workers | share |
|---|---|---|
| 12-14 | 287,789 | 0.6% |
| 15-17 | 1,005,197 | 1.9% |
| 18-64 | 48,513,684 | 92.8% |
| 65+ | 2,444,259 | **4.7%** |

Bounding at 18-64 would discard **7.2%** of the workforce — 2.4M workers over 65,
and 1.3M under 18, the latter being exactly the group that makes the school
question interesting.

There is no row below 12, because `CONACT` is not asked below 12. The consuming
config filters on `age >= 12` for that reason — and also because the engine's
categorical filter *skips* null values rather than rejecting them, so an under-12
with no `work_status` would otherwise pass a `values: [...]` filter and be given a
job.

## 3. Transformation

```
key    = ("MX" + ENT + MUN, "male" | "female", age band)
status = employee | own_account | not_working
```

Accumulate `FACTOR`, emit one row per key.

### Output schema

```
geo_unit,Sex,age_band,Total,employee,own_account,not_working
MX01001,female,12-14,25891,37,486,25368
MX01001,female,15-17,27597,1741,1663,24193
```

### National output

29,626 rows over all 2,469 municipios. `Total` equals the status sum in every
row; every municipio is present.

| status | weighted (12+) | share |
|---|---|---|
| not_working | 47,894,003 | 47.8% |
| employee | 34,238,074 | **34.2%** |
| own_account | 17,980,429 | 18.0% |

**Employees are 65.6% of workers nationally, against 75.1% in Aguascalientes.**
The informal share is half again as large across the country as in the test
state.

Two cells are missing of 29,628 possible, both `MX29048` La Magdalena Tlaltelulco
(female 12-14, female 65+) — a municipio already absent from the commuting matrix
for the same reason. See the cross-source table in [README.md](README.md).

## 4. Validation

Measured in the Aguascalientes world build (`create_world.py`, exit 0):

```
work_status        1,117,857 assigned, 0 failed
  employee           477,060 (42.7%)
  not_working        482,078 (43.1%)
  own_account        158,719 (14.2%)
workplace_location   635,779 assigned, 0 failed
work_sector          635,779 assigned, 0 failed
```

`477,060 + 158,719 = 635,779` exactly — the people given a workplace are precisely
those this file marks as working, so the filter gates as intended. Assigned
`employee` (477,060) reproduces the source table's Aguascalientes figure
(472,723) to within 0.9%, the residual being sampling in the draw.

## 5. Limitation: the model is binary where the data is not

`employee` / `own_account` is a split by *where* someone works, not *how much*.
The census carries `HORTRA` (hours worked per week, continuous) and `ASISTEN`
(school attendance, `V344`: `1` = Si, `3` = No), and together they show the binary
is wrong for under-18s.

Nationally, the 13,050,569 people aged 12-17:

| | weighted | share |
|---|---|---|
| studies only | 10,026,679 | 76.8% |
| neither | 1,730,904 | 13.3% |
| works only | 794,000 | 6.1% |
| **works and studies** | **498,986** | **3.8%** |

**Of the 1,292,986 working 12-17s, 38.6% also attend school.** And the hours
separate them cleanly:

| national 12-17 workers | median hours | ≤20h/week |
|---|---|---|
| also attending school | **15h** | 72% |
| not attending school | **40h** | 18% |
| adults 18+, for comparison | 45h | 14% |

Working students are genuinely part-time — 15h a week, 72% at or under 20h.
Working non-students are at 40h against an adult median of 45h: they are
school-leavers doing adult jobs, not part-timers. Half a million children hold
both a school place and a job, and roughly 800,000 more have left school for
full-time work; collapsing either into the other misrepresents both.

The Aguascalientes test state agrees on the shape but not the size: 43.3% of its
working 12-17s attend school against 38.6% nationally, and its non-student
workers report 48h against 40h. The 15h part-time finding is identical (71% vs
72% at ≤20h), so it is a real national regularity rather than a local artefact.

A correct model would split `employee` into a `student_worker` state holding both
a school and a part-time job — the engine permits it (`activity_map` is
`activity -> venue type -> subsets`, and `require_unassigned: false` allows a
second venue), and `HORTRA` could weight contact intensity by hours: 15 hours in
a shop is not the exposure that 45 hours is.

It is not built, because **the Mexican world has no school step yet**: the
261,842-row school list is not in `venues_config.yaml` and no school distributor
exists. Until then `exclude: ["primary_activity"]` matches nobody and every
working 12-17 takes a company as their primary activity. Harmless in a world
without schools; wrong the moment one is added, which is when `student_worker`
starts to mean something.
