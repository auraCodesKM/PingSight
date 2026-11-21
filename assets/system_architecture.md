# System Architecture Diagrams — PingSight

## Diagram 1: High-Level System

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                            PINGSIGHT SYSTEM                               ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                           ║
║  ┌─────────┐   ┌──────────────────────────┐   ┌────────────────────────┐  ║
║  │  VIDEO  │   │    PERCEPTION LAYER      │   │    GEOMETRY LAYER      │  ║
║  │  INPUT  │──▶│  Ball detector  (×2)     │──▶│  DLT + LM + RANSAC     │  ║
║  │  (MP4)  │   │  Table detector (×2)     │   │  Camera geometry gate  │  ║
║  └─────────┘   │  13 keypoints · DBSCAN   │   │  K, R, C per clip      │  ║
║                └──────────────────────────┘   └───────────┬────────────┘  ║
║                                                           │               ║
║  ┌────────────────────────────────────────────────────────▼────────────┐  ║
║  │                     CONTACT + RECONSTRUCTION LAYER                   │  ║
║  │                                                                      │  ║
║  │   2D track ──▶ despike ──▶ contact candidates ──▶ bounce / hit      │  ║
║  │                                                        │             │  ║
║  │   learned uplifting model ──▶ height prior             ▼             │  ║
║  │                                   ┌─────────────────────────────┐   │  ║
║  │                                   │  Joint physics fit           │   │  ║
║  │                                   │  Z(t) = Z₀ + V_z·t − ½g·t²   │   │  ║
║  │                                   │  X,Y = ray ∩ plane z = Z(t)  │   │  ║
║  │                                   └─────────────────────────────┘   │  ║
║  └───────────────────────────────────────────┬──────────────────────────┘  ║
║                                              │                             ║
║  ┌───────────────────────────────────────────▼──────────────────────────┐  ║
║  │                         ANALYTICS LAYER                               │  ║
║  │   Zone classifier (18 zones, prev / curr / next) · Physics audit      │  ║
║  └──────────────────────┬─────────────────────────────┬─────────────────┘  ║
║                         │                             │                    ║
║  ┌──────────────────────▼─────────┐   ┌───────────────▼─────────────────┐  ║
║  │  DATA                           │   │  VISUALIZATION                  │  ║
║  │  trajectory.csv · events.json   │   │  Annotated frames · MP4 · GIF   │  ║
║  │  calibration.json · manifest    │   │  3D trajectory plot             │  ║
║  └────────────────────────────────┘   └─────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## Diagram 2: Zone Grid

See [`zone_grid_diagram.md`](zone_grid_diagram.md).

```
        NEAR HALF (x < 0, primed)          FAR HALF (x ≥ 0)

     ┌────────┬────────┬────────┐  ║  ┌────────┬────────┬────────┐
BASE │  a1'   │  a2'   │  a3'   │  ║  │  a1    │  a2    │  a3    │ BASE
     ├────────┼────────┼────────┤  ║  ├────────┼────────┼────────┤
MID  │  b1'   │  b2'   │  b3'   │  ║  │  b1    │  b2    │  b3    │ MID
     ├────────┼────────┼────────┤  ║  ├────────┼────────┼────────┤
NET  │  c1'   │  c2'   │  c3'   │  ║  │  c1    │  c2    │  c3    │ NET
     └────────┴────────┴────────┘ NET └────────┴────────┴────────┘

     Row: a = baseline, b = middle, c = next to the net
     Col: 1, 2, 3 = thirds of the table width
     "0" = bounce outside the table
```

---

## Diagram 3: Data Flow Timeline

```
TIME ──────────────────────────────────────────────────────────────────▶

  VIDEO FRAMES     F1  F2  F3  F4  F5  F6  F7  F8  F9 ...  Fn
                    └───┴───┘   frame triplets (t-1, t, t+1)

  BALL DETECTION   (u, v) per frame from two models, kept where they agree

  TABLE DETECTION  13 keypoints per frame → DBSCAN over the clip → one table

  CALIBRATION      one camera per clip → geometry gate

  CONTACTS         image-height extrema → bounce / hit
                   reference clip: bounce(f13) ── bounce(f24)

  PHYSICS FIT      arcs between contacts, solved jointly
                   bounce → height pinned to the table · hit → free

  ZONES            bounce(f13) → a1'      bounce(f24) → a1
                   rendered per frame as (prev, curr, next)

  OUTPUT           trajectory.csv · events.json · frames · MP4 · GIF
```

---

## Diagram 4: Reconstruction Geometry

```
                 camera centre C
                        ●
                         \
                          \  ray through detection (u, v)
                           \
                            \        Z(t) from the gravity parabola
                             ●───────────────────────── plane z = Z(t)
                            (X, Y, Z)
                              \
                               \
            ════════════════════\════════════  table surface (z = 0.76 m)
                                 ●  bounce: Z pinned to the table
```
