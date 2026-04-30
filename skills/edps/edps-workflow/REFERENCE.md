# edps-workflow Reference

## `task()` builder

```python
from edps import task, SCIENCE
t = (task("task_name")
    .with_recipe("recipe_name")
    .with_main_input(datasource)
    .with_associated_input(task_or_ds, [PRODUCT_TYPE])
    .build())
```

| Method | Description |
|---|---|
| `.with_recipe(name)` | esorex recipe name (must match exactly) |
| `.with_function(fn, recipes=[…])` | dynamic recipe selection; `fn` receives job context and returns recipe name; list all possible recipes in `recipes=` |
| `.with_main_input(ds)` | primary raw datasource |
| `.with_associated_input(src, [tags], condition=fn, match_rules=obj, min_ret=N, max_ret=N)` | add calibration input; `src` can be a task or datasource |
| `.with_alternative_associated_inputs(alt)` | arm/mode-dependent input via `alternative_associated_inputs()` or `alternative_association()` |
| `.with_input_filter(*PRODUCT_TYPES)` | whitelist product types forwarded to the recipe |
| `.with_input_map({SRC_TYPE: DST_TYPE})` | rename a product type before passing to the recipe |
| `.with_output_filter(*PRODUCT_TYPES)` | keep only these product types from the recipe output |
| `.with_meta_targets([SCIENCE])` | mark as a science-producing task (required for final outputs) |
| `.with_dynamic_parameter(name, fn)` | call `fn` at runtime to compute a recipe parameter |
| `.with_condition(fn)` | skip the task entirely if `fn` returns `False` for the current input |
| `.with_report(template, ReportInput.X, driver='png')` | attach a QC report template |
| `.with_job_processing(fn)` | inject recipe parameters via a job-editing function (can be called multiple times) |
| `.with_cluster(keyword, max_diameter=, max_separation=)` | group observations spatially; values reference `parameters.yaml` keys |
| `.with_min_group_size(N)` | minimum inputs per group before the task runs |
| `.with_grouping_keywords([…])` | group inputs by these FITS keywords |

`min_ret=0` makes an input optional (default is 1). `max_ret=N` caps the number of associated files.

## `alternative_associated_inputs()` builder

The named version of `alternative_association()`. Define as a variable and reuse across multiple tasks. Supports `match_rules=` per branch.

```python
from edps import alternative_associated_inputs

# Define once, reuse everywhere
bias_calibrations = (alternative_associated_inputs()
    .with_associated_input(bias, [MASTER_BIAS_BLUE],            condition=is_blue)
    .with_associated_input(bias, [MASTER_BIAS_REDL, MASTER_BIAS_REDU], condition=is_red))

# Override match rules for science (different validity range)
bias_calibrations_for_science = (alternative_associated_inputs()
    .with_associated_input(bias, [MASTER_BIAS_BLUE], condition=is_blue,
                           match_rules=science_bias_rules)
    .with_associated_input(bias, [MASTER_BIAS_REDL, MASTER_BIAS_REDU], condition=is_red,
                           match_rules=science_bias_rules))
```

Attach with `.with_alternative_associated_inputs(bias_calibrations)` on a task.

`alternative_association()` (from ESPRESSO examples) is the inline anonymous form; `alternative_associated_inputs()` is the named reusable form. Both work with `.with_alternative_associated_inputs()`.



Override the association time ranges for a specific `.with_associated_input()` without changing the datasource definition. Pass the result as `match_rules=` on a task's associated input.

```python
from edps import match_rules, RelativeTimeRange
from edps.generator.time_range import ONE_DAY, TWO_WEEKS, UNLIMITED

next_day = (match_rules()
    .with_match_keywords(instrument_setup, time_range=RelativeTimeRange(0, 1), level=0)
    .with_match_keywords(instrument_setup, time_range=ONE_DAY, level=1)
    .with_match_keywords(instrument_setup, time_range=TWO_WEEKS, level=2)
    .with_match_keywords(instrument_setup, time_range=UNLIMITED, level=3))

task(...).with_associated_input(wave_ds, [WAVE_MATRIX], match_rules=next_day, condition=is_ut)
```

## `ReportInput` constants

```python
from edps import ReportInput
```

| Constant | Data passed to the QC report template |
|---|---|
| `ReportInput.RECIPE_INPUTS` | Input files only |
| `ReportInput.RECIPE_INPUTS_OUTPUTS` | Input + output files |
| `ReportInput.RECIPE_OUTPUTS` | Output files only |



```python
from edps import data_source, RelativeTimeRange
from edps.generator.time_range import UNLIMITED, ONE_DAY, ONE_WEEK, SAME_NIGHT

ds = (data_source("LABEL")
    .with_classification_rule(rule)
    .with_min_group_size(N)
    .with_setup_keywords([kwd.instrume, kwd.arm, kwd.ins_slit])
    .with_grouping_keywords([kwd.tpl_start, kwd.arm])
    .with_match_keywords([kwd.arm], time_range=ONE_DAY, level=0)
    .with_match_function(fn, time_range=UNLIMITED)
    .build())
```

| Method | Description |
|---|---|
| `.with_classification_rule(rule)` | which files belong to this datasource; multiple calls allowed |
| `.with_min_group_size(N)` | minimum files per group (e.g. 3 for biases) |
| `.with_setup_keywords([…])` | FITS keywords that define the observing setup |
| `.with_grouping_keywords([…])` | how to batch files together |
| `.with_match_keywords([…], time_range, level)` | association rule at a given quality level |
| `.with_match_function(fn, time_range)` | custom association function |

**Level convention:**
| Level | Meaning |
|---|---|
| < 0 | Stricter than calibration plan |
| 0 | Follows the calibration plan |
| 1 | QC1-certifiable quality |
| 2 | Probably still acceptable |
| 3 | Significant quality risk |

## `classification_rule()`

```python
from edps import classification_rule

# Raw file classification (uses DPR keywords)
raw_rule = classification_rule('LABEL', {
    kwd.instrume: "INSTR",
    kwd.dpr_catg: "CALIB",   # or "SCIENCE"
    kwd.dpr_type: "BIAS",
})

# Pipeline product classification (uses PRO keywords)
prod_rule = classification_rule('PRODUCT', {
    kwd.instrume: "INSTR",
    kwd.pro_catg: "MASTERBIAS",
})
```

## Time ranges

| Constant | Meaning |
|---|---|
| `ONE_DAY` | ±1 day from observation |
| `ONE_WEEK` | ±7 days |
| `TWO_WEEKS` | ±14 days |
| `ONE_MONTH` | ±30 days |
| `QUARTERLY` | ±90 days |
| `SAME_NIGHT` | same night only |
| `IN_THE_PAST` | any file before the observation (use for static calibrations) |
| `UNLIMITED` | no time constraint |
| `RelativeTimeRange(-N, N)` | custom ±N days |

## Workflow file naming convention

| File | Convention | Example |
|---|---|---|
| Main workflow | `<instr>_wkf.py` | `muse_wkf.py` |
| Datasources | `<instr>_datasources.py` | `muse_datasources.py` |
| Classification | `<instr>_classification.py` | `muse_classification.py` |
| Rules | `<instr>_rules.py` | `muse_rules.py` |
| Keywords | `<instr>_keywords.py` | `muse_keywords.py` |
| Task functions | `<instr>_task_functions.py` | `muse_task_functions.py` |
| Parameters | `<instr>_parameters.yaml` | `muse_parameters.yaml` |
| Subworkflow | `<instr>_<name>.py` | `muse_calib.py` |

## `application.properties` configuration

Located at `~/.edps/application.properties`. Key fields:

```properties
workflow_dir = /home/user/edps_workflows
esorex = /path/to/esorex
```

The `workflow_dir` must contain subdirectories named after the instrument (e.g. `muse/`, `xshoo/`).

## Common keywords (`_keywords.py` pattern)

```python
instrume = "INSTRUME"
dpr_catg = "DPR.CATG"
dpr_type = "DPR.TYPE"
pro_catg = "PRO.CATG"
arm      = "INS.ARM"
ins_slit = "INS.SLIT"
tpl_start = "TPL.START"
arcfile  = "ARCFILE"
```

Defining these as variables (rather than inline strings) makes rules easier to read and reduces typos.

## Subworkflows

A subworkflow is a Python function that creates and returns one or more tasks, allowing reuse:

```python
# in instr_subworkflow.py
from edps import task

def bias_arm(raw_bias_ds, arm_suffix):
    return (task(f"bias_{arm_suffix}")
        .with_recipe("run_bias")
        .with_main_input(raw_bias_ds)
        .build())
```

```python
# in instr_wkf.py
from .instr_subworkflow import bias_arm
bias_vis_task = bias_arm(raw_bias_vis, "vis")
bias_nir_task = bias_arm(raw_bias_nir, "nir")
```
