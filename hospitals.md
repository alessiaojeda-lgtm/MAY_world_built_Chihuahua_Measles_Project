# Hospital venues (Mexico, DENUE April 2020)

How `data/Mexico_data/venues/primary_activities/hospitals.csv` is produced,
and why each decision was taken. Built by
`data/Mexico_data/source/build_hospitals.py`.

Companion to [companies.md](companies.md): same source (DENUE), same **April
2020** vintage, same per-establishment transformation. Placed alongside
`companies.csv` in `primary_activities/` rather than in
`leisure_activities/` (see [leisure_venues.md](leisure_venues.md)) because a
hospital is not a leisure/hospitality destination.

## 1. The source

Same DENUE register, same archived April 2020 by-activity release, same
reasoning for the vintage choice as [companies.md](companies.md) S1 and
[leisure_venues.md](leisure_venues.md) S1: kept consistent with
`companies.csv` so a world combining workplaces and hospitals describes one
economic snapshot, not a mix of 2020 and present-day Mexico.

## 2. Which SCIAN codes are "hospitals"

Sector 62 (servicios de salud y asistencia social) is broader than hospitals
alone. Its four subsectors:

| subsector | covers | kept? |
|---|---|---|
| 621xxx | Consultorios (doctor's offices: general/specialist medicine, dental, chiropractic, optometry, psychology, therapy, nutrition), medical labs, ambulance services, home nursing | no -- outpatient, not a hospital |
| **622xxx** | **Hospitales** | **yes** |
| 623xxx | Residencias (nursing homes, group homes for mental/intellectual disability, elder care, orphanages) | no -- residential care, not acute hospital care |
| 624xxx | Social assistance (daycare/guarderias, youth social work, addiction support groups, community meal/shelter services, job training) | no -- social services, not medical care |

`622xxx` -- SCIAN's own "Hospitales" subsector -- splits into three hospital
types, each private/public:

```
622111  Hospitales generales del sector privado
622112  Hospitales generales del sector publico
622211  Hospitales psiquiatricos y para el tratamiento por adiccion del sector privado
622212  Hospitales psiquiatricos y para el tratamiento por adiccion del sector publico
622311  Hospitales del sector privado de otras especialidades medicas
622312  Hospitales del sector publico de otras especialidades medicas
```

All six are kept as `hospitals.csv`. `industry_code` stays at the full
6-digit code (not truncated to sector `62` the way `companies.csv` truncates
`industry_code`) so general/psychiatric/specialty and private/public remain
distinguishable -- truncating would make every row read `62`, since only
hospital codes are selected here in the first place, and there is no census
work-sector column being matched against that would require the truncation
(same reasoning as [leisure_venues.md](leisure_venues.md) S4).

## 3. Transformation

Identical to `companies.csv` (see [companies.md](companies.md) S2-S3 for the
full justification, not repeated here):

```
geo_unit       = "MX" + cve_ent + cve_mun
name           = "denue_" + id
industry_code  = codigo_act verbatim (6-digit SCIAN, see S2 above)
sizeband       = per_ocu verbatim
employee_count = uniform draw within band; Pareto tail for the open band,
                 seeded (SEED = 20200315, same seed as companies.csv)
latitude       = latitud
longitude      = longitud
```

## 4. Output

```
data/Mexico_data/venues/primary_activities/hospitals.csv   (5,649 establishments)
```

National count (all 32 entidades), by `industry_code`:

| code | meaning | count |
|---|---|---|
| 622112 | Hospitales generales, sector publico | 2,616 |
| 622111 | Hospitales generales, sector privado | 1,900 |
| 622311 | Otras especialidades, sector privado | 873 |
| 622312 | Otras especialidades, sector publico | 155 |
| 622211 | Psiquiatricos/adiccion, sector privado | 67 |
| 622212 | Psiquiatricos/adiccion, sector publico | 38 |

Public general hospitals are the largest single category, consistent with
Mexico's public healthcare system (IMSS, ISSSTE, Secretaria de Salud)
carrying most general acute care nationally; private facilities dominate the
"other specialties" category. `ageb`/`manzana` are dropped for the same
reason given in `companies.md` S3: no population data exists below municipio
to make use of them.

## 5. Limitations carried over from `companies.csv`

- `employee_count` measures staffing (doctors, nurses, admin, support staff),
  not bed count or patient capacity. A hospital's contact capacity for
  patients/visitors is not the same thing as its staff headcount, and this
  file should not be read as the former.
- The top-band Pareto tail is assumed, not measured (see `companies.md` S2).
- Same pandemic-timing caveat as `companies.md` S5: April 2020 fieldwork
  observed establishments during the first weeks of Mexico's COVID-19
  shutdown, which is a particularly live caveat for hospitals specifically --
  facility counts, and especially staffing levels, were unusually volatile
  in exactly this window nationally.
- DENUE lists fixed, formally registered premises. Public-sector hospital
  identity in DENUE (IMSS, ISSSTE, Secretaria de Salud facilities are all
  tagged "del sector publico" within each subsector, not separately
  identified by operator) is not broken out by institution here; if that
  matters, it would need cross-referencing against a health-ministry
  facility registry, which this build does not attempt.
