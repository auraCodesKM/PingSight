# System Architecture — PingSight

> Diagrams: [`assets/system_architecture.md`](../assets/system_architecture.md) and
> [`assets/pipeline_flow.md`](../assets/pipeline_flow.md).

## 1. Overview

One call processes a clip end to end: video in; 3D trajectory, contact events, zones and renders out. The stages run in
sequence and hand typed arrays to each other. Model loading goes through `torch.hub`, and every run seeds its RNGs and
requests deterministic cuDNN kernels so results are reproducible.

```
Video (.mp4)
   │
   ▼
1. Ball detection ─────────── two models per frame triplet → agreement filter
2. Table detection ────────── two models → 13 keypoints → DBSCAN per keypoint
3. Calibration ────────────── DLT / LM / RANSAC → camera geometry gate
4. Despike ────────────────── metric-space jump check on the table plane
5. Contact candidates ─────── extrema of smoothed image height
6. Contact classification ─── bounce vs racket hit
7. 3D reconstruction ──────── joint gravity-parabola fit + camera-ray intersection
8. Physics audit ──────────── hard and soft physical checks
9. Zone classification ────── 18 zones, prev / curr / next
10. Output ─────────────────── trajectory.csv, events.json, calibration.json, frames, MP4, GIF, run manifest
```

## 2. Stages

### 2.1 Ball detection (Kienzle et al.)

Each model takes an image triple (previous, current, next) and outputs a heatmap; the ball position is the heatmap
peak, refined with a 3×3 Gaussian fit. Two models run (SegFormer++ B2 and WASB) and a frame is kept when they agree
within 20 px.

### 2.2 Table detection (Kienzle et al.)

SegFormer++ B2 and HRNet predict the 13 standard table keypoints. A per-keypoint DBSCAN over the clip removes temporal
outliers.

### 2.3 Calibration

DLT on the visible keypoints gives an initial projection; Levenberg-Marquardt refines it and RANSAC rejects outlier
keypoints (upstream). PingSight adds a **camera geometry gate**: the camera is validated before it is published to the
rest of the pipeline, and only a validated camera drives 3D reconstruction.

### 2.4 Despike

A detection whose table-plane position jumps more than 5 m in one step is interpolated from its neighbours, keeping the
arrays aligned.

### 2.5 Contacts

Contact candidates are local extrema of the smoothed image-height track (minimum prominence 1.5 px, non-maximum
suppression over 5 steps). Each candidate is then classified:

- outside an out-of-play envelope (|X| > 4 m or |Y| > 2.5 m) it is skipped;
- the tolerance is the ball radius plus the calibration residual scaled by the local metres-per-pixel;
- it is a **bounce** when, within ±1 frame, the ball's table-plane shadow lies inside the table footprint and either the
  margin is non-negative or the vertical motion reverses; otherwise it is a racket **hit**;
- the contact instant is the admissible instant with the largest margin (the lowest ball).

### 2.6 3D reconstruction

Given the camera, X and Y at any frame follow from the height Z along that frame's ray, so the unknown is Z(t): a gravity
parabola per arc. Arcs are split at contacts and solved jointly with least squares; a bounce pins the height to the
table surface, a racket hit does not. The learned uplifting model (Kienzle et al. architecture, checkpoint retrained in
this project) supplies the height prior and the initial value. X and Y are smoothed per arc with a Savitzky-Golay filter.

### 2.7 Physics audit

Hard checks: every bounce inside the table footprint, consecutive bounces on opposite sides, the ball above the net when
it crosses, never below the table surface. Soft checks: gravity, speed band and a plausible envelope.

### 2.8 Zones

Each bounce maps to one of 18 zones (see [`assets/zone_grid_diagram.md`](../assets/zone_grid_diagram.md)), and every frame
carries the previous, current and next bounce zone.

## 3. Outputs

```
outputs/<clip>/
├── frames/                 side-by-side 2D annotated frame | 3D zone view
├── viz/trajectory_3d.png   full-clip arc with bounce markers
├── data/
│   ├── trajectory.csv      frame_idx, time_s, x_m, y_m, z_m, ball_visible, confidence, zone
│   ├── events.json         bounces (video frame, time, x, y, zone) and racket hits (time)
│   └── calibration.json    intrinsics, extrinsics, calibration residual, camera geometry
├── analysis_normal.mp4
├── analysis_slow.gif
└── run_manifest.json       commit, model bundle version, model hashes, input hash, environment
```

Each bounce also records the frame-supported contact interval and whether its zone is the same across that interval.

## 4. Reproducibility

- Six checkpoints in one pinned bundle, each SHA256-checked on load.
- Seeded RNGs and deterministic cuDNN kernels.
- A run manifest per clip answers "what produced this result?".
- A reference-clip golden regression and an 18-case human-labelled event test.
