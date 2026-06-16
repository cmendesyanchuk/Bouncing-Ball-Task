# Bouncing Ball Task — Bug Diagnosis
**Date:** 2026-06-16
**Scope:** Full `src/bouncing_ball_task/` tree, prompted by a review of `bouncing_ball.py`

---

## Summary

| # | File | Severity | Status |
|---|------|----------|--------|
| 1 | `bouncing_ball.py` | Low (silent, wrong value) | Confirmed |
| 2 | `bouncing_ball.py` | Medium (`AttributeError` on `str()`/`print()`) | Confirmed |
| 3 | `bouncing_ball.py` | Low (wrong exception type/message) | Confirmed |
| 4 | `bouncing_ball.py` | Low (narrow edge case) | Confirmed |
| 5 | `bouncing_ball.py` | Low (dead code path today) | Confirmed |
| 6 | `bouncing_ball.py` | Low (no current output impact) | Confirmed |
| 7 | `human_bouncing_ball/dataset.py` | **High** (breaks default pipeline) | Confirmed |
| 8 | `model_bouncing_ball/dataset.py` | **High** (breaks default pipeline) | Confirmed |
| 9 | `model_bouncing_ball/dataset.py` | **High** (breaks default pipeline) | Confirmed |
| 10 | `human_bouncing_ball/dataset.py` | **High** (corrupts saved data) | Confirmed |
| 11 | `human_bouncing_ball/dataset.py` | Medium (crashes for model-style configs) | Confirmed |
| 12 | `human_bouncing_ball/nonwall.py` | Medium (dead/broken entry point) | Confirmed |
| 13 | `human_bouncing_ball/dataset.py` | Low (dead code) | Confirmed |
| 14 | `human_bouncing_ball/dataset.py` | Cosmetic (logging only) | Confirmed |
| 15 | `utils/pyutils.py` | Medium (breaks retry-with-delay) | Confirmed |
| 16 | `utils/pyutils.py` | Low (unused today) | Confirmed |
| 17 | `utils/gif.py` | Low (masked by coincidence) | Confirmed |
| 18 | various `defaults.py` | Cosmetic | Confirmed |
| 19 | `human_bouncing_ball/dataset.py` | Cosmetic | Confirmed |
| 20 | `model_bouncing_ball/defaults.py` | Low (design ambiguity) | Confirmed |
| 21 | `utils/htaskutils.py` | Low (latent pitfall, currently inert) | Confirmed |
| 22 | `bouncing_ball.py` | **Critical** (guaranteed crash) — shape-feature regression | Confirmed (reproduced) |
| 23 | `bouncing_ball.py` | Low (dormant, unused) — shape-feature regression | Confirmed |
| 24 | `human_bouncing_ball/dataset.py` | High (unreachable only because #22 crashes first) — shape-feature regression | Confirmed |
| 25 | `human_bouncing_ball/dataset.py` | High (downstream of #22) — shape-feature regression | Confirmed (reproduced) |
| 26 | all `human_bouncing_ball/*.py`, `model_bouncing_ball/*.py` | Design gap, not a crash | Confirmed |
| 27 | `tests/test_bouncing_ball.py` | Coverage gap — shape-feature regression went undetected | Confirmed |
| 28 | `tests/test_bouncing_ball.py` | Medium — test is unreliable (≈75% fail rate observed) | Confirmed (reproduced) |

**Key conclusion (revised):** an initial pass concluded the shape-change feature introduced no bugs of its own. That conclusion was **wrong** and is corrected in [Relationship to the shape-change feature](#relationship-to-the-shape-change-feature) below — `model_samples` and its downstream consumers are broken specifically because of the shape addition, and the feature was never wired into the `human_bouncing_ball`/`model_bouncing_ball` dataset-generation pipelines at all. Everything else in the list below (#1–#6, #7–#21) is a pre-existing defect that predates shapes.

---

## 1–6. `src/bouncing_ball_task/bouncing_ball.py`

### 1. `transitioning_change_mode` property reads the wrong backing field
**Location:** [bouncing_ball.py:566-587](src/bouncing_ball_task/bouncing_ball.py#L566-L587)

```python
@property
def transitioning_change_mode(self) -> str:
    return self._transitioning_change_mode

@return_change_mode.setter        # <-- should be @transitioning_change_mode.setter
def transitioning_change_mode(self, mode: Optional[str]):
    ...
    self._transitioning_change_mode = mode
    if mode == "all":
        self.transition_value = np.inf
    elif mode == "half":
        self.transition_value = self.ball_radius
    elif mode == None:
        self.transition_value = 0
```

**Why it's a bug:** `@return_change_mode.setter` takes the *already-defined* `return_change_mode` property and attaches this function as its setter, producing a **new** property object whose getter is still `return_change_mode`'s original getter (`return self._return_change_mode`). That new object then overwrites the class attribute `transitioning_change_mode`, clobbering the correct getter defined three lines above.

**Effect:** Writing `self.transitioning_change_mode = "half"` works correctly (sets `self._transitioning_change_mode` and `self.transition_value`). But *reading* `task.transitioning_change_mode` returns `self._return_change_mode` instead — a value from a completely unrelated setting. The simulation itself isn't affected because the core loop reads `self.transition_value` directly, never the property. But any code or debugging session that inspects `task.transitioning_change_mode` gets a silently wrong answer.

**Provenance:** Verified with `git log -L` — present since the property was first introduced in commit `849fb54` ("Fixing an issue with grayzone color changes", Oct 2024). Pre-dates the shape feature by over a year.

**Fix:** Change the decorator to `@transitioning_change_mode.setter`.

---

### 2. `__str__` references a non-existent attribute
**Location:** [bouncing_ball.py:2318](src/bouncing_ball_task/bouncing_ball.py#L2318)

```python
f"    min_t_color_change={self.min_t_color_change},\n"
```

**Why it's a bug:** There is no `self.min_t_color_change` anywhere in the class. Only the more specific `min_t_color_change_after_random`, `min_t_color_change_after_bounce`, `min_t_color_change_after_shape_change`, and `min_t_bounce_color_change_after_random` exist.

**Effect:** Calling `str(task)` or `print(task)` raises `AttributeError: 'BouncingBallTask' object has no attribute 'min_t_color_change'`.

**Provenance:** Present since the very first commit (`dac2e07`, Sept 2024) — predates the per-cause cooldown split and the shape feature entirely.

**Fix:** Reference one of the actual attributes (or list all of them).

---

### 3. Undefined variable in an error message
**Location:** [bouncing_ball.py:1140](src/bouncing_ball_task/bouncing_ball.py#L1140)

```python
else:
    raise ValueError(f"Invalid str color input, '{valid_color}'")
```

**Why it's a bug:** The parameter is named `valid_colors` (plural); `valid_color` (singular) is never defined in this scope.

**Effect:** Passing an unsupported string for `valid_colors` (anything besides `"default"`/`"constant"`) raises `NameError: name 'valid_color' is not defined` instead of the intended, more informative `ValueError`.

**Fix:** Use `valid_colors` in the f-string.

---

### 4. Likely copy-paste bug in multi-color initial-color merge logic
**Location:** [bouncing_ball.py:1144-1155](src/bouncing_ball_task/bouncing_ball.py#L1144-L1155)

```python
if initial_color is not None and len(initial_color) == 3:
    if np.array(initial_color).ndim > 1:
        for color in initial_color:
            if (
                not (initial_color[0] == valid_colors)   # always checks initial_color[0]
                .all(axis=1)
                .any()
            ):
                valid_colors = [color] + valid_colors
```

**Why it's a bug:** The loop iterates `color in initial_color`, but the membership test inside always checks `initial_color[0]` against `valid_colors` — never the current loop variable `color`. Every iteration is effectively asking "is the *first* initial color already registered?" rather than checking each color individually.

**Effect:** Only triggers when `initial_color` is a 2D array of exactly 3 colors (a narrow condition). When triggered, the logic doesn't behave as intended — it can prepend the wrong colors to `valid_colors` or skip colors that should have been added.

**Fix:** Replace `initial_color[0]` with `color` in the comparison.

---

### 5. `same_xy_velocity` repeat axis assumes 2-D shape
**Location:** [bouncing_ball.py:1128-1129](src/bouncing_ball_task/bouncing_ball.py#L1128-L1129)

```python
if self.same_xy_velocity:
    vel = np.repeat(vel, 2, axis=1)
```

**Why it's a bug:** This assumes `vel` is always 2-D (`batch, 1`). If `sample_velocity` is ever called with `timesteps > 1`, `vel` is 3-D (`timesteps, batch, 1`) and `axis=1` would repeat along the *batch* dimension instead of the velocity dimension.

**Effect:** Currently inert — `sample_velocity` is only ever called with the default `timesteps=1` in this codebase. Also, `same_xy_velocity` is already flagged in-code (`logger.warning(...)`) as "not thoroughly tested." Low priority, but a real latent defect.

---

### 6. `reverse_ball_sequence` doesn't negate velocity
**Location:** [bouncing_ball.py:1925-1948](src/bouncing_ball_task/bouncing_ball.py#L1925-L1948)

**Why it's a bug:** When a trajectory is played in reverse, velocity should flip sign — a ball moving right becomes, in reverse, a ball moving left. The function reverses position, color, and shape correctly but carries `vel` straight through unflipped.

**Effect:** No current visible impact, because velocity is never included in any sample/target transformation that gets returned to callers. It would only matter if velocity were ever surfaced (e.g., via `debug=True` tracking).

**Provenance:** The velocity-handling code in this function predates the shape feature; shape support (`shp`/`shp_ch`) was added later by mirroring the *existing* color-handling pattern, not by touching the velocity lines.

---

## 7–21. The rest of the repository

### 7. `generate_video_dataset` return-arity mismatch breaks effective-hazard-rate estimation
**Location:** [human_bouncing_ball/dataset.py:700](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L700)

```python
task, output_samples, output_targets, df_data, dict_metadata = generate_video_dataset(
    dataset_parameters, task_parameters, dict_trial_type_generation_funcs,
    shuffle=True, validate=True, defaults=defaults, _adjust_labels=False,
)
```

**Why it's a bug:** `generate_video_dataset` returns **6** values everywhere else it's used or defined —
`task, output_samples, output_model_samples, output_targets, df_data, dict_metadata`
(confirmed at the `return` on [line 197](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L197) and inside `adjust_dataset_labels` at [line 819](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L819)). This call site unpacks only **5** names, dropping `output_model_samples`.

**Effect:** `ValueError: too many values to unpack (expected 5)`. This call lives inside `estimate_effective_hazard_rates`, which `generate_video_dataset` calls **automatically by default** (`_adjust_labels=True`, `estimate_mult=100` are the defaults). So the *default* code path of the main human-dataset generation pipeline is broken.

**Likely cause:** A refactor added `output_model_samples` to `generate_video_dataset`'s return signature and updated the `__main__` call site ([line 1210](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L1210), which correctly unpacks 6 values) but missed this internal call.

---

### 8. Same bug, second call site
**Location:** [model_bouncing_ball/dataset.py:39](src/bouncing_ball_task/model_bouncing_ball/dataset.py#L39)

```python
task, output_samples, output_targets, df_data, dict_metadata = hds.generate_video_dataset(...)
```

Identical issue to #7 — 5-value unpacking of a 6-value return. Same `ValueError`.

---

### 9. `save_video_dataset` call is missing the `output_model_samples` argument
**Location:** [model_bouncing_ball/dataset.py:105-112](src/bouncing_ball_task/model_bouncing_ball/dataset.py#L105-L112)

```python
path_videos = hds.save_video_dataset(
    dir_base, name_dataset, df_data, dict_metadata,
    samples, targets, task,        # <-- 3 positional args
    duration=args.duration, ...
)
```

`save_video_dataset`'s signature is `(dir_base, name_dataset, df_data, dict_metadata, output_samples, output_model_samples, output_targets, task, ...)` — **4** required positional args after `dict_metadata`, but only 3 are supplied.

**Effect:** `TypeError: save_video_dataset() missing 1 required positional argument: 'task'` (the values that *are* passed shift into the wrong slots: `targets` lands in `output_model_samples`, `task` lands in `output_targets`, and nothing is left for the real `task` parameter).

**Pattern:** Bugs #7, #8, and #9 are all symptoms of the same root cause — the `output_model_samples` plumbing was added to `human_bouncing_ball/dataset.py` but never propagated to `model_bouncing_ball/dataset.py`. **The model-dataset generation pipeline cannot currently run end-to-end.**

---

### 10. `save_video_dataset` writes model-samples over the real samples file
**Location:** [human_bouncing_ball/dataset.py:932-936](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L932-L936)

```python
df_target.to_csv(str(path_df_target))
df_sample.to_csv(str(path_df_sample))
df_model_sample.to_csv(str(path_df_sample))   # should be path_df_model_sample
df_color_change.to_csv(str(path_df_color_change))
```

`path_df_model_sample` is defined right above (`dir_video / f"video_{idx_block_video}_model_samples.csv"`) but never used.

**Effect:** Each video's `*_samples.csv` file gets immediately overwritten with model-sample data after the correct sample data was written one line earlier. No `*_model_samples.csv` file is ever produced. Saved datasets silently lose the real per-video sample data.

---

### 11. `estimate_effective_hazard_rates` crashes when `total_dataset_length` is `None`
**Location:** [human_bouncing_ball/dataset.py:698](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L698)

```python
dataset_parameters["total_dataset_length"] *= estimate_mult
```

**Why it's a bug:** Model-style dataset configs size themselves via `total_videos` and leave `total_dataset_length=None` (see `ModelDatasetParameters`). `None *= 100` raises `TypeError: unsupported operand type(s) for *=: 'NoneType' and 'int'`.

**Effect:** Compounds bug #7/#8 — even if the unpacking were fixed, this function would still fail for any dataset configured by video count rather than by time.

---

### 12. `nonwall.py`'s `main_test` cannot run
**Location:** [human_bouncing_ball/nonwall.py:191-307](src/bouncing_ball_task/human_bouncing_ball/nonwall.py#L191-L307)

Three independent problems, each fatal on its own:

a) **Missing required argument.** Calls `htaskutils.compute_dataset_size(...)` without the required `trial_types` positional argument → `TypeError: compute_dataset_size() missing 1 required positional argument: 'trial_types'`.

b) **Undefined names.** Further down, it passes `num_pos_x_linspace_bounce`, `idx_linspace_bounce`, `bounce_timestep`, and `repeat_factor` to `generate_initial_dict_metadata` — none of these are parameters of `main_test`, nor are they defined locally → `NameError`.

c) **Extra/misplaced positional argument.** It also inserts a `num_pos_bounce` value that has no corresponding parameter in `generate_initial_dict_metadata`'s signature. If (a) and (b) were fixed, this would silently shift every subsequent positional argument (`border_tolerance_outer`, `border_tolerance_inner`, `bounce_timestep`, `repeat_factor`, `seed`, ...) one slot to the right.

**Effect:** `main_test()` — and the `if __name__ == "__main__":` block that calls it — is entirely non-functional. This looks like an abandoned/stale development scaffold: the real entry point in `human_bouncing_ball/dataset.py` calls the same helper function correctly, with matching argument order.

---

### 13. `dataset.py`'s `plot_effective_stats` is dead and broken
**Location:** [human_bouncing_ball/dataset.py:1041-1172](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L1041-L1172)

- References `visualization.get_color_palette`, but `visualization` is never imported (only commented out: `# from hmdcpd import visualization`).
- `plt.suptitle(f"... {batch_size} Videos")` references `batch_size`, which is never defined or passed into this function.

**Effect:** `NameError` if ever called. Not called anywhere in `src/`, so currently dead code. A working duplicate of this function exists in `utils/visualize.py`.

---

### 14. Non-f-strings used where interpolation was clearly intended
**Location:** [human_bouncing_ball/dataset.py:903](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L903) and [907](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L907)

```python
display_path = "/dir_dataset/videos/{dir_block.stem}/{dir_video.stem}"
```

Missing the `f` prefix — the literal text `{dir_block.stem}` gets logged instead of the actual path. Cosmetic (logging only), but clearly not the intended behavior.

---

### 15. `retry()` decorator references an unimported `time` module
**Location:** [utils/pyutils.py:180](src/bouncing_ball_task/utils/pyutils.py#L180)

```python
time.sleep(ndelay)
```

`time` is never imported anywhere in `pyutils.py` (only `os, re, ast, inspect, shutil, tempfile, random, dataclasses, numpy, loguru`).

**Effect:** Any function wrapped with `@retry(...)` that actually triggers a delay after catching an exception (`ndelay` truthy) raises `NameError: name 'time' is not defined` instead of backing off and retrying — the decorator fails exactly when it's supposed to do its job.

---

### 16. `repeat_sequence_imbalanced` indexes by raw value instead of position
**Location:** [utils/pyutils.py:561-589](src/bouncing_ball_task/utils/pyutils.py#L561-L589)

```python
value_counter = np.zeros(num_values)
...
value = int(unique_values[i % num_values])
output.append(splits[value][int(value_counter[value] % len(splits[value]))])
value_counter[value] += 1
```

**Why it's a bug:** `value_counter` is sized `num_values` and is meant to be indexed by *position* among the unique values (0, 1, 2, ...), but it's indexed by the raw `value` itself. If `balance`'s unique values aren't exactly `0..num_values-1` (e.g., `[0, 2, 5]`), `value_counter[5]` raises `IndexError: index 5 is out of bounds for axis 0 with size 3`.

**Effect:** Currently unused anywhere in `src/` (verified via `grep`), so this is a latent bug rather than an active one — but it would break the moment anyone calls this function with non-contiguous balance values.

---

### 17. Wrong PIL keyword argument for the timestamp overlay color
**Location:** [utils/gif.py:170](src/bouncing_ball_task/utils/gif.py#L170)

```python
ImageDraw.Draw(img).text(..., color="white", font=font)
```

**Why it's a bug:** `PIL.ImageDraw.ImageDraw.text()` has no `color` parameter — the correct keyword is `fill`. Verified empirically:

```python
>>> d.text((5,5), 'A', color='red')   # silently ignored — text renders in default ink, NOT red
>>> d.text((5,5), 'A', fill='red')    # actually red
```

**Effect:** Currently masked by coincidence — in the installed Pillow version, `text()` accepts arbitrary `**kwargs` and silently drops unrecognized ones, and the *default* ink this Pillow version falls back to happens to render close to white anyway, so the on-screen result looks correct by accident. It would misbehave the moment someone changes the intended color, or on older Pillow versions where `text()` doesn't accept `**kwargs` and would raise `TypeError` outright. Present since the very first commit — unrelated to any later feature work.

---

### 18. Dataclass field type annotations don't match their defaults
**Locations:** e.g. [defaults.py:22](src/bouncing_ball_task/defaults.py#L22) (`mask_color: list[...] = (127, 127, 127)` — the default is actually a tuple), [human_bouncing_ball/defaults.py:25-29](src/bouncing_ball_task/human_bouncing_ball/defaults.py#L25-L29) (fields annotated `int`/`float` whose real default, inherited from `TaskParameters.size_frame` etc., is a tuple).

Cosmetic only — Python dataclasses don't enforce annotations at runtime, so this doesn't cause failures, just misleading type hints.

---

### 19. Duplicate import
**Location:** [human_bouncing_ball/dataset.py:1,3](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L1-L3) — `import copy` appears twice. Harmless no-op.

---

### 20. `NongrayDatasetParameters.timestep_change` is fixed at class-definition time
**Location:** [model_bouncing_ball/defaults.py:32-35](src/bouncing_ball_task/model_bouncing_ball/defaults.py#L32-L35)

```python
@dataclass
class NongrayDatasetParameters(ModelDatasetParameters):
    ncc_nvc_timesteps: int = 20
    timestep_change: int = ncc_nvc_timesteps // 2   # evaluated once, at class-body execution
    timestep_from_wall: int = 5
```

This works syntactically (class-body statements execute top-to-bottom, so `ncc_nvc_timesteps` is already `20` in the namespace when `timestep_change` is computed), but if a user later constructs `NongrayDatasetParameters(ncc_nvc_timesteps=40)`, `timestep_change` stays hard-coded at `10` instead of tracking the override. Possibly intentional, but worth confirming with whoever relies on this relationship.

---

### 21. Shared mutable default lists in `group_trial_data`
**Location:** [utils/htaskutils.py:699-706](src/bouncing_ball_task/utils/htaskutils.py#L699-L706)

```python
if bounce_index_x is None:
    bounce_index_x = [[],] * num_trials
```

This is the classic Python pitfall: `[[],] * num_trials` creates `num_trials` references to the *same* empty list object, not independent lists. Currently inert — nothing downstream mutates an individual trial's slot in place — but it's a landmine for any future code that does.

---

## Relationship to the shape-change feature

**Correction:** an earlier pass through this diagnosis concluded the shape-change feature introduced no bugs of its own, and that all defects predated it. That conclusion was wrong for bugs #1–#6 and #17 (those genuinely do predate shapes, see provenance below) — but a deeper, shape-specific pass found a critical regression that the first pass missed (#22–#25 below), plus confirmation that the feature was simply never wired into the dataset-generation pipelines (#26).

### What does predate shapes (confirmed unrelated)

- The shape feature touches exactly four files, confirmed via `git show --stat` on the introducing commits `bcb0aaa` ("included shapes square and diamond in the gif utility") and `ea756b9` ("core logic with shapes"): `bouncing_ball.py`, `constants.py`, `defaults.py`, and `utils/gif.py`.
- None of the dataset-pipeline files where bugs #7–#14 live (`human_bouncing_ball/dataset.py`, `model_bouncing_ball/dataset.py`, `nonwall.py`) were touched by either commit.
- Bugs #1–#3 in `bouncing_ball.py` were traced with `git log -L` to their introducing commits: the `transitioning_change_mode` property bug to `849fb54` (Oct 2024), and the `__str__`/`min_t_color_change` and `valid_color` typo bugs to the very first commit `dac2e07` (Sept 2024) — both well over a year before shapes were added.
- The `utils/gif.py` color-keyword bug (#17) was also traced to `dac2e07`, untouched by the shape-drawing additions (`draw_ball`/`draw_frame`) in `bcb0aaa`.
- The `pccosc`/`pccovasc` probability logic, the `rand_for_shape_color` cooldown gating, `reverse_ball_sequence`'s `shp`/`shp_ch` handling, and `gif.draw_ball`'s square/diamond rendering were all re-read directly and correctly mirror the pre-existing, working color-change machinery. No defect found there.
- Commit `bcb0aaa` also removed a special case in `draw_circle` (`color=[255,255,255] if color == mask_color else list(color)` → `list(color)`) — an intentional, correct fix bundled into the same commit (a fully-masked ball was being drawn white/visible instead of blending into the grayzone), not a regression.

### What is genuinely broken by the shape feature (the part the first pass missed)

#### 22. `model_samples` is broken by the new `shape` column — guaranteed crash
**Location:** [bouncing_ball.py:926](src/bouncing_ball_task/bouncing_ball.py#L926)

```python
samples[mask_locations_samples, 2:] = self.targets[mask_locations_targets, 2:5]
```

Before shapes, sample rows were `[x, y, r, g, b]` and target rows were `[x, y, r, g, b, ...changes]`, so `samples[..., 2:]` (3 cols) matched `targets[..., 2:5]` (3 cols). The shape commits appended a `shape` column to **both** structures (`[x, y, r, g, b, shape]`), growing `samples[..., 2:]` to 4 columns, but left this target-side slice hardcoded at `2:5` (still 3 columns). Reproduced directly:

```text
samples shape: (4, 50, 6)
targets shape: (4, 50, 11)
model_samples FAILED: ValueError('shape mismatch: value array of shape (0,3) could not be broadcast to indexing result of shape (0,4)')
```

This fails on the column-count mismatch alone — independent of how many rows the grayzone mask selects, so it is **not** an edge case; every call to `task.model_samples` raises. `git log -L` shows `model_samples` was added in `8c54d437` (Jul 2025) — well before shapes — confirming this is an integration gap left behind by the shape commits, not a pre-existing bug. `model_samples` is called 4 times in `human_bouncing_ball/dataset.py` (`shorten_trials_and_update_meta`, twice in `adjust_dataset_labels`), so **this, not bug #7, is the actual first failure point of the dataset-generation pipeline** — bug #7 is never reached in practice.

**Fix:** change the slice to `self.targets[mask_locations_targets, 2:6]` (or compute the color-block width from `self.num_colors`/array structure rather than hardcoding it).

#### 23. `target_to_sample` has the identical hardcoded-slice problem
**Location:** [bouncing_ball.py:1955](src/bouncing_ball_task/bouncing_ball.py#L1955)

```python
colors = target[:, 2:]
```

Assumes target rows are exactly `position + color`. With shape (and the change columns) appended, this slice now sweeps in `shape` and every change column as if they were color channels. Currently unused anywhere in `src/`, so dormant rather than actively crashing — but broken the moment anything calls it.

#### 24. `adjust_dataset_labels` has the same mismatched-width assertion
**Location:** [human_bouncing_ball/dataset.py:783-788](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L783-L788)

```python
assert (
    (
        targets[mask_model_samples_nongray, 2:5] ==
        model_samples[mask_model_samples_nongray, 2:]
    ).all()
)
```

Same `2:5` (3 cols) vs `2:` (now 4 cols) mismatch as bug #22 — would itself raise `ValueError` on the broadcast. Currently unreachable only because `model_samples` (#22) already crashes earlier in the same call path.

#### 25. CSV-export column names in `save_video_dataset` were never updated for the new column
**Location:** [human_bouncing_ball/dataset.py:874-875](src/bouncing_ball_task/human_bouncing_ball/dataset.py#L874-L875)

```python
sample_columns = ["x", "y", "r", "g", "b"]
target_columns = sample_columns + ["vc_bounce", "vc_random", "cc_bounce", "cc_random"]
```

5 names for what's now a 6-column sample array (missing `shape`), and 9 names for what's now an 11-column target array under the default `return_change_mode="source"` (missing `shape` plus a 5th change column — `source` mode now emits `vcb, vcr, ccb, ccr, scr`, not 4). Reproduced directly:

```text
sample_columns mismatch -> ValueError('Shape of passed values is (5, 6), indices imply (5, 5)')
target_columns mismatch -> ValueError('Shape of passed values is (5, 11), indices imply (5, 9)')
```

This would fire once bug #22 is fixed and the pipeline gets far enough to reach `save_video_dataset`.

#### 26. Shape is simply not wired into any trial-generation pipeline
**Location:** all of `human_bouncing_ball/{catch,straight,bounce,nonwall}.py` and `model_bouncing_ball/{cc,ncc}_*.py`

Confirmed via `grep` across the whole `src/` tree: only `bouncing_ball.py`, `constants.py`, `defaults.py`, and `TEST1.py` reference real shape-feature names (`valid_shapes`, `initial_shape`, `forced_shape_changes`, `probability_shape_change`, `shape_index`, `DEFAULT_SHAPES`, etc.) — every match in the `human_bouncing_ball`/`model_bouncing_ball` files is an incidental `.shape` (NumPy array attribute), not the feature.

Unlike `initial_position`, `initial_velocity`, `initial_color`, and `forced_velocity_bounce_x/y`/`forced_color_changes` — all of which every trial generator computes explicitly per trial — no trial generator ever sets `initial_shape`, `valid_shapes`, or `forced_shape_changes`. Shape only varies via the random default (`probability_shape_change=0.001`, inherited from the base `TaskParameters` dataclass), is never tracked in any metadata/stats helper (`print_type_stats`, `print_block_stats`, `compute_trial_color_and_stats` have no shape equivalent of the color-split logic), and never appears in a `default_idx_to_color_dict`-style mapping. This isn't a crash — it's a design gap: even once bugs #22–#25 are fixed, shape remains an uncontrolled, untracked side effect in every generated dataset rather than a real, designed-for trial feature.

### Revised conclusion

The shape feature's *simulation* logic in `bouncing_ball.py` (the probability/cooldown machinery, `reverse_ball_sequence`, `gif.draw_ball`) was implemented carefully and mirrors the existing color-change patterns correctly. But the feature's *integration* was incomplete: it grew the sample/target array width without updating every consumer of that fixed-width assumption (`model_samples`, `target_to_sample`, the `adjust_dataset_labels` assertion, and the CSV column names in `save_video_dataset`), and it was never connected to any of the `human_bouncing_ball`/`model_bouncing_ball` trial-generation pipelines at all. So, to directly answer "is shape accounted for in all the Python files in `src`": **no** — it's accounted for only in `bouncing_ball.py`'s simulation core and `utils/gif.py`'s rendering, and even within `bouncing_ball.py` itself one property (`model_samples`) and one dormant classmethod (`target_to_sample`) were missed.

---

## Test suite coverage of the shape-change feature

`tests/` has exactly two files: `test_bouncing_ball.py` (964 lines) and `test_utils_pyutils.py` (33 lines, tests only `create_sequence_splits` — irrelevant to shape).

### 27. The shape-change feature is almost entirely untested

Grepping `test_bouncing_ball.py` for every real shape-feature name — `valid_shapes`, `initial_shape`, `forced_shape_changes`, `probability_shape_change`, `num_shapes`, `shape_index`, `DEFAULT_SHAPES`, `min_t_shape_change_after_random`, `warmup_t_no_rand_shape_change`, `probability_color_change_on_shape_change`, `probability_color_change_on_velocity_and_shape_change`, `min_t_color_change_after_shape_change`, `model_samples`, `target_to_sample` — returns **zero matches**. Every other `shape` hit in the file is the unrelated NumPy `.shape` attribute.

The *only* place the ball-shape concept is touched at all is two lines, inside `test_sample_target_has_correct_min_max_values`:
```python
shape_vals = array[:, :, 5]
assert np.all(shape_vals >= 0) and np.all(shape_vals <= 2)
```
A single passive bounds check — it confirms the shape-index column never goes outside `[0, 2]`, but says nothing about whether shape actually changes at the right rate, respects its cooldowns, or interacts correctly with color.

By contrast, every other change mechanism gets a dedicated, statistically-rigorous test: `test_pvc_causes_correct_random_velocity_changes`, `test_pccovc_causes_correct_color_changes`, `test_pccnvc_causes_correct_color_changes`, `test_min_t_color_change_causes_predictable_change_statistics`, `test_color_change_bounce_delay_causes_correct_color_changes`. **None of these has a shape equivalent.** Nothing exercises `forced_shape_changes`, `valid_shapes`/`initial_shape`, the shape-change cooldowns, pccosc/pccovasc firing rates, or `model_samples` — which is exactly why bug #22 (`model_samples`'s broken slice) was never caught by the suite.

Relatedly, `test_correct_sequences_for_each_sequence_modes` parametrizes generically over `sequence_mode` including `"reverse"`, but the dedicated correctness test for that mode is a commented-out stub: `# def test_reverse_sequence_mode_correctly_reverses_sequences()` ([test_bouncing_ball.py:369](tests/test_bouncing_ball.py#L369)). So `reverse_ball_sequence`'s shape-handling (`shp`/`shp_ch`) is only ever checked structurally (shapes/dtypes line up), never for actual correctness.

### 28. `test_min_t_color_change_causes_predictable_change_statistics` passes a dead parameter name — and is measurably broken

**Location:** [tests/test_bouncing_ball.py:785-913](tests/test_bouncing_ball.py#L785-L913)

```python
@pytest.mark.parametrize("min_t_color_change", [0, 5, 10, 20])
def test_min_t_color_change_causes_predictable_change_statistics(
    pvc, pccovc, pccnvc, min_t_color_change,
):
    ...
    task = BouncingBallTask(
        ...
        min_t_color_change=min_t_color_change,
        ...
    )
```

Not shape-related, but found while auditing this file. `BouncingBallTask.__init__` has no `min_t_color_change` parameter (only `min_t_color_change_after_random`/`_after_bounce`/`_after_shape_change` exist) — it's the same dead-parameter-name pattern already noted for `__str__` (#2) and `htaskutils.plot_params`. The constructor's trailing `**kwargs` silently absorbs it, so the task is always built with the *default* cooldowns (5) regardless of the parametrized value, while the test's own statistical predictions (`total_deadzone_timesteps`, `corrected_pccnvc`, etc.) are computed *as if* the parametrized value had been applied.

Reproduced by running the test directly:
```text
$ pytest tests/test_bouncing_ball.py -k test_min_t_color_change_causes_predictable_change_statistics -v
...
36 failed, 12 passed
```
Most parametrizations fail outright (only combinations close to the real default of 5 pass by coincidence), confirming this isn't just a theoretical concern — the test is actively unreliable today.
