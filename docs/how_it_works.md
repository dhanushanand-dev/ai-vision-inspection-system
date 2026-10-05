# How It Works

> **Note:** The source code for this project is kept in a private repository. Module and file names below refer to that repository. Access is [available on request](https://github.com/dhanushanand-dev).

The inspection runs in four steps: **teach**, **find the part**, **correct the pose**, and **inspect the zones**.

## 1. Teach

The operator draws an ROI around a known-good part and presses **T** (or presses **A** for an automatic ROI). `InspectionPlan.set_master_ok()` stores this **master** image.

On the master the operator then marks:

- **Reference zones** (key **P**, at least two): distinctive features used only to work out where the part is and how it's rotated.
- **Inspection zones** (key **V**): the features that must be present, such as holes, screws and clips. Each zone has a pass threshold (default 0.62). The **+** and **−** keys adjust it.

For each inspection zone, `ZoneFeatureProfileEngine` learns what kind of feature it is (for example a hole or blob, a mark or text, or a generic feature) and sets a suitable threshold. The plan and tracker are saved with pickle under `models/`, so the next session can reload them.

## 2. Find the part: `MultiTemplateTracker`

The tracker runs on a background thread, so the UI stays responsive.

1. **Template matching (NCC)** against the master, pre-rotated to several angles. After the first lock, only angles near the last one are searched, and only in a window around the last position.
2. **ORB feature matching** runs every *N* frames. It recovers the part after big jumps.
3. **Contour matching** runs every *M* frames. It's a shape-based fallback for low-texture parts.
4. The best candidate is checked with an edge-similarity score and a bounding-box sanity check, then smoothed. If the part disappears for a moment, the last good box is held for a few frames.

## 3. Correct the pose: `PositionCorrector`

Each reference zone is searched for near its expected place in the live frame. From the matched reference centres, the corrector estimates the shift and rotation between the master and the live part. The live image is then warped back into master coordinates, so every inspection zone lines up with the same feature.

## 4. Inspect the zones: `ROIComparator` + `PresenceAbsenceEngine`

- Each aligned zone is compared with the master zone. `PresenceAbsenceEngine` combines grey-level, edge and dark-blob similarity into a zone score.
- A zone passes when its score is at or above the threshold. The part is **OK** only when every required zone passes. Otherwise it is **NG**, and the failing zones are listed and drawn in red.
- To avoid flicker, the verdict changes only after it has been the same for `confirm_frames` frames in a row, with a hysteresis gap around the threshold.

## Hardware

The system is deployed on an **NVIDIA Jetson Orin Nano** edge computer running Linux (JetPack/Ubuntu), with a Basler industrial camera at the inspection station. All the processing is CPU-based OpenCV and NumPy, so the same code also runs on a regular PC for development.

## Cameras

`processing/camera/camera_manager.py` uses a Basler camera through `pypylon` when one is available. Otherwise it falls back to an OpenCV device. The inspection app can also take a webcam index or phone URL (`--phone`), or use a software mock camera (`--mock`). The exposure, gain and resolution settings live in `config/camera_settings.json` and can be edited with `camera_settings_gui.py`.

## Configuration

| File | Purpose |
|---|---|
| `config/camera_settings.json` | Resolution, exposure (µs), gain, auto-exposure, OpenCV fallback |
| `config/app_config.json` *(optional)* | Overrides the display layout: `width`, `height`, `view_w`, `view_h` |
