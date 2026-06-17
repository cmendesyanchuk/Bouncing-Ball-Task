# Wire the shape-change feature into trial-generation pipelines (Bug #26)

**Date:** 2026-06-16 (revised)

## What changed from the original plan

The original plan handled only `psc`, `initial_shape`, and `forced_shape_changes`. This revision adds:
- `pccosc` and `pccovasc` as per-trial linspace-cycled parameters (follow `pccovc` exactly — computed in `compute_trial_shape_stats` and included in the `group_trial_data` tuple)
- `valid_shapes`, `min_t_color_change_after_shape_change`, `min_t_shape_change_after_random`, and `warmup_t_no_rand_shape_change` as **overrides-only parameters** — they are NOT dataset-level fields, exactly as `min_t_color_change_after_random`, `valid_colors`, and `warmup_t_no_rand_velocity_change` are not
- `warmup_t_no_rand_shape_change` must be added to `catch.py`'s and model `ncc_nvc.py`'s overrides dicts (not just infrastructure-only)
- The `group_trial_data` tuple grows from 10 fields to 15 (adds 5: `initial_shape`, `psc`, `shape_change_index`, `pccosc`, `pccovasc`)

---

## Confirmed baseline facts (read from source)

| Parameter in `BouncingBallTask` | Default value | Analogous existing param |
|---|---|---|
| `probability_shape_change` | 0.001 | `probability_velocity_change` (pvc) |
| `probability_color_change_on_shape_change` | 0.0 | `probability_color_change_on_velocity_change` (pccovc) |
| `probability_color_change_on_velocity_and_shape_change` | 1.0 | `probability_color_change_on_velocity_change` (pccovc) |
| `min_t_color_change_after_shape_change` | 5 | `min_t_color_change_after_random` (overrides-only) |
| `min_t_shape_change_after_random` | 15 | `min_t_color_change_after_random` (overrides-only) |
| `warmup_t_no_rand_shape_change` | 3 | `warmup_t_no_rand_velocity_change` (overrides-only) |
| `valid_shapes` | None (all shapes) | `valid_colors` (overrides-only) |
| `forced_shape_changes` | None | `forced_color_changes` |
| `initial_shape` | None (random) | `initial_color` |

Target array column layout (confirmed from `save_video_dataset`):

```
[x=0, y=1, r=2, g=3, b=4, shape=5, vc_bounce=6, vc_random=7, cc_bounce=8, cc_random=9, sc_random=10]
```

---

## Design decisions

1. **`psc` is a scalar per-trial-type** (like `pvc`, not like `pccovc`). It is added to `HumanDatasetParameters`, stored in `dict_metadata`, broadcast per-trial in `group_trial_data`, and threaded to `BouncingBallTask` as `probability_shape_change`. Default: `0.001` (matches `BouncingBallTask`'s own default — no behavior change for existing datasets).

2. **`pccosc` and `pccovasc` are per-trial linspace-cycled values** (like `pccovc`). They require `HumanDatasetParameters` linspace triplets. Backward-compatible defaults: `pccosc_lower = pccosc_upper = 0.0, num_pccosc = 1` and `pccovasc_lower = pccovasc_upper = 1.0, num_pccovasc = 1` — these collapse to single-value linspaces that exactly match `BouncingBallTask`'s defaults, so no behavior change for existing datasets.

3. **`valid_shapes`, `min_t_color_change_after_shape_change`, `min_t_shape_change_after_random`, `warmup_t_no_rand_shape_change` are NOT dataset parameters.** They follow the overrides-only pattern: `BouncingBallTask`'s own defaults apply unless a specific trial type requires a deviation via the `overrides` dict. `catch.py` and model `ncc_nvc.py` must add `warmup_t_no_rand_shape_change` to their existing overrides dicts.

4. **`initial_shape` is always computed per trial** via `pyutils.repeat_sequence(np.arange(len(DEFAULT_SHAPES)), num_trials, shuffle=True, roll=False, shift=1)` — fixed cycling order, exactly mirroring color's call signature.

5. **`pccosc` and `pccovasc` are computed in `compute_trial_shape_stats`**, not in `compute_trial_color_and_stats`. This keeps the color function's return signature unchanged (important since all 9 generators call it), and consolidates all shape-triggered parameters in one place.

6. **`group_trial_data` tuple grows to 15 fields** (14 + optional meta). New fields are appended at the end before meta to avoid reordering existing fields:

   ```
   position, velocity, color, pccnvc, pccovc, pvc,
   bounce_index_x, bounce_index_y, color_change_index,
   initial_shape, psc, shape_change_index, pccosc, pccovasc,
   [meta]
   ```

   `shorten_trials_and_update_meta`'s `*_` absorber currently captures fields 6–8; with the new tuple it would absorb 6–13, still landing `meta_trial` as the last element — but since we need to extract `initial_shape`, `pccosc`, `pccovasc` from that middle range, the unpack must be made fully explicit.

---

## Implementation

### A. `src/bouncing_ball_task/human_bouncing_ball/defaults.py`

Add to `HumanDatasetParameters` next to `pvc` / `pccovc*`:

```python
psc: float = 0.001

pccosc_lower: float = 0.0
pccosc_upper: float = 0.0
num_pccosc: int = 1

pccovasc_lower: float = 1.0
pccovasc_upper: float = 1.0
num_pccovasc: int = 1
```

`ModelDatasetParameters` and `NongrayDatasetParameters` inherit these automatically.

---

### B. `src/bouncing_ball_task/utils/htaskutils.py`

**`generate_initial_dict_metadata`:**
- Add `psc`, `pccosc_lower`, `pccosc_upper`, `num_pccosc`, `pccovasc_lower`, `pccovasc_upper`, `num_pccovasc` as explicit parameters (next to `pvc` and the existing `pccovc*` parameters; currently they would fall through `**kwargs`).
- Store them in `dict_metadata`:
  - `dict_metadata["psc"] = psc`
  - `dict_metadata["pccosc_linspace"] = np.linspace(pccosc_lower, pccosc_upper, num_pccosc, endpoint=True)`
  - `dict_metadata["pccovasc_linspace"] = np.linspace(pccovasc_lower, pccovasc_upper, num_pccovasc, endpoint=True)`

**New function `compute_trial_shape_stats(num_trials, dict_meta, dict_meta_type)`** — placed right after `compute_trial_color_and_stats`:

```python
def compute_trial_shape_stats(num_trials, dict_meta, dict_meta_type):
    # Fixed cycling order for initial shape, mirroring color
    initial_shape = pyutils.repeat_sequence(
        np.arange(len(DEFAULT_SHAPES)),
        num_trials,
        shuffle=True,
        roll=False,
        shift=1,
    ).astype(int).tolist()
    dict_meta_type["initial_shape_counts"] = np.unique(initial_shape, return_counts=True)

    # Per-trial pccosc, cycling through linspace like pccovc
    pccosc = pyutils.repeat_sequence(
        dict_meta["pccosc_linspace"], num_trials, shuffle=False, roll=True,
    ).tolist()
    dict_meta_type["pccosc_counts"] = np.unique(pccosc, return_counts=True)

    # Per-trial pccovasc, cycling through linspace like pccovc
    pccovasc = pyutils.repeat_sequence(
        dict_meta["pccovasc_linspace"], num_trials, shuffle=False,
    ).tolist()
    dict_meta_type["pccovasc_counts"] = np.unique(pccovasc, return_counts=True)

    return initial_shape, pccosc, pccovasc, dict_meta_type
```

Extend the existing import at the top: `from bouncing_ball_task.constants import DEFAULT_COLORS, DEFAULT_SHAPES`.

**`group_trial_data`:** Add 5 new keyword parameters with `None` defaults after `color_change_index`, before `dict_meta_trials`:

```python
initial_shape=None,
psc=None,
shape_change_index=None,
pccosc=None,
pccovasc=None,
```

Handle defaults and broadcasting:
- `shape_change_index`: `[[],] * num_trials` if None (same as `bounce_index_x`)
- `psc`: broadcast with `if not isiterable(psc): psc = [psc,] * num_trials`; default scalar `0.001`
- `pccosc`: broadcast with same guard; default scalar `0.0`
- `pccovasc`: broadcast with same guard; default scalar `1.0`
- `initial_shape`: in practice always provided by `compute_trial_shape_stats`

Append all 5 to `list_to_zip` in order: `initial_shape, psc, shape_change_index, pccosc, pccovasc`.

**`print_type_stats`:** Update unpack on line 47:

```python
# Before
position, velocity, color, pccnvc, pccovc, pvc, fxvc, fyvc, fcc, meta = zip(*trials)

# After
position, velocity, color, pccnvc, pccovc, pvc, fxvc, fyvc, fcc, initial_shapes, pscs, fsc, pccosc_vals, pccovasc_vals, meta = zip(*trials)
```

Add to `trial_type_stats`:
```python
("Shape Splits:", Counter(initial_shapes)),
("pccosc Splits:", Counter(pccosc_vals)),
("pccovasc Splits:", Counter(pccovasc_vals)),
```

**`print_block_stats`:** Add to `block_stats`:
```python
("Shape Counts:", Counter(df_block["Final Shape"].values)),
("PSC Counts:", Counter(df_block["PSC"])),
("pccosc Counts:", Counter(df_block["PCCOSC"])),
("pccovasc Counts:", Counter(df_block["PCCOVASC"])),
```

---

### C. `src/bouncing_ball_task/human_bouncing_ball/dataset.py`

**`generate_video_parameters`:** Add the 7 new parameters to the signature, defaulting from `defaults.*`, and pass them all into `htaskutils.generate_initial_dict_metadata`.

**`generate_video_dataset`:** Update the per-trial-type unpack from 10 to 15 fields:

```python
# Before
positions, velocities, colors, pccnvcs, pccovcs, pvcs, fxvc, fyvc, fcc, meta_trials = (
    list(param) for param in zip(*params)
)

# After
positions, velocities, colors, pccnvcs, pccovcs, pvcs, fxvc, fyvc, fcc, shapes, pscs, fsc, pccoscs, pccovAscs, meta_trials = (
    list(param) for param in zip(*params)
)
```

Add to `task_parameters_type`:
```python
task_parameters_type["initial_shape"] = shapes
task_parameters_type["probability_shape_change"] = pscs
task_parameters_type["forced_shape_changes"] = fsc
task_parameters_type["probability_color_change_on_shape_change"] = pccoscs
task_parameters_type["probability_color_change_on_velocity_and_shape_change"] = pccovAscs
```

**`shorten_trials_and_update_meta`:** Replace the current `*_` absorbing unpack with a fully explicit one:

```python
# Before
position, velocity, _, pccnvc, pccovc, pvc, *_, meta_trial = param

# After
position, velocity, _, pccnvc, pccovc, pvc, _, _, _, initial_shape, psc_val, _, pccosc_val, pccovasc_val, meta_trial = param
```

Add to `meta_trial.update(...)`:
```python
"Final Shape": DEFAULT_SHAPES[int(target[-1, 5])],
"PSC": psc_val,
"PCCOSC": pccosc_val,
"PCCOVASC": pccovasc_val,
```

Fix existing latent bug in the same edit — change `target[-1, 2:]` to `target[-1, 2:5]` in the `"Final Color"` entry. Without this fix, `np.argmax` runs over `[r, g, b, shape, vc_bounce, ...]` instead of just `[r, g, b]`.

Add `DEFAULT_SHAPES` to the imports from `bouncing_ball_task.constants`.

**`adjust_dataset_labels`:** Fix `samples[range(batch_size), last_idx, 2:]` → `samples[range(batch_size), last_idx, 2:5]` for `last_visible_color` (same latent bug, same root cause).

---

### D. All 9 trial generators

In each of `catch.py`, `straight.py`, `bounce.py`, `nonwall.py` (human) and `ncc_nvc.py`, `cc_nvc.py`, `ncc_vc.py`, `cc_vc.py`, `ncc_rvc.py`, `cc_rvc.py` (model) — same 3-part change:

**Step 1.** Right after the existing `compute_trial_color_and_stats(...)` call, add:

```python
initial_shape, pccosc, pccovasc, dict_meta_type = htaskutils.compute_trial_shape_stats(
    num_trials, dict_meta, dict_meta_type,
)
```

**Step 2.** In the `group_trial_data(...)` call, add keyword arguments:

```python
initial_shape=initial_shape,
psc=dict_meta["psc"],
pccosc=pccosc,
pccovasc=pccovasc,
# shape_change_index omitted → defaults to [[],]*num_trials
```

**Step 3. `catch.py` and model `ncc_nvc.py` only** — extend their existing overrides dict to include `warmup_t_no_rand_shape_change`:

```python
dict_meta_type["overrides"] = {
    "warmup_t_no_rand_velocity_change": catch_ncc_nvc_timesteps,
    "warmup_t_no_rand_color_change": catch_ncc_nvc_timesteps,
    "warmup_t_no_rand_shape_change": catch_ncc_nvc_timesteps,  # NEW
}
```

---

### E. Overrides-only parameters — no dataset-level wiring needed

These are already supported by `BouncingBallTask` and reachable via any trial type's `overrides` dict. They are NOT added to `HumanDatasetParameters` or `generate_initial_dict_metadata`.

| Parameter | BouncingBallTask default | When to set via overrides |
|---|---|---|
| `valid_shapes` | None (all shapes) | If a trial type must restrict available shapes |
| `min_t_color_change_after_shape_change` | 5 | If a trial type needs a non-default cooldown |
| `min_t_shape_change_after_random` | 15 | If a trial type needs a non-default shape cooldown |
| `warmup_t_no_rand_shape_change` | 3 | Handled in step D3 above for catch/ncc_nvc |

---

### F. Constants — no changes needed

`DEFAULT_SHAPES` already exists in `constants.py`. It is used directly for index-to-name mapping in `shorten_trials_and_update_meta` and for arange indexing in `compute_trial_shape_stats`.

---

## Verification

1. Run `pytest tests/test_bouncing_ball.py tests/test_utils_pyutils.py` — confirm no regressions (htaskutils/dataset.py changes are inert to existing test fixtures).

2. Generate a small end-to-end human dataset and confirm:
   - `task_parameters_type` for each trial type has `initial_shape`, `probability_shape_change`, `probability_color_change_on_shape_change`, `probability_color_change_on_velocity_and_shape_change`, and `forced_shape_changes` populated.
   - `df_data` has `"Final Shape"`, `"Final Color"`, `"PSC"`, `"PCCOSC"`, `"PCCOVASC"` columns with sane values.
   - `print_type_stats` output includes `Shape Splits:`, `pccosc Splits:`, `pccovasc Splits:`.
   - `print_block_stats` output includes `Shape Counts:`, `PSC Counts:`, `pccosc Counts:`, `pccovasc Counts:`.

3. Repeat for one model generator (`ncc_nvc`). Confirm `warmup_t_no_rand_shape_change` is set to `ncc_nvc_timesteps` in the `task_parameters` for that trial type.

4. Confirm `"Final Color"` is correct (not contaminated by shape column) by cross-checking against known initial color values across a short run — this validates the `2:5` slice fix.

5. Confirm `compute_trial_shape_stats` cycling: assert that no shape index is over- or under-represented by more than 1 across `num_trials` (same guarantee `repeat_sequence` gives color).
