---
name: eso-tap-obs
description: Query the ESO Science Archive (ivoa.ObsCore) using the TAP protocol. Translates natural-language requests into ADQL queries, TAP sync URLs, and ESO Science Portal links. Resolves target names via SIMBAD. Executes live queries via curl when the user says "search", "find", or "show me" in the context of ESO archive data.
---

# ESO TAP Obs

Expert querying of the **ESO Science Archive** via TAP (`ivoa.ObsCore`).
For PI-delivered scientific catalogues from public surveys use the `eso-tap-cat` skill instead.

## Workflow

For each request, follow these steps in order:

1. **Resolve target name** (if given) via SIMBAD:
   ```
   curl -s "https://simbad.cds.unistra.fr/simbad/sim-tap/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=json&QUERY=SELECT+ra,dec+FROM+basic+JOIN+ident+ON+ident.oidref=oid+WHERE+id='TARGET_NAME'"
   ```
   Extract `ra` and `dec` from the JSON response. Report: *"Resolved TARGET → RA: X°, Dec: Y°"*.

2. **Build ADQL query** targeting `ivoa.ObsCore` (see [REFERENCE.md](REFERENCE.md) for columns and enums).

3. **Build TAP sync URL**:
   - Base: `https://archive.eso.org/tap_obs/sync`
   - Fixed params: `REQUEST=doQuery&LANG=ADQL&FORMAT=txt&MAXREC=200`
   - URL-encode the ADQL for the `QUERY=` parameter.

4. **Build ESO Science Portal URL**:
   - Base: `https://archive.eso.org/scienceportal/home`
   - Cone searches: `pos=RA,Dec&r=RADIUS_DEG` (default `r=0.1`)
   - Append `ins_id=INSTRUMENT` when an instrument is specified.

5. **Execute live** when the user says "search", "find", "show me", or similar:
   ```
   curl -s "TAP_URL"
   ```
   Display the first 10 rows as a formatted table. Convert `t_min`/`t_max` MJD → ISO-8601 (MJD 0 = 1858-11-17).

## Query Construction Rules

- Use `SELECT *` unless specific columns are requested.
- Use `TOP N` only when the user specifies a limit.
- Do **not** add extra constraints (e.g. `calib_level`, `dataproduct_type`) unless the user asks.
- Spatial queries: use `INTERSECTS(s_region, CIRCLE('ICRS', ra, dec, radius))` — never filter on `s_ra`/`s_dec` alone.
- Default cone radius: **0.1 degrees**.
- Instrument filters: use `LIKE '%INSTRUMENT%'` (names may have suffixes).
- VLT telescopes: `facility_name LIKE '%VLT%'` matches all unit telescopes.

## Output Format (always in this order)

1. `sql` code block — ADQL query
2. `url` code block — TAP sync URL
3. `url` code block — ESO Science Portal URL
4. Formatted results table (top 10 rows) — only when live execution is triggered
5. One-sentence explanation of what the query returns
6. Footer: `⚠️ Use at your own risk. Results may be partial or incomplete.`

## Reference

Column catalogue, allowed enums, ADQL examples, and URL construction examples → [REFERENCE.md](REFERENCE.md)
