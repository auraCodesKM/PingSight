# Methodology — PingSight

> Figures referenced here are in [`assets/`](../assets/). The descriptions are mathematical; implementation code lives in
> a private repository.

## 1. Principle

Use physics where the physics is known and learning where it is not. Detection is learned (Kienzle et al.); flight
between contacts is ballistic, so height is fitted analytically and the learned 3D model is used as a prior.

## 2. Camera model

With intrinsics `K`, rotation `R` and camera centre `C`, the viewing ray through pixel `(u, v)` is

```
ray(λ) = C + λ · d,      d = R⁻¹ · K⁻¹ · (u, v, 1)ᵀ
```

Calibration estimates `K, R, C` per clip from the 13 table keypoints (DLT initialisation, Levenberg-Marquardt refinement,
RANSAC over keypoints). The camera is then checked by the geometry gate before use.

## 3. Height model

Between two contacts the ball follows

```
Z(t) = Z₀ + V_z · t − ½ · g · t²,      g = 9.81 m/s²
```

For each frame, the point on its ray at height `Z(t)` is

```
λ*     = (Z(t) − C_z) / d_z
(X, Y) = C_xy + λ* · d_xy
```

All arcs in a clip are solved together by least squares over `(Z₀, V_z)` per arc:

- at a **bounce** the height is constrained to the table surface (0.76 m);
- at a **racket hit** the arc boundary is free;
- a weighted prior term pulls `Z` towards the learned uplifting model's estimate, which also supplies the initial value.

X and Y are then smoothed per arc with a Savitzky-Golay filter (polynomial order 2, window 5–7); Z stays analytic.

## 4. Contacts

Contacts are found in image space, before any 3D exists:

1. Smooth the image-height track of the ball.
2. Take local extrema with prominence ≥ 1.5 px, at least 5 steps apart.
3. For each candidate, project the ball onto the table plane at instants within ±1 frame and test whether that shadow
   lies inside the table footprint, with tolerance = ball radius + calibration residual × local metres-per-pixel.
4. Accept a **bounce** when an admissible instant exists and the margin is non-negative or the vertical motion reverses;
   otherwise label a **hit**.
5. The contact instant is the admissible instant with the largest margin, which is where the ball is lowest.

## 5. Zones

The table (2.74 m × 1.525 m) is split at the net into two halves. Each half is a 3 × 3 grid: rows by distance from the net
(`c` next to the net, `b` middle, `a` baseline), columns by thirds of the width (`1`, `2`, `3`). Zones on the near half
(`x < 0`) carry a prime: `a1'`. A bounce outside the table is zone `0`.

## 6. Physics audit

Every run can be checked against hard constraints (bounce inside the footprint, alternating sides, above the net at the
crossing, never below the table) and soft ones (gravity consistency, speed band, plausible envelope).

## 7. Uplifting model training

The uplifting transformer (Kienzle et al.) is trained on MuJoCo-simulated trajectories rendered through virtual cameras
(azimuth 29°–209°, elevation 30°–70°, distance 7–17 m), with AdamW and an EMA of the weights. PingSight adds a
table-canonical homography to the training transform and retrains the checkpoint used as the height prior.

## 8. Evaluation

- Pipeline completion and runtime over 519 clips from six World Tour matches.
- Reprojection error of the reconstructed track against the 2D detections, per clip and per match.
- An 18-case human-labelled event test and a reference-clip golden regression.
- Blind human review before adopting any detector change.

Results: [`results/performance_benchmarks.md`](../results/performance_benchmarks.md).
