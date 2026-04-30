# ESO Skills — Domain Language

A shared vocabulary for skills and agents working with ESO data services and astronomy concepts.

## Services

**ESO Science Archive**:
The ESO long-term archive of science-ready data products, raw observations, and calibrations, accessible at `archive.eso.org`. Also called the Archive.
_Avoid_: ESO archive (ambiguous), data portal

**TAP** (Table Access Protocol):
The IVOA-standard HTTP protocol for querying astronomical tables using ADQL. The ESO Science Archive exposes TAP at `https://archive.eso.org/tap_obs/sync`.
_Avoid_: TAP service (just say TAP)

**ESO Science Portal**:
The web UI for the ESO Science Archive, at `https://archive.eso.org/scienceportal/home`. Useful for visual cone-search exploration.
_Avoid_: Science Archive Portal, ESO Portal

## Data Model

**ObsCore** (`ivoa.ObsCore`):
The IVOA-standard table exposed by the ESO TAP endpoint. Each row is one data product. The primary table for all archive queries.
_Avoid_: observations table, ESO table

**Data product**:
A single downloadable file in the Archive (raw frame, reduced cube, spectrum, catalogue, etc.), corresponding to one row in ObsCore.
_Avoid_: observation (overloaded), file

**Cone search**:
A spatial query that selects data products whose footprint (`s_region`) overlaps a circle defined by RA, Dec, and radius. The preferred spatial query pattern.
_Avoid_: circular search, radius search

**calib_level**:
Integer column in ObsCore encoding the processing level of a data product: 0 = raw, 1 = reduced, 2 = science-ready, 3 = enhanced.

## Query Language

**ADQL** (Astronomical Data Query Language):
The SQL dialect used to query ObsCore via TAP. Supports geometry functions (`CIRCLE`, `INTERSECTS`, `POINT`).

**TAP sync URL**:
A GET URL to `https://archive.eso.org/tap_obs/sync` with `REQUEST=doQuery&LANG=ADQL&FORMAT=txt&MAXREC=200&QUERY=<url-encoded ADQL>`. Returns plain-text tabular results.

## Telescopes & Instruments

**VLT** (Very Large Telescope):
ESO's four 8.2-metre unit telescopes (UT1–UT4) at Cerro Paranal. Match in ADQL with `facility_name LIKE '%VLT%'`.

**VLTI** (Very Large Telescope Interferometer):
Interferometric combination of VLT unit and auxiliary telescopes. Match with `facility_name LIKE '%VLTI%'`.

**Instrument name**:
The value of the `instrument_name` column in ObsCore (e.g. `MUSE`, `XSHOOTER`, `UVES`). Always filter with `LIKE '%NAME%'` to handle suffix variants.

## Coordinates

**MJD** (Modified Julian Date):
The time system used in ObsCore `t_min` / `t_max` columns. MJD 0 = 1858-11-17. Convert to ISO-8601 for display.

**ICRS**:
The default celestial reference frame. Use `CIRCLE('ICRS', ra, dec, radius)` in cone searches.
