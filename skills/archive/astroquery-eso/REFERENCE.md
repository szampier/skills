# astroquery.eso — Reference

## Shared parameters (all query methods)

| Parameter | Type | Description |
|-----------|------|-------------|
| `column_filters` | `dict` | ADQL `WHERE` expressions |
| `columns` | `list[str]` / `str` | Columns to return; `None` = all |
| `cone_ra` / `cone_dec` / `cone_radius` | `float` | Cone search in degrees |
| `top` | `int` | `SELECT TOP N` |
| `count_only` | `bool` | Return row count as `int` |
| `get_query_payload` | `bool` | Return the ADQL string instead of running it |
| `help` | `bool` | Print table columns and exit |
| `authenticated` | `bool` | Run as logged-in user (requires `login()`) |

---

## Get FITS headers — `get_headers`

```python
table = eso.query_instrument("midi", column_filters={"object": "NGC4151"}, columns=["dp_id"])
headers = eso.get_headers(table["dp_id"])
# Returns Table: rows = products, columns = FITS header keywords
```

---

## Download data — `retrieve_data`

```python
# Single dataset
local_file = eso.retrieve_data("AMBER.2006-03-14T07:40:19.830")

# Multiple datasets
files = eso.retrieve_data(
    table["dp_id"][:5],
    destination="/tmp/eso_data",   # default: astropy cache
    continuation=False,             # True = force re-download even if already present
    with_calib="raw",               # None | 'raw' | 'processed'
    unzip=True,
)
```

Returns `str` for a single dataset string, `list[str]` otherwise.

---

## Associated calibration files — `get_associated_files`

```python
calib_ids = eso.get_associated_files(
    ["MIDI.2007-02-07T07:01:51.000"],
    mode="raw",       # 'raw' or 'processed'
    savexml=True,
    destination="/tmp/calib_xml",
)
files = eso.retrieve_data(calib_ids)
```

---

## Diagnostic workflow

1. **List what's available**: `eso.list_instruments()` / `eso.list_surveys()`
2. **Inspect columns**: `eso.query_instrument("name", help=True)`
3. **Dry-run**: `get_query_payload=True` — returns the ADQL string without executing
4. **Count first**: `count_only=True` before fetching large tables
5. **Limit rows in dev**: `eso.ROW_LIMIT = 50`
6. **Clear stale cache**: `Eso.clear_cache()`

---

## TAP tables

| Table | Contents |
|-------|----------|
| `dbo.raw` | All raw instrument data |
| `ivoa.ObsCore` | Phase 3 / reduced data products |
| `ist.<instrument>` | Instrument-specific raw data |
| `ist.apex_quicklooks` | APEX quicklook products |

ESO Programmatic Access: https://archive.eso.org/programmatic/#TAP
