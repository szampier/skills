# VISIR Workflow Examples

VISIR is a mid-IR imager and spectrograph with three observation modes: imaging,
long-slit spectroscopy, and échelle spectroscopy. Each mode has its own `_wkf.py`;
`visir_wkf.py` assembles them via `__all__`.

---

## 1. Four `@subworkflow` wrappers on one internal function

When two independent axes (here: `data_type` × `data_format`) produce related but
distinct task graphs, write one shared internal function and wrap it with as many
`@subworkflow` decorators as needed:

```python
# visir_reduce_spectra.py
def process_spectra(raw_input, data_type, data_format):
    suffix  = data_format + '_' + data_type        # e.g. "long_slit_science"
    # ... build tasks using suffix for names, product types, and conditionals ...
    return VisirSpectra(repack=repack, undistort=undistort, destripe=destripe, processed=process)

@subworkflow("process_science_longslit", "")
def process_science_longslit(raw_input):
    return process_spectra(raw_input, "science", "long_slit")

@subworkflow("process_standard_longslit", "")
def process_standard_longslit(raw_input):
    return process_spectra(raw_input, "standard", "long_slit")

@subworkflow("process_science_echelle", "")
def process_science_echelle(raw_input):
    return process_spectra(raw_input, "science", "echelle")

@subworkflow("process_standard_echelle", "")
def process_standard_echelle(raw_input):
    return process_spectra(raw_input, "standard", "echelle")
```

> Each wrapper is a distinct named node in the EDPS graph with its own inputs.
> The internal function is never called directly by EDPS — only through the wrappers.

---

## 2. String-driven task naming and product-type renaming

Use string parameters to drive task names **and** choose the right product-type
constant for `with_input_map()` — all in one place:

```python
def process_spectra(raw_input, data_type, data_format):
    suffix = data_format + '_' + data_type   # "long_slit_science", "echelle_standard", …

    # Task name derived from suffix
    repack = task("repack_" + suffix).with_recipe("visir_util_repack")...build()
    undistort = task("undistort_" + suffix)...build()
    destripe  = task("destripe_" + suffix)...build()

    # Product type constant chosen by (data_type, data_format)
    if data_type == "science":
        renamed = SPEC_OBS_LMR_PREPROCESSED if data_format == "long_slit" \
                  else SPEC_OBS_HRG_PREPROCESSED
    else:
        renamed = SPEC_CAL_PHOT_PREPROCESSED if data_format == "long_slit" \
                  else SPEC_CAL_PHOT_HRG_PREPROCESSED

    process = (task("process_" + suffix)
               .with_recipe("visir_old_spc_obs")
               .with_main_input(destripe)
               .with_input_map({DESTRIPED: renamed})         # ← chosen above
               .with_input_filter(UNDISTORTED, mode=FilterMode.REJECT)
               .build())
```

This avoids duplicate code while producing clearly-named tasks for each mode
combination in the EDPS provenance graph.

---

## 3. Conditional `.with_report()` inside a factory function

Attach QC reports only for specific `(data_type, data_format)` combinations by
branching on the builder before calling `.build()`:

```python
def process_spectra(raw_input, data_type, data_format):
    ...
    repack_building = (task("repack_" + suffix)
                       .with_recipe("visir_util_repack")
                       .with_main_input(raw_input)
                       .with_meta_targets(metatargets))

    # Add rawdisp report only for standard star observations
    if data_type == "standard":
        repack = repack_building.with_report("visir_rawdisp", ReportInput.RECIPE_INPUTS).build()
    else:
        repack = repack_building.build()

    ...
    process_building = task("process_" + suffix).with_recipe(...)...

    # Add specphot report only for long-slit standard
    if suffix == 'long_slit_standard':
        process = process_building.with_report("visir_specphot_std",
                                               ReportInput.RECIPE_INPUTS_OUTPUTS).build()
    else:
        process = process_building.build()
```

---

## 4. Multi-key `with_input_map()`

`with_input_map()` accepts any number of `{SRC: DST}` pairs in a single call — useful
when a recipe expects several product types under different names than those produced
upstream:

```python
photometry = (task("photometry_standard")
              .with_recipe("visir_old_img_phot")
              .with_main_input(qc)
              .with_associated_input(undistort, match_rules=match_calib)
              .with_input_filter(QC_HEADER, COADDED_CONTRIBUTION_COMBINED, mode=FilterMode.REJECT)
              .with_input_map({
                  COADDED_IMAGE:          IM_CAL_PHOT_PREPROCESSED,
                  COADDED_IMAGE_COMBINED: IM_CAL_PHOT_PREPROCESSED,
                  COADDED_WEIGHT:         WEIGHT_MAP,
                  COADDED_WEIGHT_COMBINED: WEIGHT_MAP,
                  QC_HEADER_COMBINED:     QC_HEADER,   # rename while REJECT removes the original
              })
              .with_meta_targets([qc1calib])
              .build())
```

> `with_input_filter(..., REJECT)` and `with_input_map()` can be combined: reject the
> original type while renaming a related type to the same destination name.

---

## 5. Placeholder tasks for unsupported observation modes

Register observation types that the pipeline does not yet process by defining tasks
**with no recipe**.  EDPS tracks and groups the raw data without invoking any recipe:

```python
# visir_unsupported_wkf.py
@subworkflow("raster_coronagraphy_and_sam", "")
def coronagraphy_and_sam(raw_sci_coro, raw_sci_sam, raw_sci_raster, ...):
    science_coronagraphy = (task("science_coronagraphy")
                            .with_main_input(raw_sci_coro)
                            .with_associated_input(raw_standard_others)
                            .with_meta_targets([qc0, calchecker])
                            .build())   # ← no .with_recipe()

    science_sam = (task("science_sam")
                   .with_main_input(raw_sci_sam)
                   .with_associated_input(raw_standard_others)
                   .with_meta_targets([qc0, calchecker])
                   .build())
    ...
    return science_coronagraphy, science_sam, science_raster
```

> A task without `.with_recipe()` is a **report-only or no-op task**.  It ingests the
> raw data into the EDPS provenance graph and makes it visible in the UI, but runs
> no recipe.  Use it to handle unsupported modes gracefully rather than silently
> ignoring them.

---

## 6. `__all__` to declare sub-workflow modules

When `visir_wkf.py` imports multiple sub-workflow modules, declare them in `__all__`
so EDPS discovers all tasks across all modules:

```python
# visir_wkf.py
from . import visir_echelle_wkf
from . import visir_imaging_wkf
from . import visir_long_slit_wkf
from . import visir_unsupported_wkf

__all__ = [
    'visir_unsupported_wkf',
    'visir_imaging_wkf',
    'visir_long_slit_wkf',
    'visir_echelle_wkf',
]
```

> This is the module-level equivalent of FORS's `imported_tasks = [...]` list.
> Both serve the same purpose: tell EDPS which additional modules contain tasks
> that belong to this workflow.
