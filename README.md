![logo](go2_survey_logo.png)

# go2-survey

go2-survey walks a Unitree Go2 quadruped robot along surveyed routes using an RTK
GPS and photographs the ground ahead as it goes. Every photograph is saved with
the robot's position, to centimeter accuracy, and its bearing at the time of capture.

The software collected the ground images of the
[MPG Ranch Multimodal Survey](https://github.com/mosscoder/multimodal_survey)
dataset. Tag `v0.36.0` marks the version of the last survey run in that
dataset.

## Features

- **Line surveys.** The robot walks straight legs between corner waypoints,
turns in place at each corner, and takes a photograph at a fixed interval of
travel.
- **Route planning.** `make-waypoints` lays out the legs of a lawnmower route
from a polygon or a point and draws a preview of it.
- **Geotagging.** Each photograph gets a JSON sidecar and an EXIF header with
its RTK position, reported accuracy, capture time and bearing. Bearings are
computed from the RTK track after the run.
- **Waypoint navigation.** A mission without a capture strategy walks the robot
from point to point.
- **Resuming.** `run --resume` continues an interrupted survey from its last
photograph, into the same run folder.
- **Monitoring.** A run pauses when the GPS fix degrades or the corrections go
stale, and logs battery state and motor temperatures throughout.

It runs on a companion computer on the robot (we used an NVIDIA Jetson Orin
Nano) with a u-blox ZED-F9P or ZED-F9R receiver and an NTRIP correction
service.

## Quick start

```bash
pip install -e .                          # installs go2-survey and its dependencies
go2-survey list                           # the missions under dev/missions/
go2-survey run 00_parking_lot --dry-run   # load the configuration and waypoints, skip hardware
go2-survey run 00_parking_lot             # walk the mission
```

[`docs/install.md`](docs/install.md) covers installation, configuration,
robot discovery and troubleshooting.
[`docs/field-walkthrough.md`](docs/field-walkthrough.md) is the operator
procedure for a line survey in the field.

## Commands

| Command | What it does |
|---|---|
| `go2-survey run <mission>` | Runs a mission, given by name under `dev/missions/` or by the path of any folder with a `mission.toml` |
| `go2-survey run <mission> --dry-run` | Loads the configuration and waypoints and logs the plan without touching hardware |
| `go2-survey run --resume <run_dir>` | Resumes an interrupted line survey from the last photograph of that run |
| `go2-survey list` | Lists the missions under `dev/missions/` |
| `go2-survey discover-ip` | Finds the robot's address on the local network |
| `go2-survey make-waypoints <file> --leg-space-m <m> --epsg <code>` | Generates the corner waypoints of a lawnmower route from a polygon or a point (GeoJSON or KML), and a preview image |
| `go2-survey plot-waypoints <waypoints.geojson>` | Draws `mission_layout.png` for a waypoint file |
| `go2-survey finalize-bearings <run_dir>` | Recomputes the bearing of every photograph in a run from the RTK track. A line survey does this itself when it ends |

`go2-survey <command> --help` lists every option.

## A mission

A mission is a folder with two files. `mission.toml` specifies the robot
survey details, and `waypoints.geojson` gives the route.

### `mission.toml`

| Section | What it sets |
|---|---|
| Top level | `name`, `description`, `mode` (`nav` for a survey, or one of the static camera, static geotag and probe tests), and `output_dir` |
| `[gps]` | Serial port and baud rate of the receiver |
| `[ntrip]` | Host and port of the correction service, and its `mountpoints`, `usernames` and `passwords` as parallel lists tried in order. `on_unavailable` chooses between continuing without corrections and aborting |
| `[robot]` | Connection mode, and optionally the robot's address or serial number |
| `[navigation]` | Speed, turn rate, arrival tolerance, and the limits on GPS quality that pause a run (fix type, horizontal accuracy, age of corrections) |
| `[capture]` | The capture `strategy` (`none`, `frame_only`, `waypoint_forward` or `line_survey`), the interval between photographs, and how the robot steers along a leg |
| `[probe]` | Duration and sample rate of the GPS probe test |

A line survey like those of the dataset looks like this.

```toml
name        = "site_1_strip_1"
description = "Lawnmower line survey of site 1, strip 1"
mode        = "nav"

[ntrip]
host        = "caster.emlid.com"
port        = 2101
mountpoints = ["MOUNTPOINT"]
usernames   = ["USERNAME"]
passwords   = ["PASSWORD"]

[navigation]
max_velocity   = 1.0             # m/s
max_hacc       = 0.10            # m, pause when horizontal accuracy is worse than this
max_rtcm_age_s = 12.0            # s, pause when corrections are older than this

[capture]
strategy           = "line_survey"
capture_interval_m = 1.0            # one photograph per meter of travel
leg_steering       = "cross_track"  # follow the leg line on the GPS course
```

Every field except `name` is optional. Unset fields take the defaults in
`src/go2_survey/config.py`, and `dev/missions/_template/mission.toml` documents
each one. An environment variable overrides the file (`GPS_PORT`,
`EMLID_USERNAMES`, `ROBOT_IP`, `GO2_SURVEY_OUTPUT_DIR` and others), so the
order of precedence is defaults, then `mission.toml`, then the environment.

### `waypoints.geojson`

A GeoJSON `FeatureCollection` of `Point` features, with coordinates as
`[longitude, latitude]`. For a line survey the points are the corners of the
legs, in walking order, which is what `make-waypoints` writes. A
`properties.name` on each feature is optional and appears in the log as the
robot reaches it.

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "name": "wp_001" },
      "geometry": { "type": "Point", "coordinates": [-113.99780057, 46.86164631] }
    }
  ]
}
```

To start a new mission, copy the template and edit the two files.

```bash
cp -r dev/missions/_template dev/missions/mission_NAME
go2-survey run mission_NAME --dry-run
```

## Run outputs

Each run gets its own folder, `<mission>/runs/<name>_<timestamp>/`, or
`<output_dir>/<mission>/<name>_<timestamp>/` when the mission sets
`output_dir`.

| File | Contents |
|---|---|
| `main.log` | The narrative of the run, also printed to the console |
| `gps.log` | GPS and RTK telemetry once a second, one JSON object per line |
| `imu.log` | The robot's state stream |
| `battery.log`, `motor.log` | Battery state and motor temperatures |
| `captures/` | One JPEG per photograph and a JSON sidecar beside it |

## Project Layout

```
go2-survey/
├── src/go2_survey/          the installable package
│   ├── cli.py, mission_runner.py, config.py        commands, the mission loop, settings
│   ├── logging_utils.py                            log formatting
│   ├── gps.py, ntrip.py                            receiver and corrections
│   ├── robot.py, discovery.py                      robot connection
│   ├── navigator.py, geometry.py                   steering
│   ├── waypoints.py, waypoint_gen.py               loading and generating routes
│   ├── capture.py, vision/                         photographs and geotagging
│   ├── bearings.py, resume.py                      bearings after a run, resuming a run
│   └── battery.py, motors.py, health.py, probes.py monitoring and diagnostics
├── dev/
│   ├── missions/            test missions and the template
│   ├── changelog.md         dated log of every version
│   ├── scripts/             one-off maintenance scripts
│   └── archive/             retired scripts, missions and logs
├── docs/
│   ├── install.md           installation and first run
│   ├── field-walkthrough.md operator procedure for a line survey
│   ├── gps/                 receiver reference documents and configuration notes
│   ├── webrtc/              notes on the robot's WebRTC protocol
│   └── jetson_orin_nano/    companion computer setup
└── pyproject.toml           dependencies and the entry point
```

## Project history

[`dev/changelog.md`](dev/changelog.md) is the dated log of every change, from
the first autonomous two-waypoint run on 2026-04-01 to version 0.36.0 on
2026-06-18.

## License

MIT, see [`LICENSE`](LICENSE).
