# Zone Grid Diagram — PingSight

## Zone Taxonomy

```
Table coordinate system (metres, origin at table centre, net at x = 0):

  Table length: 2.74 m    (x from −1.37 to +1.37)
  Table width:  1.525 m   (y from −0.7625 to +0.7625)
  Table height: 0.76 m    (world z of the playing surface)

  2 halves × 3 rows × 3 columns = 18 zones, plus "0" for outside the table
```

## Grid (top-down)

```
  x: −1.37      −0.91      −0.46       0        0.46       0.91      1.37
      ║  a  ║  b  ║  c  ║   NET   ║  c  ║  b  ║  a  ║
      ╠═════╬═════╬═════╬═════════╬═════╬═════╬═════╣
 col 3║ a3' ║ b3' ║ c3' ║         ║ c3  ║ b3  ║ a3  ║   y: +0.25 … +0.76
 col 2║ a2' ║ b2' ║ c2' ║         ║ c2  ║ b2  ║ a2  ║   y: −0.25 … +0.25
 col 1║ a1' ║ b1' ║ c1' ║         ║ c1  ║ b1  ║ a1  ║   y: −0.76 … −0.25
      ╚═════╩═════╩═════╩═════════╩═════╩═════╩═════╝
        NEAR HALF (x < 0, primed)     FAR HALF (x ≥ 0)
```

## Rules

```
  Row by distance from the net, |x|:
    c   0.00 – 0.46 m   next to the net
    b   0.46 – 0.91 m   middle
    a   0.91 – 1.37 m   baseline

  Column by position across the width, y (same on both halves):
    1   −0.7625 – −0.254 m
    2   −0.254  –  0.254 m
    3    0.254  –  0.7625 m

  Half:
    x < 0   near half, zone name carries a prime   (e.g. b2')
    x ≥ 0   far half                               (e.g. b2)

  Outside the table footprint → "0"
```

## Per-frame zone context

`trajectory.csv` has a `zone` column, and each rendered frame shows the previous, current and next bounce zone (see
`demo/annotated_frame.png`). On the reference clip the two bounces land in `a1'` (frame 13) and `a1` (frame 24).

## Colour scheme in renders

| Row | Colour |
|---|---|
| `c` (net) | orange `#FF9800` |
| `b` (middle) | green `#4CAF50` |
| `a` (baseline) | blue `#2196F3` |

The active bounce zone is highlighted in the 3D table view.
