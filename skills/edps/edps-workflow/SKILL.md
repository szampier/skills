---
name: edps-workflow
description: "Modify EDPS (ESO Data Processing System) workflows written in Python. Add or change tasks, datasources, and classification rules in instrument workflow packages. Use when editing _wkf.py, _datasources.py, _classification.py, or related workflow files for any ESO instrument pipeline."
---

# edps-workflow

## What is an EDPS workflow?

An EDPS workflow package is a set of Python files that tell the ESO Data Processing System how to classify, group, and process raw data with instrument pipeline recipes. The files follow strict naming conventions: `<instrument>_wkf.py`, `_datasources.py`, `_classification.py`, `_rules.py`, `_keywords.py`, `_task_functions.py`, and `_parameters.yaml`.

## Quick start

Before editing, verify the workflow loads:

```bash
edps -lw                                    # lists all registered workflows
edps -lt -w uves.uves_wkf                   # lists tasks in the UVES workflow (smoke-test)
edps -lt -w uves.uves_wkf -P 5001          # same, if EDPS runs on a non-default port
```

`application.properties` in `~/.edps/` must include a `workflow_dir` pointing to your working directory. Instrument subdirectories must exist (e.g. `muse/muse_wkf.py`).

## Adding a new task (primary workflow)

Follow this checklist in order — each step depends on the previous.

### 1. Understand what exists

Read `_wkf.py` to see all current tasks and their dependency chain. Note:
- Which tasks produce outputs that your new task will use as calibration
- Which `data_source` objects are already defined in `_datasources.py`
- The `PRODUCT_TYPE` classification rules already in `_classification.py`

### 2. Add a classification rule (if a new file type is introduced)

In `_classification.py`, add a rule for any new raw or product type:

```python
from edps import classification_rule

new_type_class = classification_rule('NEW_TYPE', {
    kwd.instrume: "INSTR",
    kwd.dpr_catg: "CALIB",
    kwd.dpr_type: "NEW_TYPE",
})
# For pipeline products (output of a previous recipe):
NEW_PRODUCT = classification_rule("NEW_PRODUCT", {kwd.instrume: "INSTR", kwd.pro_catg: "NEW_PRODUCT"})
```

### 3. Add a datasource (if a new input is needed)

In `_datasources.py`, define a datasource for the new raw input and import the classification rule:

```python
from edps import data_source, RelativeTimeRange
from edps.generator.time_range import UNLIMITED, ONE_DAY

raw_new = (data_source("NEW_TYPE")
    .with_classification_rule(new_type_class)
    .with_min_group_size(3)
    .with_setup_keywords(setup)           # list of FITS keywords defining the setup
    .with_grouping_keywords(grouping)     # how to group files into batches
    .with_match_keywords([kwd.arm, kwd.ins_slit], time_range=ONE_DAY,    level=0)
    .with_match_keywords([kwd.arm, kwd.ins_slit], time_range=UNLIMITED,  level=3)
    .build())
```

No datasource is needed for products from a previous task — pass the task object directly.

### 4. Add condition/parameter functions in `_task_functions.py` (if needed)

If the task requires conditional associations, runtime parameters, or job-time
parameter injection, add the functions in `_task_functions.py` before wiring the task:

```python
# _task_functions.py
from edps import List, ClassifiedFitsFile, JobParameters, get_parameter, Job

def which_arm(files: List[ClassifiedFitsFile]) -> str:
    return files[0].get_keyword_value(kwd.seq_arm, None)

def is_uvb(params: JobParameters) -> bool:
    return get_parameter(params, "arm") == "UVB"

def set_parameters(job: Job):
    job.parameters.recipe_parameters["recipe.param"] = job.input_files[0].get_keyword_value(kwd.some_kwd, None)
```

Skip this step entirely if the task uses only static associations with no conditions.

### 5. Add the task in `_wkf.py`

```python
from edps import task
from .instr_datasources import raw_new

new_task = (task("new_task_name")
    .with_recipe("recipe_name")           # must match the recipe's esorex name
    .with_main_input(raw_new)             # primary raw datasource
    .with_associated_input(prev_task, [PREV_PRODUCT])    # upstream task output
    .with_meta_targets([SCIENCE])         # required for final science outputs
    .build())
```

Optional modifiers — add as needed:
```python
    .with_associated_input(ds, min_ret=0)                        # optional input
    .with_associated_input(ds, condition=fn, match_rules=obj)    # conditional/override
    .with_alternative_associated_inputs(alt_assoc)               # arm/mode-dependent
    .with_dynamic_parameter("name", fn)                          # runtime parameter
    .with_condition(fn)                                          # skip task entirely
    .with_job_processing(fn)                                     # inject recipe params
    .with_report("template", ReportInput.RECIPE_INPUTS_OUTPUTS)  # QC report
    .with_input_filter(PRODUCT_TYPE)                             # whitelist recipe inputs
    .with_grouping_keywords([kwd.tpl_start])                     # group by FITS keyword
```

See [REFERENCE.md](REFERENCE.md) for the full list of builder methods.

### 6. Extract to a subworkflow (if complexity warrants it)

If the task group grows beyond ~5 tasks, or the same group is called multiple times
with different inputs, move it to a separate `_<name>.py` file and decorate with
`@subworkflow`:

```python
# instr_calibrations.py
from edps import subworkflow, task

@subworkflow("my_calibrations", "")
def my_calibrations(bias, raw_input):
    step1 = task("step1").with_recipe(...).with_main_input(raw_input).build()
    step2 = task("step2").with_recipe(...).with_main_input(step1).build()
    return step1, step2
```

Keep `_wkf.py` as a thin wiring file. Move conditions and parameter logic to
`_task_functions.py`, multi-task groups to subworkflow files.

### 7. Verify

```bash
edps -lw                                    # confirm workflow still loads
edps -lt -w uves.uves_wkf                   # list tasks — check your new task appears
edps -lt -w uves.uves_wkf -P 5001          # same, if EDPS runs on a non-default port (default: 5000)
```

## Modifying an existing task

Change `.with_associated_input()`, `.with_match_keywords()`, or `time_range` to adjust
how inputs are selected. Always check:

- `_datasources.py` and `_classification.py` — other tasks may share the same datasource
- `_task_functions.py` — condition functions and job-processing functions may be shared across tasks; changes affect all callers
- Subworkflow files — if the task lives in a `_<name>.py` subworkflow, edits there propagate to every call site in `_wkf.py`

## See also

- [REFERENCE.md](REFERENCE.md) — full API for `task`, `data_source`, `classification_rule`, time ranges, association patterns
- [examples/espresso.md](examples/espresso.md) — subworkflows, `match_rules`, conditional inputs, reports, dual-arm products
- [examples/uves.md](examples/uves.md) — reusable `alternative_associated_inputs`, parameter-driven calibration, `with_function`, `with_cluster`, `@dataclass` subworkflow returns
- [examples/muse.md](examples/muse.md) — `copy_upstream`, `FilterMode.SELECT` on output filters, `with_grouping_function`, custom string meta targets, multiple geometry alternatives per consumer
- [examples/fors.md](examples/fors.md) — multiple `_wkf.py` files per instrument, shared `fors_common.py`, `imported_tasks` list, task factory with conditional builder chaining, report-only task
- [examples/kmos.md](examples/kmos.md) — `FilterMode.REJECT`, `copy_all` grouping task, `$`-prefixed dynamic grouping keywords, fine-grained time constants, low-level recipe loop API
- [examples/eris.md](examples/eris.md) — builder-extension helper functions, `get_parameter()` in condition chains, two `@subworkflow` wrappers for one internal function, metatargets as function argument, `Optional[Task]` dataclass fields
- [examples/giraffe.md](examples/giraffe.md) — `job.setup` mutation for dynamic grouping keys, workflow parameters in condition functions, factory function vs `@subworkflow`, `with_function()` count-based dispatch, Python-level FITS grouping, MJD-epoch recipe parameters
- [examples/hawki.md](examples/hawki.md) — `params.get_workflow_param()`, `job.parameters.workflow_parameters` in job processing, `job.associated_files` mutation, `job.task_name`, `alternative_association()` stored as variable, `.with_alternatives()` alias, product-isolation task pattern, `num_inputs` dynamic parameter
- [examples/gravity.md](examples/gravity.md) — dual reduction paths (raw + pre-computed) from one subworkflow, two tasks sharing a recipe with different datasources, minimal declarative workflow structure
- [examples/visir.md](examples/visir.md) — four `@subworkflow` wrappers on one function, string-driven task naming + product-type renaming, conditional `.with_report()` in factory, multi-key `with_input_map()`, placeholder tasks for unsupported modes, `__all__` for module discovery
- [examples/xshooter.md](examples/xshooter.md) — reports as `dict` parameter to subworkflow, `input_type` as task name prefix and meta target, compound condition functions, `copy_upstream` relay for calibration strategy switching, large `alternative_associated_inputs()` matrices
