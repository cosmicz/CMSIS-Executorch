# Cube Tutor

A camera-grounded Rubik's Cube tutor on the **Alif Ensemble E8**. Start with
a scrambled cube: the board finds a visible face, reads its nine sticker
colours, and displays observation-based prompts beside a live, mirrored
preview. No solved-cube starting position or cloud connection is required.

## How the code works

1. **Capture:** the OV5675 camera supplies RGB frames. The camera adapter
   rotates/crops the view and mirrors it for natural interaction.
2. **Locate:** a locally fine-tuned YOLOv8 cube-face detector receives a
   white-balanced 192×192 RGB image. INT8 conversion and Arm Vela compilation
   target the Ethos-U85; TensorFlow Lite Micro runs the deployed detector.
3. **Read:** C code samples a 3×3 grid inside each selected face box, applies
   the saved white-balance profile, and classifies HSV colours on the M55.
   Uncertain colours are rejected rather than silently filled in.
4. **Respond:** LVGL shows the preview, face box, colour letters and a short
   prompt based on visible white-edge stickers. It does not assume a cube
   orientation or claim that an unseen edge is solved.

The camera, inference, colour reader and feedback all run on the board. A
laptop was used for training, flashing and inspecting intermediate buffers.

## Source

Public project repository: [cosmicz/alif-cube-tutor](https://github.com/cosmicz/alif-cube-tutor).
It contains the full firmware source, upstream history and project README.

[`cube-tutor.patch`](cube-tutor.patch) contains the firmware changes and host
regression tests, relative to [npow/alif-mlek-yolo at
`bbc9f4f`](https://github.com/npow/alif-mlek-yolo/tree/bbc9f4f164b89b7a29ef84bd99f0d56917dc7225).
It belongs to that **separate MLEK application**, not the CMSIS pico-faces
starter. This submission leaves the starter application unchanged.

To inspect the complete Cube Tutor source:

```sh
git clone https://github.com/npow/alif-mlek-yolo.git cube-tutor
cd cube-tutor
git checkout bbc9f4f164b89b7a29ef84bd99f0d56917dc7225
git apply /absolute/path/to/submissions/cube-tutor/cube-tutor.patch
```

Key files after applying the patch:

- `source/app/use_case/alif_object_detection/src/UseCaseHandler.cc`:
  inference, face selection, sticker reads and display updates.
- `source/app/use_case/alif_object_detection/include/CubeFaceMultiSelect.hpp`:
  confidence/area filtering and non-maximum suppression.
- `source/hal/source/components/camera/source/alif/source/cube_preview.h`:
  white balance and sticker-colour classification.
- `source/hal/source/components/camera/source/alif/source/hal_camera_alif.c`:
  camera preprocessing, saved table calibration and diagnostics.
- `source/app/use_case/alif_object_detection/include/cube_cross_hint.h`:
  allocation-free, single-face observation prompts.

The base repository documents its toolchain and dependencies. Our final
configuration uses the fitted **OV5675**, UART4, display rotation 90°,
Ethos-U85-256 and LVGL software rendering. Do not use the base repository's
default human-face model for this application. The custom trained model and
firmware binary are **not included in this source-only submission**; this
patch alone is not a ready-to-flash package. Private training captures are
also excluded. Existing source notices and dependency licences still apply.

## What we demonstrated

Live board-camera preview, full-face detection and calibrated sticker
recognition. The operator accepted the current colour calibration on
October 18, 2026. A table-specific calibration profile is compiled into
startup; subsequent button-triggered calibration is runtime-only.

We adapted the detector using an actual board-camera frame after the
webcam-trained model missed the cube. We also isolated display corruption
from camera input by inspecting both buffers. These are prototype results,
not a representative accuracy benchmark.

Demo source identity: `cae1a9bf392177d4660361ae6ff4a87ba74f1e18`.
Demo AXF SHA-256:
`42607084c72c512379d3eaef14bcb9f7f7c93ec6c84cc2de7b4dc794bdb46fc9`.

## ExecuTorch and current limits

We also built and exercised a **separate ExecuTorch sticker-classification
prototype on the E8**, but its colour accuracy was not accepted. The final
demonstrated YOLO pipeline uses **TFLM, not ExecuTorch**. This patch does not
include that separate ExecuTorch runner or port Cube Tutor to the CMSIS
pico-faces application.

The current tutor closes the perception-to-feedback loop, not the entire
solve-and-verify loop. Six-face state reconstruction, a full solver and
visual move verification are next. Multi-face decoding is implemented but
has not been acceptance-demonstrated on hardware. Colour boundaries are
tuned to the demonstration table and are not lighting-independent.

Built with C/C++, Python, PyTorch, Ultralytics YOLOv8, TensorFlow Lite Micro,
Arm Vela, Ethos-U85 and LVGL; ExecuTorch was explored in the separate runner.
