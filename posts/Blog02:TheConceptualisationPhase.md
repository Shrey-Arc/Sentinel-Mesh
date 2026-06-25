---
title: "Blog 02: The Conceptualisation Phase"
description: "The Journey taking a SHAPE."
tags: ["ideas", "technology", "uav", "drone"]
type: "blog" # Set to "blog" or "log"
---

# Sentinel-Mesh — Blog 02: The Conceptualisation Phase
## *From Research Outputs to Architecture: Designing a System That Cannot Lie to Itself*

**Author:** Shrey Kumar (Sentinel-Mesh)
**Date:** 25 June 2026
**Series:** Sentinel-Mesh Development Blog — Entry 02 of N
**Previous Entry:** [Blog 01 — The Thinking Phase](https://github.com/Shrey-Arc/Sentinel-Mesh)
**Tags:** `ROS2` `System-Architecture` `Node-Graph` `Interface-Design` `SITL` `PX4` `Software-Design` `UAV-Fleet`

---

> *"An architecture is not a collection of components. It is a collection of decisions."*
> — Ruth Malan

The thinking phase produced validated research and a clearly bounded problem statement. The conceptualisation phase answers a different question: **given what we now know, what exactly are we building, and how do we build it without the pieces contradicting each other?**

This blog documents that process in full technical detail. It covers the translation of research conclusions into architectural decisions, the design of the node graph, the definition of interface contracts between all four team workstreams, the rationale behind the simulation-first strategy, and the multi-vehicle namespacing scheme that allows the software stack to scale from one simulated drone on a laptop to a physical fleet without a single line change in application code.

The conceptualisation phase is where a project either gains structural integrity or accumulates hidden debt. Every ambiguous interface left undefined here becomes a blocking conflict in implementation. Every architectural assumption left implicit here becomes an untestable claim in the final system. The discipline of this phase is the discipline of making every decision explicit, in writing, before writing any code.

---

## 1. The Translation Problem

The thinking phase ended with a body of validated analysis and an open architectural risk (the carrier drone specification). Translating that into a buildable system requires answering three questions in order:

1. **What are the system's logical layers, and what is each layer responsible for?**
2. **What are the computational units within each layer, and what are the contracts between them?**
3. **In what order do we build and validate each unit, given that some are blocked on others?**

None of these questions can be answered in isolation. The layering decision determines the interface points; the interface points determine the build order; the build order determines what can be simulated and what cannot. The conceptualisation phase is the process of working through all three simultaneously.

---

## 2. Layer Decomposition

The first architectural decision was the highest-level one: how many layers does the system have, what is each responsible for, and where are the layer boundaries?

We decomposed Sentinel-Mesh into three orthogonal layers, each with a single, well-defined responsibility.

### Layer 1: The Physical Layer

**Responsibility:** All physical phenomena — aerodynamics, structural mechanics, propulsion, power electronics, and hardware interfaces with the flight controller.

**Owned by:** Person 1 (mechanical) and Person 2 (electrical/propulsion).

**Outputs to upper layers:**
- Mass and geometry data → feeds the W^1.5 power model used by the energy monitor
- Measured power consumption at operating points → validates the analytical model
- Motor vibration characterisation → informs IMU filter configuration in PX4
- ESC telemetry (current, RPM, temperature) → provides ground truth for SOC estimation

The physical layer is the only layer that cannot be simulated with full fidelity before hardware exists. Everything above it can be. This makes it the critical-path dependency: the software stack must be designed so it can run completely without the physical layer during development, substituting high-fidelity simulation models wherever hardware is absent.

### Layer 2: The Control and Communication Layer

**Responsibility:** Real-time flight control, state estimation, inter-drone telemetry, fleet coordination, and the translation boundary between raw sensor data and mission-level decisions.

**Owned by:** Person 3 (avionics/SLAM) and the CS lead (fleet software/comms).

**Inputs from Layer 1:** Raw IMU, GPS, barometer, ESC telemetry (in hardware), or simulated equivalents (in SITL).

**Outputs to Layer 3:** Abstract drone state (`DroneState` messages), coverage map updates, obstacle data.

This layer is the most complex in terms of internal structure and contains the highest density of novel contributions: the SOC-constrained fleet scheduler, the TDMA communication manager, and the MAVLink-to-ROS2 bridge.

### Layer 3: The Cognitive Layer

**Responsibility:** AI-driven perception, object detection, terrain mapping, spatial priority filtering, and energy-adaptive inference management.

**Owned by:** CS lead.

**Inputs from Layer 2:** Abstract pose estimates, mission assignments, SOC state.

**Outputs to Layer 2:** Detected objects, obstacle maps, priority zone updates.

The cognitive layer is deliberately isolated from flight control. It does not issue motor commands, does not interact with MAVLink directly, and has no knowledge of individual drone hardware. It consumes abstract state and produces abstract perception outputs. This isolation is an explicit architectural choice: it means the AI pipeline can be developed and tested on a GPU workstation, containerised, and deployed to the Jetson Orin NX with minimal integration risk.

---

## 3. The Middleware Decision: Why ROS2 Humble

The choice of Robot Operating System 2 (ROS2) Humble as the software middleware is not incidental and warrants formal justification.

ROS1 reached End of Life on 31 May 2025. The entire robotics community has shifted to ROS2, and a modular architecture built on ROS1 today would be entering technical debt immediately. More substantively, ROS2's architecture is qualitatively different from its predecessor in ways that matter directly to this project.

The most significant change is the replacement of the centralised ROS Master with the **Data Distribution Service (DDS)** standard as the underlying communication middleware. DDS is a publish-subscribe middleware that provides a data-centric communication model, enabling efficient and reliable data exchange between distributed systems. Its decentralised architecture means that node discovery and communication do not require a single point of failure — a critical property for a distributed fleet of drones where any individual node may go offline.

ROS2 also provides formal Quality of Service (QoS) policies on topics: RELIABLE vs. BEST_EFFORT delivery, TRANSIENT_LOCAL vs. VOLATILE history, and deadline enforcement. For a fleet coordination system, the distinction matters: drone state telemetry should be BEST_EFFORT with a short history (stale state is worse than no state), while mission assignments should be RELIABLE (a lost assignment may cause a coverage gap). These distinctions are first-class concepts in ROS2 and would require application-level workarounds in ROS1.

ROS2 also enables seamless integration between high-level mission planning and low-level actuator control, facilitating tight integration of perception, planning, and action modules.

The specific choice of **Humble** (over Jazzy or Rolling) is a stability decision: Humble is an LTS release with guaranteed support through May 2027, and it is the release for which PX4's uXRCE-DDS bridge has the most mature documentation and community support.

---

## 4. The PX4-ROS2 Bridge Architecture

The most technically intricate boundary in the system is the interface between PX4 — the flight controller firmware running on the Pixhawk 6C — and the ROS2 application layer running on the companion computer (Jetson Orin NX). Getting this boundary right is critical because it is the data pathway for every flight command and every piece of vehicle state.

In the legacy ROS1 ecosystem, this boundary was managed by MAVROS: a ROS node that translated between MAVLink messages (PX4's native protocol) and ROS topics. MAVROS is still available for ROS2 but represents a conceptual mismatch: it inserts a MAVLink serialisation step where none is necessary.

The modern approach, and the one selected for Sentinel-Mesh, is **uXRCE-DDS** (Micro eXtreme Resource Constrained Environments DDS): PX4's native ROS2 integration layer. The uXRCE-DDS middleware consists of a client running on PX4 and an agent running on a companion computer, which exchange bidirectional data over a serial or UDP link. The agent acts as a proxy for the client, enabling it to publish and subscribe to topics in the global DDS data space.

The result is that PX4's internal uORB topics — `VehicleLocalPosition`, `BatteryStatus`, `VehicleAttitude`, `TrajectorySetpoint` — appear directly as ROS2 topics on the companion computer. There is no MAVLink serialisation layer; messages are exchanged as DDS data samples with nanosecond-precision timestamps. ROS2 applications need to be built in a workspace that includes the same message definitions that were used to create the uXRCE-DDS client module in the PX4 firmware — a constraint that makes version management of the `px4_msgs` package a first-class concern.

### 4.1 Multi-Vehicle Namespacing

The most consequential architectural decision in the bridge configuration is **per-vehicle namespacing**. In a single-vehicle system, PX4 topics appear at the root namespace: `/fmu/out/vehicle_local_position`, `/fmu/in/trajectory_setpoint`. In a multi-vehicle system, if all drones publish to the same topic names, the ROS2 topic graph is ambiguous — there is no way to associate a position estimate with a specific vehicle.

PX4's uXRCE-DDS client solves this by appending a unique namespace prefix to each vehicle instance. In simulation, each PX4 instance receives a unique `px4_instance` number starting from 0, and a unique ROS2 namespace prefix in the form `px4_$px4_instance` is added to all topics. The first instance does not receive this prefix for backward compatibility, but all subsequent instances (1, 2, 3...) produce namespaced topics:

```
/px4_1/fmu/out/vehicle_local_position
/px4_1/fmu/out/battery_status
/px4_1/fmu/in/trajectory_setpoint

/px4_2/fmu/out/vehicle_local_position
/px4_2/fmu/out/battery_status
/px4_2/fmu/in/trajectory_setpoint
```

This namespacing scheme is mirrored exactly in the workspace's custom `px4_bridge` package: each bridge node is instantiated with a namespace parameter corresponding to a specific drone ID. The fleet scheduler and command node consume namespaced topics and publish namespaced commands. No application-layer code contains hardcoded topic names; everything is parameterised by drone ID at launch time.

This design decision has a critical implication: **adding a new drone to the fleet requires only a new launch file entry, not a code change.** This is a concrete statement of the system's scalability — not a claim but an architectural guarantee.

---

## 5. The Workspace Structure

The ROS2 workspace (`uav_fleet_ws`) is the shared integration artefact for all four team members. Its package structure defines the team's integration boundaries as concretely as any interface specification document.

```
uav_fleet_ws/
├── src/
│   ├── fleet_interfaces/          # Interface contracts — owned by CS lead
│   │   ├── msg/
│   │   │   ├── DroneState.msg     # Per-drone state vector
│   │   │   ├── FleetStatus.msg    # Aggregated fleet state
│   │   │   ├── DockStatus.msg     # Docking station occupancy and charge state
│   │   │   └── CoverageMap.msg    # Spatial priority grid from cognitive layer
│   │   ├── srv/
│   │   │   ├── AssignMission.srv  # Scheduler → drone mission assignment
│   │   │   ├── RequestRTH.srv     # Force return-to-home for a specific drone
│   │   │   └── SetInferenceMode.srv  # GCS → cognitive layer power mode change
│   │   └── action/
│   │       └── ExecuteMission.action  # Long-running mission with feedback
│   │
│   ├── px4_bridge/                # MAVLink ↔ ROS2 translation — CS lead
│   │   └── px4_bridge_node.py     # Per-drone uXRCE-DDS subscriber/publisher
│   │
│   ├── fleet_scheduler/           # SOC-constrained CBBA — CS lead
│   │   ├── scheduler.py           # Core CBBA with SOC scoring
│   │   └── scheduler_node.py      # ROS2 node wrapper
│   │
│   ├── energy_monitor/            # SOC estimation and throttle triggers — CS lead
│   │   ├── soc_estimator.py       # Coulomb counting + voltage correction
│   │   └── energy_monitor_node.py
│   │
│   ├── comms_manager/             # ESP-NOW TDMA mesh abstraction — CS lead
│   │   └── comms_manager_node.py
│   │
│   ├── perception_server/         # AI inference pipeline — CS lead + Jetson
│   │   ├── yolo_inference.py      # TensorRT YOLOv8n wrapper
│   │   ├── spf_filter.py          # Spatial Priority Filtering
│   │   └── perception_node.py
│   │
│   ├── local_planner/             # Per-drone obstacle avoidance — CS lead
│   │   └── local_planner_node.py
│   │
│   ├── slam_bridge/               # LiDAR-Inertial SLAM interface — Person 3
│   │   └── slam_bridge_node.py    # Publishes odometry to ROS2 graph
│   │
│   ├── dock_controller/           # Docking station management — Person 2
│   │   └── dock_controller_node.py
│   │
│   └── fleet_gcs/                 # Ground Control Station UI — CS lead
│       ├── gcs_dashboard.py       # PyQt5/tkinter fleet visualisation
│       └── mission_interface.py   # Operator mission input
│
├── launch/
│   ├── sitl_single.launch.py      # Single drone SITL
│   ├── sitl_multi_2.launch.py     # Two-drone SITL (Phase 1 target)
│   ├── sitl_multi_5.launch.py     # Five-drone SITL (Phase 2 target)
│   └── hardware_deploy.launch.py  # Physical hardware deployment
│
├── config/
│   ├── fleet_params.yaml          # Fleet-wide parameters (SOC thresholds, etc.)
│   └── drone_profiles/            # Per-drone hardware configuration
│       ├── worker_default.yaml
│       └── command_drone.yaml
│
└── CMakeLists.txt / package.xml
```

The `fleet_interfaces` package is the most critical package in the workspace. It contains no executable code — only message, service, and action definitions. It is built first, and all other packages depend on it. The principle it encodes is that interface contracts are established before implementation. Message types in ROS are defined in `.msg` files with typed fields, enforcing interface contracts across the system. Services provide synchronous request-response communication, structured as pairs of `.srv` files defining request and response message types.

This package is the CS lead's highest-priority first-day deliverable, because it unblocks every other team member from writing code against a stable contract.

---

## 6. The Node Graph in Full

The node graph is the system's computational architecture made visible. Every rectangle is a running process; every arrow is a data flow. Designing it before any node is implemented forces clarity about what each component knows and does not know, and where the coordination boundaries lie.

```
╔══════════════════════════════════════════════════════════════════════╗
║                         COMMAND NODE CLUSTER                         ║
║                    (Runs on GCS laptop / companion)                  ║
║                                                                      ║
║  /fleet_scheduler      ← DroneState[]  → AssignMission.srv          ║
║  /mission_planner      ← operator intent → CoverageMap              ║
║  /comms_manager        ← ESP-NOW TDMA frames → worker telemetry      ║
║  /energy_monitor       ← BatteryStatus → throttle trigger, RTH      ║
║  /perception_server    ← CoverageMap → SPF zones                    ║
║  /fleet_gcs            ← FleetStatus → operator display             ║
╚═══════════════════════════╦══════════════╦═══════════════════════════╝
                            │              │
                    uXRCE-DDS/UDP  uXRCE-DDS/UDP
                      namespace     namespace
                      /px4_1/       /px4_2/
                            │              │
              ╔═════════════╩══╗     ╔═════╩═════════════╗
              ║  WORKER 1 NODE ║     ║  WORKER 2 NODE    ║
              ║                ║     ║                   ║
              ║ /px4_bridge    ║     ║ /px4_bridge       ║
              ║ /soc_estimator ║     ║ /soc_estimator    ║
              ║ /local_planner ║     ║ /local_planner    ║
              ║ /slam_bridge   ║     ║ /slam_bridge      ║
              ╚═══════╦════════╝     ╚═════════╦═════════╝
                      │                        │
            PX4 uXRCE-DDS              PX4 uXRCE-DDS
            Pixhawk 6C W1              Pixhawk 6C W2
                      │                        │
                      └────────────┬───────────┘
                                   │
                         ╔═════════╩════════╗
                         ║  DOCK CONTROLLER ║
                         ║                  ║
                         ║ /dock_controller ║
                         ║ /charge_monitor  ║
                         ╚══════════════════╝
```

Each cluster in this graph has a well-defined computational role and a formally specified data contract with every other cluster. The following subsections describe each cluster's responsibilities and contracts precisely.

### 6.1 The Command Node Cluster

The command cluster is the system's decision-making brain. It runs on the ground station laptop during testing and on the carrier drone's onboard computer in field deployment. It is the only cluster with full visibility of the fleet state.

**`/fleet_scheduler`** — Subscribes to `DroneState` from all worker nodes. Maintains a fleet state vector. On each SOC update, evaluates the SOC-constrained CBBA bidding function. If a drone's SOC drops to or below `RTH_TRIGGER_SOC` (40%), publishes a `RequestRTH.srv` call to that drone's `px4_bridge` node and triggers an `AssignMission.srv` call to dispatch the highest-SOC available replacement. Publishes `FleetStatus` at 1 Hz for the GCS.

**`/mission_planner`** — Accepts operator mission input (coverage polygon, priority zones, mission duration) and translates it into a waypoint sequence and `CoverageMap` message. Does not know about individual drone states — it describes *what* to cover, not *who* covers it. The `fleet_scheduler` consumes the coverage map and assigns specific drones to specific waypoint blocks.

**`/comms_manager`** — Manages the ESP-NOW TDMA physical layer. Assigns transmission slots to registered fleet members, publishes received telemetry to the ROS2 graph as additional data sources (secondary to the uXRCE-DDS bridge), and monitors link quality. In Phase 1 (SITL), this node runs in simulation mode, injecting synthetic latency to validate scheduler behaviour under degraded communication.

**`/energy_monitor`** — Subscribes to `BatteryStatus` from all drones. Implements the three-threshold decision logic:
- `SOC > 60%`: nominal operation, full AI inference enabled
- `40% ≤ SOC ≤ 60%`: reduced inference resolution (320px), SOC-adjusted bid penalty in scheduler
- `SOC ≤ 40%`: RTH trigger published, inference degraded to obstacle-avoidance only

**`/perception_server`** — Manages AI inference jobs. Receives frame data via the RealSense D435i driver, runs TensorRT YOLOv8n inference, applies SPF zone weighting from the mission planner, and publishes detection results to the fleet-wide topic graph. In simulation, receives synthetic camera frames from Gazebo's camera plugin.

**`/fleet_gcs`** — Reads `FleetStatus` at 1 Hz and renders the operator dashboard: per-drone SOC bars, dock occupancy, active mission waypoints, detection events, and manual override controls.

### 6.2 Worker Node Clusters

Each worker drone runs an identical node cluster, differentiated only by its ROS2 namespace and drone ID parameter.

**`/px4_bridge`** — The namespace-aware uXRCE-DDS subscriber and publisher. Translates between the fleet application layer's `DroneState` message format and PX4's native uORB topic format (`VehicleLocalPosition`, `BatteryStatus`, `VehicleAttitude`, `TrajectorySetpoint`). This node is the only node in the system that speaks PX4's message vocabulary directly; all nodes above it consume the project's own `DroneState` type.

This deliberate isolation means that a hardware change — say, migrating from PX4 to ArduPilot, or from a Pixhawk 6C to a different flight controller — requires changes only in `px4_bridge`, not in any scheduling or planning logic.

**`/soc_estimator`** — Implements a Coulomb-counting SOC estimator with voltage-based correction. Subscribes to ESC telemetry (current draw per motor) and battery voltage from the flight controller. The estimator accounts for the non-linear voltage-vs-SOC relationship of LiPo cells and the temperature-dependent capacity loss. Its output — a single float between 0.0 and 1.0 — is published as part of the `DroneState` message at 2 Hz.

**`/local_planner`** — A lightweight reactive obstacle avoidance layer that operates independently of the mission planner. Subscribes to depth data from the RealSense D435i and publishes velocity adjustments to the PX4 offboard control interface. It is designed to be preemptive: even if the command cluster loses communication, the local planner continues to prevent collisions and hold position.

**`/slam_bridge`** — Person 3's integration point. Publishes LiDAR-Inertial odometry estimates to the ROS2 graph as a source of GPS-denied localisation. In Phase 1, this node is stubbed: it publishes GPS-derived position as if it were SLAM output, allowing the rest of the stack to run without a working SLAM implementation. The stub's API is identical to the production SLAM bridge's API — the `slam_bridge` node publishes to the same topic with the same message type regardless of the underlying implementation.

### 6.3 The Dock Controller

The docking station is the system's energy source. It runs a single controller node that manages charge cycles, reports occupancy, and interacts with the fleet scheduler to announce when a drone has reached sufficient SOC for redeployment.

**`/dock_controller`** — Subscribes to `DockStatus` requests from the fleet scheduler and publishes `DockStatus` updates at 1 Hz. Manages up to N docking bays (N configurable, default 3 for Phase 1). When a drone docks, the controller reports its bay ID and initiates a charge cycle. When the drone reaches 90% SOC, it publishes a `ready_for_deployment` flag that the fleet scheduler can act on.

In Phase 1, the docking station is simulated: the dock controller node accepts MAVLink `COMMAND_LONG` landing commands and simulates a charge cycle with a configurable time constant, then publishes readiness. The physical docking mechanism design is Person 2's domain; the software interface is defined here and frozen.

---

## 7. Interface Contracts: The Message Schemas

The message schemas are the most important artefacts produced in the conceptualisation phase. They are the contracts that allow all four team members to write code in parallel without stepping on each other.

### 7.1 `DroneState.msg`

```
# fleet_interfaces/msg/DroneState.msg
# Per-drone state vector published by each px4_bridge node at 10 Hz

std_msgs/Header header          # Timestamp, drone frame ID

string drone_id                 # Unique drone identifier ("worker_1", "worker_2", ...)
uint8 drone_role                # 0=WORKER, 1=COMMAND, 2=CARRIER
uint8 mission_state             # 0=IDLE, 1=ACTIVE, 2=RTH, 3=DOCKING, 4=CHARGING, 5=FAULT

# Position and velocity (NED frame, metres)
geometry_msgs/Point position    # x=North, y=East, z=-Down
geometry_msgs/Vector3 velocity  # m/s in NED frame

# Attitude
geometry_msgs/Quaternion attitude

# Energy state
float32 soc                     # State of Charge, 0.0 to 1.0
float32 voltage                 # Battery voltage (V)
float32 current_draw            # Total current (A)
float32 flight_time_remaining   # Estimated remaining flight time (minutes)

# Mission state
uint32 current_waypoint_index
float32 mission_completion_pct  # 0.0 to 1.0
bool obstacle_detected

# Communication
float32 link_quality            # 0.0 to 1.0 (ESP-NOW RSSI-derived)
uint8 comms_mode                # 0=ESP_NOW, 1=WIFI_AP, 2=TELEMETRY_RADIO

# Inference state
uint8 inference_mode            # 0=FULL (640px), 1=REDUCED (320px), 2=MINIMAL
float32 inference_fps           # Current inference throughput
```

This message is the system's primary data contract. Every scheduling decision, every GCS display element, and every energy management action is derived from fields in this message. Defining it precisely before implementation prevents the most common category of integration failure: two nodes that disagree about what a field means.

### 7.2 `FleetStatus.msg`

```
# fleet_interfaces/msg/FleetStatus.msg
# Aggregated fleet state published by fleet_scheduler at 1 Hz

std_msgs/Header header

DroneState[] drone_states       # Full state of all fleet members
uint8 active_drones             # Count of drones currently on mission
uint8 docked_drones             # Count of drones currently charging
float32 fleet_coverage_pct      # Fraction of coverage zone currently observed
float32 estimated_endurance_hrs # Fleet-level estimated remaining endurance
bool carrier_available          # Whether the carrier drone is in service
```

### 7.3 `AssignMission.srv`

```
# fleet_interfaces/srv/AssignMission.srv
# Request/response for mission assignment from scheduler to a worker drone

# REQUEST
string drone_id
geometry_msgs/Point[] waypoints   # Ordered waypoint sequence
float32 mission_priority          # 0.0 to 1.0
float32 max_mission_duration_min  # Time limit in minutes
uint8 inference_requirement       # Minimum inference mode required

---

# RESPONSE
bool accepted
string rejection_reason           # If not accepted, why
float32 estimated_completion_min  # Estimated time to complete mission
```

### 7.4 `DockStatus.msg`

```
# fleet_interfaces/msg/DockStatus.msg
# Docking station state published by dock_controller at 1 Hz

std_msgs/Header header
uint8 total_bays
uint8 occupied_bays

BayState[] bay_states

---
# Nested type
string bay_id
bool occupied
string occupant_drone_id        # Empty if not occupied
float32 charge_soc              # Current charge level of occupant
bool ready_for_deployment       # True when SOC >= 0.90
float32 estimated_ready_min     # Minutes until ready_for_deployment
```

These four schemas — `DroneState`, `FleetStatus`, `AssignMission`, `DockStatus` — are the complete interface surface between all four team workstreams. Person 2 writes to `DockStatus`. Person 3 writes to the `position`, `velocity`, and `attitude` fields of `DroneState` (via the `slam_bridge`). Person 1's output appears indirectly through the mass and power parameters that calibrate the `soc_estimator`. The CS lead writes everything that consumes and produces these messages.

---

## 8. The Simulation-First Strategy and the Sim2Real Gap

The decision to develop the entire software stack in SITL before any hardware integration is not merely a convenience — it is a deliberate risk management strategy grounded in two observations.

**Observation 1:** The most expensive bugs are integration bugs. A logic error in the fleet scheduler discovered in simulation costs one git commit to fix. The same error discovered during a hardware test costs a crashed drone.

**Observation 2:** Simulation fidelity is sufficient for the Phase 1 validation target. The core Phase 1 claim — that the SOC-constrained CBBA scheduler can maintain 95%+ coverage continuity for two drones across a 60-minute simulated session — does not require physical hardware. It requires a faithful power consumption model, a faithful battery discharge model, and a faithful flight dynamic model. All three are available in PX4 SITL with Gazebo Classic.

The simulation framework uses a critical-path design: the simulation-image contains Gazebo Sim, the compiled PX4/ArduPilot SITL binaries, and the complete library of aircraft and world SDF assets serving as the digital twin of the physical environment. The aircraft-image contains the ROS2 autonomy stack, the autopilot interface middleware, and the perception pipelines; crucially, it is created from the same Dockerfile for both simulation and deployment, minimising the sim2real gap.

The primary sim2real gap in this system is not in the flight dynamics (PX4 SITL is a well-validated physics model) but in two specific areas:

**Gap 1: Power consumption accuracy.** The SITL power model uses a default constant-current assumption. Real LiPo cells have a non-linear voltage drop under load, a temperature coefficient, and a cycle-dependent capacity fade. The `soc_estimator` node must be tuned against bench measurements (Person 2's domain) before Phase 2 hardware integration, or the SOC thresholds that drive the scheduler will be calibrated against an inaccurate model.

**Gap 2: Companion computer architecture.** Hardware-in-the-loop simulation is an essential validation step for safety-critical aerospace systems, narrowing the gap between pure simulation and flight tests. More critically in this project, the AI inference performance on the Jetson Orin NX (ARM64, CUDA 11.8, TensorRT 8.6) differs substantially from performance on a development workstation (x86_64). YOLOv8n achieves 52 FPS on the Jetson Orin NX and 65 fps with INT8 quantisation; the development workstation typically exceeds 200 FPS. The energy-adaptive inference thresholds are calibrated against the Jetson's performance, not the workstation's — so the perception pipeline must be benchmarked on target hardware before Phase 2.

Both gaps are documented as **known deferred risks** with explicit mitigation actions: ESC bench characterisation in Week 1 (Person 2), and TensorRT model conversion and Jetson benchmarking in Month 2 (CS lead). They are not architectural surprises; they are scheduled work items.

---

## 9. The Development Sequence

The build order follows from the dependency graph. Each milestone is stated in terms of a concrete, testable output — not a subjective assessment of "progress."

### Phase 0: Interface Foundations (Week 0–1)
**Owner:** CS lead
**Deliverable:** `fleet_interfaces` package committed to the shared Git repository, with all `.msg`, `.srv`, and `.action` definitions reviewed and frozen.

A frozen interface definition means: no field additions, removals, or type changes without a team-wide review. The purpose of this constraint is not bureaucratic — it is to prevent the silent divergence where two team members write to the same message type but disagree about the semantics of a field.

**Also in Week 0:** Ubuntu 22.04 base environment, ROS2 Humble installation, PX4 SITL with Gazebo Classic running and verified, QGroundControl connected.

### Phase 1A: Single Vehicle SITL (Week 1–2)
**Owner:** CS lead + Person 3
**Deliverable:** A single simulated drone takes off, flies a five-waypoint mission at 10 metres altitude, and returns to home — all commanded via the `px4_bridge` ROS2 node, with telemetry logging to ROS2 bag.

This milestone validates the uXRCE-DDS bridge, the PX4 offboard control interface, and the `DroneState` publisher. It is the foundation for everything that follows.

### Phase 1B: Energy Monitor + SOC-Triggered RTH (Week 3)
**Owner:** CS lead
**Deliverable:** The `energy_monitor` node is running. SOC is estimated from the SITL battery model. At 40% SOC, the node publishes a `RequestRTH.srv` call, and the simulated drone returns to home and lands.

This milestone validates the complete RTH trigger chain: battery model → `soc_estimator` → `energy_monitor` → `px4_bridge` → PX4 RTH mode → landing.

### Phase 1C: Two-Drone Fleet with Basic Rotation (Week 4)
**Owner:** CS lead
**Deliverable:** Two simulated drones fly simultaneously, each on an independent mission. When Drone 1 reaches 40% SOC, the fleet scheduler triggers RTH and dispatches Drone 2 to continue coverage. Coverage continuity is logged and verified to exceed 90% during the handover.

This milestone is the Phase 1 keystone. It is the first empirical measurement (in simulation) of the system's central architectural claim. The exact coverage continuity number, the handover latency, and the SOC at which replacement drone arrives on-station are all logged as baseline measurements for comparison against Phase 2 hardware results.

### Phase 2: Hardware Integration (Months 2–4, pending hardware)
The Phase 2 sequence — ESC characterisation, Pixhawk 6C flashing, Jetson Orin NX bring-up, TensorRT pipeline, hardware-in-loop testing, outdoor flight — is planned but not scheduled in detail here. Phase 2 begins when Phase 1C is complete and hardware has been procured.

---

## 10. Conceptualisation Outputs: What This Phase Produced

At the conclusion of the conceptualisation phase, the following concrete documents and decisions existed in written form and were distributed to all team members:

1. **Layer decomposition document** — Physical, Control/Communication, Cognitive layers with explicit responsibility boundaries and data flow specifications.

2. **Node graph diagram** — Full resolution diagram with all node names, topic names, service names, and inter-node data flows. Published to the shared repository.

3. **Message schema definitions** — `DroneState.msg`, `FleetStatus.msg`, `DockStatus.msg`, `AssignMission.srv`, `RequestRTH.srv`, `SetInferenceMode.srv`, `ExecuteMission.action` — all defined, versioned, and frozen for Phase 1.

4. **Workspace directory structure** — Defined with package-level ownership assigned to specific team members. No ambiguity about who owns what.

5. **Multi-vehicle namespacing spec** — `px4_$N` namespace convention, launch file parameterisation scheme, and the constraint that no application-layer code may contain hardcoded topic names.

6. **Sim2real gap register** — Two documented gaps (power model and Jetson inference performance) with explicitly assigned mitigation owners and delivery dates.

7. **Development sequence** — Four milestones with concrete, binary pass/fail criteria and dependency chains.

8. **SOC threshold parameter table:**

| Parameter | Value | Justification |
|---|---|---|
| `RTH_TRIGGER_SOC` | 0.40 (40%) | 10% margin above minimum 30% needed for RTH transit + landing |
| `INFERENCE_THROTTLE_SOC` | 0.40 (40%) | Matches RTH trigger; inference degrades at same point to save power |
| `DOCK_RESERVE_SOC` | 0.25 (25%) | Hard floor; below this, drone must not fly regardless of scheduler state |
| `DEPLOYMENT_READY_SOC` | 0.90 (90%) | Ensures newly deployed drone can complete a full mission cycle |
| `FULL_INFERENCE_SOC` | 0.60 (60%) | Inference throttling begins when coverage gap risk first appears |

These thresholds are not arbitrary — they derive from the W^1.5 power analysis and the fleet endurance model. They are the analytical outputs of the thinking phase made operational.

---

## 11. What the Conceptualisation Phase Does Not Decide

An honest accounting of architectural scope also requires documenting what was explicitly deferred.

**The carrier drone architecture** — as noted in Blog 01, this remains the highest architectural risk. The conceptualisation phase added one concretisation: the carrier's software interface to the fleet stack. When the carrier drone eventually exists, its companion computer will run a `carrier_bridge` node that publishes a single `CarrierState.msg` (position, heading, payload bay status, onboard power available) to the command cluster. The interface is defined; the hardware is not.

**Federated EKF for GPS-denied operation** — The theoretical framework exists in the research corpus, but the ROS2 node for federated state estimation across multiple drones is a Phase 3 item. It requires a validated SLAM implementation (Person 3's Phase 2 deliverable) before meaningful integration work can begin.

**Carrier drone fleet logistics software** — The scheduling logic for the carrier drone's own repositioning — determining when and where to move to minimise average worker transit time — is an optimisation problem with an interesting analytical structure (a variant of the facility location problem with mobile facilities). It is in scope for the project but cannot be designed without empirical data on worker drone range and carrier transit dynamics. It is deferred to Phase 3.

These deferrals are features, not failures. A conceptualisation phase that attempts to fully specify every component of a multi-phase system produces either an overfit design or a paralysed team. The discipline is deciding what must be specified now — the interfaces that unblock immediate work — and what can be safely deferred until more data exists.

---

## 12. Reflection: What Conceptualisation Is Really About

The conceptualisation phase is sometimes described as "system design" or "software architecture," but those labels obscure its real function: **it is the process of making the system's future behaviour predictable from its present decisions.**

A well-conceived architecture is one where, at any point during implementation, you can look at a bug and say: "this bug is in component X, it crosses interface Y, and fixing it requires changing Z." That level of diagnostic clarity is not a property of clever code — it is a property of clear interfaces. A poorly conceived architecture is one where a bug anywhere might be caused by anything, because the components are not truly decoupled.

The node graph, the message schemas, the namespace convention, the package ownership map — all of these are instruments for ensuring that when something goes wrong (and in a four-person multi-hardware robotics project, things will go wrong), the failure mode is localised, attributable, and fixable without cascading through the entire system.

The next blog in this series covers the **Research Deep-Dive Phase**: a detailed technical examination of each of the nine research documents — the W^1.5 power model, the FOC-IMU analysis, the CFRP FEA, the power isolation study, the AI nav benchmarks, the fleet endurance model, the SOC throttling analysis, the comms range model, and the carrier drone architecture — with the analysis, results, and architectural conclusions of each.

---

## References

- ROS2 Humble architecture and DDS middleware: Macenski et al., *Science Robotics*, 2022
- PX4 uXRCE-DDS bridge and multi-vehicle namespacing: PX4 Developer Guide v1.14+
- Modular heterogeneous UAV swarm architecture: arXiv:2510.27327, 2025
- Aerial autonomy stack sim2real minimisation: arXiv:2602.07264, 2026
- SITL in UAV rapid development lifecycle: IEEE/ASME *Aerospace Science and Technology*, 2021
- Sim2real gap characterisation in industrial UAV inspection: MDPI *Aerospace*, 2023
- ROS2 interface contract enforcement via `.msg`/`.srv` files: roboticsarchitectureauthority.com

---

*This blog is Part 2 of the Sentinel-Mesh Development Log. The repository is maintained at [github.com/Shrey-Arc/Sentinel-Mesh](https://github.com/Shrey-Arc/Sentinel-Mesh).*

---
*© 2026 Shrey, Sentinel-Mesh Project. All rights reserved.*
