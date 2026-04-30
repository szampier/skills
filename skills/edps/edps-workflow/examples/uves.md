# UVES workflow examples

Examples drawn from the UVES instrument pipeline (`uves_wkf.py`, `uves_datasources.py`, `uves_science.py`).

UVES is a cross-dispersed echelle spectrograph with RED and BLUE arms and a FLAMES fibre feed (MOS mode). Its workflow is a good reference for: reusable `alternative_associated_inputs()` variables, parameter-driven calibration selection, different validity ranges for science vs calibration, `with_function()`, `with_cluster()`, and `@dataclass` subworkflow returns.

---

## Reusable `alternative_associated_inputs()` variables

UVES has RED and BLUE arms. Rather than repeating the arm-selection logic on every task, named `alternative_associated_inputs()` objects are defined once and attached to many tasks.

```python
from edps import alternative_associated_inputs

# Defined once in uves_wkf.py — used by bias, dark, flat, arc, science, ...
bias_calibrations = (alternative_associated_inputs()
    .with_associated_input(bias, [MASTER_BIAS_BLUE], condition=is_blue)
    .with_associated_input(bias, [MASTER_BIAS_REDL, MASTER_BIAS_REDU], condition=is_red))

orderpos_calibrations = (alternative_associated_inputs()
    .with_associated_input(orderpos, [ORDER_TABLE_BLUE], condition=is_blue)
    .with_associated_input(orderpos, [ORDER_TABLE_REDL, ORDER_TABLE_REDU], condition=is_red))
```

```python
# Every task that needs biases just reuses the same variable:
flat = (task("flat")
    .with_recipe("uves_cal_mflat")
    .with_main_input(raw_flat)
    .with_alternative_associated_inputs(bias_calibrations)      # reused
    .with_alternative_associated_inputs(orderpos_calibrations)  # reused
    .with_dynamic_parameter("arm_used", which_arm)
    .with_meta_targets([QC1_CALIB, CALCHECKER])
    .build())
```

---

## Different validity ranges for science vs calibrations

Some calibrations (bias, flat, arc) have tighter time-range requirements when associated to *science* data than when associated to other calibrations. Define a separate `match_rules()` object and use it in a separate `alternative_associated_inputs()`:

```python
# In uves_datasources.py:
associate_bias_to_science = (match_rules()
    .with_match_keywords(match_bias_dark, time_range=FOUR_DAYS, level=0)
    .with_match_keywords(match_bias_dark, time_range=ONE_WEEK,  level=1)
    .with_match_keywords(match_bias_dark, time_range=UNLIMITED, level=3))
```

```python
# In uves_wkf.py: two separate alternative objects — one for calibs, one for science
bias_calibrations = (alternative_associated_inputs()
    .with_associated_input(bias, [MASTER_BIAS_BLUE],            condition=is_blue)
    .with_associated_input(bias, [MASTER_BIAS_REDL, MASTER_BIAS_REDU], condition=is_red))

bias_calibrations_for_science = (alternative_associated_inputs()
    .with_associated_input(bias, [MASTER_BIAS_BLUE],            condition=is_blue,
                           match_rules=associate_bias_to_science)
    .with_associated_input(bias, [MASTER_BIAS_REDL, MASTER_BIAS_REDU], condition=is_red,
                           match_rules=associate_bias_to_science))
```

Calib tasks use `bias_calibrations`; the science task uses `bias_calibrations_for_science`.

---

## Parameter-driven calibration selection

UVES lets users choose at runtime (via `parameters.yaml`) which flat type to use — regular, combined, or iodine. An `alternative_associated_inputs()` with many condition branches encodes all choices:

```python
# Each condition function reads parameters.yaml at runtime
flat_for_science = (alternative_associated_inputs()
    .with_associated_input(combined_flat, [MASTER_FLAT_BLUE],
                           condition=use_combined_flat_in_science_blue,
                           match_rules=associate_flat_to_science)
    .with_associated_input(flat, [MASTER_FLAT_BLUE],
                           condition=use_regular_flat_in_science_blue,
                           match_rules=associate_flat_to_science)
    .with_associated_input(flat, [MASTER_FLAT_REDL, MASTER_FLAT_REDU],
                           condition=use_regular_flat_in_science_red,
                           match_rules=associate_flat_to_science)
    .with_associated_input(iflat, [MASTER_IFLAT_BLUE],
                           condition=use_iodine_flat_blue,
                           match_rules=associate_flat_to_science)
    .with_associated_input(iflat, [MASTER_IFLAT_REDL, MASTER_IFLAT_REDU],
                           condition=use_iodine_flat_red,
                           match_rules=associate_flat_to_science))
```

---

## `with_function()` — dynamic recipe selection

When the recipe to run depends on the input data (e.g. slit vs fibre mode), use `.with_function()` instead of `.with_recipe()`. The function receives the job context and returns the recipe name:

```python
science_slit = (task("object")
    .with_function(run_obs_scired, recipes=["uves_obs_scired"])  # fn decides which recipe
    .with_job_processing(object_type)
    .with_main_input(raw_science)
    .with_alternative_associated_inputs(bias_calibrations_for_science)
    ...
    .with_meta_targets([SCIENCE, QC0, CALCHECKER, IDP])
    .build())
```

`recipes=` lists all recipe names that `run_obs_scired` might select (needed for SOF generation).

---

## `with_cluster()` — spatial grouping of science exposures

Use `.with_cluster()` to group observations that are spatially close (e.g., multiple exposures of the same target from different OBs):

```python
science_slit_idp = (task("science_slit")
    .with_recipe("uves_utl_idp")
    .with_main_input(science_slit)
    .with_condition(is_pointlike)
    .with_cluster('SKY.POSITION',
                  max_diameter="max_diameter",    # from parameters.yaml
                  max_separation="max_separation")
    .with_grouping_keywords([kwd.det_chips, "$combination_kwd"])
    .with_min_group_size(2)
    .with_meta_targets([SCIENCE, IDP])
    .build())
```

`max_diameter` and `max_separation` reference parameter names from `parameters.yaml`.

---

## `with_output_filter()` — keeping only specific output products

Use `.with_output_filter()` to drop output product types you don't need downstream:

```python
science_idp_combined = (task("combine_spectra")
    .with_recipe("esotk_spectrum1d_combine")
    .with_main_input(science_slit_idp)
    .with_input_map({RED_SCI_POINT_BLU:    SPECTRUM_1D,
                     RED_SCI_POINT_RED:    SPECTRUM_1D,
                     FLUXCAL_SCI_POINT_BLU: SPECTRUM_1D,
                     FLUXCAL_SCI_POINT_RED: SPECTRUM_1D})
    .with_output_filter(ESOTK_SPECTRUM_IDP_FORMAT)   # keep only this output type
    .with_meta_targets([IDP, SCIENCE])
    .build())
```

---

## `@dataclass` return from subworkflow

When a subworkflow returns many tasks, a `@dataclass` is cleaner than a tuple — callers access tasks by name rather than by position:

```python
# uves_science.py
from dataclasses import dataclass
from typing import Optional
from edps import Task

@dataclass
class ScienceReduced:
    science_slit: Optional[Task] = None
    solar_slit:   Optional[Task] = None
    science_mos:  Optional[Task] = None
    science_slit_idp: Optional[Task] = None
    science_idp_combined: Optional[Task] = None

@subworkflow("science", "")
def uves_science(...):
    ...
    return ScienceReduced(science_slit=science_slit, solar_slit=solar_slit, ...)
```

```python
# uves_wkf.py
from .uves_science import uves_science

result = uves_science(bias_calibrations_for_science, ...)
# Access by name — no positional index fragility:
_ = result.science_slit
_ = result.science_idp_combined
```

---

## Multiple `with_job_processing()` calls

A single task can call multiple job-processing functions in sequence. UVES uses this to set both the flat size and the master sflat selection on the same task:

```python
sff_ofpos_mos = (task("sff_ofpos_mos")
    .with_recipe("flames_cal_prep_sff_ofpos")
    .with_main_input(raw_fib_ff_odd_mos)
    ...
    .with_job_processing(set_flat_size)       # first function
    .with_job_processing(change_master_sflat) # second function, applied after
    .with_meta_targets([QC1_CALIB, CALCHECKER])
    .build())
```
