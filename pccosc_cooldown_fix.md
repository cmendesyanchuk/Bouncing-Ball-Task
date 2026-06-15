# Fix: pccosc cooldown semantic inconsistency

## The problem

`pccosc` (probability color changes given shape changed, velocity not changed) is supposed to be an unconditional probability — if you set `pccosc=0.8`, a shape change should produce a color change 80% of the time, every time.

The implementation violated this. After a shape-triggered color change, the cooldown block set:

```python
rand_for_color[t+1 : t+min_t_color_change_after_shape_change+1, ..., 1] = 1.0
```

`rand_for_color[:, :, 1]` is a single channel shared by both pccosc and pccnvc. Setting it to 1.0 was intended to block the next *random* (pccnvc) color changes during the cooldown window. But it also blocked the next *shape-triggered* (pccosc) color change, because both drew from the same array.

The check `rand_for_color[t, i, 1] <= pccosc` would evaluate `1.0 <= 0.8` — False — silently suppressing a color change that pccosc=0.8 should have permitted.

The only case where this was not a problem was `pccosc=1.0`, because `1.0 <= 1.0` is True. For any `pccosc < 1.0` the guarantee was broken.

### Why this happened

The cooldown mechanism was originally designed for pccnvc — a purely random color change. It suppresses the random channel for N steps after a color change to prevent clustering. When pccosc was introduced, it was routed through the same `rand_for_color[:, :, 1]` channel, inheriting the cooldown suppression unintentionally.

---

## The fix

Give pccosc its own dedicated random array, `rand_for_shape_color`, fully independent from `rand_for_color[:, :, 1]`. The pccnvc cooldown can never reach it, and the pccosc cooldown can never reach pccnvc.

### Col-1 is now two independent paths

**Before:** a single `effective_prob` stacked the two probabilities and drew from one array:

```python
effective_prob[:, 1] = np.where(shape_changes_random, pccosc, pccnvc)
color_changes_combined[:, 1] = rand_for_color[t, :, 1] <= effective_prob[:, 1]
```

**After:** two independent draws, OR'd together:

```python
pccnvc_fires = vel_not_changed AND ~shape_changed AND rand_for_color[t, :, 1] <= pccnvc
pccosc_fires = vel_not_changed AND  shape_changed AND rand_for_shape_color[t]  <= pccosc

color_changes_combined[:, 1] = pccnvc_fires OR pccosc_fires
```

### Cooldowns now route to separate arrays

| Color change source | Cooldown suppresses |
|---|---|
| pccosc (shape-triggered) | `rand_for_shape_color` for `min_t_color_change_after_shape_change` steps |
| pccnvc (purely random) | `rand_for_color[:, :, 1]` for `min_t_color_change_after_random` steps |

Neither can interfere with the other.

---

## What is not affected

- Col-0 (vel changed): pccovc and pccovasc still share `rand_for_color[:, :, 1]`... 

  wait — col-0 uses `rand_for_color[:, :, 0]`, not col-1. The col-0 cooldown (`min_t_color_change_after_bounce`) suppresses col-1, not col-0 itself. A symmetric issue could in principle arise for pccovasc but was not raised and is not addressed here.

- The cross-cooldown (`min_t_bounce_color_change_after_random`): after any col-1 change, suppress col-0 for N steps. This still applies to both pccnvc and pccosc fires — unchanged.

- `min_t_color_change_after_bounce`: after a col-0 color change, suppress `rand_for_color[:, :, 1]` for N steps. This still applies — and notably it does NOT suppress `rand_for_shape_color`, so a bounce-triggered color change no longer blocks the next pccosc-triggered one either.
