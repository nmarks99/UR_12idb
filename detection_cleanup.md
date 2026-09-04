# AprilTag Detection Cleanup Plan

## Goal

Remove the IOC one-shot `DetectTag` command and retain continuous AprilTag
tracking only for the camera overlay and diagnostic tag-result PVs. Separate
this display-oriented feature from robot-operation support before adding the
runtime user-script interface.

User scripts will continue to perform robot-facing AprilTag work through the
connected robot's camera APIs, such as `robot.camera.decodeAT()` and the
existing helpers in `camera_tools.py`. They must not depend on IOC tag-result
PVs for robot motion decisions.

## Current State

`iocBoot/ioc12idUR/robot_operations.py` currently contains two unrelated
features:

- Robot command support: camera capture and three fixed user scripts.
- AprilTag support: one-shot detection, continuous tracking, shared tag
  result state, and pyDevSup result-record support.

Both one-shot `DetectTag` and continuous `TrackTag` write the same
`TagResult`. The lock prevents a corrupted tuple, but the result is ambiguous
when they overlap. This ambiguity disappears when one-shot detection is
removed.

`TrackTag` reads the `image1:` areaDetector stream. It does not use the
connected robot or the serialized robot worker. Its bounding-box result PVs
feed `TagOver:1` in `iocsh/ADURL.iocsh`; `Py:TagOverlayUse` enables the overlay
when `Py:TagFound` is true.

## Intended Interface

Retain these records in a dedicated tag-tracking database:

```text
$(P)Py:TrackTag
$(P)Py:TagId
$(P)Py:TagCenterX
$(P)Py:TagCenterY
$(P)Py:TagBoxMinX
$(P)Py:TagBoxMinY
$(P)Py:TagBoxSizeX
$(P)Py:TagBoxSizeY
$(P)Py:TagFound
$(P)Py:TagOverlayUse
```

Remove these one-shot detection records:

```text
$(P)Py:DetectTag
$(P)Py:DetectTagRetries
$(P)Py:DetectTagAttempt
```

`TrackTag` remains an explicit enable/disable command. It remains responsible
for starting and stopping the background detector, and it is stopped at IOC
exit.

## Implementation Steps

1. Create `iocBoot/ioc12idUR/tag_tracking.py`.

   Move the following from `robot_operations.py` into the new module:

   - `TagResult` and the global tag result instance.
   - The tag I/O Intr scan list.
   - `TagTracker` and the global tracker instance.
   - `TrackTagSupport`.
   - `TagResultSupport`.
   - The AprilTag detector imports and `EpicsImageSource` import.
   - A module-level `build(record, args)` that accepts `TrackTag <image-prefix>`
     and a tag-result field name.

   The tracker must continue to reset and publish the result when no image is
   available, zero or multiple tags are found, an error occurs, tracking is
   stopped, or the IOC exits.

2. Simplify tag-result state.

   Remove `DetectTagAttempt` from the field map and result tuple. The remaining
   fields are `TagFound`, `TagId`, `TagCenterX`, `TagCenterY`, `TagBoxMinX`,
   `TagBoxMinY`, `TagBoxSizeX`, and `TagBoxSizeY`.

   Preserve current values for the no-detection state:

   ```python
   (False, -1, 0.0, 0.0, 0, 0, 0, 0)
   ```

   Preserve bounding-box clamping and minimum size behavior, because the
   areaDetector overlay consumes those values directly.

3. Remove one-shot detection from `robot_operations.py`.

   Delete:

   - `detect_tag()`.
   - The `DetectTag` dispatcher branch.
   - Imports used exclusively by tag support or one-shot detection, including
     `math`, `threading`, `time`, `partial`, `IOScanListThread`, `getRecord`,
     `addHook`, `AprilTagDetector`, `decodeAT`, and `EpicsImageSource`.

   Keep `CaptureCamera` and fixed user-script support unchanged in this cleanup.
   Do not move or alter the fixed user scripts until the separate runtime
   user-script feature is implemented.

4. Split the database.

   Create `iocBoot/ioc12idUR/tag_tracking.db` containing the retained tracking,
   tag-result, and overlay-use records. Use `@tag_tracking` links in this file.

   Remove all tag records from `robot_operations.db`, leaving camera capture and
   fixed user-script records.

   Preserve the `TagOverlayUse` closed-loop link from `Py:TagFound` to
   `TagOver:1:Use` so the overlay is visible only while exactly one tag is
   detected.

5. Load the new database during IOC startup.

   In `iocBoot/ioc12idUR/st.cmd.Linux`, add:

   ```iocsh
   dbLoadRecords("tag_tracking.db", "P=$(PREFIX)")
   ```

   Load it after the areaDetector configuration and alongside the other Python
   support databases. The overlay records loaded by `ADURL.iocsh` must exist
   before `TagOverlayUse` can forward values to `TagOver:1:Use`.

6. Clean up `12idURApp/op/ui/12id_robot.ui`.

   Remove widgets and labels for `DetectTagRetries` and `DetectTagAttempt`.

   Keep the tag-found, tag-ID, and tag-center diagnostic displays. Keep the
   `TrackTag` control, but relabel it from `Detect Tag` to `Track Tag` or
   `AprilTag Tracking` so the UI describes continuous enable/disable behavior
   accurately.

   Do not change unrelated UI layout or controls.

7. Add focused no-hardware tests in the IOC repository.

   Tests should avoid importing the full robot-control stack and should stub
   pyDevSup, the image source, and AprilTag detector dependencies as needed.
   Cover:

   - Initial and reset tag-result values.
   - Detection conversion to tag ID, center, and clamped bounding box.
   - Reset when zero or multiple tags are returned.
   - Result-record support reads and I/O Intr registration.
   - Idempotent tracker start and stop behavior.
   - Tracker shutdown resets the result and notifies scan clients.
   - Error and frame-timeout paths reset the result without killing the IOC.

   Do not construct `UR3`, `UR5`, or `common.robUR.UR`; no test may contact
   robot hardware, a dashboard, or a camera.

## Validation

1. Run syntax parsing on changed Python modules without importing the complete
   robot stack.
2. Run focused tag-tracking tests only.
3. Run any available IOC Python lint or format checks only on changed files.
4. Run `git diff --check`.
5. Rebuild and start the IOC only under an approved IOC test procedure. Do not
   connect to a robot as part of this cleanup.
6. In an approved non-hardware IOC session with a valid image source, confirm:

   - `Py:TrackTag` starts and stops tracking.
   - A single visible tag updates diagnostics and the overlay.
   - Zero or multiple tags clears `Py:TagFound` and hides the overlay.
   - Removed `DetectTag`, retry, and attempt PVs are absent.

## Non-Goals

- No changes to `UR_12idb` camera or robot APIs.
- No changes to robot geometry, TCP, payload, motion paths, or calibration.
- No arbitrary motion through IOC records.
- No runtime user-script implementation in this change.
- No conversion of continuous tracking into a robot-worker operation. It is an
  independent display/diagnostic consumer of the areaDetector image stream.
