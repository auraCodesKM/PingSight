# Sample Outputs — PingSight

Outputs on WTT broadcast footage: Fukuoka (Wang Chuqin vs Harimoto), Frankfurt (Lebrun vs Moregard) and the Incheon
example sequence distributed with Kienzle et al.'s code.

## 1. Annotated frames

![Annotated Frame](../assets/demo/annotated_frame.png)
*Pipeline frame render, Fukuoka. Left: broadcast with frame counter, running reprojection error and previous / current /
next bounce zone. Right: the table in 3D with the 18-zone grid and the ball.*

![Full Pipeline Output](../assets/demo/full_pipeline_output.png)
*Learned uplifting model's trajectory reprojected onto the Incheon example sequence (upstream tutorial run).*

Each rendered frame contains:

- the broadcast frame with the ball's recent track;
- a text panel: frame number, reprojection error, previous / current / next bounce zone;
- a 3D panel: table with the zone grid coloured by row, the 13 table keypoints, and the ball.

## 2. Zones

![Zone Grid](../assets/demo/zones.png)
*Zone coding reference behind the 18-zone grid. Full definition: [`assets/zone_grid_diagram.md`](../assets/zone_grid_diagram.md).*

## 3. 3D reconstruction

![Rally 3D Reconstruction](../assets/demo/rally_3d_reconstruction.png)
*Frankfurt: broadcast frame beside the recovered 3D table with its 13 keypoints and the ball position.*

![Rally 3D Reconstruction GIF](../assets/demo/rally_3d_reconstruction.gif)

| Rotating view | Split view |
|:---:|:---:|
| ![3D anim](../assets/demo/3d_trajectory_animation.gif) | ![split](../assets/demo/3d_trajectory_split_animation.gif) |

## 4. Data files

A reference-clip run produces, among others:

```
data/trajectory.csv
frame_idx,time_s,x_m,y_m,z_m,ball_visible,confidence,zone
...

data/events.json
[
  {"type": "bounce", "video_frame": 13, "zone": "a1'", "time_s": ..., "x_m": ..., "y_m": ..., ...},
  {"type": "bounce", "video_frame": 24, "zone": "a1",  "time_s": ..., "x_m": ..., "y_m": ..., ...}
]
```

Field definitions: [`docs/data_pipeline.md`](../docs/data_pipeline.md).

## 5. Intermediate stages

| | Single model, raw | Two models, filtered |
|---|:---:|:---:|
| **Ball** | ![rb](../assets/detection/raw_ball_detections.png) | ![fb](../assets/detection/filtered_ball_detections.png) |
| **Table** | ![rt](../assets/detection/raw_table_keypoints.png) | ![ft](../assets/detection/filtered_table_keypoints.png) |

*Rendered by the tutorial script on the Incheon example sequence; detectors and filter from Kienzle et al.*

| 2D input to uplifting | 3D → 2D reprojection | Debug overlay |
|:---:|:---:|:---:|
| ![2d](../assets/uplifting/2d_input_to_uplifting.png) | ![rep](../assets/uplifting/2d_reprojection_from_3d.png) | ![dbg](../assets/demo/debug_overlay.png) |
