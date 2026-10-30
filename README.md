# Eye-Galvo Control

English | [中文](README.zh-CN.md)

[![Python Verification](https://github.com/LiTongShuo-edu/eye-galvo-control/actions/workflows/python.yml/badge.svg)](https://github.com/LiTongShuo-edu/eye-galvo-control/actions/workflows/python.yml)

An experimental eye-target localization and galvanometer control project. An Intel RealSense depth camera supplies color and depth data; MediaPipe Face Landmarker locates both eye centers; the target is converted into a 3D position in camera coordinates; and a RIGOL DP832 supplies galvanometer control voltages. The real-device functional chain has been validated within the scope described in the [hardware validation record](./docs/hardware-validation.md).

## Workflow

```text
Color image + aligned depth
        |
MediaPipe eye-center detection
        |
Pixel/depth back-projection to a 3D target
        |
Voltage inversion using calibrated spatial model, or uncalibrated angle mapping
        |
PyVISA -> RIGOL DP832 -> galvanometer
```

## Modules

| Module | Purpose |
| --- | --- |
| [`eye_galvo/geometry.py`](./eye_galvo/geometry.py) | Depth pixel back-projection, target angles, basic voltage mapping |
| [`eye_galvo/calibration.py`](./eye_galvo/calibration.py) | Nine-point spot sampling on near, working, and far planes; model export |
| [`eye_galvo/tracking.py`](./eye_galvo/tracking.py) | Real-time tracking, model validation, voltage inversion and boundary protection |
| [`eye_galvo/instrument.py`](./eye_galvo/instrument.py) | DP832 VISA communication, output updates, centering and shutdown |
| [`eye_galvo/cli.py`](./eye_galvo/cli.py) | `track`, `link`, `scan`, and `calibrate` commands |

## Three-plane calibration

`calibrate` samples a `3 x 3` voltage grid of `4.0 V`, `5.0 V`, and `6.0 V` on near, working, and far planes, collecting 27 valid 3D spot positions. It fits a ray to the three points for each voltage pair and produces `galvo_calibration.json` with nine rays, residuals, and voltage limits.

During linked tracking:

- A valid calibration file enables voltage inversion for targets within the calibrated volume.
- If no file is found, a bounded angle-based mapping remains available for basic checks.
- A corrupt, incompatible, or out-of-bounds calibration file prevents linked output from starting.
- A target outside the permitted volume holds the current output instead of extrapolating without bounds.

See the [spatial calibration model](./docs/calibration-model.md) for the mathematics.

## Installation

Python 3.12 in a virtual environment is recommended:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Download the [official MediaPipe Face Landmarker model](https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task) and save it as `face_landmarker.task` in the repository root. It is not distributed with the source.

## Usage

```powershell
# Vision tracking without an instrument
.\.venv\Scripts\python.exe -m eye_galvo track --model .\face_landmarker.task
# Scan VISA devices
.\.venv\Scripts\python.exe -m eye_galvo scan
# Collect calibration data (default output: galvo_calibration.json)
.\.venv\Scripts\python.exe -m eye_galvo calibrate --resource "YOUR_VISA_RESOURCE"
# Linked tracking with a valid calibration and DP832
.\.venv\Scripts\python.exe -m eye_galvo link --resource "YOUR_VISA_RESOURCE" --model .\face_landmarker.task
```

The instrument resource can also be supplied via `DP832_RESOURCE`.

## Tests and validation

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-test.txt
.\.venv\Scripts\python.exe -m pytest
```

Hardware-free tests cover geometry, voltage limits, three-plane data completeness, ray-model validation, configuration I/O and rejection, and safe instrument shutdown. GitHub Actions runs the same tests. Real-device functional validation is documented separately in [the hardware record](./docs/hardware-validation.md); positioning accuracy, response latency, and long-term stability have not been quantified as conclusions.

## Hardware and safety

- The experimental setup uses an Intel RealSense D435i or compatible depth camera, RIGOL DP832, galvanometer assembly, and VISA backend.
- On exit, linked operation centers the control voltages and disables the output channels. Before first connection, manually check voltage limits, wiring, and optical safety.
- The model, calibration file, real instrument resource address, and experimental images are not committed.
