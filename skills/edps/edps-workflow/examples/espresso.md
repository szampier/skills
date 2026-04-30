# ESPRESSO workflow examples

Examples drawn from the ESPRESSO instrument pipeline (`espresso_wkf.py` and subworkflows).

---

## Subworkflows

Use subworkflows to group related tasks into a reusable function. The `@subworkflow` decorator registers the function with EDPS and keeps `_wkf.py` clean.

```python
# espresso_detector_properties.py
from edps import task, subworkflow, ReportInput, QC1_CALIB, CALCHECKER
from .espresso_datasources import *

@subworkflow("detector_properties", "")
def detector_properties():
    bias = (task('bias')
            .with_recipe('espdr_mbias')
            .with_report('espresso_master_bias', ReportInput.RECIPE_INPUTS_OUTPUTS, driver='png')
            .with_main_input(raw_bias)
            .with_associated_input(ccd_geom)
            .with_associated_input(inst_config)
            .with_meta_targets([QC1_CALIB])
            .build())

    dark = (task('dark')
            .with_recipe('espdr_mdark')
            .with_main_input(raw_dark)
            .with_associated_input(bias, [MASTER_BIAS_RES])   # uses output of bias task
            .with_associated_input(ccd_geom)
            .with_meta_targets([QC1_CALIB, CALCHECKER])
            .build())

    return bias, dark   # caller receives task objects to use as upstream calibrations
```

```python
# espresso_wkf.py
from .espresso_detector_properties import detector_properties

bias, dark, detmon = detector_properties()   # unpack the returned tasks
```

---

## `match_rules()` — per-task association overrides

Use `match_rules()` when one task needs different time-range logic than the datasource default. ESPRESSO science uses wavelength calibrations preferably taken the *next* day (not the same night):

```python
from edps import match_rules, RelativeTimeRange
from edps.generator.time_range import ONE_DAY, TWO_WEEKS, UNLIMITED

# Prefer calibrations from the day *after* the observation
next_day = (match_rules()
    .with_match_keywords(instrument_setup, time_range=RelativeTimeRange(-0, 1), level=0)
    .with_match_keywords(instrument_setup, time_range=ONE_DAY, level=1)
    .with_match_keywords(instrument_setup, time_range=TWO_WEEKS, level=2)
    .with_match_keywords(instrument_setup, time_range=UNLIMITED, level=3))

science = (task('object')
    .with_recipe('espdr_sci_red')
    .with_main_input(raw_science)
    .with_associated_input(wave_thar_fp,
                           [WAVE_MATRIX_THAR_FP_A, DLL_MATRIX_THAR_FP_A],
                           match_rules=next_day,     # override datasource default
                           condition=is_ut)          # only for UT telescope
    .build())
```

---

## Condition-dependent associated inputs

Use `condition=fn` on `.with_associated_input()` to attach calibrations only when a runtime condition is met. ESPRESSO has two telescope modes (UT and PoET) that need different flat and wavelength associations:

```python
science = (task('object')
    .with_recipe('espdr_sci_red')
    .with_main_input(raw_science)
    # UT telescope: use default (same-night) flat association
    .with_associated_input(flat,
                           [ORDER_PROFILE_A, ORDER_PROFILE_B, FSPECTRUM_A, FSPECTRUM_B],
                           condition=is_ut)
    # PoET telescope: use ±1-day flat association
    .with_associated_input(flat,
                           [ORDER_PROFILE_A, ORDER_PROFILE_B, FSPECTRUM_A, FSPECTRUM_B],
                           match_rules=poet_assoc_rules,
                           condition=is_poet)
    # Optional input (min_ret=0): only required for IDP stream
    .with_associated_input(flux, [ABS_EFF_A], min_ret=0, condition=is_not_idp_stream)
    .with_associated_input(flux, [ABS_EFF_A], min_ret=1, condition=is_idp_stream)
    .build())
```

`condition` functions receive the current observation's metadata and return `True`/`False`.

---

## Task-level conditions

Use `.with_condition(fn)` to skip an entire task if it should not run for a given input:

```python
combine_science = (task("combine_science")
    .with_condition(combination_limit)   # skip if > 250 input files
    .with_recipe("esotk_spectrum1d_combine")
    .with_main_input(science)
    .with_dynamic_parameter("n_inputs", n_inputs)
    .with_input_filter(S1D_FINAL_A)
    .with_input_map({S1D_FINAL_A: SPECTRUM_1D})   # rename product type for the recipe
    .with_meta_targets([SCIENCE])
    .build())
```

---

## `with_input_map()` — renaming product types

When a recipe expects a product type name that differs from what the upstream task produces, use `with_input_map()` to rename it:

```python
.with_input_map({S1D_FINAL_A: SPECTRUM_1D})
# S1D_FINAL_A is the output classification from the upstream task;
# SPECTRUM_1D is the tag the combine recipe reads from its input SOF.
```

---

## `with_report()` — QC reports

Attach QC report templates to a task for automatic HTML/PNG generation:

```python
bias = (task('bias')
    .with_recipe('espdr_mbias')
    .with_report('espresso_rawdisp', ReportInput.RECIPE_INPUTS, driver='png')
    .with_report('espresso_master_bias', ReportInput.RECIPE_INPUTS_OUTPUTS, driver='png')
    ...
    .build())
```

| `ReportInput` value | What data is passed to the template |
|---|---|
| `RECIPE_INPUTS` | input files only |
| `RECIPE_INPUTS_OUTPUTS` | input + output files |
| `RECIPE_OUTPUTS` | output files only |

---

## `max_ret` — upper bound on associated inputs

Use `max_ret=N` to cap how many files of a given type are associated. ESPRESSO caps contamination frames at 1:

```python
.with_associated_input(contam_fp, [MASTER_CONTAM_FP], min_ret=0, max_ret=1)
```

---

## Multiple classification rules on one datasource

A single datasource can cover several raw types. ESPRESSO science accepts five simultaneous-calibration modes:

```python
raw_science = (data_source("OBJECT")
    .with_classification_rule(science_fp_class)    # simultaneous FP
    .with_classification_rule(science_thar_class)  # simultaneous ThAr
    .with_classification_rule(science_sky_class)   # simultaneous sky
    .with_classification_rule(science_lfc_class)   # simultaneous LFC
    .with_classification_rule(science_dark_class)  # dark fibre
    .with_setup_keywords(setup + [kwd.telescop])
    .with_grouping_keywords([kwd.unique])
    .build())
```

---

## Static calibrations with `IN_THE_PAST`

Static files (geometry tables, line lists, masks) are always in the past. Use `IN_THE_PAST` as the time range:

```python
from edps.generator.time_range import IN_THE_PAST, UNLIMITED

ccd_geom = (data_source()
    .with_classification_rule(ccd_geom_class)
    .with_match_keywords([kwd.det_binx, kwd.det_biny], time_range=IN_THE_PAST)
    .with_match_keywords([kwd.det_binx, kwd.det_biny], time_range=UNLIMITED, level=3)
    .build())
```

Pass it directly to `.with_associated_input()` without specifying product tags (EDPS infers them from the classification rule):

```python
.with_associated_input(ccd_geom)
.with_associated_input(inst_config)
```

---

## Dual-output instruments (A/B fibres)

For instruments with two arms or fibres, use separate product-type tags per arm in the associated-input list. ESPRESSO produces arm-A and arm-B products from flat and wavelength tasks:

```python
.with_associated_input(flat,
                       [ORDER_PROFILE_A, ORDER_PROFILE_B,
                        FSPECTRUM_A,     FSPECTRUM_B,
                        BLAZE_A,         BLAZE_B])
.with_associated_input(wave_thar_fp,
                       [WAVE_MATRIX_THAR_FP_A, DLL_MATRIX_THAR_FP_A])
.with_associated_input(wave_fp_thar,
                       [WAVE_MATRIX_FP_THAR_B, DLL_MATRIX_FP_THAR_B])
```

When only a subset of products from a task should be passed to a recipe, use `.with_input_filter()` to whitelist them:

```python
.with_input_filter(MASTER_BIAS_RES, HOT_PIXEL_MASK, BAD_PIXEL_MASK,
                   ORDER_TABLE_A, ORDER_TABLE_B,
                   ORDER_PROFILE_A, ORDER_PROFILE_B,
                   FSPECTRUM_A, FSPECTRUM_B, BLAZE_A, BLAZE_B,
                   WAVE_MATRIX_THAR_FP_A, DLL_MATRIX_THAR_FP_A,
                   WAVE_MATRIX_FP_THAR_B, DLL_MATRIX_FP_THAR_B,
                   inst_config_class, ccd_geom_class)
```
