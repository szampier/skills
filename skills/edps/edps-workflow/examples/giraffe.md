# GIRAFFE Workflow Examples

GIRAFFE (FLAMES/GIRAFFE) is a multi-object spectrograph with several observing modes:
**Medusa1**, **Medusa2**, **ARGUS**, **IFU1**, **IFU2** — all handled by a single workflow.

---

## 1. `job.setup` mutation inside `with_job_processing()`

Normally `with_job_processing()` functions modify recipe parameters.  They can also
**add a keyword to the job's grouping setup** by writing to `job.setup`.  This changes
which FITS keywords EDPS uses to group input files for that job:

```python
from edps import Job

def set_science_parameters_and_setup_kwd(job: Job):
    # Normal parameter injection
    dpr_type = job.input_files[0].get_keyword_value(kwd.dpr_type, None)
    if dpr_type == "OBJECT,SimCal":
        job.parameters.recipe_parameters["giraffe.sgcalibration.zmax"] = -1.
    else:
        job.parameters.recipe_parameters["giraffe.sgcalibration.zmax"] = 10**4.

    # Dynamically add a setup keyword for ARGUS mode only
    ins_mode = job.input_files[0].get_keyword_value(kwd.ins_mode, None)
    if ins_mode == "ARG":
        ins1_opti1_pos = job.input_files[0].get_keyword_value(kwd.ins1_opti1_pos, None)
        job.setup[kwd.ins1_opti1_pos] = ins1_opti1_pos   # ← adds to grouping key
```

> `job.setup` is a dict of `{keyword: value}` pairs that EDPS uses to match files into
> the same processing group.  Writing a new entry here restricts grouping further —
> useful when one observing mode needs an extra discriminating keyword that other modes
> don't have.

---

## 2. Condition functions reading workflow parameters (not dynamic parameters)

`get_parameter()` can read **workflow parameters** defined in `parameters.yaml`, not
only values set by `with_dynamic_parameter()`.  This allows task-level conditions to be
toggled at deployment time:

```python
from edps import JobParameters, get_parameter

def process_data(params: JobParameters) -> bool:
    # process_ifu_data is a workflow parameter from parameters.yaml
    # It can be set to "FALSE" to skip IFU data processing entirely
    return get_parameter(params, "process_ifu_data") == "TRUE" or is_medusa(params)

def associate_darks(params: JobParameters) -> bool:
    # Another workflow-level toggle
    return get_parameter(params, "associate_darks") == "TRUE"

# Both used as task-level conditions (skip entire task if False):
science_task = (task("science")
                .with_condition(process_data)           # skip if IFU-only mode disabled
                .with_dynamic_parameter("instrument_mode", which_instrument_mode)
                ...
                .with_associated_input(dark, [master_dark_class], min_ret=0,
                                       condition=associate_darks)  # also on associated input
                .build())
```

> **Key distinction**:
> - `with_dynamic_parameter(name, fn)` → computes `name` from the input files at runtime
> - Workflow parameters → set in `parameters.yaml`; readable via `get_parameter(params, name)` in any condition function

---

## 3. Task factory function vs `@subworkflow`

When two tasks share the same recipe and calibration structure but differ only in their
raw input and name, use a **plain factory function** (no decorator) rather than a
`@subworkflow`:

```python
# giraffe_response.py — plain factory, not a subworkflow
def create_response_task(raw_input, taskname, bias, dark, fibre_flat, wavecal):
    associate_flats = (match_rules()
                       .with_match_keywords(setup, time_range=RelativeTimeRange(-1.5, 1.5), level=0)
                       .with_match_keywords(setup, time_range=TWO_WEEKS, level=1)
                       .with_match_keywords(setup, time_range=UNLIMITED, level=3))

    return (task(taskname)
            .with_recipe('gistandard')
            .with_main_input(raw_input)
            .with_associated_input(fibre_flat, [...], match_rules=associate_flats)
            .with_associated_input(wavecal, [...], match_rules=associate_arcs)
            ...
            .build())

@subworkflow("response", "")
def process_standard_star(raw_std_argus, raw_std_ifu, bias, dark, fibre_flat, wavecal):
    response_argus = create_response_task(raw_std_argus, "response_argus", bias, dark, fibre_flat, wavecal)
    response_ifu   = create_response_task(raw_std_ifu,   "response_ifu",   bias, dark, fibre_flat, wavecal)
    return response_argus, response_ifu
```

> **Rule of thumb**: use `@subworkflow` for the named entry point exposed to EDPS; use
> a plain factory function for internal reuse within the same subworkflow.

---

## 4. `with_function()` with count-based recipe dispatch

A `with_function()` handler can inspect the **number of input files** and choose a
different recipe invocation (or skip the recipe entirely for single-file pass-through):

```python
from edps import RecipeInvocationArguments, InvokerProvider, ProductRenamer, RecipeInvocationResult, \
                 FitsFile, File

def combine_function(input_files, args, invoker_provider, renamer) -> RecipeInvocationResult:
    if len(input_files) == 1:
        # Single file: return it directly without invoking any recipe
        f = input_files[0]
        return RecipeInvocationResult(
            return_code=0,
            output_files=[FitsFile(name=f.file_path, category=f.category)]
        )
    elif len(input_files) == 2:
        # Two files: use strict bad-pixel rejection
        params = args.parameters.copy()
        params["esotk_spectrum1d_combine.bpm.enable"]  = "TRUE"
        params["esotk_spectrum1d_combine.bpm.kappa-high"] = "3."
        return run_recipe(input_files, [], params, 'esotk_spectrum1d_combine', args, ...)
    else:
        # Many files: disable bad-pixel rejection
        params = args.parameters.copy()
        params["esotk_spectrum1d_combine.bpm.enable"] = "FALSE"
        return run_recipe(input_files, [], params, 'esotk_spectrum1d_combine', args, ...)
```

---

## 5. Python-level FITS grouping inside `with_function()`

`with_function()` handlers receive the **flat list** of all input files; Python code
inside must do any sub-grouping.  GIRAFFE groups science spectra by the `FPS` FITS
keyword before combining them:

```python
from collections import defaultdict
from itertools import chain
from astropy.io import fits
from edps import RecipeInvocationArguments, InvokerProvider, ProductRenamer, RecipeInvocationResult

def combine_spectra(args: RecipeInvocationArguments, invoker_provider: InvokerProvider,
                    renamer: ProductRenamer) -> RecipeInvocationResult:
    # Reclassify science files to the expected downstream category
    science_files = [File(f.name, "SPECTRUM_1D", "")
                     for f in args.inputs.combined if f.category == "SCIENCE_RBNSPEC_IDP"]

    # Group by FPS keyword read from FITS headers
    files_same_fps = defaultdict(list)
    for f in science_files:
        with fits.open(f.file_path) as hdul:
            fps = hdul[0].header.get('FPS', 'UNDEFINED')
            files_same_fps[fps].append(f)

    # Run one combine per group and merge results
    results = [combine_function(group, args, invoker_provider, renamer)
               for group in files_same_fps.values()]

    ret_code = 1 if any(r.return_code != 0 for r in results) else 0
    output_files = list(chain.from_iterable(r.output_files for r in results))
    return RecipeInvocationResult(return_code=ret_code, output_files=output_files)

# Wired into the task:
combine_science = (task("combine_science")
                   .with_function(combine_spectra)
                   .with_main_input(science_task)
                   .with_input_map({SCIENCE_RBNSPEC_IDP: SPECTRUM_1D})
                   .with_meta_targets([idp])
                   .build())
```

> Use `with_input_map()` to rename the upstream product type before it enters the
> function — here `SCIENCE_RBNSPEC_IDP → SPECTRUM_1D` so the function works with a
> stable category name.

---

## 6. MJD-epoch-driven recipe parameters in `with_job_processing()`

For instruments with historical configuration changes, `with_job_processing()` can
encode **time-epoch logic** using `MJD-OBS`:

```python
from edps import Job

def set_flat_recipe_parameters(job: Job):
    slit_name = job.input_files[0].get_keyword_value(kwd.ins_slit_name, None)
    exp_mode  = job.input_files[0].get_keyword_value(kwd.ins_exp_mode, None)
    night     = int(job.input_files[0].get_keyword_value(kwd.mjd_obs, None) - 0.5)
    flat_type = job.input_files[0].get_keyword_value(kwd.dpr_type, None)

    # Complex lookup: different fibre lists by slit, mode, AND epoch
    splist = set_splist(slit_name, exp_mode, night, default="setup")
    noise  = set_noise(slit_name, exp_mode, night, flat_type, default=7)

    job.parameters.recipe_parameters["giraffe.fibers.spectra"]   = splist
    job.parameters.recipe_parameters["giraffe.localization.noise"] = noise
```

> Subtract `0.5` from `MJD-OBS` and `int()` it to get the **MJD of the observing
> night's noon** — a convenient epoch key for grouping by calendar date.
