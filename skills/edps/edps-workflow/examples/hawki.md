# HAWK-I Workflow Examples

HAWK-I is a wide-field near-IR imager with a 4-detector mosaic.  The workflow supports
regular observations and a fast "HIT" (High-cadence Imaging Technology) sub-mode.

---

## 1. `params.get_workflow_param()` — OO-style parameter access

An object-oriented alternative to the `get_parameter(params, name)` function.
Both read workflow parameters from `parameters.yaml` **and** values set by
`with_dynamic_parameter()`:

```python
from edps import JobParameters

# OO style (HAWK-I convention)
def readgain_from_static(params: JobParameters) -> bool:
    return params.get_workflow_param('use_static_readgain') == "TRUE"

def astrom_local_2mass(params: JobParameters) -> bool:
    return (params.get_workflow_param('astrom_catalog')     == '2MASS'
            and params.get_workflow_param('cds_astrom_catalog') == 'none')

# Which_band is set via with_dynamic_parameter() — also readable via get_workflow_param()
def is_broad_and_requested(params: JobParameters) -> bool:
    return (params.get_workflow_param("which_band") == "BROAD"
            and params.get_workflow_param("associate_night_standard_star") == "TRUE")
```

> `params.get_workflow_param(name)` and `get_parameter(params, name)` access the same
> pool — both workflow parameters (from `parameters.yaml`) and runtime-computed values
> (from `with_dynamic_parameter()`).  Choose whichever style is consistent in your codebase.

---

## 2. `job.parameters.workflow_parameters` in `with_job_processing()`

Inside a `with_job_processing()` function, read **workflow parameters** from
`job.parameters.workflow_parameters` (a dict) and use them to set recipe parameters.
This lets recipe behaviour be configured at the workflow level:

```python
from edps import Job

def catalogs(job: Job):
    # Read workflow parameters and forward them as recipe parameters.
    # job.command is the recipe name — enables generic parameter keys like "hawki.{recipe}.cdssearch_astrom"
    job.parameters.recipe_parameters[f'hawki.{job.command}.cdssearch_astrom'] = \
        job.parameters.workflow_parameters['cds_astrom_catalog']
    job.parameters.recipe_parameters[f'hawki.{job.command}.cdssearch_photom'] = \
        job.parameters.workflow_parameters['cds_photom_catalog']
```

> `job.command` is the recipe name of the current task.  Using it to build the
> parameter key makes the same `catalogs()` function reusable across tasks that call
> different recipes (e.g. `standard`, `object`, `object_postprocess`).

---

## 3. `job.associated_files` mutation — filtering inputs at job time

In the same `with_job_processing()` function, inspect `job.task_name` to apply
task-specific logic, and **filter `job.associated_files`** to remove unwanted files
before the recipe runs:

```python
from edps import Job

def catalogs(job: Job):
    # ... set recipe parameters (section 2 above) ...

    # Only for science tasks: strip MATCHSTD_PHOTOM files that propagated from upstream.
    # (The with_input_filter(REJECT) handles new associations; this handles inherited ones.)
    if job.task_name in ["object", "object_postprocess"]:
        job.associated_files = [f for f in job.associated_files
                                if f.classification != "MATCHSTD_PHOTOM"]
```

> `job.task_name` is the string name passed to `task('name')`.  Checking it inside a
> shared `with_job_processing()` function lets one function handle multiple tasks with
> slightly different behaviour — without duplicating the function.

---

## 4. `alternative_association()` stored as variable + `.with_alternatives()`

The **anonymous** `alternative_association()` can be stored in a variable and reused
across tasks (not just the named `alternative_associated_inputs()` form).
HAWK-I also uses the alias `.with_alternatives()` instead of `.with_alternative_associated_inputs()`:

```python
from edps import alternative_association

# Anonymous form stored as variable — reusable just like alternative_associated_inputs()
astrometric_catalogs = (alternative_association()
                        .with_associated_input(astrom_2mass,   condition=astrom_local_2mass,  min_ret=0)
                        .with_associated_input(astrom_ppmxl,   condition=astrom_local_ppmxl,  min_ret=0)
                        .with_associated_input(astrom_custom,  condition=astrom_local_custom, min_ret=0))

readgain_calibrations = (alternative_association()
                         .with_associated_input(master_readgain, condition=readgain_from_static)
                         .with_associated_input(detector_noise, [MASTER_READGAIN],
                                                condition=readgain_from_raw_calib, min_ret=0))

# .with_alternatives() is an alias for .with_alternative_associated_inputs()
standard = (task('standard')
            .with_recipe('hawki_standard_process')
            .with_main_input(raw_standard)
            .with_alternatives(readgain_calibrations)     # alias
            .with_alternatives(astrometric_catalogs)
            .with_alternatives(photometric_catalogs)
            ...
            .build())
```

> All three forms (`.with_alternatives()`, `.with_alternative_associated_inputs()`,
> `alternative_association()`, `alternative_associated_inputs()`) are interchangeable —
> choose based on codebase convention.

---

## 5. Product-isolation task: `copy_all` + `FilterMode.SELECT` as a named relay

When downstream tasks need to conditionally include **specific products** from an
upstream task — while also excluding those same products when they arrive via other
paths — insert a dedicated isolation task:

```python
from edps import copy_all, FilterMode

# Step 1: create an isolation task that extracts only MATCHSTD_PHOTOM from standard
filter_standard_products = (task("filter_standard_products")
                            .with_main_input(standard)
                            .with_function(copy_all)
                            .with_input_filter(MATCHSTD_PHOTOM, mode=FilterMode.SELECT)
                            .build())

# Step 2: associate the isolated product conditionally
science = (task('object')
           ...
           # Include the product only for narrow-band (optional) or broad-band (required)
           .with_associated_input(filter_standard_products, [MATCHSTD_PHOTOM], min_ret=0,
                                  condition=is_narrow_and_requested)
           .with_associated_input(filter_standard_products, [MATCHSTD_PHOTOM], min_ret=1,
                                  condition=is_broad_and_requested)
           # REJECT any MATCHSTD_PHOTOM that arrives via inheritance from upstream
           .with_input_filter(MATCHSTD_PHOTOM, mode=FilterMode.REJECT)
           .build())
```

> The two-step is necessary because EDPS can propagate products transitively.  The
> `FilterMode.REJECT` removes files that arrive automatically; the conditional
> `.with_associated_input()` puts back only the ones explicitly requested.  The
> isolation task (`copy_all` + `FilterMode.SELECT`) provides a clean single-product
> datasource to associate against.

---

## 6. `num_inputs` dynamic parameter — exposing file count to conditions

Compute the input count as a dynamic parameter so condition functions can branch on
it without reading files themselves:

```python
from edps import List, ClassifiedFitsFile, JobParameters

def num_inputs(files: List[ClassifiedFitsFile]) -> int:
    return len(files)

def should_combine(params: JobParameters) -> bool:
    # Skip the task if running in IDP mode with fewer than 4 inputs
    return (params.get_workflow_param("is_idp") == "FALSE"
            or params.get_workflow_param("num_inputs") > 3)

science = (task('object')
           .with_condition(should_combine)
           .with_dynamic_parameter("num_inputs", num_inputs)   # count is now accessible
           .with_dynamic_parameter("which_band", which_band)
           ...
           .build())
```
