# Pipeline Flow Diagrams — PingSight

## Full Pipeline: Video to Analytics

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              INPUT STAGE                                 │
│   rally_clip.mp4   (single broadcast camera)                             │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │  Frame decode      │
                          │  OpenCV            │
                          └─────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼──────────┐       ┌────────────▼──────────┐
          │  BALL DETECTION    │       │  TABLE DETECTION       │
          │  Input: triplet    │       │  Input: triplet        │
          │  SegFormer++ B2    │       │  SegFormer++ B2        │
          │  + WASB            │       │  + HRNet               │
          │  agree ≤ 20 px     │       │  13 kps · DBSCAN       │
          └─────────┬──────────┘       └────────────┬──────────┘
                    │                               │
                    │               ┌───────────────▼──────────┐
                    │               │  CALIBRATION             │
                    │               │  DLT → LM → RANSAC       │
                    │               │  camera geometry gate    │
                    │               └───────────────┬──────────┘
                    │                               │
          ┌─────────▼───────────────────────────────▼──────────┐
          │  CONTACTS                                          │
          │  despike (> 5 m jump) → image-height extrema       │
          │  footprint test ±1 frame → bounce / hit            │
          └─────────────────────────┬──────────────────────────┘
                                    │
          ┌─────────────────────────▼──────────────────────────┐
          │  PHYSICS FIT                                       │
          │  Z(t) parabola per arc, all arcs solved jointly    │
          │  X,Y from camera ray · height prior from the       │
          │  learned uplifting model · per-arc XY smoothing    │
          └─────────────────────────┬──────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼──────────┐       ┌────────────▼───────────┐
          │  ANALYTICS          │       │  VISUALIZER            │
          │  18-zone classifier │       │  annotated frames      │
          │  prev / curr / next │       │  3D trajectory plot    │
          │  physics audit      │       │  MP4 · slow-motion GIF │
          │  Output:            │       │                        │
          │  trajectory.csv     │       │                        │
          │  events.json        │       │                        │
          └────────────────────┘       └────────────────────────┘
```

---

## Synthetic Data Generation (uplifting model)

```
┌──────────────────────────────────────────────────────────────┐
│                   MUJOCO SIMULATION                           │
│                                                               │
│  ┌────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │ Ball physics   │  │ Initial state   │  │ Virtual camera│  │
│  │ gravity, drag  │  │ position        │  │ azimuth       │  │
│  │ spin, bounces  │  │ velocity, spin  │  │ elevation     │  │
│  │                │  │                 │  │ distance      │  │
│  └────────────────┘  └─────────────────┘  └───────────────┘  │
│                ↓                ↓                ↓            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Simulate flight → 3D trajectory with exact labels       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Project through the virtual camera → 2D sequence        │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Dataset: (2D sequence, table keypoints, 3D trajectory)  │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
              Trains the uplifting transformer, whose output
              serves as the height prior in the physics fit
```

---

## Validation Flow

```
                    CHANGE VALIDATION

     Proposed change (detector, contact rule, gate)
                         │
               ┌─────────▼──────────┐
               │  Reference clip     │
               │  golden regression  │
               └─────────┬──────────┘
                         │
               ┌─────────▼──────────┐
               │  18 human-labelled  │
               │  event cases        │
               └─────────┬──────────┘
                         │
               ┌─────────▼──────────┐
               │  Blind human review │
               │  of changed clips   │
               └─────────┬──────────┘
                         │
               ┌─────────▼──────────┐
               │  Adopt or reject    │
               └────────────────────┘
```
