# Grey Zone Color Fade

## Overview

Modified the visual rendering so the ball's color gradually blends toward grey as it enters the grey zone, becoming fully invisible once completely inside.

---

## File Modified

### `src/bouncing_ball_task/utils/gif.py`

Three changes were made to this file.

---

### Change 1 — Remove white override from `draw_circle`

**Before:**
```python
def draw_circle(...):
    return cv2.circle(
        np.copy(background),
        ...,
        color=[255, 255, 255] if color == mask_color else list(color),
        thickness=thickness,
    )
```

**After:**
```python
def draw_circle(...):
    return cv2.circle(
        np.copy(background),
        ...,
        color=list(color),
        thickness=thickness,
    )
```

**Why:** The original code drew the ball in white when its stored color matched `mask_color`. This caused a white flash on the visible portion of the ball the moment the `color_mask_mode` triggered masking (e.g. `"outer"` mode), before the grey paint step could cover it.

---

### Change 2 — Remove white override from `draw_ball`

**Before:**
```python
def draw_ball(...):
    draw_color = [255, 255, 255] if (np.array(color) == np.array(mask_color)).all() else [int(c) for c in color]
```

**After:**
```python
def draw_ball(...):
    draw_color = [int(c) for c in color]
```

**Why:** Same issue as `draw_circle`. The white override is unnecessary because `draw_frame` already handles occlusion via the grey paint step.

---

### Change 3 — Gradual color blend in `draw_frame`

**Before:**
```python
def draw_frame(position, color, ball_radius, mask_color, size_frame, mask_start, mask_end, ...):
    frame = np.zeros((*size_frame[::-1], 3), dtype=np.uint8)
    frame = draw_ball(position, color, frame, ball_radius, mask_color, ...)
    frame[:, mask_start:mask_end, :] = mask_color
    ...
```

**After:**
```python
def draw_frame(position, color, ball_radius, mask_color, size_frame, mask_start, mask_end, ...):
    frame = np.zeros((*size_frame[::-1], 3), dtype=np.uint8)
    x_position = position[0]

    # Blend draw color toward mask_color proportional to overlap with the grey zone
    overlap_width = max(min(x_position + ball_radius, mask_end) - max(x_position - ball_radius, mask_start), 0)
    overlap_proportion = overlap_width / (2 * ball_radius)
    blended_color = [
        int(round(overlap_proportion * mc + (1 - overlap_proportion) * c))
        for c, mc in zip(color, mask_color)
    ]

    frame = draw_ball(position, blended_color, frame, ball_radius, mask_color, ...)
    frame[:, mask_start:mask_end, :] = mask_color
    ...
```

**Why:** `overlap_proportion` is the fraction of the ball's diameter that overlaps with the grey zone (0 = fully outside, 1 = fully inside). The draw color is linearly interpolated between the ball's actual color and `mask_color` by that proportion. After drawing, the grey paint step covers the interior of the grey zone as before. The result is that the visible portion of the ball (outside the grey zone) smoothly fades from its actual color to grey as it enters.

---

## Verified Behavior

| Overlap | Draw color | Visible area |
|---|---|---|
| 0% | actual color | full ball |
| 25% | 25% towards grey | 75% of ball |
| 50% | 50% towards grey | 50% of ball |
| 75% | 75% towards grey | 25% of ball |
| 100% | grey (invisible) | none |
