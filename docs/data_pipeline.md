# Data Pipeline — PingSight

## 1. Input

| Property | Value |
|---|---|
| Format | MP4, single broadcast camera |
| Clip | one rally segment |
| Evaluation corpus | 519 clips from six World Tour matches (30,417 frames) |

A run takes one video or a directory of videos; each clip gets its own output directory.

## 2. Flow

```
MP4
 │  decode frames
 ▼
Frame triplets ──► Ball detection (2 models) ──► agreement filter ──► (u, v) per frame
            └────► Table detection (2 models) ─► DBSCAN ─────────────► 13 keypoints
                                                                          │
                                                       Calibration + geometry gate
                                                                          │
(u, v) track ──► despike ──► contact candidates ──► bounce / hit ──► joint physics fit
                                                                          │
                                                   Zones · physics audit · rendering
                                                                          │
                                                       data/ · frames/ · viz/ · MP4 · GIF
```

## 3. Output formats

### `data/trajectory.csv`

One row per frame:

| Column | Meaning |
|---|---|
| `frame_idx`, `time_s` | frame index and time in seconds |
| `x_m`, `y_m`, `z_m` | ball position in metres, table-centred world frame |
| `ball_visible`, `confidence` | detection flag and confidence |
| `zone` | current bounce zone |

### `data/events.json`

One record per contact, sorted by time. Bounces: `type`, `video_frame`, `time_s`, `x_m`, `y_m`, `zone`, plus the
frame-supported contact interval and whether the zone is the same across it. Racket hits: `type`, `time_s`.

### `data/calibration.json`

Intrinsics (`Mint`) and extrinsics (`Mext`), the calibration reprojection residual, input properties (resolution,
frame count, fps, visible keypoints) and a camera geometry summary.

### `run_manifest.json`

Git commit, model bundle version and per-model hashes, input file hash, PyTorch / CUDA / GPU fingerprint.

## 4. Runtime

Measured over the 519-clip run on an RTX 4090: **73.5 s per clip** on average, 635.7 min in total.

## 5. Training data for the uplifting model

The uplifting transformer is trained on MuJoCo-simulated ball flights (gravity, bounces, drag and spin) projected
through randomly sampled virtual broadcast cameras, which gives unlimited trajectories with exact 3D labels.
