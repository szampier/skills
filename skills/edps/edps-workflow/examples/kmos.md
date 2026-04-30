# KMOS workflow examples

Examples drawn from the KMOS instrument pipeline (`kmos_wkf.py`, `kmos_datasources.py`, `kmos_illumination.py`, `kmos_combination.py`, `kmos_molecfit.py`).

KMOS is a multi-object near-infrared IFU spectrograph. Its workflow is a good reference for: `FilterMode.REJECT`, `with_function(copy_all)` for grouping tasks, `$`-prefixed dynamic grouping keywords, fine-grained time constants, `alternative_association()` returned from subworkflows, and the low-level recipe invocation loop API.

---

## `FilterMode.REJECT` — blacklist input filter

By default, `with_input_filter()` is a **whitelist**: only the listed types are forwarded to the recipe. Pass `mode=FilterMode.REJECT` to **blacklist** instead — all types are forwarded *except* the listed ones:

```python
from edps import FilterMode

telluric_correction = (task('telluric_correction')
    .with_recipe('kmos_molecfit_correct')
    .with_main_input(science)
    .with_condition(use_molecfit)
    .with_associated_input(transmission_from_standard, condition=molecfit_on_standard)
    .with_associated_input(transmission_from_science,  condition=molecfit_on_science)
    .with_input_filter(SCI_RECONSTRUCTED_COLL, CALCTRANS_ATMOS_PARM,
                       mode=FilterMode.REJECT)   # pass everything EXCEPT these two types
    .build())
```

Use `FilterMode.REJECT` when the recipe accepts most product types but a few need to be suppressed.

---

## `with_function(copy_all)` — grouping task without running a recipe

Use `copy_all` (a built-in EDPS function) when you need to *collect and group* inputs from multiple upstream tasks without running an actual recipe. The outputs are the combined inputs, re-grouped:

```python
from edps import copy_all

grouped_science = (task("group_science")
    .with_function(copy_all)                   # no recipe — just groups the inputs
    .with_main_input(science)
    .with_associated_input(telluric_correction,
                           condition=use_molecfit,
                           max_ret=10000,
                           match_rules=same_dataset)
    .with_grouping_keywords(['$combine_exposures',
                             kwd.grat1_name, kwd.grat2_name, kwd.grat3_name])
    .with_job_processing(change_output_filter)
    .build())
```

The outputs of `group_science` become the main input of the combination recipe downstream.

---

## `$`-prefixed dynamic grouping keywords

Grouping keywords prefixed with `$` are resolved at runtime from `parameters.yaml`, not from FITS headers. Use them to let users control grouping strategy without editing the workflow:

```python
# groups science exposures; '$process_exposures' resolved from parameters.yaml at runtime
raw_science = (data_source()
    .with_classification_rule(science_class)
    .with_setup_keywords(setup)
    .with_match_keywords([kwd.grat1_id])
    .with_grouping_keywords(['$process_exposures'])
    .build())

# multiple keywords — static FITS keywords + dynamic parameter
grouped_science = (task("group_science")
    .with_function(copy_all)
    .with_grouping_keywords(['$combine_exposures',
                             kwd.grat1_name, kwd.grat2_name, kwd.grat3_name])
    ...
    .build())
```

Define the values in `parameters.yaml`:
```yaml
process_exposures: "tpl_start"   # or "ob_id", "arcfile", etc.
combine_exposures: "ob_id"
```

---

## Fine-grained built-in time constants

KMOS uses sub-day time constants available directly from `edps`:

```python
from edps import ONE_AND_HALF_HOURS, NEXT_DAY, SAME_NIGHT

raw_arc_on = (data_source()
    .with_classification_rule(arc_on_class)
    .with_setup_keywords(setup)
    .with_match_keywords(setup, time_range=ONE_AND_HALF_HOURS, level=-1)  # very strict
    .with_match_keywords(setup, time_range=NEXT_DAY,           level=0)
    .with_match_keywords(setup, time_range=TWO_DAYS,           level=1)
    .with_match_keywords(setup, time_range=TWO_WEEKS,          level=2)
    .with_match_keywords(setup, time_range=UNLIMITED,          level=3)
    .build())
```

For bespoke windows use `RelativeTimeRange(days_before, days_after)` with fractional days:

```python
PAST_4_HRS = RelativeTimeRange(-0.16, 0)
NEXT_4_HRS = RelativeTimeRange(0, 0.16)

flat_for_arcs = (match_rules()
    .with_match_keywords(setup, time_range=PAST_4_HRS, level=0)
    .with_match_keywords(setup, time_range=NEXT_4_HRS, level=0)   # same level — either direction
    .with_match_keywords(setup, time_range=TWO_DAYS,   level=1)
    .with_match_keywords(setup, time_range=UNLIMITED,  level=3))
```

---

## `alternative_association()` returned from a subworkflow

An `alternative_association()` object can be created inside a subworkflow and returned for use by the caller:

```python
# kmos_illumination.py
from edps import subworkflow, task, alternative_association

@subworkflow("illumination", "")
def select_illumination(dark, lamp_flat, wavelength):
    sky_flat    = (task('sky_flat')    ...).build()
    illumination = (task('illumination') ...).build()

    # return an alternative that caller attaches to science / standard tasks
    illumination_correction = (alternative_association()
        .with_associated_input(sky_flat,    [ILLUM_CORR], min_ret=0, max_ret=1,
                               condition=use_sky_flats)
        .with_associated_input(illumination,[ILLUM_CORR], min_ret=0, max_ret=1,
                               condition=use_lamp_flats))

    return illumination_correction, sky_flat, illumination
```

```python
# kmos_wkf.py
illumination_correction, sky_flat, illumination = select_illumination(dark, lamp_flat, wavelength)

standard = (task('standard')
    ...
    .with_alternative_associated_inputs(illumination_correction)   # attach here
    .build())

science = (task('object')
    ...
    .with_alternatives(illumination_correction)                    # alias form
    .build())
```

Both `.with_alternative_associated_inputs()` and `.with_alternatives()` are accepted.

---

## Low-level recipe invocation: looping a recipe file-by-file

When `with_function()` needs to run a recipe on each input file individually (rather than passing the whole group at once), use the low-level invocation API:

```python
from edps import (
    task, File, FitsFile,
    ProductRenamer, RecipeInvocationArguments, RecipeInvocationResult,
    InvokerProvider, RecipeInputs
)

def loop_recipe(recipe, args, invoker_provider, renamer, categories="all"):
    invoker = invoker_provider.recipe_invoker

    if categories == "all":
        loop_files   = [File(f.name, f.category, "") for f in args.inputs.combined]
        calibrations = None
    else:
        loop_files   = [File(f.name, f.category, "") for f in args.inputs.combined
                        if f.category in categories]
        calibrations = [File(f.name, f.category, "") for f in args.inputs.combined
                        if f.category not in categories]

    results, ret_codes = [], []
    for input_file in loop_files:
        inputs    = RecipeInputs(main_upstream_inputs=[input_file],
                                 associated_upstream_inputs=calibrations)
        arguments = RecipeInvocationArguments(inputs=inputs, parameters={},
                                              job_dir=args.job_dir,
                                              input_map={},
                                              logging_prefix=args.logging_prefix)
        result = invoker.invoke(recipe, arguments, renamer, create_subdir=True)
        ret_codes.append(result.return_code)
        results.append(result)

    output_files = [FitsFile(name=f[0].name, category=f[0].category)
                    for f in [r.output_files for r in results]]
    return RecipeInvocationResult(return_code=min(ret_codes), output_files=output_files)
```

Use this inside a `with_function()` task when the recipe cannot handle a batch SOF and must be run separately per exposure.
