---
name: eso-tap-cat
description: Query the ESO Science Archive catalog service (tap_cat) using the TAP protocol. Translates natural-language requests into ADQL queries and TAP sync URLs targeting PI-delivered scientific catalogues from ESO public surveys. Resolves target names via SIMBAD. Executes live queries via curl when the user says "search", "find", or "show me" in the context of ESO catalogue data.
---

# ESO TAP Cat

Expert querying of the **ESO catalogue TAP service** (`tap_cat`).
For observation metadata and data products use the `eso-tap-obs` skill instead.

## Workflow

For each request, follow these steps in order:

1. **Discover the right catalog** if the user hasn't named one. Query `TAP_SCHEMA.tables`:
   ```
   curl -s "https://archive.eso.org/tap_cat/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=text&MAXREC=200&QUERY=SELECT+table_name,title,collection,instrument,number_rows+FROM+TAP_SCHEMA.tables+WHERE+schema_name+NOT+IN+('TAP_SCHEMA')+ORDER+BY+table_name"
   ```
   Filter by `collection` or `instrument` to narrow down. Show the user a short list.

2. **Inspect columns** for the chosen table to find the correct RA/Dec column names (they vary per catalog):
   ```
   curl -s "https://archive.eso.org/tap_cat/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=text&MAXREC=200&QUERY=SELECT+column_name,unit,ucd,description+FROM+TAP_SCHEMA.columns+WHERE+table_name='TABLE_NAME'"
   ```
   Look for columns with UCD `pos.eq.ra` and `pos.eq.dec`.

3. **Resolve target name** (if given) via SIMBAD:
   ```
   curl -s "https://simbad.cds.unistra.fr/simbad/sim-tap/sync?REQUEST=doQuery&LANG=ADQL&FORMAT=json&QUERY=SELECT+ra,dec+FROM+basic+JOIN+ident+ON+ident.oidref=oid+WHERE+id='TARGET_NAME'"
   ```
   Report: *"Resolved TARGET → RA: X°, Dec: Y°"*.

4. **Build ADQL query** — see Query Construction Rules below.

5. **Build TAP sync URL**:
   - Base: `https://archive.eso.org/tap_cat/sync`
   - Fixed params: `REQUEST=doQuery&LANG=ADQL&FORMAT=text&MAXREC=200`
   - URL-encode the ADQL for the `QUERY=` parameter.
   - **Note**: `tap_cat` uses `FORMAT=text`, not `FORMAT=txt`.

6. **Execute live** when the user says "search", "find", "show me", or similar:
   ```
   curl -s "TAP_URL"
   ```
   Display the first 10 rows as a formatted table.

## Query Construction Rules

- Table names must be **double-quoted** in ADQL: `"VHS_CAT_V3"`, `"VVVX_VIRAC_V2_SOURCES"`.
- Use `SELECT *` unless specific columns are requested; use `TOP N` only when asked.
- **Cone search** — `tap_cat` supports only `CONTAINS/POINT/CIRCLE`, not `INTERSECTS/s_region`:
  ```sql
  SELECT * FROM "VHS_CAT_V3"
  WHERE CONTAINS(POINT('', raj2000, dej2000), CIRCLE('', 83.82, -5.39, 0.1)) = 1
  ```
  Replace `raj2000`/`dej2000` with the actual RA/Dec column names for the target table.
- Default cone radius: **0.1 degrees**.
- Do **not** add extra constraints unless the user specifies them.

## Output Format (always in this order)

1. `sql` code block — ADQL query
2. `url` code block — TAP sync URL
3. Formatted results table (top 10 rows) — only when live execution is triggered
4. One-sentence explanation of what the query returns
5. Footer: `⚠️ Use at your own risk. Results may be partial or incomplete.`

## Reference

TAP_SCHEMA metadata columns, notable collections, URL construction examples → [REFERENCE.md](REFERENCE.md)
