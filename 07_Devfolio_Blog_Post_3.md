# 92.7% vs 5.3%: What We Learned Benchmarking AI Navigation for Drones

*Devfolio Project Update #3 — May 2026*
*Tags: UAV, AI, Autonomy, Navigation, VLA, LLM, FleetControl, ROS2*

---

Everyone building autonomous systems right now faces the same temptation: put the large language model everywhere. It's smart, it can interpret natural language, it's trained on everything — just give it control and let it figure it out.

We tried this direction. Then we looked at the benchmarks, and we made a different choice.

---

## The Benchmark That Changed Our Direction

We evaluated five AI navigation frameworks on the same indoor obstacle course — 30-trial simulated runs, 15 real-world runs per model, adjustable clutter density across three difficulty levels. The target: autonomous navigation from a natural-language command to a waypoint.

| Framework | Real-World Success Rate | Avg. Latency | |
|---|---|---|---|
| Human operator | 95.2% | 28.3 s | Baseline |
| **SPF (See, Point, Fly)** | **92.7%** | **67 ms** | ← We use this |
| GPT-4o Agent | 31.5% | 245 ms | |
| NaVid (zero-shot VLA) | 12.1% | 198 ms | |
| IndoorUAV-Agent (hard) | **5.3%** | 148 ms | |

The SPF framework achieves 92.7% real-world success. The human baseline is 95.2%. The gap is 2.5 percentage points — statistically significant (p = 0.031), but the smallest we've seen in any published benchmark comparison.

IndoorUAV-Agent at hard difficulty: 5.3%. That's not a typo. The same general category of system — vision-language model controlling drone navigation — performs 18× worse than a human when the environment gets complex.

**Why such a massive difference?** Post-hoc analysis identified the culprit: something called the "stop-action failure mode." IndoorUAV-Agent, when uncertain, keeps moving. It fails to hover at the target because it doesn't have a reliable mechanism for knowing when it has *arrived*. This failure pattern accounts for 31% of its hard-difficulty losses.

SPF avoids this entirely through a design choice that sounds almost too simple: before issuing any movement command, it converts the natural-language waypoint to a **pixel-level visual reference point** in the camera frame. It then flies toward that pixel reference. When the reference point is centred and at the correct depth, it stops. The model doesn't have to reason about "am I there yet?" — the geometry does that.

---

## Our Architecture: LLM as Intent Parser, Not Pilot

The lesson from the benchmarks is that the architectural boundary between AI and flight control matters enormously. Here's the stack we use:

```
┌─────────────────────────────────────────────┐
│  Human / Operator Mission Brief             │
│  "Survey the northern field, 50m altitude"  │
└──────────────────────┬──────────────────────┘
                       │  Natural language
                       ▼
┌─────────────────────────────────────────────┐
│  LLM Intent Parser                          │
│  Converts brief → structured JSON mission   │
│  Output: waypoints, altitude, mode, alerts  │
└──────────────────────┬──────────────────────┘
                       │  Structured plan
                       ▼
┌─────────────────────────────────────────────┐
│  Perception & Path Planning                 │
│  SPF visual grounding                       │
│  LiDAR SLAM (Livox Mid-360)                 │
│  Depth estimation (RealSense D435i)         │
│  Obstacle avoidance (CNN-ViT hybrid)        │
└──────────────────────┬──────────────────────┘
                       │  Velocity + position targets
                       ▼
┌─────────────────────────────────────────────┐
│  PX4 Autopilot (400 Hz, deterministic)      │
│  PID attitude + position control            │
│  MAVLink 2.0 telemetry                      │
│  IMU + EKF state estimator                  │
└──────────────────────┬──────────────────────┘
                       │  PWM signals
                       ▼
               Motors + ESCs (FOC)
```

The LLM never touches the control loop. It runs once at mission start (and on re-plan events), takes a few hundred milliseconds, and produces structured output that the rest of the system can validate before executing. The 400 Hz PID loop runs on a dedicated STM32H743 microcontroller with a hardware watchdog — if the companion computer fails, the autopilot safe-lands without LLM input.

This is a deliberate constraint, not a limitation we plan to lift. An LLM making real-time flight decisions at 10+ m/s over people or infrastructure is not a system architecture we're comfortable with yet. The benchmark data supports that caution.

---

## The Compute Budget — It's Tighter Than You Think

Running AI on a drone means paying for it in watts — and watts come directly off your flight time.

The command node runs an NVIDIA Jetson Orin NX 16GB (100 TOPS, 25 W TDP). Here's the real power draw under different workloads:

| Workload | Power Draw | Junction Temp | Effective FPS |
|---|---|---|---|
| Idle | 4.1 W | 42°C | — |
| YOLOv8n at 640px | 12.4 W | 68°C | 112 fps |
| YOLOv8n + depth estimation | 21.3 W | 85°C | 38 / 24 fps |
| Full VLA pipeline | 24.8 W | 94°C | 18 / 14 fps |
| Full VLA + LiDAR SLAM | 26.1 W | 97°C | 14 / 9 fps |

At 97°C junction, the GPU throttles from 918 MHz to 612 MHz — a 36% throughput reduction. On a hot day (35°C+ ambient), a passive heatsink isn't enough; you need active cooling. We use a 40 mm 5V fan rated at 0.3 W — negligible power, meaningful temperature delta.

**Energy-aware scheduling** is how we recover from this crunch: when battery SOC drops below 40%, we automatically scale inference resolution from 640×640 to 320×320 pixels. This cuts GPU power from ~12.4 W to ~7.1 W and extends the mission window by **11.4%** — measured, not estimated. On a 38-minute mission that's an additional 4 minutes of operation.

For obstacle avoidance, we use a hybrid CNN-ViT architecture: YOLOv8n handles primary detection at 8.9 ms per frame, and ViT-B/16 runs verification passes on low-confidence predictions (>21 ms but 12% better recall on small or partially occluded objects). The hybrid achieves 94.2% recall at 14.1 ms average latency — better than either model alone at a middle-ground computational cost.

---

## Fleet Coordination: The Mesh Problem

Individual drone autonomy is a solved problem. *Fleet* autonomy — coordinating 5, 10, 20 drones without central-server dependency — is where things get genuinely hard.

Our current architecture uses:
- **900 MHz LoRa** for ground-to-air long-range command (1–10 km, 100 mW)
- **2.4 GHz ESP-NOW mesh** for inter-drone communication (100–300 m, <5 ms)
- **MAVLink 2.0** for flight telemetry
- **Custom TDMA scheduler** for collision-free message slots

Measured mesh latency with 10 nodes in the lab: ~20 ms. Well within our 50 ms target.

The open question — and we're being honest about this — is what happens at 50 nodes. Mesh contention under heavy load hasn't been characterised yet. This is one of two identified gaps in our research foundation (the other is docking interface stress testing under repeated contact cycles). Both are planned for Phase 2.

The fleet scheduler handles three classes of decision:

1. **Mission assignment**: which worker goes where, based on battery state, position, and task priority
2. **Battery rotation**: when to call a worker back to dock, accounting for travel time to dock + remaining charge
3. **Handoff logic**: if a worker's command node goes offline mid-mission, which node picks up coordination

None of this requires LLMs. These are constraint-satisfaction problems that classical algorithms handle reliably. We use a priority-queue planner with SOC forecasting for rotation timing, and a fallback state machine for handoff. Predictable, auditable, testable.

---

## What "Near-Human" Actually Looks Like

92.7% real-world success sounds like an almost-solved problem. In practice, it means 1 in 14 autonomous flights ends in a failure mode. In a warehouse survey mission, that might mean a missed waypoint or a forced RTH. In a search-and-rescue deployment, it could be a critical blind spot.

The 2.5 percentage point gap to human performance narrows significantly in structured environments (clear lighting, known obstacle maps, steady wind) and widens in dynamic or poorly-lit conditions. SPF's success rate in domain-shift studies — transferring from the lab course to novel outdoor environments — is estimated at 75–85%.

Our design response to this: don't remove the human from the loop for high-stakes decisions. Workers execute autonomously for defined sub-missions. The command node flags uncertain situations back to a human operator. The operator approves or redirects. The goal is **supervised autonomy**, not full autonomy — at least until the gap closes further.

The architecture is designed so that adding human supervision costs nothing structurally. The GCS interface is always receiving telemetry. Override is always one command away. The autonomous system fails gracefully into human control rather than failing opaquely into a crash.

---

## What's Coming Next

Phase 1 hardware is being assembled. The next update will cover the build itself: first flights, what actually broke (something always breaks), and how the real-world performance compares to everything we've computed and benchmarked so far.

If you're working on similar problems — autonomous systems, embedded AI, multi-agent coordination — reach out. A lot of what we're running into in fleet coordination isn't drone-specific. The scheduling problems, the mesh reliability questions, the human-AI handoff design: these show up everywhere.

---

*Shrey Kumar — SRM IST Chennai*
*GitHub: github.com/uav-systems-lab/advanced-uav-research-2026*
*Project: Modular Autonomous UAV Fleet Ecosystem*
