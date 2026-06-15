# Free Parameters — Velocity, Color, Shape

## Velocity

| Parameter | Default | Description |
|---|---|---|
| `initial_velocity` | `None` | Fixed starting velocity; sampled if `None` |
| `velocity_x_lower_multiplier` | `1/12.5` | Lower bound for x velocity magnitude as fraction of frame width |
| `velocity_x_upper_multiplier` | `1/7.5` | Upper bound for x velocity magnitude as fraction of frame width |
| `velocity_y_lower_multiplier` | `1/12.5` | Lower bound for y velocity magnitude as fraction of frame height |
| `velocity_y_upper_multiplier` | `1/7.5` | Upper bound for y velocity magnitude as fraction of frame height |
| `same_xy_velocity` | `False` | Force x and y velocity magnitudes to be equal |
| `sample_velocity_discretely` | `False` | Sample from a fixed linspace of velocities instead of continuous uniform |
| `num_x_velocities` | `1` | Number of discrete x velocity values (if `sample_velocity_discretely`) |
| `num_y_velocities` | `2` | Number of discrete y velocity values (if `sample_velocity_discretely`) |
| `initial_velocity_points_away_from_grayzone` | `True` | On first sample, force velocity to point away from the grayzone |
| `probability_velocity_change` | `0.001` | Probability of a random velocity resample per timestep |
| `min_t_velocity_change_after_bounce` | `5` | Cooldown: min timesteps before a random velocity change after a wall bounce |
| `min_t_velocity_change_after_random` | `5` | Cooldown: min timesteps before another random velocity change after one occurred |
| `warmup_t_no_rand_velocity_change` | `3` | Initial timesteps during which random velocity changes are blocked |
| `forced_velocity_bounce_x` | `None` | Timesteps to force an x-wall bounce |
| `forced_velocity_bounce_y` | `None` | Timesteps to force a y-wall bounce |
| `forced_velocity_resamples` | `None` | Timesteps to force a random velocity resample |

---

## Color

| Parameter | Default | Description |
|---|---|---|
| `initial_color` | `None` | Fixed starting color; sampled if `None` |
| `valid_colors` | `"default"` | Set of allowed colors (`"default"`, `"constant"`, or explicit list) |
| `num_colors` | `None` | Number of colors to generate if `valid_colors` is `None` |
| `probability_initial_colors` | `None` | Probability distribution over colors for initial color sampling |
| `color_sampling` | `"fixed"` | How next color is picked: `"fixed"` (cycle) or `"random"` |
| `probability_color_change_no_velocity_change` (pccnvc) | `0.01` | P(color changes \| vel not changed, shape not changed) |
| `probability_color_change_on_velocity_change` (pccovc) | `1.0` | P(color changes \| vel changed, shape not changed) |
| `probability_color_change_on_shape_change` (pccosc) | `0.0` | P(color changes \| shape changed, vel not changed) |
| `probability_color_change_on_velocity_and_shape_change` (pccovasc) | `1.0` | P(color changes \| vel changed AND shape changed) |
| `pccnvc_lower` / `pccnvc_upper` | `None` | Range to sample pccnvc from across batch elements |
| `num_pccnvc` | `None` | Number of bins when sampling pccnvc from range |
| `pccovc_lower` / `pccovc_upper` | `None` | Range to sample pccovc from across batch elements |
| `num_pccovc` | `None` | Number of bins when sampling pccovc from range |
| `min_t_color_change_after_bounce` | `5` | Cooldown: min timesteps before a pccnvc change after a col-0 (vel-triggered) color change |
| `min_t_color_change_after_random` | `5` | Cooldown: min timesteps before another pccnvc change after one occurred |
| `min_t_color_change_after_shape_change` | `5` | Cooldown: min timesteps before another pccosc change after one occurred |
| `min_t_bounce_color_change_after_random` | `3` | Cooldown: min timesteps before a col-0 color change after any col-1 change |
| `warmup_t_no_rand_color_change` | `3` | Initial timesteps during which all color changes are blocked |
| `color_change_bounce_delay` | `0` | Timesteps between a vel-triggered event and the resulting color change |
| `color_change_random_delay` | `0` | Timesteps between a random/shape-triggered event and the resulting color change |
| `forced_color_changes` | `None` | Timesteps to force a color change |
| `color_mask_mode` | `"inner"` | How the grayzone masks the ball color (`"inner"`, `"outer"`, `"centroid"`, `"fade"`) |
| `transitioning_change_mode` | `None` | Whether random color changes are allowed while the ball is entering the grayzone |
| `transition_tol` | `5` | Pixel tolerance around grayzone edge for transition detection |

---

## Shape

| Parameter | Default | Description |
|---|---|---|
| `initial_shape` | `None` | Fixed starting shape index; sampled if `None` |
| `valid_shapes` | `None` (→ circle, square, diamond) | Set of allowed shapes |
| `probability_shape_change` | `0.001` | Probability of a random shape change per timestep |
| `min_t_shape_change_after_random` | `15` | Cooldown: min timesteps before another shape change after one occurred |
| `warmup_t_no_rand_shape_change` | `3` | Initial timesteps during which shape changes are blocked |
| `forced_shape_changes` | `None` | Timesteps to force a shape change |
