# PingSight

**Recovers metric 3D ball trajectories, bounce events and landing zones from a single broadcast camera, run across 519
World Tour broadcast clips.**

PingSight takes an ordinary monocular table-tennis broadcast and produces the ball's 3D flight arc in metres, every
bounce and racket hit, and which of 18 table zones each bounce landed in. No multi-camera rig, markers or sensors.

It builds on **Kienzle et al., *Uplifting Table Tennis: A Robust, Real-World Application for 3D Trajectory and Spin
Estimation* (WACV 2026)**, which provides the ball and table detectors, the camera-calibration code and the learned
2D→3D uplifting model. On top of that, PingSight adds a physics-first 3D reconstruction, bounce/hit classification, a
camera geometry gate and zone analytics ([details](#what-pingsight-adds)).

This repository presents results, figures and write-ups. Implementation is in a private repository; available on
request. Built during a TEEP research internship (2026).

## Results gallery

### Pipeline output frame

![Annotated Frame with 3D Zone Model](assets/demo/annotated_frame.png)
*Frame rendered by the pipeline. Left: broadcast with the running reprojection error and previous / current / next
bounce zone. Right: the table in 3D with the 18-zone grid. WTT Finals Fukuoka, Wang Chuqin vs Tomokazu Harimoto.*

### Broadcast frame and 3D table model

![Rally 3D Reconstruction](assets/demo/rally_3d_reconstruction.png)
*Broadcast frame beside the recovered 3D table: the 13 calibration keypoints (green), the zone grid, and the ball
position (orange). WTT Champions Frankfurt, Félix Lebrun vs Truls Moregard.*

![Rally 3D Reconstruction GIF](assets/demo/rally_3d_reconstruction.gif)
*The same view animated over the rally.*

### 3D trajectory views

| Trajectory animation | Split-view animation |
|:---:|:---:|
| ![3D Trajectory](assets/demo/3d_trajectory_animation.gif) | ![3D Split Animation](assets/demo/3d_trajectory_split_animation.gif) |
| *Rotating 3D view of a reconstructed trajectory: flight arc (orange) and table keypoints (green).* | *Frame and 3D trajectory side by side, rendered by the 3D animation script on the example sequence from Kienzle et al.* |

![3D Trajectory Visualization](assets/demo/3d_trajectory_visualization.png)
*Static 3D plot of the same example sequence, rendered by the 3D plotting script.*

### Learned uplifting model (Kienzle et al.)

![Full Pipeline Output](assets/demo/full_pipeline_output.png)
*The learned uplifting model's 3D trajectory reprojected onto the example sequence (WTT Incheon), from the upstream
tutorial run. In PingSight this model provides the height prior for the physics fit.*

| 2D input to the uplifting model | 3D → 2D reprojection |
|:---:|:---:|
| ![2D Input](assets/uplifting/2d_input_to_uplifting.png) | ![2D Reprojection](assets/uplifting/2d_reprojection_from_3d.png) |
| *2D ball detections fed to the uplifting model. WTT Champions Frankfurt, Lebrun vs Moregard.* | *Table keypoints (blue) and a 3D ball position reprojected into the image (green).* |

### Detection

Detectors and the dual-model filter are from Kienzle et al.; these were rendered by the tutorial script on the example
sequence.

| Single model, raw | Two models, filtered |
|:---:|:---:|
| ![Raw Ball Detections](assets/detection/raw_ball_detections.png) | ![Filtered Ball Detections](assets/detection/filtered_ball_detections.png) |
| ![Raw Table Keypoints](assets/detection/raw_table_keypoints.png) | ![Filtered Table Keypoints](assets/detection/filtered_table_keypoints.png) |

![Debug Overlay](assets/demo/debug_overlay.png)
*Combined overlay: table keypoints (magenta), 2D ball detections (green) and the reconstructed trajectory reprojected onto
the broadcast frame.*

## The problem

Broadcast table tennis is filmed by one camera from a different angle at every venue. The ball is a few pixels wide and
often blurred or hidden by players. Recovering 3D needs the camera's pose, which must be estimated from the table in each
clip, and depth along a single camera ray is ambiguous.

## What it does

- Detects the ball and 13 table keypoints in every frame with two models each.
- Calibrates the camera from the table keypoints and validates the camera geometry before any 3D is produced.
- Finds contact candidates in the 2D track and classifies each as a table **bounce** or a racket **hit**.
- Reconstructs 3D with a physics fit: a gravity parabola for height per arc, X/Y from each frame's camera ray.
- Labels each bounce with one of 18 zones and tracks previous / current / next zone per frame.
- Writes `trajectory.csv` (x, y, z, zone per frame), `events.json` (bounces with frame, time, position and zone; racket
  hits with time), `calibration.json`, rendered frames, MP4 and GIF, and a run manifest (commit, model hashes, input
  hash, GPU).

## What PingSight adds

From Kienzle et al.: the SegFormer++ / WASB ball detectors, the SegFormer++ / HRNet table detectors, the dual-model
filter, DLT → Levenberg-Marquardt → RANSAC calibration, the uplifting transformer and the MuJoCo data generator.

Added in PingSight:

| Addition | What it does |
|---|---|
| Physics-first 3D reconstruction | Height per arc is a gravity parabola `Z(t) = Z₀ + V_z·t − ½g·t²`; X, Y come from intersecting the camera ray with `z = Z(t)`. Arcs are split at contacts and solved jointly: a bounce pins the height to the table, a racket hit does not. The learned uplifting model supplies the height prior and initial value. |
| 2D contact detection | Contacts are found on the 2D track (extrema of smoothed image height) before any 3D exists, so bounce finding doesn't depend on the 3D fit it feeds. |
| Bounce vs hit classification | Accepts a bounce when the ball's table-plane shadow falls inside the table within ±1 frame, with a tolerance from ball radius and calibration residual. |
| Camera geometry gate | Validates each calibration before use, so 3D is produced only from trustworthy camera geometry. |
| Metric-space despike | Interpolates detections that jump more than 5 m between steps on the table plane. |
| Zone analytics | 18 zones (rows c/b/a from the net out, columns 1–3 across), with previous / current / next zone per frame. |
| Physics audit | Per-run checks: bounces inside the table, consecutive bounces on opposite sides, ball above the net when crossing, never below the table. |
| Reproducibility | Pinned SHA256-checked model bundle, deterministic cuDNN, a run manifest per clip, and a reference-clip regression test. |

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         PINGSIGHT PIPELINE                         │
│                                                                    │
│  MP4 Input                                                         │
│     │                                                              │
│     ▼  ┌────────────────────────────────────────────────────┐     │
│        │           STAGE 1 — 2D PERCEPTION                  │     │
│        │  Frame triplet (t-1, t, t+1)                       │     │
│        │  ┌─────────────────────┐  ┌──────────────────────┐ │     │
│        │  │  Ball Detection     │  │  Table Detection     │ │     │
│        │  │  Model A + Model B  │  │  Model A + Model B   │ │     │
│        │  │  → agree ≤ 20 px    │  │  → 13 keypoints      │ │     │
│        │  │                     │  │  → DBSCAN filter     │ │     │
│        │  └──────────┬──────────┘  └──────────┬───────────┘ │     │
│        └─────────────┼─────────────────────────┼─────────────┘     │
│                      │                         │                   │
│     ▼  ┌─────────────┼─────────────────────────▼─────────────┐    │
│        │           STAGE 2 — CAMERA CALIBRATION               │    │
│        │  DLT + Levenberg-Marquardt + RANSAC                   │    │
│        │  + camera geometry gate                               │    │
│        └─────────────────────────────────┬────────────────────┘    │
│                                          │                         │
│     ▼  ┌─────────────────────────────────▼────────────────────┐    │
│        │           STAGE 3 — CONTACTS + PHYSICS FIT            │    │
│        │  2D contact candidates → bounce / hit classification  │    │
│        │  Z(t) gravity parabola per arc, solved jointly        │    │
│        │  X,Y = camera ray ∩ plane z = Z(t)                    │    │
│        │  learned uplifting model → height prior               │    │
│        └─────────────────────────────────┬────────────────────┘    │
│                                          │                         │
│     ▼  ┌─────────────────────────────────▼────────────────────┐    │
│        │           STAGE 4 — ANALYTICS                          │    │
│        │  Zone classification (18 zones) · prev/curr/next       │    │
│        │  Physics audit                                         │    │
│        └──────────────────┬──────────────────┬─────────────────┘    │
│                           │                  │                      │
│     ┌─────────────────────▼──┐   ┌───────────▼───────────────┐     │
│     │   DATA                  │   │   VISUALIZER              │     │
│     │  trajectory.csv         │   │  frames · MP4 · GIF       │     │
│     │  events.json            │   │  3D trajectory plot       │     │
│     └────────────────────────┘   └───────────────────────────┘     │
└────────────────────────────────────────────────────────────────────┘
```

### Zone grid

![Zone Grid](assets/demo/zones.png)
*Zone coding reference (substitution table) the grid is based on.*

```
                    NEAR SIDE (primed, x < 0)  ║  FAR SIDE (x ≥ 0)
                                               ║
              Col 1    Col 2    Col 3          ║          Col 1    Col 2    Col 3
             ┌────────┬────────┬────────┐      ║      ┌────────┬────────┬────────┐
  BASELINE   │  a1'   │  a2'   │  a3'   │      ║      │  a1    │  a2    │  a3    │  BASELINE
             ├────────┼────────┼────────┤      ║      ├────────┼────────┼────────┤
  MIDDLE     │  b1'   │  b2'   │  b3'   │      ║      │  b1    │  b2    │  b3    │  MIDDLE
             ├────────┼────────┼────────┤      ║      ├────────┼────────┼────────┤
  NET        │  c1'   │  c2'   │  c3'   │      ║      │  c1    │  c2    │  c3    │  NET
             └────────┴────────┴────────┘     NET     └────────┴────────┴────────┘
```

Rows are thirds of each half by distance from the net (c = next to the net, a = baseline); columns are thirds of the
width. A bounce outside the table is zone `0`. More: [`assets/zone_grid_diagram.md`](assets/zone_grid_diagram.md).

## Evaluation

All runs on an NVIDIA RTX 4090 (PyTorch 2.6, CUDA 12.4). Full tables: [`results/performance_benchmarks.md`](results/performance_benchmarks.md).

| What | Result | Conditions |
|---|---|---|
| Pipeline completion | 518 of 519 clips (99.8%) | 6 World Tour matches, 30,417 frames |
| Runtime | 73.5 s per clip on average | Same 519 clips, 635.7 min total |
| Reprojection error after the physics fit | Median 3.25 px | Clips that pass the geometry gate, same run |
| Event identity | 18 / 18 correct | Human-labelled bounce / no-contact cases, regression test |
| Reference clip | 2 bounces at frames 13 and 24 (zones a1′, a1) | Matches the golden regression record (fixed torch / cuDNN / GPU environment) |

Pipeline changes are measured against blind human labels before they ship: the Rule B contact classifier was adopted
this way, and three alternative detector-identity fixes were rejected on the same evidence.

## Tech stack

| Layer | Used |
|---|---|
| Language | Python 3.10 |
| Deep learning | PyTorch 2.6 (CUDA 12.4), loaded through `torch.hub` |
| Detection models | SegFormer++ B2 + WASB (ball), SegFormer++ B2 + HRNet (table), from Kienzle et al. |
| Geometry and fitting | NumPy, SciPy (`least_squares`), OpenCV, scikit-learn (DBSCAN) |
| Synthetic training data | MuJoCo 3.3.2, for the uplifting model |
| Rendering | Matplotlib, imageio |
| Hardware | NVIDIA RTX 4090 |

## Repository structure

```
README.md
docs/       architecture, methodology, design decisions, data formats
results/    performance_benchmarks.md, sample_outputs.md
assets/     demo/, detection/, uplifting/ figures; architecture, pipeline and zone diagrams
```

## Design choices and roadmap

- **Physics over learned depth.** A gravity parabola plus camera-ray intersection keeps every 3D point consistent with the
  observed 2D track and with ballistic motion; the learned model contributes where it is strongest, as a height prior.
- **Bounces from 2D first.** Detecting contacts on the image track before fitting 3D removes the circular dependency
  between finding bounces and fitting arcs.
- **Gate the camera, not the output.** Each clip's calibration is validated before 3D is produced, so downstream
  analytics run on geometry that passed the check.
- **Roadmap:** a coaching assistant that turns zone sequences into written tactical feedback (retrieval over coaching
  material plus an LLM), 3D ground truth from multi-camera capture for metric accuracy, a drag and Magnus-effect flight
  model, and faster inference.

## Credits

- Kienzle, Ludwig, Lorenz, Satoh, Lienhart. *Uplifting Table Tennis: A Robust, Real-World Application for 3D Trajectory
  and Spin Estimation.* WACV 2026. Code: [KieDani/UpliftingTableTennis](https://github.com/KieDani/UpliftingTableTennis).
- WASB and HRNet authors for the detector backbones.
- Footage: World Table Tennis (WTT) and ITTF broadcasts.
