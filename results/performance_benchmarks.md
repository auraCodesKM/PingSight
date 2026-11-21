# Performance Benchmarks — PingSight

All numbers come from recorded runs of the pipeline on an NVIDIA RTX 4090 (PyTorch 2.6.0, CUDA 12.4).

## 1. Large-scale run

Six World Tour matches, every rally clip processed end to end.

| Match | Games | Clips | Frames |
|---|---:|---:|---:|
| 2024-10-27 · Félix Lebrun vs Lin Shidong | 4 | 76 | 5,270 |
| 2024-10-28 · Félix Lebrun vs Tomokazu Harimoto | 5 | 82 | 5,421 |
| 2025-02-05 · Tomokazu Harimoto vs Xue Fei | 4 | 66 | 4,863 |
| 2025-02-06 · Lin Yun-Ju vs Hugo Calderano | 4 | 83 | 3,657 |
| 2025-02-08 · Wang Chuqin vs Liang Jingkun | 1 | 133 | 8,776 |
| 2025-02-07 · Lin Yun-Ju vs Lin Shidong | 1 | 79 | 2,430 |
| **Total** | | **519** | **30,417** |

| Metric | Value |
|---|---|
| Clips completed | 518 / 519 (99.8%) |
| Average runtime per clip | 73.5 s |
| Total runtime | 635.7 min |

## 2. Reprojection error after the physics fit

Reprojection error measures how closely the reconstructed 3D track projects back onto the 2D ball detections. Computed
per clip from the run log, over clips with a valid calibration.

| Percentile | Reprojection error |
|---|---:|
| p25 | 1.43 px |
| **Median** | **3.25 px** |
| p75 | 11.60 px |

Per match (each match is one venue and camera setup):

| Match | Median |
|---|---:|
| Lebrun vs Lin Shidong | 1.75 px |
| Lebrun vs Harimoto | 1.79 px |
| Wang Chuqin vs Liang Jingkun | 2.17 px |
| Harimoto vs Xue Fei | 2.89 px |
| Lin Yun-Ju vs Lin Shidong | 7.27 px |
| Lin Yun-Ju vs Calderano | 18.77 px |

## 3. Event identity

Regression test over 18 human-labelled cases (bounce at a given frame, or no contact, including frames where the
detector sits on a player's clothing): **18 / 18 correct**.

## 4. Reference clip

The reference clip is reconstructed with two bounces at video frames 13 and 24, in zones `a1'` and `a1`, and the output
matches the golden regression record field by field in the recorded environment. Frame-by-frame human review confirmed
frame 13 as the first contact.

## 5. Change review

Proposed changes are compared against blind human labels before they are adopted. Reviews with recorded labels:

| Change | Reviewed cases | Outcome |
|---|---:|---|
| Contact classification ("Rule B") | 6 | adopted |
| Despike / bridging gate | 6 | rejected (5 of 6 changes were regressions) |
| Top-K association override | 10 | rejected (5 correct, 5 wrong) |
