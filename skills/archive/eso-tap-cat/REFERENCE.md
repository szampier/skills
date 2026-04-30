# ESO TAP Cat — Reference

## TAP endpoints

```
https://archive.eso.org/tap_cat/sync        # synchronous queries
https://archive.eso.org/tap_cat/async       # asynchronous queries
https://archive.eso.org/tap_cat/tables      # table discovery
https://archive.eso.org/tap_cat/capabilities
```

**Important**: `tap_cat` uses `FORMAT=text` (not `FORMAT=txt` as in `tap_obs`).

---

## TAP_SCHEMA.tables — catalogue metadata columns

| column | description |
|--------|-------------|
| `table_name` | Fully qualified table name (use double-quoted in ADQL) |
| `title` | Human-readable title of the catalogue |
| `description` | Short description |
| `collection` | Phase 3 collection name (e.g. `VHS`, `VVV`, `KIDS`, `GAIAESO`) |
| `instrument` | Instrument used (e.g. `VIRCAM`, `OMEGACAM`, `GIRAFFE`) |
| `telescope` | Telescope used (e.g. `VISTA`, `VST`, `VLT`) |
| `number_rows` | Number of sources in the catalogue |
| `skysqdeg` | Sky coverage in square degrees |
| `mjd_obs` / `mjd_end` | MJD range of the underlying observations |
| `cat_id` | Internal ID; web UI at `https://www.eso.org/qi/catalogQuery/index/<cat_id>` |
| `acknowledgment` | Required acknowledgment text for publications |
| `bibliography` | BIBCODE or DOI of the associated paper |

---

## Spatial query syntax

`tap_cat` supports **only cone search**, not the full `INTERSECTS(s_region, ...)` syntax of `tap_obs`.

```sql
-- Standard cone search pattern:
SELECT *
FROM "CATALOG_TABLE_NAME"
WHERE CONTAINS(
  POINT('', ra_column, dec_column),
  CIRCLE('', 83.82, -5.39, 0.1)
) = 1
```

> The RA/Dec column names differ per catalog. Always inspect `TAP_SCHEMA.columns` first.
> Look for columns with UCD `pos.eq.ra;meta.main` and `pos.eq.dec;meta.main`.

---

## ADQL query examples

### Discover all catalogs from a collection
```sql
SELECT table_name, title, number_rows, skysqdeg
FROM TAP_SCHEMA.tables
WHERE collection = 'VHS'
ORDER BY table_name
```

### Discover catalogs by instrument
```sql
SELECT table_name, title, collection, number_rows
FROM TAP_SCHEMA.tables
WHERE instrument LIKE '%VIRCAM%'
  AND schema_name NOT IN ('TAP_SCHEMA')
ORDER BY number_rows DESC
```

### Inspect columns of a catalog
```sql
SELECT column_name, unit, ucd, description
FROM TAP_SCHEMA.columns
WHERE table_name = 'VHS_CAT_V3'
ORDER BY column_name
```

### Cone search in VHS catalogue
```sql
SELECT *
FROM "VHS_CAT_V3"
WHERE CONTAINS(POINT('', raj2000, dej2000), CIRCLE('', 83.82, -5.39, 0.1)) = 1
```

### Cone search in PESSTO transient catalogue
```sql
SELECT *
FROM "pessto_tran_cat_fits_V2"
WHERE CONTAINS(
  POINT('', transient_raj2000, transient_decj2000),
  CIRCLE('', 41.2863, -55.7406, 0.04)
) = 1
```

### All spectroscopic surveys (GaiaESO, VANDELS, VIPERS…)
```sql
SELECT table_name, title, collection, number_rows
FROM TAP_SCHEMA.tables
WHERE telescope LIKE '%VLT%'
  AND schema_name NOT IN ('TAP_SCHEMA')
ORDER BY collection
```

---

## TAP URL construction

ADQL → TAP URL (VHS cone search):
```
https://archive.eso.org/tap_cat/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=text&MAXREC=200&QUERY=SELECT%20*%0aFROM%20%22VHS_CAT_V3%22%0aWHERE%20CONTAINS(POINT('',raj2000,dej2000),CIRCLE('',83.82,-5.39,0.1))=1
```

ADQL → TAP URL (catalog discovery):
```
https://archive.eso.org/tap_cat/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=text&MAXREC=200&QUERY=SELECT+table_name,title,collection,instrument,number_rows+FROM+TAP_SCHEMA.tables+WHERE+schema_name+NOT+IN+('TAP_SCHEMA')+ORDER+BY+table_name
```

Web UI for a specific catalog (use `cat_id` from `TAP_SCHEMA.tables`):
```
https://www.eso.org/qi/catalogQuery/index/<cat_id>
```

---

## Notable collections (examples)

| Collection | Instrument | Description |
|------------|-----------|-------------|
| `VHS` | VIRCAM/VISTA | VISTA Hemisphere Survey — NIR photometry |
| `VVV` / `VVVX` | VIRCAM/VISTA | VISTA Variables in the Via Lactea — Galactic bulge/disk |
| `VMC` | VIRCAM/VISTA | VISTA Magellanic Survey |
| `KIDS` | OMEGACAM/VST | Kilo-Degree Survey — weak lensing photometry |
| `VPHASplus` | OMEGACAM/VST | VST Photometric Hα Survey of the Southern Galactic Plane |
| `VIKING` | VIRCAM/VISTA | VISTA Kilo-degree Infrared Galaxy survey |
| `GAIAESO` | GIRAFFE+UVES/VLT | Gaia-ESO spectroscopic survey |
| `VANDELS` | VIMOS/VLT | Deep spectroscopic survey of high-z galaxies |
| `VIPERS` | VIMOS/VLT | VIMOS Public Extragalactic Redshift Survey |
| `LEGA-C` | VIMOS/VLT | Large Early Galaxy Astrophysics Census |
| `PESSTO` | EFOSC2/NTT | Public ESO Spectroscopic Survey of Transient Objects |
| `NGTS` | NGTS | Next Generation Transit Survey — light curves |
| `ATLASGAL` | APEX | APEX Telescope Large Area Survey of the Galaxy |
