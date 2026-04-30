# MUSE workflow examples

Examples drawn from the MUSE instrument pipeline (`muse_wkf.py`, `muse_science.py`, `muse_sky.py`, `muse_monitoring_and_long_term_cals.py`).

MUSE is a wide-field integral-field spectrograph. Its workflow is a good reference for: `copy_upstream`, `FilterMode.SELECT` on output filters, `with_grouping_function()` for dynamic spatial clustering, custom string meta targets, multiple `alternative_associated_inputs()` variants derived from the same upstream task for different consumers, and readable sub-day time range notation.

---

## `with_function(copy_upstream)` — selective upstream forwarding

`copy_upstream` passes a filtered subset of an upstream task's outputs to the next task, without running a recipe. Use it when you need to extract specific product types from a task's output for a downstream association:

```python
from edps import copy_upstream, FilterMode

# Extract only autocal_factors from preprocess_science outputs
# and make them available for the science task
autocalibration_from_object_sky = (task("autocalibration_from_object_sky")
    .with_condition(use_autocalibration_sky)
    .with_function(copy_upstream)
    .with_main_input(preprocess_science)
    .with_associated_input(science_sky, match_rules=associate_sky_autocalibrations)
    .with_output_filter(autocal_factors_class, mode=FilterMode.SELECT)  # keep only this type
    .build())

# science task then uses autocalibration_from_object_sky as an associated input
science = (task("object")
    .with_recipe("muse_scipost")
    ...
    .with_associated_input(autocalibration_from_object_sky, min_ret=0,
                           condition=use_autocalibration_sky,
                           match_rules=same_dataset)
    ...
    .build())
```

**`copy_upstream` vs `copy_all`**: `copy_upstream` preserves the upstream task's associations (calibrations travel with the data); `copy_all` merges all inputs into a flat group.

---

## `FilterMode.SELECT` on `with_output_filter()`

`with_output_filter()` supports both select (whitelist) and reject (blacklist) modes:

```python
from edps import FilterMode

# Keep ONLY autocal_factors from output (SELECT = whitelist)
.with_output_filter(autocal_factors_class, mode=FilterMode.SELECT)

# Keep everything EXCEPT SKY_MASK and STD_TELLURIC (REJECT = blacklist)
.with_output_filter(SKY_MASK, STD_TELLURIC, mode=FilterMode.REJECT)
```

The same `FilterMode` applies to both `with_input_filter()` and `with_output_filter()`.

---

## `with_grouping_function()` — dynamic spatial clustering

When fixed keyword-based grouping is not enough (e.g. grouping by sky proximity), use `.with_grouping_function()` to supply a custom Python function that decides which exposures belong together:

```python
from .muse_task_functions import variable_clustering_by_sky_position

science = (task("object")
    .with_recipe("muse_scipost")
    .with_main_input(preprocess_science)
    ...
    .with_grouping_keywords(['$combine_science', kwd.ins_mode])  # coarse grouping first
    .with_grouping_function(variable_clustering_by_sky_position) # then refine spatially
    .with_min_group_size(2)
    ...
    .build())
```

The grouping function receives a list of candidate files and returns groups. It is applied *after* keyword grouping.

---

## Custom string meta targets

Standard meta targets (`SCIENCE`, `QC1_CALIB`, etc.) are constants imported from `edps`. You can define your own string targets to mark tasks for pipeline-specific downstream steps:

```python
# Define at module level — just a plain string constant
IDP_DEEP = "idp_deep"

# Use alongside standard targets
science = (task("object")
    .with_recipe("muse_scipost")
    ...
    .with_meta_targets([QC0, SCIENCE, CALCHECKER, IDP, IDP_DEEP])
    .build())
```

Custom targets can be queried by external tools that consume EDPS products.

---

## Multiple `alternative_associated_inputs()` variants from one task

When the same upstream task needs to be associated with *different* time-range rules depending on which downstream task consumes it, define multiple named alternatives — one per consumer:

```python
# In muse_monitoring_and_long_term_cals.py
# geometry is one task, but three consumers need different match rules:

# 1. General use: static table fallback
geometry_calibrations = (alternative_associated_inputs()
    .with_associated_input(geometry, [GEOMETRY_TABLE], condition=recompute_geometry)
    .with_associated_input(geometry_reference))   # fallback to static

# 2. Throughput task: field-mode–only match
match_geometry_for_throughput = (match_rules()
    .with_match_keywords([kwd.ins_opti2_name], time_range=IN_THE_PAST, level=0)
    .with_match_keywords([kwd.ins_opti2_name], time_range=UNLIMITED,   level=1))

geometry_calibrations_for_throughput = (alternative_associated_inputs()
    .with_associated_input(geometry, [GEOMETRY_TABLE], condition=recompute_geometry,
                           match_rules=match_geometry_for_throughput, min_ret=0)
    .with_associated_input(geometry_reference, match_rules=match_geometry_for_throughput, min_ret=0))

# 3. Astrometry task: geometry taken *after* the observation (future 2-day window)
match_geometry_for_astrometry_future = (match_rules()
    .with_match_function(rules.is_associated, time_range=RelativeTimeRange(0, 2), level=0))
match_geometry_for_astrometry_static = (match_rules()
    .with_match_function(rules.is_associated, time_range=IN_THE_PAST, level=1))

geometry_calibrations_for_astrometry = (alternative_associated_inputs()
    .with_associated_input(geometry, [GEOMETRY_TABLE],
                           match_rules=match_geometry_for_astrometry_future)
    .with_associated_input(geometry_reference,
                           match_rules=match_geometry_for_astrometry_static))
```

```python
# Each consumer picks its own variant:
sky_flat.with_alternative_associated_inputs(geometry_calibrations)
throughput.with_alternative_associated_inputs(geometry_calibrations_for_throughput)
astrometry.with_alternative_associated_inputs(geometry_calibrations_for_astrometry)
```

---

## Readable sub-day time ranges using division

Define sub-day windows with division for readability — avoids opaque decimal numbers:

```python
# Clear and self-documenting
TWO_HOURS  = RelativeTimeRange(-2.0 / 24.0,  2.0 / 24.0)
FOUR_HOURS = RelativeTimeRange(-4.0 / 24.0,  4.0 / 24.0)

# Use like any other time range constant
raw_flat_illum = (data_source()
    .with_classification_rule(flat_illum_class)
    .with_match_keywords(field_and_readout_setup + [kwd.prog_id, kwd.obs_id],
                         time_range=TWO_HOURS,  level=-1)   # same OB, ±2 hrs
    .with_match_keywords(field_and_readout_setup,
                         time_range=TWO_HOURS,  level=0)
    .with_match_keywords(field_and_readout_setup,
                         time_range=FOUR_HOURS, level=1)
    .build())
```

---

## Highly decomposed subworkflow structure

MUSE's `muse_wkf.py` is extremely lean — every logical concern lives in its own subworkflow module:

```python
# muse_wkf.py — the entire science chain is one line per subworkflow
lsf_calibrations, lsf_from_lsf, lsf_from_arcs = lsf(bias, dark, lamp_flat, wavelength)
linearity_and_gain, geometry_calibrations, geometry_calibrations_for_astrometry, throughput = monitoring(bias, dark, lamp_flat, wavelength)
response = process_standard(bias, dark, lamp_flat, wavelength, geometry_calibrations, sky_flat)
astrometry, astrometry_calibrations = process_astrometry(response, bias, dark, lamp_flat, wavelength, geometry_calibrations_for_astrometry, sky_flat)
sky, science_sky, sky_combination = process_sky(response, lsf_calibrations, astrometry_calibrations, bias, dark, lamp_flat, wavelength, geometry_calibrations, sky_flat)
science, science_combination = process_science(sky, response, lsf_calibrations, astrometry_calibrations, bias, dark, lamp_flat, wavelength, geometry_calibrations, sky_flat, science_sky)
```

Each subworkflow receives its upstream task dependencies as arguments, making the data-flow graph explicit at the top level.
