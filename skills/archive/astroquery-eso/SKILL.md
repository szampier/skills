---
name: astroquery-eso
description: "Query the ESO Science Archive using the astroquery.eso Python API. Use when writing Python code to search ESO raw data, Phase 3 survey data, or instrument-specific tables; downloading datasets; filtering with ADQL column_filters; listing instruments or surveys; issuing free TAP queries; or retrieving calibration files. Trigger on: astroquery eso, query ESO archive, ESO TAP python, eso.query_instrument, eso.query_main, eso.query_surveys, eso.retrieve_data, eso.get_headers."
argument-hint: "What do you want to query? e.g. 'HARPS data for HD203608', 'MUSE observations of NGC 1234', 'download MIDI datasets'"
---

# astroquery.eso — ESO Archive Queries

Query the ESO Science Archive using `astroquery.eso` (Python).
For raw ADQL/curl-based queries use the `eso-tap-obs` skill instead.

> **Requires the latest pre-release version of astroquery:**
> ```bash
> pip install --pre astroquery
> ```
> The stable release does not include the current TAP-based interface.

> **WDB → TAP migration:** `column_filters` values must now be ADQL expressions
> (e.g. `"between '2024-01-01' and '2024-12-31'"`), not WDB-style keys like `stime`/`etime`.
> See the `column_filters` section below.

## Quick setup

```python
from astroquery.eso import Eso
eso = Eso()
eso.ROW_LIMIT = 100   # None → no limit (can return millions of rows)
```

```python
eso.login(username="YOUR_USERNAME", store_password=True)
# NOTE: TAP queries are NOT authenticated by default even after login().
# Pass authenticated=True explicitly to each query to include proprietary data.
```

## Core query methods

### `query_instrument` — raw data for one instrument
```python
table = eso.query_instrument(
    "midi",
    column_filters={"object": "NGC4151", "exp_start": "between '2008-01-01' and '2009-05-12'"},
    columns=["object", "date_obs", "dp_id"],
)
eso.list_instruments()           # discover instrument names
eso.query_instrument("midi", help=True)  # inspect columns
```

### `query_main` — all raw data (`dbo.raw`)
Use when the instrument has no dedicated table (HARPS, FEROS, APICAM…) or when spanning multiple instruments.
```python
table = eso.query_main(column_filters={"instrument": "APICAM", "filter_path": "LUMINANCE"})
```

### `query_surveys` — Phase 3 / reduced data (`ivoa.ObsCore`)
```python
surveys = eso.list_surveys()
table = eso.query_surveys(surveys="HARPS", target_name="HD203608")
```

### `query_apex_quicklooks`
```python
table = eso.query_apex_quicklooks(project_id="E-095.F-9802")
```

### `query_tap` — free ADQL
```python
table = eso.query_tap("SELECT TOP 10 * FROM ivoa.ObsCore")
```

## `column_filters` — ADQL syntax

Values are bare ADQL expressions appended after the column name:

```python
column_filters = {
    "object":    "NGC4151",                               # string equality (auto-quoted)
    "exp_start": "between '2024-01-01' and '2024-12-31'",
    "exp_start": "> '2024-06-01'",
    "det_dit":   "< 30",
    "filter_path":    "like '%LUMINANCE%'",
    "obs_collection": "in ('HARPS', 'ESPRESSO')",
}
```

WDB → TAP migration: `stime` → `exp_start > 'YYYY-MM-DD'`, `etime` → `exp_start < 'YYYY-MM-DD'`

## Row limit

```python
eso.ROW_LIMIT = 500    # keep first 500 rows
eso.ROW_LIMIT = None   # unlimited — caution with large tables
```
A `MaxResultsWarning` fires when results are truncated.

## Reference

Full parameter table, download, FITS headers, calibration files, diagnostics → [REFERENCE.md](REFERENCE.md)


## Core query methods

### `query_instrument` — raw data for one instrument
```python
table = eso.query_instrument(
    "midi",
    column_filters={"object": "NGC4151", "exp_start": "between '2008-01-01' and '2009-05-12'"},
    columns=["object", "date_obs", "dp_id"],
)
eso.list_instruments()           # discover instrument names
eso.query_instrument("midi", help=True)  # inspect columns
```

### `query_main` — all raw data (`dbo.raw`)
Use when the instrument has no dedicated table (HARPS, FEROS, APICAM…) or when spanning multiple instruments.
```python
table = eso.query_main(column_filters={"instrument": "APICAM", "filter_path": "LUMINANCE"})
```

### `query_surveys` — Phase 3 / reduced data (`ivoa.ObsCore`)
```python
surveys = eso.list_surveys()
table = eso.query_surveys(surveys="HARPS", target_name="HD203608")
```

### `query_apex_quicklooks`
```python
table = eso.query_apex_quicklooks(project_id="E-095.F-9802")
```

### `query_tap` — free ADQL
```python
table = eso.query_tap("SELECT TOP 10 * FROM ivoa.ObsCore")
```

## `column_filters` — ADQL syntax

Values are bare ADQL expressions appended after the column name:

```python
column_filters = {
    "object":    "NGC4151",                               # string equality (auto-quoted)
    "exp_start": "between '2024-01-01' and '2024-12-31'",
    "exp_start": "> '2024-06-01'",
    "det_dit":   "< 30",
    "filter_path":    "like '%LUMINANCE%'",
    "obs_collection": "in ('HARPS', 'ESPRESSO')",
}
```

Old WDB keys → TAP: `stime` → `exp_start > 'YYYY-MM-DD'`, `etime` → `exp_start < 'YYYY-MM-DD'`

## Row limit

```python
eso.ROW_LIMIT = 500    # keep first 500 rows
eso.ROW_LIMIT = None   # unlimited — caution with large tables
```
A `MaxResultsWarning` fires when results are truncated.

## Reference

Full parameter table, download, FITS headers, calibration files, diagnostics → [REFERENCE.md](REFERENCE.md)
