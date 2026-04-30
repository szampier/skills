# GRAVITY Workflow Examples

GRAVITY is a VLTI near-IR interferometer.  Its workflow is notably lean — no task
functions, no dynamic parameters, no conditions — but introduces a useful architectural
pattern: **parallel reduction paths for raw and pre-computed data**.

---

## 1. Dual reduction paths from a single subworkflow

For instruments where users often download pre-reduced visibilities from the archive and
re-calibrate them, provide **both** a raw-to-calibrated and a pre-computed-to-calibrated
task inside the same subworkflow.  Both tasks use the same recipe; only the main input
datasource differs:

```python
# gravity_calibrate_visibilities.py
from edps import subworkflow, task, science, calchecker

idp = "idp"

@subworkflow("calibrate_visibilities", "")
def calibrate_visibilities(target_visibilities, calibrator_visibilities):

    # Path A: calibrate from freshly reduced raw data
    calibrate_visibilities = (task('calibrate_visibilities')
                              .with_recipe('gravity_viscal')
                              .with_main_input(target_visibilities)          # upstream task
                              .with_associated_input(calibrator_visibilities, min_ret=1, max_ret=100)
                              .with_associated_input(diameter_catalog, min_ret=0)
                              .with_meta_targets([idp, science, calchecker])
                              .build())

    # Path B: calibrate from pre-processed data (e.g. downloaded from the ESO Science Portal)
    calibrate_pre_computed = (task('calibrate_pre_computed_visibilities')
                              .with_recipe('gravity_viscal')
                              .with_main_input(target_pre_computed_visibilities)  # datasource on disk
                              .with_associated_input(calibrator_pre_computed_visibilities,
                                                     min_ret=1, max_ret=20)
                              .with_associated_input(diameter_catalog, min_ret=0)
                              .with_meta_targets([idp, science])
                              .build())

    return calibrate_visibilities, calibrate_pre_computed
```

The main workflow unpacks both tasks with one call:

```python
# gravity_wkf.py
calibrate_vis, calibrate_pre_computed_vis = calibrate_visibilities(
    target_visibilities, calibrator_visibilities
)
```

> **When to use this pattern**: when the instrument produces archive products
> (e.g., uncalibrated visibilities) that users may want to re-process without
> re-running the full raw reduction.  Adding a second task with a datasource that
> matches archived product types gives EDPS two independent entry points into the
> pipeline — one from raw, one from disk.

---

## 2. Two tasks sharing a recipe with different datasources

When the same recipe processes two distinct input types (here: calibrator star vs
science target, each with their own SKY pairing), define separate tasks rather than
overloading one:

```python
# Calibrator: optional sky (up to 10), delivers QC calibration products
calibrator_visibilities = (task('calibrator_visibilities')
                           .with_recipe('gravity_vis')
                           .with_main_input(raw_cal_object)
                           .with_associated_input(raw_cal_sky, min_ret=0, max_ret=10)
                           .with_associated_input(calib_dark, [DARK], match_rules=match_dark_normal)
                           .with_associated_input(pixel_to_visibility,
                                                  [MASTER_FLAT, MASTER_WAVE, MASTER_P2VM, MASTER_BAD])
                           .with_meta_targets([qc1calib, calchecker, idp])
                           .build())

# Science target: optional sky (at most 1), delivers science products
target_visibilities = (task('target_visibilities')
                       .with_recipe('gravity_vis')
                       .with_main_input(raw_sci_object)
                       .with_associated_input(raw_sci_sky, min_ret=0, max_ret=1)
                       .with_associated_input(calib_dark, [DARK], match_rules=match_dark_normal)
                       .with_associated_input(pixel_to_visibility,
                                              [MASTER_FLAT, MASTER_WAVE, MASTER_P2VM, MASTER_BAD])
                       .with_meta_targets([qc0, calchecker, idp])
                       .build())
```

> Separate tasks appear as distinct nodes in the EDPS provenance graph and can have
> different `max_ret`, `match_rules`, and `meta_targets` — even when the underlying
> recipe is identical.

---

## 3. Minimal workflow structure (no task functions or conditions)

Not every workflow needs dynamic parameters or condition functions.  GRAVITY's
`gravity_wkf.py` is purely declarative — it wires subworkflows together and declares
tasks with static inputs:

```python
# gravity_wkf.py — entire file (simplified)
from .gravity_calibrate_visibilities import calibrate_visibilities
from .gravity_quality_control import quality_control
from .gravity_datasources import *

__title__ = "GRAVITY workflow"
idp = "idp"

calib_dark          = task('dark').with_recipe('gravity_dark').with_main_input(raw_dark)...build()
pixel_to_visibility = task('pixel_to_visibility').with_recipe('gravity_p2vm')...build()

actuators_response, arc, dispersion = quality_control(calib_dark, pixel_to_visibility)

calibrator_visibilities = task('calibrator_visibilities').with_recipe('gravity_vis')...build()
target_visibilities     = task('target_visibilities').with_recipe('gravity_vis')...build()

calibrate_vis, calibrate_pre_computed_vis = calibrate_visibilities(
    target_visibilities, calibrator_visibilities
)
```

> Keep `_wkf.py` as a pure wiring file.  Move all conditionals and parameter logic
> into `_task_functions.py` only when the instrument genuinely needs them.
