# Cube Tutor

A camera-grounded Rubik's Cube tutor on the **Alif Ensemble E8**. Start with
a scrambled cube: the board finds a visible face, reads its nine sticker
colours, and displays feedback beside a live, mirrored preview. The latest
source remembers stable face reads and adds rudimentary white-cross next-move
guidance after six faces are recorded. No solved-cube starting position or
cloud connection is required.

## How the code works

1. **Capture:** the OV5675 camera supplies RGB frames. The camera adapter
   rotates/crops the view and mirrors it for natural interaction.
2. **Locate:** a locally fine-tuned YOLOv8 cube-face detector receives a
   white-balanced 192×192 RGB image. INT8 conversion and Arm Vela compilation
   target the Ethos-U85; TensorFlow Lite Micro runs the deployed detector.
3. **Read:** C code samples a 3×3 grid inside each selected face box, applies
   the saved white-balance profile, and classifies HSV colours on the M55.
   Uncertain colours are rejected rather than silently filled in.
4. **Remember and guide:** LVGL shows simple bounding boxes, colour letters
   and visible-sticker observations. Three identical complete reads are
   required before remembering a face. Once six faces are available, a C
   planner searches up to five turns to home another white-cross edge while
   preserving already-home edges. Planning assumes consistent scan orientation;
   physical move verification remains future work. The 3D wireframe is removed.

The camera, inference, colour reader and feedback all run on the board. A
laptop was used for training, flashing and inspecting intermediate buffers.

## Source

Public project repository: [cosmicz/alif-cube-tutor](https://github.com/cosmicz/alif-cube-tutor).
It contains the full firmware source, upstream history and project README.
This submission matches public revision
[`5f4ea5d9`](https://github.com/cosmicz/alif-cube-tutor/tree/5f4ea5d9750d2bebfd37afc69cb1a61ac00aed70),
including the polished README, model details and standalone host tests.

[`cube-tutor.patch`](cube-tutor.patch) contains the firmware changes,
documentation and host regression tests, relative to [npow/alif-mlek-yolo at
`bbc9f4f`](https://github.com/npow/alif-mlek-yolo/tree/bbc9f4f164b89b7a29ef84bd99f0d56917dc7225).
It belongs to that **separate MLEK application**, not the CMSIS pico-faces
starter. This submission leaves the starter application unchanged.

To inspect the complete Cube Tutor source:

```sh
git clone https://github.com/npow/alif-mlek-yolo.git cube-tutor
cd cube-tutor
git checkout bbc9f4f164b89b7a29ef84bd99f0d56917dc7225
git apply /absolute/path/to/submissions/cube-tutor/cube-tutor.patch
bash tests/run_cube_tests.sh
# Optional ASan + UBSan; no board, weights or submodules needed:
SANITIZE=1 bash tests/run_cube_tests.sh
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
- `source/app/use_case/alif_object_detection/include/cube_face_memory.h`:
  stable reads and six-face memory.
- `source/app/use_case/alif_object_detection/include/cube_cross_planner.h`:
  bounded white-cross next-move search.
- `docs/cube-tutor-status.md`: exact artifact identities, build configuration
  and remaining validation work.

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

Latest flashed runtime source: `c7e9187bcecd6ce59198e365b76c970607a22bc3`.
AXF SHA-256:
`da4f21a386af15e526f72fbecbeeb040b8d21b07c998fc02fad4b61ace25afde`.
Public revision `5f4ea5d9` retains that runtime source and adds documentation
and tests. The latest six-face planner has host-test and programming evidence;
this update does not claim a completed on-board six-face acceptance rehearsal.

## Model sizes and execution

| Component | Size | Execution |
| --- | --- | --- |
| YOLOv8n-style detector | 3,138,003 parameters; 3,033,904-byte Vela model (2.89 MiB) | INT8 neural graph on Ethos-U85-256 through TFLM |
| Colour reader | C code, saved white balance and HSV rules | Cortex-M55 CPU |
| Face memory, planner, LVGL | C/C++ | Cortex-M55 CPU |

The detector uses float32 `[1,3,192,192]` input and `[1,5,756]` output.
Quantize/dequantize boundaries, preprocessing and box decoding remain on the
CPU. Vela's 771.5 KiB model SRAM and 11.87 ms inference time are compiler
estimates, not total application RAM or measured end-to-end FPS.

## Verification

The patch reconstructs public revision `5f4ea5d9750d2bebfd37afc69cb1a61ac00aed70`
exactly from the pinned baseline, including tree
`b1d7188a2e9c3b0401debb91daeb51106319fe78`. Applying with `git apply --index`
and checking `git write-tree` reproduces that identity.

All 14 standalone host executions pass normally and with ASan/UBSan, including
four display rotations, stable face memory, asymmetric mirror/turn checks
and planner progress over 20 deterministic scrambles. This is not the upstream
MLEK FVP/model integration suite, which needs additional resources.

One test explicitly characterizes a **known red/orange defect**, retaining
the original reference labels: expected `BYRGRYRRR`, shipped `BYRGOYRRR`.
A passing characterization test does not mean that colour error is fixed.

## ExecuTorch and current limits

We also built and exercised a **separate ExecuTorch sticker-classification
prototype on the E8**, but its colour accuracy was not accepted. The final
demonstrated YOLO pipeline uses **TFLM, not ExecuTorch**. This patch does not
include that separate ExecuTorch runner or port Cube Tutor to the CMSIS
pico-faces application.

The current tutor closes the perception-to-feedback loop and includes a
rudimentary cross planner, not a complete solve-and-verify system. Cube-state
legality, stale face reads after a turn, worst-case planning latency and visual
move verification still need work. Multi-face hardware acceptance is not
claimed. Colour boundaries are tuned to the demonstration table and are not
lighting-independent. No firmware is changed by this submission update.

Built with C/C++, Python, PyTorch, Ultralytics YOLOv8, TensorFlow Lite Micro,
Arm Vela, Ethos-U85 and LVGL; ExecuTorch was explored in the separate runner.
