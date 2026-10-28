# skydock2 developer guide

Autonomous agricultural spray UAV. A Raspberry Pi 5 companion computer runs a
mission finite-state machine that scans a field with a Hailo AI camera, clusters
weed detections, then flies to and "sprays" each weed. Flight is executed by an
ArduPilot flight controller; the Pi talks to it over MAVLink (`pymavlink`).

The same code runs against ArduPilot **SITL** for development, with a simulated
camera (`sim_ai.py`) that projects known weed locations into synthetic detections.
Every run is logged to `missions/NNNN/`, and a Flask web app (`tools/log_server/`)
replays, analyses and compares missions from both sim and real flights.

## Stack

| Layer | What |
|-------|------|
| Flight controller | ArduPilot (Copter), MAVLink over serial (real) or UDP (sim) |
| Companion computer | Raspberry Pi 5 |
| AI camera | Hailo-8 on RPi5 (`hailo-rpi5-examples/`, vendored); sim equivalent in `sim_ai.py` |
| Language | Python 3 (numpy, pymavlink, pyserial, SQLAlchemy; Flask for the log viewer) |
| Persistence | Per-mission SQLite DB + `mission.jsonl` event log |

## Repository layout

| Path | Purpose |
|------|---------|
| [main.py](../main.py) | Entry point: connect telemetry, load mission, run the FSM loop |
| [fsm.py](../fsm.py) | `StateMachine`: dispatches per-tick to a state handler |
| [states/](../states/) | One module per state: `override`, `scan`, `goto`, `homing`, `spray`, `rtl` (+ `enum`, `shared_data`) |
| [telemetry.py](../telemetry.py) | MAVLink connection, background reader thread, movement commands |
| [drone_state.py](../drone_state.py) | `DroneStateForHoming` telemetry snapshot + attitude/position interpolation |
| [ai_class.py](../ai_class.py) | `Detection`, `Frame`, and the thread-safe `ai_storage_singleton` |
| [ai_callback.py](../ai_callback.py) | Real camera: Hailo GStreamer pipeline callback (symlink to `hailo-rpi5-examples/basic_pipelines/detection_simple.py`) |
| [sim_ai.py](../sim_ai.py) | Simulated camera: projects weed GPS to pixel detections |
| [server.py](../server.py) | Live camera feed with detection overlay, served on port 8080 during a mission |
| [preflight_tests.py](../preflight_tests.py) | Pre-flight checks to run on the Pi before each flight session |
| [utils.py](../utils.py) | Pure geometry: haversine, pixel <-> NED <-> lat/lon projection |
| [DB.py](../DB.py) / [DB_abstraction.py](../DB_abstraction.py) | SQLAlchemy models and the high-level DB API |
| [mission_gen.py](../mission_gen.py) | Build a lawnmower scan path / mission JSON from weed locations |
| [mission_logging.py](../mission_logging.py) | Allocate `missions/NNNN/`, write `mission.jsonl` |
| [sitl.py](../sitl.py) | Launch and configure ArduPilot SITL |
| [constants.py](../constants.py) | Mission tunables (heights, speeds, thresholds, timeouts) |
| [tools/log_server/](../tools/log_server/) | Flask mission viewer: dashboard, log analysis, compare, training-data review, weed marking |
| [tools/](../tools/) | Offline analysis (`fsm_analyze`, `make_video`, `sim_accuracy`, `tune_clustering`), camera-mount calibration (`calibration_orbit`, `estimate_camera_mount`, `camera_orientation_from_flow`), and Pi sync (`sync_rpi_logs.sh`, `push_real_missions.sh`) |
| [sim_data/](../sim_data/) / [real_missions/](../real_missions/) | Mission files (weed locations + scan path) for sim and real flights |
| [tests/](../tests/) | Pytest suite (runs with no hardware/SITL) |
| [tests_sitl/](../tests_sitl/) | Integration tests against a headless ArduPilot SITL |
| [docs/]() | Architecture, testing guide, SITL guide, camera specs, code review, refactor plan |
| `archive/`, `hailo-rpi5-examples/` | Dead/experimental code and vendored Hailo examples (out of scope) |

See [docs/architecture.md](architecture.md) for diagrams and data flow.

## Running in simulation

Requires ArduPilot SITL installed at `~/ardupilot/` (`sim_vehicle.py` via
`~/venv-ardupilot`) and `xterm`. If SITL is already listening on UDP 14550 it is
reused. See [docs/skydock_sitl_guide.md](skydock_sitl_guide.md) for full setup.

```bash
pip install -r requirements.txt
python main.py --sim                       # first mission file in sim_data/
python main.py --sim sim_data/0010.json    # a specific mission
python main.py --sim --speed 5             # 5x SITL speedup
```

`--sim` starts SITL, arms and takes off automatically, runs the simulated camera,
and drives the FSM. Mission artifacts are written to `missions/NNNN/`.

## Running on the real UAV

On the Pi, with the flight controller connected over USB and the Hailo camera
pipeline available:

```bash
pip install -r requirements.txt
python preflight_tests.py            # pre-flight checks (--no-ai skips the camera)
python main.py                       # prompts for a mission file, then takeoff
```

`main.py` lists existing `real_missions/*.json` or records a new one by walking
the UAV over each weed and capturing GPS. While it runs, the live camera feed is
at `http://<pi>:8080/`.

Autonomy is gated by a 3-position RC switch on channel 16: down = autonomy on,
middle = autonomy off, up = force homing. Battery / failsafe handling is delegated
to the ArduPilot flight controller: when the FC switches to RTL, the FSM detects
`mode == "RTL"` and stops.

Pull mission logs from the Pi with `tools/sync_rpi_logs.sh` (or the log server's
**Sync RPi** button).

## Mission artifacts

Each run creates `missions/NNNN/`:

| File | Contents |
|------|----------|
| `droneDB.db` | SQLite: waypoints, weeds, drone_states, detections |
| `mission.jsonl` | One JSON object per line: state transitions, telemetry samples, move commands, detections, spray events |
| `database_snapshot.json` | JSON dump of the DB written by `backup_and_clear` at mission load |

Review them with the log server, then open <http://localhost:5050/>:

```bash
python tools/log_server/app.py
```

Command-line alternatives: `tools/fsm_analyze.py`, `tools/sim_accuracy.py` and
`tools/make_video.py`, each taking a mission's `mission.jsonl` or directory. See
[tools/log_server/README.md](../tools/log_server/README.md) for the port setting and the
per-mission SQLite index.

## Tests

The flight-stack suite runs on a laptop with no UAV and no SITL (telemetry is mocked,
the DB uses temp SQLite):

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
python -m pytest                                  # flight stack (tests/)
pip install flask && python -m pytest tools/log_server/tests/   # log viewer
python -m pytest tests_sitl/                      # SITL integration (~2 min)
```

The log viewer suite includes Playwright browser tests. They need
`python -m playwright install chromium` once and are skipped if Chromium is missing.
The SITL suite boots a headless SITL, or reuses one on UDP 14550, and is skipped if
ArduPilot isn't installed.

See [docs/testing.md](testing.md) for how the suite stubs hardware and how to
add tests for new states.

## More docs

- [docs/architecture.md](architecture.md): system diagram, FSM, homing flow, DB schema, design-vs-implementation drift
- [docs/testing.md](testing.md): test architecture and conventions
- [docs/skydock_sitl_guide.md](skydock_sitl_guide.md): SITL setup
- [docs/ai_and_cammera_spics.md](ai_and_cammera_spics.md): camera and AI hardware specs
- [docs/how_to_set_up_ssh_rfd900.md](how_to_set_up_ssh_rfd900.md): SSH over the RFD900 radio link
- [docs/code_review_2026-06-13.md](code_review_2026-06-13.md): code review findings
- [docs/refactor_plan.md](refactor_plan.md): phased follow-up work
