# FORS workflow examples

Examples drawn from the FORS instrument pipeline (`fors_wkf.py`, `fors_common.py`, `fors_imaging_wkf.py`, `fors_spec_wkf.py`, `fors_spectra_calibrations.py`).

FORS supports five observing modes (imaging, LSS/MOS/MXU spectroscopy, polarimetry, HIT, PMOS), each with its own workflow file. Its patterns are a good reference for: splitting one instrument into multiple workflow files, sharing common tasks across modules, task factory functions with conditional builder chaining, a report-only task without a recipe, and exhaustive mode-dispatch via repeated `with_associated_input()`.

---

## Multiple workflow files for one instrument

When an instrument has several very different observing modes, split each mode into its own `_wkf.py` file and collect them in a master `fors_wkf.py` that imports them all:

```python
# fors_wkf.py — master entry point, nothing else
from . import fors_common
from . import fors_imaging_wkf
from . import fors_spec_wkf
from . import fors_pmos_wkf
from . import fors_ipol_wkf
from . import fors_hit_wkf

__title__ = "FORS workflow"

__all__ = ['fors_common', 'fors_imaging_wkf', 'fors_spec_wkf',
           'fors_pmos_wkf', 'fors_ipol_wkf', 'fors_hit_wkf']
```

Each file ending in `_wkf.py` is a valid EDPS workflow on its own. The master file registers them all under the same instrument name.

---

## Shared tasks in `fors_common.py`

Tasks used by more than one mode-specific workflow are defined once in `fors_common.py` and imported by each mode file. This avoids duplicating `bias`, `sky_flat`, etc.:

```python
# fors_common.py — defines bias, skyflat_img, screenflat_img, and shared calibrations
from edps import task, QC1_CALIB, ReportInput
from .fors_datasources import *
from .fors_spectra_calibrations import spectra_calibrations

science_notc = "science_notc"   # custom meta target shared across modes

bias = (task('bias')
    .with_recipe('fors_bias')
    .with_main_input(raw_bias)
    .with_meta_targets([QC1_CALIB])
    .build())

calibration_lss, calibration_mos, calibration_mxu, calibration_std, \
    calibration_pmos, calibration_hc_lss, calibration_hc_std = spectra_calibrations(bias)
```

```python
# fors_spec_wkf.py — imports shared tasks by name
from .fors_common import (bias, calibration_lss, calibration_std,
                          calibration_mxu, calibration_mos, science_notc)
```

---

## `imported_tasks` — declaring cross-module task dependencies

When a workflow file uses tasks defined in another module, declare them explicitly in an `imported_tasks` list so EDPS can discover and register the full dependency graph:

```python
# fors_spec_wkf.py
imported_tasks = [bias, calibration_lss, calibration_std,
                  calibration_mxu, calibration_mos,
                  calibration_hc_std, calibration_hc_lss]
```

Without this list, EDPS may not resolve upstream dependencies defined in a different file.

---

## Task factory function with conditional builder chaining

Use a plain Python function (not a subworkflow) to build multiple similar tasks. The builder's fluent API lets you conditionally add methods before calling `.build()`:

```python
# fors_spectra_calibrations.py
def create_calibration(main_input, bias, wave, ins_mode,
                       task_recipe, report_flat, report_wave, maxret):
    # Build the common part of the task
    task_building = (task('calibration_' + ins_mode)
        .with_recipe(task_recipe)
        .with_report('fors_rawdisp',  ReportInput.RECIPE_INPUTS)
        .with_report(report_flat,     ReportInput.RECIPE_INPUTS_OUTPUTS)
        .with_report(report_wave,     ReportInput.RECIPE_INPUTS_OUTPUTS)
        .with_main_input(main_input)
        .with_associated_input(wave, max_ret=maxret)
        .with_associated_input(bias, [MASTERBIAS])
        .with_associated_input(grism_table)
        .with_dynamic_parameter('ins_mode', which_observation_type)
        .with_meta_targets([QC1_CALIB]))

    # Conditionally add mode-specific processing before building
    if ins_mode == 'mxu':
        task_building = task_building.with_job_processing(set_fors_calib_mxu)
    elif ins_mode == 'hc_lss':
        task_building = task_building.with_job_processing(set_endwavelength)

    return task_building.build()   # .build() called once at the end
```

```python
@subworkflow("spectra_calibrations", "")
def spectra_calibrations(bias):
    calibration_lss = create_calibration(raw_screen_flat_lss, bias, raw_wave,
                                         'lss', 'fors_calib', 'fors_flat_spec_lss', 'fors_wave_cal', 1)
    calibration_mxu = create_calibration(raw_screen_flat_mxu, bias, raw_wave,
                                         'mxu', 'fors_calib', 'fors_flat_spec_mos', 'fors_wave_cal', 1)
    # ... repeat for mos, std, pmos, hc_lss, hc_std
    return calibration_lss, calibration_mos, calibration_mxu, ...
```

The key insight: the builder is an object you can hold in a variable and pass to `if/elif` branches — `.build()` is always called last.

---

## Report-only task (no recipe)

A task without `.with_recipe()` runs no recipe — it only generates QC reports from its inputs. Use this for monitoring tasks where the raw data is interesting but no data products need to be created:

```python
# Task produces a QC report but no reduced product
std_img_noc = (task("no_supported_photometric_standard")
    .with_report("fors_rawdisp", ReportInput.RECIPE_INPUTS)
    .with_main_input(raw_std_img_unsupported_filters)
    .with_associated_input(bias, [MASTERBIAS])
    .with_associated_input(skyflat_img, [pcls_masterSkyFlat_img])
    .with_meta_targets([QC1_CALIB, CALCHECKER])
    .build())
```

No `.with_recipe()` call — EDPS will process the task's inputs through the report template only.

---

## Exhaustive mode-dispatch via repeated `with_associated_input()`

When the same upstream task must be associated under several different conditions and match rules (one per observing sub-mode), chain multiple `with_associated_input()` calls for the same source — each with a different `condition` and `match_rules`:

```python
science = (task('science_spectra')
    .with_recipe('fors_science')
    .with_main_input(raw_sci)
    # standard_mos association: 6 branches — mandatory for regular obs, optional for others
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_lss_regular,  match_rules=match_std_mos_to_lss)
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_mos_regular,  match_rules=match_std_mos_to_mosmxu)
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_mxu_regular,  match_rules=match_std_mos_to_mosmxu)
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_lss_other,    match_rules=match_std_mos_to_lss,       min_ret=0)
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_mos_other,    match_rules=match_std_mos_to_mosmxu,    min_ret=0)
    .with_associated_input(standard_mos, [specphot_table],
                           condition=is_mxu_other,    match_rules=match_std_mos_to_mosmxu,    min_ret=0)
    ...
    .build())
```

Each `condition` function is mutually exclusive — only one branch activates per input file. This is preferred over `alternative_associated_inputs()` when the match rules also differ per branch.
