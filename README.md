Here's the current state summary and a concrete implementation roadmap as on 26th June 2026.

---

**Research phase: complete.** Nine documents, a journal paper, and a validated analytical foundation. The maturity breakdown matters for planning:

| Layer | Status | Represents |
|---|---|---|
| W^1.5 physics, FOC acoustics, CFRP FEA, power isolation, AI nav benchmarks | **Validated** | Your evidence base; cite these in implementation decisions |
| Fleet endurance scheduling, energy-aware SOC throttling, comms range via relays | **Modeled, not demonstrated** | Your Phase 1 target — turn models into measurements |
| Carrier drone, laminar fuselage, structural batteries, GPS-denied cooperative localization | **Conceptual only** | Phase 2/3; do not let these distract Phase 1 |

**Current hardware state:** Nothing built. You have a BOM, an FEA-validated frame selection (Tarot TL65B44), component specs, and a procurement list. Phase 1 starts now.

**Critical path item:** The 7+ hour fleet endurance claim, which is the project's central architectural contribution, depends entirely on a carrier drone that has no engineering specification in any document. Resolving this is not your immediate job — but you need to proxy it in simulation before the claim can be defended.

---

## Your Role Clarified

Your domain: **AI deployment, communications, fleet software, and GCS.**

Your team:
- Person 1 → mechanical fabrication + FEA/CFD
- Person 2 → electrical/propulsion + instrumentation
- Person 3 → flight systems/avionics + LiDAR SLAM
- You → everything in software: autonomy stack, fleet logic, communication architecture, ground control

You can begin **today**, before any hardware exists, because your entire Phase 1 workstream runs in simulation first. This is the structural advantage of being the software lead.

---

## Your Immediate Workstreams

### Workstream 1: SITL Environment (Start Week 1)

This is your first deliverable and the foundation for everything else. It lets you develop and test flight logic, fleet coordination, and AI pipelines before hardware is ready, and it gives Person 3 a shared environment to test avionics integration.

**Stack:**
```
Ubuntu 22.04
ROS2 Humble
PX4 Autopilot v1.14.x (SITL)
Gazebo Classic 11
MAVSDK-Python or pymavlink
QGroundControl (GCS baseline)
```

**Setup sequence:**

```bash
# 1. Install ROS2 Humble
sudo apt install ros-humble-desktop

# 2. Clone PX4
git clone https://github.com/PX4/PX4-Autopilot.git --recursive

# 3. Install PX4 deps
bash ./PX4-Autopilot/Tools/setup/ubuntu.sh

# 4. Build and run SITL with Gazebo
cd PX4-Autopilot
make px4_sitl gazebo-classic_iris

# 5. Install MAVSDK Python
pip install mavsdk

# 6. Verify MAVLink connection
python3 -c "import asyncio; from mavsdk import System; \
    async def run(): \
        drone = System(); \
        await drone.connect(system_address='udp://:14540'); \
        print('Connected') \
    asyncio.run(run())"
```

Target: a simulated drone that takes off, flies a waypoint mission, and returns to home — all controlled via Python by end of Week 2.

---

### Workstream 2: ROS2 Node Architecture (Week 2–3)

Define the node graph now, before writing any application code. This is your software architecture document. Every subsequent module will plug into this.

**Proposed node graph for Phase 1:**

```
┌─────────────────────────────────────────────────────────────┐
│                     COMMAND NODE                            │
│  /fleet_scheduler      ← SOC data → decides rotation       │
│  /mission_planner      ← operator intent → waypoints       │
│  /comms_manager        ← ESP-NOW TDMA → worker telemetry   │
│  /energy_monitor       ← battery state → throttle trigger  │
│  /perception_server    ← camera/LiDAR → obstacle data      │
└──────────────┬──────────────────────────┬───────────────────┘
               │ MAVLink                   │ MAVLink
               ▼                           ▼
┌──────────────────────┐     ┌──────────────────────┐
│    WORKER 1 NODE     │     │    WORKER 2 NODE     │
│  /px4_bridge         │     │  /px4_bridge         │
│  /soc_estimator      │     │  /soc_estimator      │
│  /local_planner      │     │  /local_planner      │
└──────────────────────┘     └──────────────────────┘
               │                           │
               └─────────┬─────────────────┘
                         ▼
               ┌──────────────────┐
               │  DOCKING STATION │
               │  /dock_controller│
               │  /charge_monitor │
               └──────────────────┘
```

**Start with these three packages:**

```
uav_fleet_ws/
├── src/
│   ├── fleet_interfaces/      # Custom msg/srv definitions
│   │   ├── msg/DroneState.msg
│   │   ├── msg/FleetStatus.msg
│   │   └── srv/AssignMission.srv
│   ├── px4_bridge/            # MAVLink ↔ ROS2 translation
│   ├── fleet_scheduler/       # Battery rotation logic
│   └── energy_monitor/        # SOC estimation + throttle
```

The `fleet_interfaces` package defines your data contracts. Build this first so all four team members can write to the same message schemas.

---

### Workstream 3: Fleet Scheduler (Month 1–2)

This is the algorithmic core of your novel contribution — the SOC-constrained fleet rotation that enables the 7+ hour endurance claim. Implement it in simulation before any hardware arrives.

**Minimum viable scheduler (Phase 1):**

```python
# fleet_scheduler/scheduler.py

from dataclasses import dataclass
from typing import List
import heapq

@dataclass
class DroneState:
    drone_id: str
    soc: float          # 0.0 to 1.0
    mission_active: bool
    flight_time_remaining: float  # minutes
    position: tuple     # (x, y, z)

class FleetScheduler:
    RTH_TRIGGER_SOC = 0.40   # Return-to-home threshold
    INFERENCE_THROTTLE_SOC = 0.40  # AI resolution downgrade
    DOCK_RESERVE_SOC = 0.25  # Minimum SOC on dock arrival

    def __init__(self):
        self.fleet: dict[str, DroneState] = {}
        self.mission_queue = []  # Priority queue

    def update_drone_state(self, state: DroneState):
        self.fleet[state.drone_id] = state
        self._evaluate_rotation(state)

    def _evaluate_rotation(self, state: DroneState):
        if state.soc <= self.RTH_TRIGGER_SOC and state.mission_active:
            self._trigger_rth(state.drone_id)
            self._assign_replacement(state.drone_id)

    def _trigger_rth(self, drone_id: str):
        # Publish RTH command via MAVLink bridge
        pass

    def _assign_replacement(self, vacated_id: str):
        # Find highest-SOC available (docked) drone
        available = [d for d in self.fleet.values()
                     if not d.mission_active and d.soc > 0.80]
        if available:
            replacement = max(available, key=lambda d: d.soc)
            self._assign_mission(replacement.drone_id)
```

This is the skeleton. The full CBBA (Consensus-Based Bundle Algorithm) — your novel invention 4 in the IP documentation — comes after this baseline runs cleanly in SITL with 2 simulated drones.

---

### Workstream 4: Communication Stack (Month 1–2)

**ESP-NOW mesh constraint you need to design around:** The standard Espressif implementation has a hard 20-peer limit. For Phase 1 (2–5 drones) this is fine. Architect the TDMA scheduler now so scaling beyond 20 nodes requires a protocol swap, not an architectural redesign.

**TDMA slot allocation (implement in firmware, inform your ROS2 comms manager):**

```
Slot size: 10 ms per drone
Cycle: 50 ms (5-drone fleet)
Payload: DroneState struct (~64 bytes)

Slot 0 (0–10 ms):   Command node beacon
Slot 1 (10–20 ms):  Worker 1 telemetry
Slot 2 (20–30 ms):  Worker 2 telemetry
...
Slot N (N×10 ms):   Dock controller heartbeat
```

Your `comms_manager` ROS2 node should abstract this so higher-level nodes don't care whether the physical layer is ESP-NOW or eventually 802.11s mesh.

---

### Workstream 5: GCS Interface (Month 2–3)

Do not build a custom GCS from scratch. Start with QGroundControl as your baseline — it has MAVLink support out of the box. Your job is to build a **fleet layer on top of it** that adds:

- Battery SOC visualization per drone
- Dock occupancy status
- Rotation schedule display
- Manual override for individual drone RTH

Minimal Phase 1 GCS can be a Python `tkinter` or `PyQt5` dashboard consuming your ROS2 topics. Build this once SITL is running.

---

### Workstream 6: AI Pipeline on Jetson (Month 3, when hardware arrives)

Your SITL work uses simulated sensors. When the Jetson Orin NX arrives, the AI pipeline needs:

```
1. JetPack 6.0 install + Docker container for ROS2
2. TensorRT conversion of YOLOv8n (fp16, batch=1)
3. RealSense D435i driver integration
4. Energy-aware inference node: 640px at SOC > 40%, 320px below
5. SPF framework port to ROS2 service call
```

Start packaging the TensorRT conversion scripts now, even on a desktop GPU, so deployment to Jetson is a container pull when hardware arrives.

---

## Team Coordination: Parallel Work from Day 1

Each person can start their domain work immediately, with interfaces defined by you.

| Person | Week 1 task | Dependency on you | Interface point |
|---|---|---|---|
| Person 1 (Mechanical) | Order CFRP frame (Tarot TL65B44), begin arm dimension CAD | None initially | Frame mass data feeds your W^1.5 model |
| Person 2 (Electrical) | Procure T-MOTOR F55A Pro II ESCs, bench test FOC vs trapezoidal on a motor rig | SITL power model for validation | Power consumption numbers feed your energy_monitor node |
| Person 3 (Avionics + SLAM) | Flash Pixhawk 6C with PX4, connect to your SITL instance via QGC | Your SITL environment | Shares the same ROS2 workspace |
| You (Software) | SITL environment live, ROS2 workspace with `fleet_interfaces` package | None — unblocked | Provides the integration foundation for everyone |

The shared artifact is the **ROS2 workspace**. Set it up on a common Git repository (GitHub) in Week 1. Everyone pulls from this. Message schema changes go through you to maintain interface contracts.

---

## 4-Week Execution Plan

**Week 1:**
- SITL live with PX4 + Gazebo
- Git repo initialized with workspace structure
- `fleet_interfaces` package: `DroneState.msg`, `FleetStatus.msg`, `AssignMission.srv` defined
- Person 2 begins ESC bench characterization

**Week 2:**
- `px4_bridge` node: MAVLink telemetry → ROS2 topics
- Single simulated drone flying a waypoint mission under ROS2 control
- Person 3: Pixhawk 6C flashed, QGC connected to SITL

**Week 3:**
- `energy_monitor` node: SOC estimation running in simulation
- SOC-triggered AI throttle behavior prototyped (threshold logic, not yet real inference)
- `fleet_scheduler` skeleton: one drone → RTH at 40% SOC, simulated replacement triggered

**Week 4:**
- Two-drone simulation running simultaneously with scheduler managing rotation
- Basic GCS dashboard showing per-drone SOC and dock status
- Review with Person 2: bench power data against your SITL power model

---

## What to Defer

These are in your domain but should not touch Phase 1 execution:

- Cooperative localization / GPS-denied navigation — requires validated carrier drone architecture first
- Federated EKF implementation — the algorithm is novel but adds no Phase 1 value
- Carrier drone fleet logistics software — can't test what doesn't physically exist
- Large-scale mesh testing beyond 5 nodes — Phase 2 problem

The carrier drone void is the biggest architectural risk. Your contribution to closing it: define a software proxy. Use a ground-based power supply station with a simulated 5 kg UAV in SITL as a stand-in carrier, so your fleet rotation logic can be validated before the physical carrier is built. This unlocks the 7+ hour coverage claim for simulation validation even in Phase 1.

---

## Summary State in One Paragraph

The research corpus is complete and sound. Your primary validated contributions (W^1.5 physics, FOC-vibration-IMU chain, power isolation) are defensible. The fleet endurance model is architecturally correct but depends on a carrier drone with zero engineering specification — proxying that in simulation is your critical path contribution. As the CS lead, you are unblocked from day one: SITL setup → ROS2 architecture → fleet scheduler → GCS layer → AI pipeline, in that order. The four-person team runs in parallel once the Git workspace and interface definitions are live. Nothing in your Phase 1 scope requires hardware to begin.
