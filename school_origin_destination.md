# Home-to-school origin-destination matrix (Mexico 2020)

How `data/Mexico_data/activities/school/MX-school-destination-likelihood.csv` is
produced, and why each decision was taken. Built by
`data/Mexico_data/source/build_school_od.py`.

It warrants its own document rather than a paragraph in
[schools.md](schools.md): it is a different source (the census, not SEP's
register), it answers a different question (where a student goes, not what exists),
and its central interpretation — the no-commute code — is a trap.

## 1. The source

**Censo 2020, Cuestionario Ampliado, person records** — the same archives as the
commuting matrix.

| column | meaning |
|---|---|
| `ASISTEN` (`V344`) | attends school: `1` Sí, `3` No, `9` NE |
| `MUN_ASI`, `ENT_PAIS_ASI` (`V586`) | municipio of the school attended |
| `TIE_TRASLADO_ESCU` (`V347`) | time taken to travel to school |
| `MED_TRASLADO_ESC1..3` (`V588`-`V590`) | mode of travel, up to three |
| `FACTOR` | expansion weight |

This is the same home-to-destination structure `MUN_TRAB` gives for work, and it
is **better populated**:

| destination of attendees | weighted | share |
|---|---|---|
| a real municipio | 32,605,739 | **98.73%** |
| blank | 252,911 | 0.77% |
| unspecified (`999`) | 166,814 | 0.51% |
| outside Mexico | 0 | **0%** |

No student in the country attends school abroad, so unlike the work matrix the
`OUTSIDE_MX` sentinel is emitted but never populated. It is kept anyway, so a
consumer excludes it explicitly rather than by assuming.

## 2. Interpretation

### The no-commute code is 6, not 7

This is the trap, and it is silent.

`TIE_TRASLADO_TRAB` uses **7** for "no se traslada", which is how the work matrix
gets its 11.1% `From_Home` category. `TIE_TRASLADO_ESCU` uses **6**, and has no
`7` at all:

| code | label | weighted | share |
|---|---|---|---|
| 1 | Hasta 15 minutos | 19,507,553 | 59.07% |
| 2 | 16 a 30 minutos | 8,045,577 | 24.36% |
| 3 | 31 minutos a 1 hora | 3,333,018 | 10.09% |
| 4 | Más de 1 hora y hasta 2 horas | 1,085,797 | 3.29% |
| 5 | Más de 2 horas | 261,579 | 0.79% |
| **6** | **No se traslada** | **581,566** | **1.76%** |
| 9 | No especificado | 210,374 | 0.64% |

INEGI's instruction for code 6 is explicit: it applies when "la persona estudia a
distancia o por Internet y, por lo tanto, no requiere asistir a una institución
educativa".

Copying `build_work_od.py`'s `NO_COMMUTE = "7"` would have matched nothing,
produced **zero** distance learners instead of 581,566, and raised no error. The
sentinel is asserted against the code list, not inherited.

**This is not a pandemic artefact.** The census reference date is 15 March 2020;
Mexico's first restrictions began 23 March. The 1.76% is ordinary distance
education, measured a week before the country closed.

### School travel is local, and the work matrix's shape does not carry over

| | share |
|---|---|
| attends school in their own municipio | **89.91%** |
| attends school in another municipio | 10.09% |

Students are far more local than workers. The 10.09% is not noise, though: **367
of 2,469 municipios have no media superior school at all**, so their 15-17 year
olds must leave the municipio to attend one. The matrix is what carries them.

### Keyed on residence, and valid per-state

Like the work matrix, this is keyed on the **origin** (residence) municipio, so
a per-state run is valid — each state archive holds its own residents. A file
keyed on *destination* (as the sector table is) would need all 32 states even to
model one. See the asymmetry note in [work_sector.md](work_sector.md).

## 3. Transformation

```
key  = (origin municipio, school municipio, mode)
mode = From_Home if TIE_TRASLADO_ESCU == 6 else Normal
```

Accumulate `FACTOR`; emit one row per key with the within-origin share.

### Output schema

```
SGU_origin_code,SGU_destination_code,Place of study indicator,Count,Total_Origin_Students,Likelihood
MX01001,MX01001,From_Home,2934,277915,0.010557184750733138
MX01001,MX01001,Normal,271668,277915,0.9775219041793354
```

`Likelihood` is the only column meant to be consumed. Counts are
extended-questionnaire estimates and run below ITER; student totals come from the
census population, never from here and never from the school register.

## 4. Validation

```
43,843 flows, 2,466 origins, 2,468 destinations
From_Home    581,566   1.76%
Normal    32,443,898  98.24%
sentinels: UNSPECIFIED only (2,743 flows), OUTSIDE_MX never populated
```

`From_Home` reproduces the `TIE_TRASLADO_ESCU = 6` count exactly (581,566), which
is the check that the sentinel is being read correctly rather than matching
nothing.

2,466 origins, against 2,469 municipios — the same three absentees as the work
matrix, for the same reason.

### The world build

`create_world.py`, exit 0, Aguascalientes (`MX01`):

```
school_location   406,626 assigned, 0 failed   11 unique destinations
```

406,626 is exactly the number of people the activity_status draw made students
(`student` + `student_employee` + `student_own_account`), so the filter admits
every student and nobody else. Zero failures means every student's origin
municipio was present in the matrix — Aguascalientes contains none of the three
absentees.

The 11 destinations are the 11 municipios of Aguascalientes: with the state
filter on, `out_of_boundary: redistribute` folds out-of-state school flows back
onto the origin's in-boundary destinations. A national run would exercise the
other 2,455.

## 5. Limitations

**Three municipios have no origin row.** Seybaplaya (`MX04012`), Honduras de la
Sierra (`MX07125`) and La Magdalena Tlaltelulco (`MX29048`) contribute no flows.

They are **not absent from the microdata** — that is the easy assumption and it is
wrong. Their residents are there, in numbers: Seybaplaya has 1,326 records,
15,299 weighted. What is absent is the *answer*: `ASISTEN` is `9` (no
especificado) or blank for every one of them, and `CONACT` is `99` or blank. So
nobody in those municipios is recorded as attending school, and an origin with no
attendees produces no flow.

This is the same fact that empties them out of
[activity_status.md](activity_status.md), where reading the unspecified as an
answer had manufactured municipios that were 100% inactive. Here the omission is
automatic — the builder keeps only `ASISTEN = 1` — which is why the matrix was
right about these three before the status table was.

A student living there has no flow to draw from, and the build fails loudly
rather than inheriting a parent's distribution. A run covering Campeche, Chiapas
or Tlaxcala must exclude them. Aguascalientes is clean.

Note the interaction with [schools.md](schools.md): `MX07125` holds **124
schools** and 4,479 places — 67 of them re-keyed in on their coordinates from the
parent SEP files them under — while holding no student origins at all. Both facts
are true of the sources and neither is patched.

**No level in the matrix.** The flows are all students together, because the
census does not record which level a student attends — only attainment
(`NIVACAD`). A 6-year-old and a postgraduate contribute to the same origin total.
Splitting the matrix by level would need the level inferred from age first.

**Mode and duration are unused.** `MED_TRASLADO_ESC1..3` (52.8% walking, 18.4%
and 17.4% on the two commonest vehicle codes) and `TIE_TRASLADO_ESCU` are
England's `mode_transport_by_oa.csv` counterpart for schools. Not built.
