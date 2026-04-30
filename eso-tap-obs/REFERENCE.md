# ESO TAP Obs — Reference

## TAP endpoint

```
https://archive.eso.org/tap_obs/sync
```

---

## ivoa.ObsCore columns

| column_name         | datatype | unit   | description |
|---------------------|----------|--------|-------------|
| abmaglim            | double   | mag    | Limiting magnitude |
| access_estsize      | long     | kbyte  | Estimated file size |
| access_format       | char     |        | MIME type of data file |
| access_url          | char     |        | URL to download the data product |
| bib_reference       | char     |        | Bibliographic reference |
| calib_level         | int      |        | Calibration level (0=raw, 1=reduced, 2=science-ready, 3=enhanced) |
| dataproduct_subtype | char     |        | Sub-type of data product |
| dataproduct_type    | char     |        | Type: cube, image, measurements, spectrum, visibility |
| dp_id               | char     |        | Data product identifier |
| em_max              | double   | m      | Maximum wavelength |
| em_min              | double   | m      | Minimum wavelength |
| em_res_power        | double   |        | Spectral resolving power |
| em_xel              | long     |        | Number of spectral bins |
| facility_name       | char     |        | Telescope (e.g. ESO-VLT-U4) |
| filter              | char     |        | Filter name |
| gal_lat             | double   | deg    | Galactic latitude |
| gal_lon             | double   | deg    | Galactic longitude |
| instrument_name     | char     |        | Instrument (e.g. MUSE, XSHOOTER, UVES) |
| multi_ob            | char     |        | Multi-OB flag |
| n_obs               | int      |        | Number of observations combined |
| obs_collection      | char     |        | Survey/programme collection name |
| obs_creator_name    | char     |        | PI name |
| obs_id              | char     |        | Observation identifier |
| obs_publisher_did   | char     |        | Archive data product identifier |
| obs_release_date    | char     |        | Public release date (timestamp) |
| obs_title           | char     |        | Title of observation |
| obstech             | char     |        | Observing technique |
| proposal_id         | char     |        | ESO programme ID |
| s_dec               | double   | deg    | Declination (J2000) |
| s_fov               | double   | deg    | Field of view diameter |
| s_pixel_scale       | double   | arcsec | Pixel scale |
| s_ra                | double   | deg    | Right Ascension (J2000) |
| s_region            | char     |        | Spatial footprint (ADQL:REGION) — use for spatial queries |
| s_resolution        | double   | arcsec | Angular resolution |
| snr                 | double   |        | Signal-to-noise ratio |
| t_exptime           | double   | s      | Exposure time |
| t_max               | double   | d      | End time (MJD) |
| t_min               | double   | d      | Start time (MJD) |
| target_name         | char     |        | Target name as given by PI |

---

## Allowed enum values

**dataproduct_type**: `cube`, `image`, `measurements`, `spectrum`, `visibility`

**dataproduct_subtype**: `catalog`, `catalogtile`, `deep pawprint`, `deep tile`, `exposure`, `fluxmap`, `ifs`, `pawprint`, `srctbl`, `tile`

**instrument_name** (common): `ALMA`, `APEXBOL`, `APEXHET`, `CRIRES`, `CRIRESplus`, `EFOSC`, `ERIS`, `ESPRESSO`, `FEROS`, `FORS1`, `FORS2`, `GIRAFFE`, `HARPS`, `HAWKI`, `ISAAC`, `KMOS`, `MUSE`, `NAOS+CONICA`, `NGTS`, `NIRPS`, `OMEGACAM`, `PIONIER`, `SINFONI`, `SOFI`, `SPHERE`, `UVES`, `VIMOS`, `VIRCAM`, `VISIR`, `XSHOOTER`

**facility_name** (common): `APEX-12m`, `ESO-3.6`, `ESO-NTT`, `ESO-VISTA`, `ESO-VLT-U1`, `ESO-VLT-U2`, `ESO-VLT-U3`, `ESO-VLT-U4`, `ESO-VLT-U1234`, `ESO-VLTI-A1234`, `ESO-VST`, `JAO`, `MPG/ESO-2.2`, `MULTI`, `NGTS`, `VISTA`

**obstech** (common): `CONTINUUM`, `ECHELLE`, `IFU`, `IMAGE`, `INTERFEROMETRY`, `MOS`, `MXU`, `NODDING`, `POLARIMETRY`, `single-dish`, `SPECTRUM`

---

## ADQL query examples

### Cone search with instrument filter
```sql
SELECT *
FROM ivoa.ObsCore
WHERE 1 = INTERSECTS(
  s_region,
  CIRCLE('ICRS', 170.06260833, 12.99128889, 0.1)
)
AND instrument_name LIKE '%MUSE%'
```

### List all instruments
```sql
SELECT DISTINCT instrument_name
FROM ivoa.ObsCore
ORDER BY instrument_name
```

### List VLT instruments only
```sql
SELECT DISTINCT instrument_name
FROM ivoa.ObsCore
WHERE facility_name LIKE '%VLT%'
ORDER BY instrument_name
```

### Count products by type
```sql
SELECT dataproduct_type, COUNT(*) AS NumProducts
FROM ivoa.ObsCore
GROUP BY dataproduct_type
ORDER BY NumProducts DESC
```

### Spectra in a cone
```sql
SELECT *
FROM ivoa.ObsCore
WHERE dataproduct_type = 'spectrum'
AND 1 = INTERSECTS(s_region, CIRCLE('ICRS', 83.82, -5.39, 0.1))
```

### Cross-match: UVES spectra at MUSE positions
```sql
SELECT DISTINCT uves.obs_id, uves.target_name, uves.s_ra, uves.s_dec, uves.access_url
FROM ivoa.ObsCore AS uves
JOIN ivoa.ObsCore AS muse
  ON 1 = INTERSECTS(uves.s_region, muse.s_region)
WHERE uves.instrument_name LIKE '%UVES%'
  AND uves.dataproduct_type = 'spectrum'
  AND muse.instrument_name LIKE '%MUSE%'
  AND muse.dataproduct_type = 'cube'
```

### NIR images near Galactic Centre (J/H/Ks bands)
```sql
SELECT TOP 100 instrument_name, em_min, em_max, s_resolution,
  distance(s_region, point('',266.42,-29.0)) AS dist_from_GC,
  dataproduct_type
FROM ivoa.ObsCore
WHERE dataproduct_type = 'image'
  AND INTERSECTS(CIRCLE('ICRS', 266.42, -29.0, 5), s_region) = 1
  AND ((em_min < 1.25E-6 AND em_max > 1.25E-6)
    OR (em_min < 1.65E-6 AND em_max > 1.65E-6)
    OR (em_min < 2.2E-6  AND em_max > 2.2E-6))
  AND em_res_power < 20
ORDER BY dist_from_GC
```

---

## TAP URL construction examples

ADQL → TAP URL (VLT instruments):
```
https://archive.eso.org/tap_obs/sync?REQUEST=doQuery&FORMAT=txt&LANG=ADQL&MAXREC=200&QUERY=SELECT+DISTINCT+instrument_name%0D%0AFROM+ivoa.ObsCore%0D%0AWHERE+facility_name+LIKE+%27%25VLT%25%27%0D%0AORDER+BY+instrument_name%0D%0A
```

ADQL → TAP URL (cone + MUSE):
```
https://archive.eso.org/tap_obs/sync?REQUEST=doQuery&LANG=ADQL&MAXREC=200&FORMAT=txt&QUERY=SELECT%20*%0aFROM%20ivoa.ObsCore%0aWHERE%201%20=%20INTERSECTS%28%0as_region,%0aCIRCLE%28%27ICRS%27,%20170.06260833,%2012.99128889,%200.1%29%0a%29%0aAND%20instrument_name%20LIKE%20%27%25MUSE%25%27
```

Science Portal URL (cone + MUSE):
```
https://archive.eso.org/scienceportal/home?pos=170.0626,12.9913&r=0.016667&ins_id=MUSE
```

> Note: the Science Portal `r` parameter is in degrees (e.g. 0.1° = 6 arcmin ≈ 0.016667°).
