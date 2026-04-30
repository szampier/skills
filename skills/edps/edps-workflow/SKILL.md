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
edps -lw                     # lists all registered workflows
edps run <instrument>.<wkf>  # smoke-test with a small dataset
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

### 4. Add the task in `_wkf.py`

```python
from edps import task
from .instr_datasources import raw_new

# A task that depends on a previous task's output as calibration:
new_task = (task("new_task_name")
    .with_recipe("recipe_name")           # must match the recipe's name in esorex
    .with_main_input(raw_new)             # the primary raw data source
    .with_associated_input(prev_task, [PREV_PRODUCT])   # output of upstream task
    .with_meta_targets([SCIENCE])         # only for final science tasks
    .build())
```

Optional modifiers — add as needed:
```python
    .with_associated_input(ds, min_ret=0)             # optional input
    .with_associated_input(ds, condition=fn, match_rules=fn)  # conditional
    .with_alternative_associated_inputs(alt_assoc)    # arm-dependent associations
    .with_dynamic_parameter("param_name", fn)         # runtime parameter
    .with_input_filter(PRODUCT_TYPE)                  # filter what goes to recipe
```

### 5. Verify

```bash
edps -lw                    # confirm workflow still loads
edps run <instr>.<wkf> -d <data_dir>   # check task appears and associations resolve
```

## Modifying an existing task

Change `.with_associated_input()`, `.with_match_keywords()`, or `time_range` to adjust how inputs are selected. Always check `_datasources.py` and `_classification.py` for side effects — other tasks may share the same datasource.

## See also

- [REFERENCE.md](REFERENCE.md) — full API for `task`, `data_source`, `classification_rule`, time ranges, association patterns
- [EXAMPLES.md](EXAMPLES.md) — annotated ESPRESSO workflow patterns: subworkflows, `match_rules`, conditional inputs, reports, dual-arm products
