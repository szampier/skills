# ERIS Workflow Examples

ERIS (Enhanced Resolution Imager and Spectrograph) has two sub-instruments:
- **NIX** — near-IR imager and coronagraph
- **SPIFFIER** — near-IR IFU spectrograph

The thin `eris_wkf.py` simply imports both sub-workflow modules; the real logic lives in
`eris_nix_wkf.py` and `eris_spiffier_wkf.py`, plus a set of helper subworkflow files.

---

## 1. Builder-extension helper function

When multiple tasks need the same block of `.with_associated_input()` calls (e.g. flat
selection logic that varies by observing mode), extract that block into a helper that
receives the **partially-built task** and returns it extended:

```python
def associate_flat_calibrations(task_building, nix_flat_lamp, nix_flat_twilight, nix_flat_sky, observ_mode):
    # Sky flats — condition selects the right wavelength range
    task_building = (task_building
                     .with_associated_input(nix_flat_sky, [MASTER_FLAT_SKY_HIFREQ, MASTER_SKY_LOFREQ],
                                            match_rules=match_nix_skyflat, condition=is_long)
                     .with_associated_input(nix_flat_sky, [MASTER_FLAT_SKY_HIFREQ, MASTER_SKY_LOFREQ],
                                            match_rules=match_skyflat_long_bra, condition=is_long_bra))

    # Lamp flats — selection depends on observ_mode
    if observ_mode in ["coronagraphy", "sam"]:
        task_building = (task_building
                         .with_associated_input(nix_flat_lamp, [...], min_ret=0, condition=is_short_broad)
                         .with_associated_input(nix_flat_lamp, [...], min_ret=1, condition=is_short))
    if observ_mode in ["cube", "images"]:
        task_building = task_building.with_associated_input(nix_flat_lamp, [...], min_ret=1,
                                                            condition=is_short_general)
    return task_building

# Usage — start the builder, delegate flat associations, then finish
task_building = (task('process_nix_science_images')
                 .with_recipe('eris_nix_cal_det')
                 .with_main_input(raw_inputs)
                 .with_associated_input(linearity_nix, [GAIN_INFO, COEFFS_CUBE, BP_MAP_NL])
                 .with_associated_input(dark_nix, [MASTER_DARK_IMG]))

task_building = associate_flat_calibrations(task_building, nix_flat_lamp, nix_flat_twilight, nix_flat_sky,
                                            observ_mode)
science_task = task_building.build()
```

This is preferable to copy-pasting the same associations into every task variant.

---

## 2. Dynamic parameters + `get_parameter()` condition chain

`with_dynamic_parameter()` reads FITS keywords at run time and stores values as named
parameters on the job.  Condition functions then call `get_parameter()` to branch on
those values — **conditions that depend on other dynamic parameters, not raw keywords**:

```python
# eris_task_function.py — dynamic parameter functions
from edps import List, ClassifiedFitsFile, JobParameters, get_parameter

def which_band(files: List[ClassifiedFitsFile]) -> str:
    band = files[0].get_keyword_value(kwd.ins2_nxfw_name, None)
    if band in ["J", "H", "Ks"]:
        return "short_broad"
    elif band in ["L-Broad", "Mp", "Lp", "Short-Lp"]:
        return "long"
    elif band in ["Br-a", "Br-a-cont"]:
        return "long_bra"
    else:
        return "short"

def is_long(params: JobParameters) -> bool:
    return get_parameter(params, "band_used") == "long" and get_parameter(params, "nix_mode") != "nixLSS"

def is_short_broad(params: JobParameters) -> bool:
    return get_parameter(params, "band_used") == "short_broad" and get_parameter(params, "nix_mode") != "nixLSS"

# In the task builder:
task_building = (task('process_nix_science_images')
                 .with_recipe('eris_nix_cal_det')
                 .with_dynamic_parameter("band_used", which_band)    # reads INS2.NXFW.NAME
                 .with_dynamic_parameter("nix_mode",  which_nix_mode)  # reads INS1.MODE
                 .with_main_input(raw_inputs)
                 # Conditions below read the dynamic params set above:
                 .with_associated_input(nix_flat_sky, [...], condition=is_long)
                 .with_associated_input(nix_flat_lamp, [...], condition=is_short_broad))
```

> **Key insight**: condition functions receive `JobParameters`, not the raw files list.  
> Use `get_parameter(params, "name")` to read a value that was previously set by a
> `with_dynamic_parameter()` call on the same task.

---

## 3. `with_job_processing()` for FITS-keyword-driven recipe parameters

`set_cube_collapse` reads a FITS keyword and sets a recipe parameter accordingly.
Multiple distinct `with_job_processing()` calls are allowed on the same task:

```python
# eris_task_function.py
from edps import Job

def set_cube_collapse(job: Job):
    fmt = job.input_files[0].get_keyword_value(kwd.det_fram_format, None)
    job.parameters.recipe_parameters["eris.eris_nix_cal_det.collapse_cube"] = 1 if fmt == "cube" else 0

def set_detmon_thresholds(job: Job):
    curname = job.input_files[0].get_keyword_value(kwd.det_read_curname, None)
    if curname == "FAST_UNCORR":
        job.parameters.recipe_parameters[saturation_limit] = 46500
        job.parameters.recipe_parameters[gain_threshold]   = 35000
    else:
        job.parameters.recipe_parameters[saturation_limit] = 11000
        job.parameters.recipe_parameters[gain_threshold]   = 8000

# In the task builder:
coronagraphy = (task('process_nix_science_coronagraphy')
                .with_recipe('eris_nix_cal_det')
                .with_job_processing(set_cube_collapse)   # sets collapse_cube from FITS
                .with_main_input(raw_coronagraphy)
                .build())
```

---

## 4. Two `@subworkflow` decorators wrapping the same internal function

When science and calibration data follow the same reduction cascade but must appear as
**separately-named subworkflows** in the EDPS graph, wrap a shared internal function
with two different `@subworkflow` decorators:

```python
from edps import subworkflow

def nix_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets):
    # ... full reduction logic shared by both ...
    return NixProcessedObservations(...)

@subworkflow("science_reduction_nix", "")
def nix_science_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets):
    return nix_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets)

@subworkflow("on_sky_calibrations_reduction_nix", "")
def nix_on_sky_calibrations_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets):
    return nix_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets)
```

> EDPS names nodes in its graph after the `@subworkflow` label, not the Python function
> name. Two wrappers → two distinct named nodes in the provenance graph.

---

## 5. Metatargets passed as argument and conditionally augmented

Metatargets are **not always hardcoded** inside a subworkflow.  Pass them as a function
argument, then conditionally add `science` based on observing context:

```python
from edps import science

def nix_reduction(raw_inputs, ..., observ_mode, observ_type, metatargets):
    # science meta-target is only added for actual science observations
    if observ_mode in ["cube", "coronagraphy", "sam"] and observ_type == "science":
        metatargets = metatargets + [science]

    science_task = (task(...)
                    .with_meta_targets(metatargets)  # uses the (possibly augmented) list
                    .build())
    ...

# Call sites supply the base metatargets; the function decides whether to add `science`
coronagraphy = nix_science_reduction(raw_coronagraphy, ..., 'coronagraphy', 'science',
                                     [qc0, calchecker])
calib = nix_on_sky_calibrations_reduction(raw_on_sky_calib_nix_image, ..., 'images',
                                          'on_sky_calib', [qc1calib, calchecker])
```

---

## 6. `@dataclass` return with `Optional[Task]` fields

When a subworkflow conditionally creates tasks (some modes produce 4 tasks, others only
1), a `@dataclass` with `Optional[Task]` fields communicates the result clearly:

```python
from dataclasses import dataclass
from typing import Optional
from edps import Task

@dataclass
class NixProcessedObservations:
    science_reduction: Task
    sky_subtraction: Optional[Task] = None
    astrometry:       Optional[Task] = None
    photometry:       Optional[Task] = None
    stacking:         Optional[Task] = None
```

The caller unpacks only the fields relevant for its wiring:

```python
image_reduction = nix_science_reduction(..., 'images', 'science', [...])
image_nix         = image_reduction.science_reduction
image_subtraction = image_reduction.sky_subtraction
image_astrometry  = image_reduction.astrometry
image_photometry  = image_reduction.photometry
image_reduced     = image_reduction.stacking

# SAM / coronagraphy: only the first step is populated
sparse_aperture_mask = nix_science_reduction(..., 'sam', 'science', [...])
# sparse_aperture_mask is a Task directly (short-circuit return, not a dataclass)
```

---

## 7. `alternative_associated_inputs()` with both flux and telluric standards optional

Group two optional standard-star products into one alternative block so the science
task can proceed without either:

```python
from edps import alternative_associated_inputs

standard_star_calibrations_spiffier = (alternative_associated_inputs()
                                       .with_associated_input(flux_standard_ifu, [RESPONSE], min_ret=0)
                                       .with_associated_input(telluric_standard_ifu, [RESPONSE], min_ret=0))

science_ifu = (task('object_ifu')
               .with_recipe('eris_ifu_jitter')
               .with_main_input(raw_object_spiffier)
               .with_alternative_associated_inputs(standard_star_calibrations_spiffier)
               # ... other inputs ...
               .build())
```

The `min_ret=0` on each branch means the task degrades gracefully when no standard star
was observed in the same night.
