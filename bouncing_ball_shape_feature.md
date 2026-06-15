# Bouncing Ball Task — Shape Feature Addition

## Overview

Added **shape** as a third dynamic variable to the `BouncingBallTask`, alongside the existing color and velocity variables.

- **Shapes**: circle (0), square (1), diamond (2)
- **Change trigger**: random only (never on bounces)
- **Change order**: fixed cycle — circle → square → diamond → circle → ...

---

## Files Modified

### `src/bouncing_ball_task/constants.py`

Added `DEFAULT_SHAPES` constant:

```python
DEFAULT_SHAPES = (
    "circle",   # shape index 0
    "square",   # shape index 1
    "diamond",  # shape index 2
)
```

---

### `src/bouncing_ball_task/defaults.py`

Added three new fields to the `TaskParameters` dataclass:

```python
probability_shape_change: float = 0.001
min_t_shape_change_after_random: int = 15
warmup_t_no_rand_shape_change: int = 3
```

---

### `src/bouncing_ball_task/utils/gif.py`

Added `draw_ball()` function that dispatches drawing based on shape index:

```python
def draw_ball(position, color, background, ball_radius, mask_color, shape=0, thickness=-1):
    # shape=0 → circle (cv2.circle)
    # shape=1 → square (cv2.rectangle)
    # shape=2 → diamond (cv2.fillPoly / cv2.polylines)
```

Updated `draw_frame()` to accept an optional `shape: int = 0` parameter and call `draw_ball()` instead of `cv2.circle()` directly.

---

### `src/bouncing_ball_task/bouncing_ball.py`

Major changes throughout:

**New `__init__` parameters:**
```python
probability_shape_change: Union[float, Callable] = 0.001,
min_t_shape_change_after_random: int = 15,
warmup_t_no_rand_shape_change: int = 3,
valid_shapes: Optional[Iterable] = None,
initial_shape: Optional[Iterable[int]] = None,
```

**New instance state:**
- `self.valid_shapes` — list of shape names (default `DEFAULT_SHAPES`)
- `self.num_shapes` — number of shapes (default 3)
- `self._shape_index` — current shape index per batch element
- `self.initial_shape` — saved initial shape for sequence reset
- `self.shape_change_indices` — 0 or 1 per batch element, controls cycling
- `self.initial_shape_changes` — initial changepoint array (shape `(batch_size, 1)`)

**New `shape_sampler()` generator** (modeled after `color_sampler`):
```python
def shape_sampler(self):
    yield self._shape_index.copy()
    while True:
        self._shape_index = (self._shape_index + self.shape_change_indices) % self.num_shapes
        yield self._shape_index.copy()
```

**`bouncing_ball_sequence()` changes:**
- Accepts `shape_sequence` generator as new argument
- Preallocates `rand_for_shape` with warmup masking (first `warmup_t_no_rand_shape_change` rows set to 1.0)
- Shape change logic (random only, no bounce trigger):
  ```python
  shape_changes_random = rand_for_shape[t] <= self.probability_shape_change
  # cooldown: block further random changes for min_t timesteps
  rand_for_shape[t+1 : t+min_t+1, shape_changes_random] = 1.0
  self.shape_change_indices = shape_changes_random.astype(int)
  shape = next(shape_sequence)
  ```
- Generator yields 8-tuple: `(position, velocity, color, shape, initial_changes, color_change_array, shape_changes_combined)`

**`get_change_arrays()` updated** to handle the `shape_change` column across all three modes:
- `"any"` — ORs velocity, color, and shape changes into a single column
- `"feature"` — returns `[any_vc, any_cc, any_sc]` (3 columns)
- `"source"` — returns `[vcb, vcr, ccb, ccr, scr]` (5 columns)

**Output array layout** (all transformation functions updated):
- `samples`: `[x, y, r, g, b, shape_idx]` → shape `(B, T, 6)`
- `targets` (`"any"`): `[x, y, r, g, b, shape_idx, any_change]` → shape `(B, T, 7)`
- `targets` (`"feature"`): `[x, y, r, g, b, shape_idx, any_vc, any_cc, any_sc]` → shape `(B, T, 9)`
- `targets` (`"source"`): `[x, y, r, g, b, shape_idx, vcb, vcr, ccb, ccr, scr]` → shape `(B, T, 11)`

---

### `tests/test_bouncing_ball.py`

Updated all tests to account for the new feature dimension (samples now have 6 features instead of 5) and the shifted target column indices:

| What changed | Old | New |
|---|---|---|
| Feature count assertion | `features == 5` | `features == 6` |
| Source mode: vcb column | `tg[..., -4]` | `tg[..., -5]` |
| Source mode: vcr column | `tg[..., -3]` | `tg[..., -4]` |
| Source mode: ccb column | `tg[..., -2]` | `tg[..., -3]` |
| Source mode: ccr column | `tg[..., -1]` | `tg[..., -2]` |
| Feature mode: any_vc column | `tg[..., -2]` | `tg[..., -3]` |
| Feature mode: any_cc column | `tg[..., -1]` | `tg[..., -2]` |
| Target column count (feature mode) | `features + 2` | `features + 3` |
| Target column count (source mode) | `features + 4` | `features + 5` |
| Color slice in grayzone tests | `samples[..., 2:]` | `samples[..., 2:5]` |

Added shape index range validation:
```python
assert np.all(shape_vals >= 0) and np.all(shape_vals <= 2)
```

---

## Test Results

| Run | Passed | Failed |
|---|---|---|
| Original repo (before changes) | 39 | 85 |
| After shape feature added | 81 | 43 |

The remaining 43 failures are all **pre-existing** and unrelated to the shape changes. They fall into three statistical tests whose correction formulas do not fully account for cooldown/warmup effects:

- `test_min_t_color_change_causes_predictable_change_statistics`
- `test_pvc_causes_correct_random_velocity_changes`
- `test_pccnvc_causes_correct_color_changes`

These tests were already failing in the original repository before any modifications.

---

## Verified Behavior

```
samples shape:           (B, T, 6)   — [x, y, r, g, b, shape_idx]
targets shape (source):  (B, T, 11)  — [x, y, r, g, b, shape_idx, vcb, vcr, ccb, ccr, scr]
targets shape (feature): (B, T, 9)   — [x, y, r, g, b, shape_idx, any_vc, any_cc, any_sc]
targets shape (any):     (B, T, 7)   — [x, y, r, g, b, shape_idx, any_change]

Shape index range:      0–2 only
Shape cycle:            circle(0) → square(1) → diamond(2) → circle(0) → ...
Shape changes on bounce (t>0, p=0.0): 0  ✓
```
