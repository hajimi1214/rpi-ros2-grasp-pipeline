# rpi-ros2-grasp-pipeline

Raspberry Pi edge inference feeding a simulated robot arm grasp pipeline (ROS2 + ONNX).

<!-- TODO: replace with a 5-second demo gif. Put the file at docs/demo.gif -->

![demo](docs/demo.gif)

## What it does

- **Perception** — a phone streams video over the network, the Raspberry Pi pulls the stream and runs a lightweight detector locally (edge inference, on-device).
- **Decision** — the detected target pose is published as a ROS2 topic.
- **Execution** — a simulated robot arm on the PC picks up the pose and completes an approach-grasp-lift-place cycle in Gazebo.

This is a hardware-in-the-loop style setup: the real device does perception, the simulation does execution.

## Hardware

| Role | Device |
|------|--------|
| Robot-side compute | Raspberry Pi <!-- TODO: model, e.g. 4B 4GB / 5 8GB --> |
| Camera source | Phone streaming over network |
| Simulation host | PC running Gazebo |
| Monitoring terminal | Tablet / PC browser |

## Environment

- Ubuntu 22.04 (Pi) / Ubuntu <!-- TODO -->  (PC)
- ROS2 <!-- TODO: Humble / Jazzy -->
- Python 3.10
- ONNX Runtime <!-- TODO: version -->

## Results

<!-- TODO: fill in real measured numbers. These are the numbers interviewers ask about. -->

| Metric | Value |
|--------|-------|
| Inference FPS (FP32) | <!-- TODO --> |
| Inference FPS (INT8) | <!-- TODO --> |
| Single-frame latency | <!-- TODO --> ms |
| CPU usage | <!-- TODO --> % |
| Temperature | <!-- TODO --> °C |
| Grasp success rate (sim, with domain randomization) | <!-- TODO --> % |

## How to run

<!-- TODO: write these after you actually run it once end to end. Someone else must be able to reproduce it. -->

1. Install ROS2 and dependencies
2. Start the camera stream on the phone
3. Launch the perception node on the Raspberry Pi
4. Launch the simulation and the grasp planner on the PC

## Known issues

Honest limits of this project:

- Results are measured in simulation, not on a real arm. The sim-to-real gap is not yet characterized.
- Simulated joint delay and camera noise are injected to partially cover the reality gap, but friction and calibration errors are not modelled.
- <!-- TODO: anything else you know is fragile -->

## Structure

```
.
├── docs/            # demo gif, figures
├── ros2_ws/src/     # ROS2 packages (perception, bridge, sim)
└── models/          # exported ONNX models
```
