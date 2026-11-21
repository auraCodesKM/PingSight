# Design Decisions — PingSight

## 1. Monocular broadcast video

**Decision:** work from the single broadcast camera.

Multi-camera tracking systems exist at top tournaments but not in archives, clubs or scouting footage. A single broadcast
feed is the format available at scale.

## 2. Build on Kienzle et al.

**Decision:** reuse the published detectors, calibration and uplifting model; invest in what happens after detection.

The WACV 2026 work already provides strong ball and table detectors trained on table-tennis data. The open problems for
analytics are contact identity, trustworthy geometry and physically consistent 3D.

## 3. Physics-first 3D

**Decision:** fit height with a gravity parabola and place X, Y on the camera ray; use the learned model as a prior.

Ballistic flight is known physics with two unknowns per arc. Placing every point on its viewing ray keeps the 3D track
consistent with what the camera saw, and a learned prior resolves the remaining height ambiguity.

## 4. Contacts from the 2D track

**Decision:** detect contacts on the image-height track before any 3D fit.

Finding bounces from a 3D fit that itself needs bounces is circular. The image track shows a contact as a sharp reversal
regardless of depth.

## 5. Bounce vs hit by table footprint

**Decision:** a contact is a bounce when the ball's table-plane shadow lands inside the table within ±1 frame.

Bounces and racket hits both reverse the ball's image motion. Only one of them happens on the table, and the footprint
test with a calibration-aware tolerance separates them.

## 6. Gate the camera

**Decision:** validate the calibration before publishing it to the rest of the pipeline.

Every 3D point depends on the camera. Checking the geometry once, up front, keeps every downstream number tied to a camera
that passed.

## 7. 18 zones

**Decision:** a 3 × 3 grid per half, rows by distance from the net and columns across the width.

It matches the zone coding used for tactical notation (see `assets/demo/zones.png`) and turns continuous positions into
labels a coach can read.

## 8. Human review gates detector changes

**Decision:** compare every proposed detector fix against blind human labels before adopting it.

Detector-identity errors are easy to "fix" in a way that looks right on one clip and regresses others. Blind review
adopted the Rule B contact classifier and rejected three alternative detector-identity fixes.

## Summary

| Decision | Benefit | Cost |
|---|---|---|
| Monocular | Works on any broadcast | Depth must come from physics and priors |
| Physics-first 3D | Ballistic, ray-consistent tracks | Needs correct contact timing |
| Contacts from 2D | No circular dependency | Relies on detection quality |
| Footprint classification | Separates bounces from hits | Needs a calibrated table plane |
| Camera gate | Downstream numbers rest on validated geometry | Some clips get no 3D |
| 18 zones | Coach-readable placement | Boundary cases between zones |
