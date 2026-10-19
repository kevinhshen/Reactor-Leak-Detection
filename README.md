# Reactor Control Panel

**A nuclear hackathon prototype for replaying sensor data, detecting possible leaks, and inspecting the evidence in a 3D dashboard.** Built for Waterloo's Fall 2026 Nuclear Innovation Challenge with Canadian Nuclear Laboratories (CNL).

![Dashboard displaying an illustrative leak scenario](dashboard/docs/dashboard-preview.jpg)

The project connects a React control-room interface to a Python ML service. Operators can replay a recording, follow detector scores over time, inspect a suspected leak location, and compare model versions. A built-in synthetic scenario makes the interface explorable without downloading a dataset or trained weights.

## What is implemented

- **Monitor:** interactive reactor cutaway, synchronized sensor charts, timeline controls, alarm acknowledgement, and an event log.
- **Evidence:** detector probabilities, location scores, model configuration, and the observations available at the current replay time.
- **Model lab:** import, validate, select, export, and roll back model packages; continue training an existing model with separate training, validation, and test recordings.
- **ML backend:** saved-scaler preprocessing, causal sliding windows, Keras LSTM inference, consecutive-window alarm confirmation, and persistent local model/data storage.

The synthetic demo uses authored readings and scores. Real inference requires matching trained models and preprocessing artifacts, which are **not included in a fresh clone**. The project does not claim a measured accuracy or detection delay on real plants.

## Run locally

Install **Python 3.12** and **Node.js 22 or newer**. The launcher uses an installed pnpm or runs the pinned version through npx. Initial setup requires internet access. A WebGL-capable browser is needed for the 3D view.

```sh
git clone https://github.com/kevinhshen/Reactor-Leak-Detection.git
cd Reactor-Leak-Detection
python3.12 dashboard/run.py --install
python3.12 dashboard/run.py
```

Open **http://127.0.0.1:5173**, choose **Demo**, and press **Play**. Keep the terminal open; Ctrl+C stops the services started by the launcher. The backend API documentation is at **http://127.0.0.1:8000/docs**.

To check an installation without starting servers:

```sh
python3.12 dashboard/run.py --check
```

For manual installation, separate terminals, troubleshooting, and model management, see the [dashboard guide](dashboard/README.md).

## Run real model inference

1. Obtain one matching export set from the team's training run: `lstm_leak_classifier.keras`, `lstm_leak_location.keras`, `scaler.pkl`, and `config.pkl`. Version suffixes such as `_v2` on model filenames are supported.
2. In **Model lab**, import the four files or a ZIP. Confirm the actual sampling interval and the locator's output-class order against the training run. Importing validates dimensions; it cannot establish artifact provenance.
3. Select the imported version. Upload a UTF-8 CSV containing `TIME` and every sensor in the exported configuration, then replay it.

No retraining is needed to replay a compatible exported model. No weights or sensor data are downloaded automatically. Imported versions and recordings remain in the ignored `dashboard/models/` and `dashboard/data/` folders. Back them up separately.

The dashboard accepts the paired Sequential LSTM format described in the [model contract](docs/model-contract.md). A separate experimental binary CNN workflow uses a different export format and is not a drop-in replacement for this detector/locator pair; see the contract for the format differences.

## How it works

| Layer | Responsibility | Source |
| --- | --- | --- |
| React, TypeScript, Vite | Playback, model-management forms, and evidence views | [frontend](dashboard/frontend/) |
| Three.js / React Three Fiber | Selectable, simplified reactor geometry | [Reactor.tsx](dashboard/frontend/src/Reactor.tsx) |
| FastAPI | Model registry, CSV uploads, replay endpoints, and training jobs | [app.py](dashboard/backend/app.py) |
| NumPy, pandas, scikit-learn, TensorFlow | CSV validation, scaling, windowing, inference, and alarm policy | [pipeline.py](dashboard/backend/pipeline.py) |

Vite forwards `/api` requests to the backend on port 8000. Sensor columns are reordered using the saved configuration, transformed with the original scaler, and grouped into windows. Each prediction is timestamped at its final observed reading; playback reveals it only once that time is reached.

The reference V3 policy uses 12 readings per window, stride 5, and 10-second sampling: the first assessment for a recording starting at zero is at **110 seconds**, followed by assessments every **50 seconds**. An alarm requires three consecutive scores at or above 0.7 and then latches for that replay. These are reference settings, not a guaranteed response time; imported package metadata determines the actual contract.

## Design decisions

- **Separate detection and localization.** A seven-output locator assumes a leak. The backend waits for a confirmed detector alarm before exposing its first-three-clip location vote. Eight-output locators can also report “No leak.”
- **Preserve preprocessing.** Feature order, sampling interval, scaler, window size, and class meanings belong to the model contract. The service rejects incompatible inputs rather than silently resampling or refitting.
- **Keep candidate models separate.** Continued training creates a new version; the baseline and active selection are preserved until a user selects another version. Recording groups cannot cross evaluation splits.
- **Make timing explicit.** Predictions use actual CSV timestamps. An assessment cannot appear before its input window is complete.
- **Keep demonstration and inference distinguishable.** The synthetic scenario remains labeled even when a real model has been imported.

The first-three-clip location policy follows the supplied V3 workflow. If those clips precede leak onset, localization can be misleading. See the [compatibility review](dashboard/docs/v3-compatibility.md) for this limitation and other differences from notebook evaluation.

## Repository layout

| Path | Contents |
| --- | --- |
| `dashboard/frontend/` | React interface and frontend lockfile |
| `dashboard/backend/` | Runnable API, inference pipeline, dependencies, and tests |
| `dashboard/run.py` | Installation, dependency checks, and local launcher |
| `dashboard/docs/` | Screenshots, presentation guide, and unchanged V3 reference notebook |
| `dashboard/models/`, `dashboard/data/` | Local runtime storage; contents excluded from Git |
| `notebooks/` | Historical training notebook and its original dependencies |
| `tools/` | Optional Access database conversion utility |
| `docs/` | Model-format and repository guidance |

The old empty `ml_algorithm.py` and misspelled `traing_algorithm.py` were placeholders. The executable backend is in `dashboard/backend/`; training continuation is implemented in its API. Historical notebooks are retained for context rather than presented as a clean training entry point.

## Development checks

```sh
# From the repository root, after installing dependencies:
cd dashboard
.venv/bin/python -m pytest backend/tests -q
cd frontend
pnpm build
```

On Windows, use `.venv\Scripts\python.exe` for the Python command. GitHub Actions runs the backend tests and frontend build on pushes and pull requests. The tests create tiny random LSTMs to check import, inference, causality, training isolation, and export/reload behavior; those fixtures do not establish model accuracy.

## Scope and credits

This is a local decision-support prototype for **recorded simulated sensor data**. It does not connect to a plant, actuate equipment, or implement live monitoring. Its simplified PWR illustration is not a validated model of the BWRX-300 or another specific SMR.

The LSTM modeling foundation and challenge materials come from the [Waterloo IDEAs Clinic Nuclear Innovation Challenge](https://github.com/IdeasClinicUWaterloo/F26-NuclearIC). Operational training data are associated with [NuclearPowerPlantAccidentData (NPPAD)](https://github.com/thu-inet/NuclearPowerPlantAccidentData), a PCTRAN simulation dataset. This is a portfolio fork of the [team repository](https://github.com/magi-wuu/CNL-Hackathon), preserving its contributors and history. The team project adds the dashboard, model-service integration, replay, and version-management workflow. Consult the original sources for their terms; no repository-wide license is declared here.
