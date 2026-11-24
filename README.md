# PNU TECH WEEK 2026 — Autonomous Search & Rescue Robot

A **Webots simulation project** for autonomous indoor search and rescue using a TurtleBot3 Burger. The robot explores an apartment, builds an occupancy map from LiDAR readings, searches for target objects with its camera, and navigates through a multi-stage mission.

> **Project status:** This repository contains the implementation, not a benchmark report. The included archive has not been runtime-tested as part of this README preparation. See [Setup notes](#setup-notes) for one configuration change required before launching it.

## Mission

The default mission is configured to:

1. Scan the environment from the starting pose.
2. Explore unknown areas and search for **two red apples**.
3. Approach and verify each detected target at close range.
4. Navigate to the designated destination / safe zone.
5. Return to the starting location and stop.

The controller also contains an alternative **football / sports ball** target configuration, but the red-apple mission is the default.

## Approach and features

| Component | Implementation |
| --- | --- |
| **Localization** | Wheel-encoder odometry, compass heading, and LiDAR-based scan matching |
| **Mapping** | Log-odds occupancy grid with 0.05 m resolution and obstacle inflation |
| **Exploration** | Frontier search, periodic 360° scanning, and camera-coverage tracking |
| **Global planning** | A* over the occupancy grid |
| **Local navigation** | Dynamic Window Approach (DWA) trajectory evaluation and obstacle-aware path following |
| **Target perception** | Camera-based red-object detection and close-range verification; optional YOLO-based object detection |
| **Semantic exploration** | YOLO object observations contribute to a semantic map and bias frontier selection toward target-relevant areas |
| **Recovery** | Front-obstacle checks, movement watchdog, backing off, and replanning |
| **Mission control** | Finite-state workflow: `SCAN → EXPLORE → APPROACH → TO_DEST → HOME → DONE` |

The exact state transitions depend on target detections and navigation progress. The mission state machine is implemented in [`controllers/alien/alien.py`](controllers/alien/alien.py).

## Repository layout

```text
PNU-Techweek-AMRHackathon/
├── controllers/
│   └── alien/
│       ├── alien.py          # Robot controller, mapping, planning, perception, mission FSM
│       └── semantic.py       # Semantic object mapping and optional frontier chooser
├── worlds/
│   └── apartment.wbt         # Webots apartment simulation
├── protos/
│   ├── RedApple.proto
│   ├── GreenApple.proto
│   ├── OrangeApple.proto
│   ├── PurpleApple.proto
│   └── icons/Apple.png
└── models/
    └── YOLO/
        ├── yolo11n.pt       # YOLO weights included in the supplied archive
        └── .gitkeep
```

Temporary or partially downloaded `.part` weight files, macOS metadata, and Webots `.wbproj` state files are omitted from this overview and should normally be excluded from Git.

## Requirements

- **Webots R2025a** (the world and PROTO files identify this version)
- Python compatible with the Webots Python controller environment
- `numpy`
- `opencv-python` (`cv2`)
- For optional YOLO inference: `torch` and `ultralytics`
- For optional TypeSafe / Jev-assisted frontier selection: `typesafe_sdk` and a `TYPESAFE_API_KEY` environment variable

Install Python packages into the **same Python interpreter configured in Webots**:

```bash
python -m pip install numpy opencv-python
# Optional semantic-object detection:
python -m pip install torch ultralytics
```

The `controller` Python module comes from Webots; it is not installed from ordinary PyPI. See [Webots documentation](https://cyberbotics.com/doc/guide/using-python).

## Setup notes

**Important — controller name mismatch in the supplied archive:** `worlds/apartment.wbt` currently specifies:

```webots
controller "tb3_sar"
```

but the Python controller is stored under `controllers/alien/alien.py`. Webots normally expects the selected controller name to match a controller directory and main script. Before running, choose **one** of the following:

- **Option A:** Rename `controllers/alien/` to `controllers/tb3_sar/` and `alien.py` to `tb3_sar.py`; keep `semantic.py` next to the renamed script.
- **Option B:** Keep the existing folder and script, and change the robot's controller setting in `worlds/apartment.wbt` from `tb3_sar` to `alien`.

**Option B is the smaller change.** The instructions below assume you use Option B.

### Run in Webots

1. Install and open Webots R2025a.
2. Clone or download this repository, preserving the `controllers/`, `worlds/`, and `protos/` folders.
3. Apply the controller name fix above.
4. Configure Webots to use the Python environment containing the required packages.
5. Open `worlds/apartment.wbt` in Webots.
6. Start the simulation. The controller should begin in the `SCAN` state and then proceed through exploration and target search.
7. Inspect the robot displays and controller output. The controller writes `sar.log` to its controller directory during execution.

This sequence is based on code inspection; a successful run on your machine has not been verified here.

## Configuration

Mission settings are defined near the top of `controllers/alien/alien.py`:

```python
START = (-0.3, -7.5, math.pi)
DEST = (-4.94, -7.33)
TARGET_NAME = "red apple"
TARGETS = {
    "red apple": dict(kind="color", diameter=0.10, count=2),
    # An alternative "football" target is also defined in the source.
}
SEMANTIC = True
SHOW_DEBUG = True
SAVE_DEBUG_FILES = False
```

The start position, destination, and map bounds are specific to the provided apartment scene. Update them when using another world.

### YOLO and semantic exploration

- The default red-apple target uses **color-based detection**, so YOLO is not required for the primary target detector.
- The controller can also load YOLO to recognize indoor objects and build a semantic map used when choosing exploration frontiers.
- It first looks for `models/YOLO/yolo_world_apartment.pt` and otherwise falls back to `models/YOLO/yolo11n.pt`. The archive includes `yolo11n.pt`, **but not** `yolo_world_apartment.pt` or the referenced `make_world_model.py` generator.
- Semantic scores combine estimated travel distance with context-dependent object priors. The heuristic does not guarantee that targets are located near the observed objects.
- If YOLO initialization fails, the main controller has an error-handling fallback; semantic object detection may be unavailable.
- Optional TypeSafe / Jev selection is only attempted when `TYPESAFE_API_KEY` is present and the relevant SDK can initialize. Otherwise, local scoring is used.

### Debugging

- `SHOW_DEBUG` enables camera/map/local-costmap visualization.
- Press **C** in the active Webots view to toggle the costmap overlay, where supported.
- Set `SAVE_DEBUG_FILES = True` to write additional debug images and data.
- Generated `sar.log`, `*_live.jpg`, `spotted.jpg`, and other diagnostic outputs should generally not be committed.

## Architecture

```text
Wheel encoders + compass ──> Pose estimation ──────────┐
LiDAR ────────────────────> Occupancy grid / scan match ├──> A* / frontier planning
Camera ───────────────────> Target detector ───────────┤          │
      └── optional YOLO ──> SemanticMap / priors ──────┘          ▼
                                                     DWA + safety / recovery
                                                               │
                                                               ▼
                                                       Differential-drive motors

Mission FSM: SCAN → EXPLORE ↔ APPROACH → TO_DEST → HOME → DONE
```

## Limitations and reproducibility

- The supplied world initially references `tb3_sar`, while the controller directory is `alien`; fix this before launching.
- The archive contains no automated tests, experiment logs, or reproducible success-rate / timing benchmark. Results should be measured rather than inferred from the code.
- The implementation depends on fixed scene coordinates and assumptions about the TurtleBot3 devices and camera/LiDAR configuration.
- Semantic search depends on YOLO availability and heuristic class priors, not a trained target-location probability model.
- The main controller is a large single Python module; further modularization would help long-term maintenance.
- The controller plans for a simulated mission. It does **not** establish validated real-world search-and-rescue performance or physical object pickup/delivery.

## Credits and asset licensing

This repository includes Webots-related fruit PROTO assets whose headers credit **Cyberbotics Ltd.** and indicate licensing for use with Webots. **Retain those headers and respect their terms**; see the [Webots asset license](https://cyberbotics.com/webots_assets_license).

Before publishing, identify and credit any workshop-provided starter world, robot configuration, or other third-party material, and confirm you are permitted to redistribute it. The authorship of individual controller files was not established by inspection of the archive. Do not apply a blanket open-source license to third-party assets without checking their terms.

## Demo and results

Add your own verified simulation screenshots, demonstration video, and performance results here before using the repository as a portfolio showcase. Suggested measurements include target-detection success, mission-completion rate, collision/recovery counts, and simulation time across repeated runs.
