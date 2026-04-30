# X-Shooter Workflow Examples

X-Shooter is a three-arm (UVB/VIS/NIR) echelle spectrograph. It is the most complex
workflow in the ESO pipeline suite, with three observing modes (stare/offset/nodding),
three input types (science/telluric/flux standard), and a configurable flat and telluric
correction strategy.

---

## 1. Reports passed as a `dict` parameter to a subworkflow

When the same subworkflow is called multiple times and each call needs different
QC reports, pass reports as a `{template: ReportInput}` dict and iterate inside:

```python
# xshooter_science.py — subworkflow accepts reports dict
@subworkflow("science_slit", "")
def xshoo_science_slit(..., reports: dict):
    task_builder = (task(input_type + "_slit_stare")
                    .with_recipe("xsh_scired_slit_stare")
                    ...)

    # Attach only the reports the caller requested
    for report_name, report_inputs in reports.items():
        task_builder = task_builder.with_report(report_name, report_inputs)

    return task_builder.build(), ...

# xshooter_wkf.py — different report sets per call
science_slit_stare, ... = xshoo_science_slit(...,
    reports={"xshooter_science": ReportInput.RECIPE_INPUTS_OUTPUTS})

telluric_slit_stare, ... = xshoo_science_slit(...,
    reports={"xshooter_rawdisp":     ReportInput.RECIPE_INPUTS,
             "xshooter_telluric_std": ReportInput.RECIPE_INPUTS_OUTPUTS,
             "xshooter_science":      ReportInput.RECIPE_INPUTS_OUTPUTS})

flux_standard_slit_stare, ... = xshoo_science_slit(...,
    reports={})   # no reports for flux standards processed as science
```

---

## 2. `input_type` as both task name prefix and custom meta target

A single `input_type` string serves double duty: it becomes part of the task name
**and** is appended directly as a custom string meta target:

```python
@subworkflow("science_slit", "")
def xshoo_science_slit(..., input_type: str, additional_metatargets: list, reports: dict):
    task_builder = (task(input_type + "_slit_stare")    # → "science_slit_stare"
                    .with_recipe("xsh_scired_slit_stare")
                    ...
                    .with_meta_targets([input_type] + additional_metatargets))
                    # → ["science", IDP, QC0, CALCHECKER]
```

Call sites choose their task names, meta targets, and reports in one place:

```python
# Science observations
xshoo_science_slit(...,
    input_type="science",
    additional_metatargets=[IDP, QC0, CALCHECKER],
    reports={"xshooter_science": ReportInput.RECIPE_INPUTS_OUTPUTS})

# Telluric standards treated as science
xshoo_science_slit(...,
    input_type="telluric_standard",
    additional_metatargets=[IDP, CALCHECKER, QC1_CALIB],
    reports={"xshooter_telluric_std": ReportInput.RECIPE_INPUTS_OUTPUTS, ...})

# Flux standards treated as science
xshoo_science_slit(...,
    input_type="flux_standard",
    additional_metatargets=[CALCHECKER],
    reports={})
```

> The string `"science"` / `"telluric_standard"` / `"flux_standard"` passed as
> `input_type` becomes a valid custom meta target — EDPS accepts any string, not just
> the built-in constants.  Using it as both the task name prefix and a meta target
> ensures consistent naming throughout the graph.

---

## 3. Compound condition functions (combining two independent predicates)

For workflows with two orthogonal axes (arm × flat-strategy), compose conditions by
`and`-ing two predicates rather than writing one large function:

```python
from edps import JobParameters

# Base predicates — each tests one axis
def use_flat_science(params: JobParameters) -> bool:
    return params.get_workflow_param("use_flat") == "science"

def is_UVB_slit(params: JobParameters) -> bool:
    return get_parameter(params, "arm") == "UVB" and get_parameter(params, "ifu") == "FALSE"

# Compound predicates — combine any two base predicates
def use_flat_science_uvb_slit(params: JobParameters) -> bool:
    return use_flat_science(params) and is_UVB_slit(params)

def use_flat_standard_nir_slit(params: JobParameters) -> bool:
    return use_flat_standard(params) and is_NIR_slit(params)
```

These compound conditions are then used as `condition=` on branches of
`alternative_associated_inputs()` to route each arm/mode combination to the
correct flat source.

---

## 4. `copy_upstream` relay task for calibration strategy selection

When a workflow parameter (e.g. `use_flat`) determines which of two upstream tasks
should feed a downstream task, use a `copy_upstream` relay task to "select and
forward" the chosen calibrations.  Two relay tasks run under mutually exclusive
conditions — only one fires:

```python
from edps import copy_upstream

@subworkflow("flat_strategy", "")
def flat_strategy(lamp_flat, flat_calibrations, raw_science, raw_standard):
    # Relay A: fires when use_flat == "science"
    flats_science = (task('flats_science')
                     .with_condition(use_flat_science)         # only one runs
                     .with_main_input(raw_science)
                     .with_function(copy_upstream)             # no recipe — just copies inputs
                     .with_job_processing(fix_setup_keywords)
                     .with_alternative_associated_inputs(flat_calibrations)
                     .build())

    # Relay B: fires when use_flat == "standard"
    flats_standard = (task('flats_standard')
                      .with_condition(use_flat_standard)       # mutually exclusive
                      .with_main_input(raw_standard)
                      .with_function(copy_upstream)
                      .with_job_processing(fix_setup_keywords)
                      .with_alternative_associated_inputs(flat_calibrations)
                      .build())

    # Large alternatives block references both relay outputs
    alternatives_flats_for_standard = (alternative_associated_inputs()
        .with_associated_input(flats_science,  [...], condition=use_flat_science_uvb_slit, ...)
        .with_associated_input(flats_science,  [...], condition=use_flat_science_vis_slit, ...)
        .with_associated_input(flats_standard, [...], condition=use_flat_standard_uvb_slit, ...)
        .with_associated_input(flats_standard, [...], condition=use_flat_standard_vis_slit, ...)
        # ... up to 12 branches (arm × mode × flat-strategy) ...
        .with_associated_input(lamp_flat, [...], condition=is_UVB_slit, ...))  # fallback

    return alternatives_flats_for_standard
```

> `copy_upstream` is like `copy_all` but propagates only the files from the
> associated inputs (not all upstream outputs).  Combine it with `with_condition()`
> and `with_alternative_associated_inputs()` to implement workflow-parameter-driven
> calibration strategy switching.

---

## 5. `alternative_associated_inputs()` with many branches (arm × mode matrix)

For multi-arm instruments, `alternative_associated_inputs()` can have many branches —
one per `(arm, mode)` combination.  Each branch selects the correct product types and
match rules:

```python
flexure_calibrations_for_science = (alternative_associated_inputs()
    # UVB slit — associate flex by OB container
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_uvb, order_tab_afc_slit_uvb, disp_tab_afc_uvb],
                           condition=is_UVB_slit, match_rules=associate_flex_using_container, min_ret=0)
    # VIS slit
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_vis, order_tab_afc_slit_vis, disp_tab_afc_vis],
                           condition=is_VIS_slit, match_rules=associate_flex_using_container, min_ret=0)
    # NIR slit
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_nir, order_tab_afc_slit_nir, disp_tab_afc_nir],
                           condition=is_NIR_slit, match_rules=associate_flex_using_container, min_ret=0)
    # IFU modes — same arms, different product types
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_uvb, order_tab_afc_ifu_uvb, disp_tab_afc_uvb],
                           condition=is_UVB_ifu, match_rules=associate_flex_using_container, min_ret=0)
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_vis, order_tab_afc_ifu_vis, disp_tab_afc_vis],
                           condition=is_VIS_ifu, match_rules=associate_flex_using_container, min_ret=0)
    .with_associated_input(flex, [xsh_mod_cfg_opt_afc_nir, order_tab_afc_ifu_nir, disp_tab_afc_nir],
                           condition=is_NIR_ifu, match_rules=associate_flex_using_container, min_ret=0))
```

> When the number of branches grows large, name each `alternative_associated_inputs()`
> variable by its consumer context (e.g., `flexure_calibrations_for_science`,
> `flexure_calibrations_for_standard`, `flexure_calibrations_for_standard_nod`) so
> the difference in match rules is clear at the call site.
