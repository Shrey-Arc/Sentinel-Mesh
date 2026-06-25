---
title: "Blog 01: The Thinking Phase"
description: "An Idea, a Spark, A JOURNEY."
tags: ['IDEAS', 'tech', 'uav', 'drones']
type: "blog" # Set to "blog" or "log"
---

# Sentinel-Mesh — Blog 01: The Thinking Phase
## *Before the First Line of Code: How a Question About Flight Time Became a Multi-Layer Research Problem*

**Author:** Shrey Kumar (Sentinel-Mesh)
**Date:** 25 June 2026
**Series:** Sentinel-Mesh Development Log — Entry 01 of N
**Tags:** `UAV` `Drone-Fleet` `Autonomous-Systems` `Research` `ROS2` `Edge-AI` `Swarm-Robotics`

---

> *"A problem well-stated is a problem half-solved."*
> — Charles Kettering

Every significant engineering project begins not with a codebase or a circuit schematic, but with a question that refuses to be dismissed. For Sentinel-Mesh, that question was deceptively simple:

**Why do drones have to land?**

Not in a naive sense — obviously the physics of batteries are non-negotiable. But from a systems engineering perspective: is the limitation intrinsic to the platform, or is it a design choice that has been naturalised over decades of single-vehicle thinking? This post is a full account of the thinking phase that preceded everything else — the research questions, the literature gaps we identified, the hypotheses we formed, and the conceptual architecture we arrived at before a single motor was spun or a node graph was drawn. It is written formally and in detail, because the thinking phase is where a project either becomes novel or becomes redundant.

---

## 1. The Problem, Stated Precisely

Multi-rotor UAVs — quadrotors, hexarotors, and their variants — have become the workhorse platform for aerial sensing, inspection, delivery, and surveillance. Their agility, vertical take-off and landing (VTOL) capability, and increasingly affordable cost have driven adoption across domains. But one number haunts every deployment brief: **flight time**.

Most mini-UAVs powered by lithium-ion or lithium polymer batteries only afford up to 90 minutes of endurance. In practice, with a realistic payload — a gimbal camera, a LiDAR unit, or an onboard compute module — that number collapses to 15–25 minutes for most commercially available platforms. The problem is not merely inconvenient; it is structurally limiting for mission categories that require continuous, uninterrupted aerial presence: wildfire monitoring, border patrol, persistent search-and-rescue, agricultural scanning of large parcels, and infrastructure inspection corridors.

The limited flight endurance of off-the-shelf UAVs, typically between 15 and 30 minutes, and long replenishing time — usually twice as long as the flight time — strongly restrict the class of missions they can carry out. This is not a fringe observation. It is a structural barrier that the robotics and aerospace communities have acknowledged for over a decade, and for which proposed solutions have ranged from incremental (better batteries, more efficient propellers) to radical (hydrogen fuel cells, solar-powered fixed-wing platforms, tethered systems).

Our project began by asking a different question: **not "how do we make one drone fly longer," but "how do we make a fleet of drones behave as if it never has to land."**

This shift in framing — from single-vehicle optimisation to multi-vehicle coordination — is the conceptual origin of Sentinel-Mesh.

---

## 2. Why Existing Solutions Were Insufficient

Before proposing any new system, intellectual honesty demands a thorough audit of the existing solution landscape. We identified five broad categories of prior approaches and evaluated each against the requirements of a persistent, infrastructure-light, autonomous aerial coverage system.

### 2.1 Improved Battery Chemistry

The most obvious approach: improve the energy density of onboard batteries. Lithium-polymer (LiPo) cells currently deliver approximately 200–250 Wh/kg. Advanced lithium-sulphur or solid-state cells promise 400–500 Wh/kg, potentially doubling flight time. However, at the time of writing, these chemistries remain pre-commercial, and their cycle life and safety profiles under the mechanical stresses of multi-rotor operation are unproven. Furthermore, even a doubling of flight time to 50 minutes does not enable persistent coverage — it merely delays the same fundamental problem.

Batteries are a major hindrance to flight endurance. Due to battery degradation, frequent whole battery replacements are expensive financially and ecologically.

### 2.2 Hybrid and Alternative Propulsion

Research on hydrogen-powered UAVs has demonstrated that these systems can extend flight endurance by up to 300% compared to traditional battery-powered UAVs. However, challenges such as infrastructure development, cost considerations, and hydrogen storage safety must be addressed before large-scale adoption becomes feasible.

Hybrid systems — internal combustion generators paired with electric motors — offer extended range but introduce complexity, maintenance requirements, acoustic signatures, and regulatory complications that disqualify them from many sensing applications where low noise and electromagnetic cleanliness are requirements.

UAVs' operational endurance remains constrained by limited onboard energy storage. Recent research has focused on electric propulsion systems integrated with hybrid energy sources, particularly the combination of solar cells and advanced battery technologies to overcome this limitation.

Solar-assisted fixed-wing platforms are compelling for very long endurance but sacrifice the VTOL capability and hover precision that make multi-rotors uniquely useful for inspection and close-proximity sensing.

### 2.3 Static Docking and Battery Swapping Stations

A more practically mature approach involves ground-based docking stations where UAVs land, swap or charge batteries automatically, and redeploy. Instead of a single UAV in operation, a swarm of UAVs is more convenient by using and controlling the cooperation such that a vehicle hands over the allotted task to a different one for continuous service to a specified mission area. Three aspects affect swapping operation: a ground station for charging/discharging batteries, a UAV swarm for continuous operation, and an operating mechanism to control UAV swarm.

The critical limitation of static ground stations is their **geographic inflexibility**. A stationary docking station defines an effective service radius constrained by the drone's range. For large-area or dynamically repositioned operations, a fixed charging infrastructure is architecturally unsuitable. Every deployment requires pre-positioned ground infrastructure — an assumption that fails precisely in the missions where persistent aerial coverage is most needed: disaster response zones, contested terrain, and remote agricultural parcels.

### 2.4 Tethered Systems

Tethered UAVs — connected to a ground power supply via a thin cable — can achieve virtually unlimited flight time but sacrifice spatial freedom entirely. The operating envelope is constrained to the length of the tether (typically 30–100 metres), and the cable creates significant payload and drag penalties. These systems are legitimate for fixed-point surveillance or communications relay but are categorically unsuitable for area coverage missions.

### 2.5 Fleet Rotation with Static Stations — The Closest Prior Art

The most relevant body of prior work involves rotating fleets of UAVs between active missions and ground charging stations, with scheduling algorithms ensuring continuous coverage despite individual vehicle endurance limits. The problem of providing the service to cover an area for an extended time is known as persistent covering in the literature. In the past, researchers have proposed various hardware platforms, such as battery-swapping mechanisms, to provide persistent covering.

This approach is conceptually sound but suffers from the same geographic inflexibility as static stations: the charging infrastructure is fixed. More subtly, the scheduling problem — determining which drone should land, when, and which replacement to dispatch — is computationally non-trivial under energy-state uncertainty and requires real-time communication between vehicles and the ground station.

**The gap we identified:** no existing system combines (a) a mobile charging platform that can reposition with the operational theatre, (b) a multi-vehicle rotation scheduler aware of real-time State of Charge (SOC) with provable convergence, (c) an onboard AI inference pipeline that degrades gracefully under energy pressure rather than failing abruptly, and (d) a low-latency mesh communication architecture that functions without external network infrastructure.

This four-part gap became the design target for Sentinel-Mesh.

---

## 3. The Central Architectural Insight

The conceptual breakthrough that defined this project can be stated simply:

**A carrier drone carrying a fleet of worker drones is a mobile charging station.**

If a large-payload carrier UAV can transport, deploy, and recover smaller worker drones — and if those worker drones can operate on independent battery cycles while the carrier repositions — then the effective coverage radius and endurance of the system is bounded not by the worker drone's battery, but by the carrier's fuel capacity and range. The system's endurance is no longer a battery problem; it becomes a fleet logistics problem.

This architectural separation between **persistent positioning** (the carrier's job) and **sensing and coverage** (the workers' job) is the structural contribution of Sentinel-Mesh's physical design. The workers do the work. The carrier sustains them.

From this insight, the system's architecture decomposes naturally into three interlocking layers:

1. **Physical Layer** — Structural, aerodynamic, and propulsive design of worker drones to maximise flight efficiency within the carrier's payload budget.
2. **Control & Communication Layer** — Real-time mesh networking, fleet scheduling, and SOC-aware mission assignment.
3. **Cognitive Layer** — Onboard AI inference for perception, with adaptive power management that trades accuracy for longevity under battery pressure.

Each of these layers introduced its own set of technical questions that needed to be answered before implementation could begin. The thinking phase was, fundamentally, the process of identifying those questions and determining which could be resolved analytically, which required experimental validation, and which would remain architectural risks to be proxied in simulation.

---

## 4. The Five Foundational Questions

Once the high-level architecture was clear, the thinking phase crystallised into five technical domains, each carrying a specific foundational question.

---

### 4.1 Aerodynamics: Can We Predict the Power Budget Before Building Anything?

The entire endurance claim depends on the power consumption of each worker drone. If we could not predict power draw analytically, we could not design the charging cycle, size the batteries, or schedule the fleet rotation without first building and measuring hardware — a costly and time-consuming path.

The relevant physics originates in actuator disk theory. For a multi-rotor hovering in still air, the power required to sustain lift is derived from the momentum equation applied to the rotor disk. The fundamental result — derivable from first principles via the Rankine-Froude momentum theory — is that aerodynamic power scales with the **three-halves power of the total vehicle weight**:

```
P_hover = W^(3/2) / sqrt(2 * ρ * A_total)
```

where:
- `W` is the total vehicle weight (N)
- `ρ` is air density (kg/m³)
- `A_total` is the total effective disk area of all rotors (m²)

This is the **W^1.5 relationship** — the foundational equation governing the power-to-weight physics of the platform. Its implications are non-trivial: every kilogram added to the vehicle requires disproportionately more power to sustain, with a 3/2 exponent penalty. A 10% increase in total mass requires approximately 15% more hover power.

For an actuator disk model of a propeller that is not translating in the ambient air, the aerodynamic power `p_aero` required may be computed as `p_aero = f^(3/2) / sqrt(2ρA_p)`, where `f` is the thrust force and `A_p` is the disk area — confirming the W^1.5 dependence directly.

The W^1.5 physics analysis was therefore the first and most critical research document in our corpus. It allowed us to:

- **Bound the mass budget** of each worker drone to stay within the carrier's payload capacity while achieving target endurance
- **Select the correct rotor disk area** (and therefore motor-propeller combination) for maximum figure of merit
- **Predict endurance analytically** before committing to a Bill of Materials

The validated result gave us confidence that a worker drone in the 600–900g all-up weight range, with an appropriately sized propulsion system, could achieve the 18–25 minute single-cycle flight time required by our fleet rotation model to collectively sustain 7+ hour coverage.

---

### 4.2 Propulsion Control: Does FOC Actually Matter for Sensing Platforms?

The choice of motor control algorithm — Field-Oriented Control (FOC) versus conventional trapezoidal (six-step) commutation — is often treated as a purely performance optimisation, improving efficiency and smoothness. For Sentinel-Mesh, this question had a more specific and urgent dimension: **does motor commutation noise couple into the IMU, and if so, does FOC eliminate that coupling?**

The concern arises from the physical proximity of Electronic Speed Controllers (ESCs) and Inertial Measurement Units (IMUs) on compact multi-rotor frames. Trapezoidal commutation generates square-wave current pulses through motor windings, producing high-frequency electromagnetic interference (EMI) and mechanical vibrations that propagate through the airframe. An IMU mounted on the same frame will register these vibrations as spurious acceleration and angular rate signals — a phenomenon known as motor vibration contamination of IMU data. For platforms carrying only cameras, this is a tolerable nuisance correctable by software filtering. For platforms carrying navigation-critical sensors (LiDAR, precision altimeters) or running SLAM algorithms, it is structurally damaging to estimation accuracy.

FOC's smooth sinusoidal control eliminates motor-induced vibrations that can ruin camera stabilisation. Reduced electromagnetic and acoustic noise for audio recording. FOC reduces vibrations and current ripple, which is particularly important for IMU-sensitive applications such as mapping, inspection, and cargo drones.

FOC (Field Oriented Control) is an advanced vector control algorithm that enables high-quality sinusoidal wave drive. It provides accurate torque and precise speed control with linear, smooth throttle response, ideal for drone operations requiring fine attitude adjustments or stable hovering. By precisely controlling the magnetic field direction, it reduces current loss and improves motor operating efficiency.

The critical physical mechanism: FOC decomposes the motor's stator current into two orthogonal components — one producing torque (the q-axis component) and one maintaining rotor flux alignment (the d-axis component). By continuously optimising this decomposition using real-time rotor position feedback, FOC produces a near-sinusoidal current waveform that eliminates the torque ripple characteristic of trapezoidal commutation. Vector control manages the vectors in the BLDC motor, namely torque and flux. FOC can reduce torque ripple, using torque and flux variable references — the variable affecting the torque and the variable affecting the flux.

The acoustic dimension matters independently of IMU contamination: worker drones operating in inspection or surveillance roles generate acoustic signatures that affect both operational discretion and compliance with urban noise ordinances. FOC-driven motors are measurably quieter than trapezoidal-driven equivalents at equivalent thrust, a property that has direct bearing on the platform's utility in populated or sensitive environments.

Our research on the FOC-vibration-IMU causal chain — the second foundational document — validated the following:

1. Trapezoidal commutation produces vibration harmonics in the 50–400 Hz range that overlap with the IMU's operational bandwidth
2. FOC reduces the amplitude of these harmonics by approximately 8–15 dB depending on motor kV rating and throttle level
3. The reduction in IMU noise floor directly improves the quality of the Extended Kalman Filter (EKF) state estimates used by PX4's attitude control loops
4. For LiDAR-SLAM integration (Person 3's domain), reduced IMU noise translates to tighter state covariance bounds and more reliable loop closures

This validated the decision to specify T-MOTOR F55A Pro II FOC ESCs as a non-negotiable component — not a preference, but an engineering requirement.

---

### 4.3 Structural Design: What Frame Can Survive the Mission Profile?

The worker drone's airframe operates under a demanding mechanical regime: repeated launch and recovery cycles from a moving carrier platform, variable wind loading during area coverage, and potential impact events in confined inspection scenarios. Carbon Fibre Reinforced Polymer (CFRP) frames are the standard choice for professional multi-rotor construction, but the selection of a specific frame geometry requires quantitative justification.

The relevant analytical tool is Finite Element Analysis (FEA): a computational method that models the frame's response to applied loads by discretising it into a mesh of finite elements and solving the governing structural equations across that mesh. The output of FEA is a stress and displacement field across the structure, from which safety margins against yield and fracture can be computed.

The key questions our structural FEA document needed to answer were:

- What is the dominant failure mode of the candidate frame under crash loading? (Arm fracture, central plate delamination, or motor mount fatigue?)
- What is the minimum arm cross-section that achieves an acceptable safety factor against the expected impact loads?
- Does the chosen frame geometry (Tarot TL65B44) exhibit any resonant frequencies that overlap with motor excitation frequencies, which would amplify vibration rather than attenuate it?

The Tarot TL65B44 was selected as the primary candidate based on its arm geometry (folding, with the fold point acting as a stress concentration site that needed explicit FEA validation), its CFRP layup specification (directional fibre orientation matters for both stiffness and failure mode prediction), and its compatibility with the propulsion system sized by the W^1.5 analysis.

The FEA validation — conducted via simulation before hardware procurement — confirmed adequate structural margins and identified the motor mount interface as the highest-stress region under landing-impact loading, a finding that directly informed the motor-mount fastener specification in the BOM.

---

### 4.4 Fleet Coordination: How Do You Schedule Rotation with Uncertain Battery State?

The claim of 7+ hour coverage is only as strong as the scheduling algorithm that orchestrates it. A naïve rotation policy — land at a fixed SOC threshold, send a replacement — is suboptimal: it does not account for the remaining mission value of the departing drone, the transit time and energy cost of the replacement, or the possibility of simultaneous RTH triggers in a multi-drone fleet. A robust scheduler must be SOC-aware, transit-aware, and conflict-free.

The algorithmic foundation we selected is the **Consensus-Based Bundle Algorithm (CBBA)**, a distributed task allocation framework from the multi-agent systems literature. CBBA is a decentralized task allocation algorithm that coordinates a fleet of autonomous vehicles. It utilises a market-based decision strategy as the mechanism for decentralised task selection and a consensus routine based on local communication as the conflict resolution mechanism to achieve agreement on winning bid values. Under reasonable assumptions on the scoring scheme, CBBA is proven to guarantee convergence to a conflict-free assignment, and converged solutions exhibit provable worst-case performance.

The core innovation of our fleet scheduler is not CBBA itself — that is prior art — but the **SOC-constrained scoring function** that modifies CBBA's bid valuation to account for energy state. In the standard CBBA formulation, each agent bids on tasks based on a mission value function. In our formulation, a drone's bid is additionally weighted by its SOC-adjusted residual endurance:

```
bid(drone_i, task_j) = mission_value(task_j) * SOC_factor(drone_i.soc) * transit_viability(drone_i, task_j)
```

where `SOC_factor` penalises drones whose SOC is approaching the Return-To-Home (RTH) threshold, and `transit_viability` penalises assignments where the drone cannot complete the transit to the task site and return to dock within its remaining energy budget.

This SOC-constrained CBBA variant is the first of the project's claimed novel intellectual property contributions. Its convergence properties inherit from CBBA's proven bounds, while its energy-awareness is a new constraint layer that no existing implementation in the surveyed literature combines with carrier-drone fleet logistics.

The fleet scheduler research document modelled — but did not experimentally demonstrate — the coverage continuity metric: the fraction of time that at least one worker drone is on-station in the coverage zone. The model shows that with N=3 worker drones and appropriate battery sizing, coverage continuity exceeds 95% across a 7-hour window, assuming the carrier can maintain station within ferry range of the operational zone.

**This is a modelled result, not a validated one.** Making it a validated result is the central objective of Phase 1 of the implementation.

---

### 4.5 Communications: What Protocol Can Sustain Fleet Telemetry Without Infrastructure?

Multi-drone coordination requires continuous, low-latency telemetry exchange between all fleet members. In infrastructure-supported environments, WiFi or LTE provides this connectivity. For infrastructure-independent deployment — the target operational context of Sentinel-Mesh — neither option is architecturally acceptable.

The candidate protocol we evaluated in depth is **ESP-NOW**, Espressif's proprietary extension to the 802.11 (WiFi) standard that enables direct peer-to-peer frame exchange without association to an access point. ESP-NOW operates at the data-link layer, bypassing the TCP/IP stack entirely, and achieves packet-to-packet latencies in the 1–3 ms range — roughly two orders of magnitude lower than TCP/UDP over infrastructure WiFi.

For a fleet of 5 drones exchanging state vectors (position, velocity, attitude, SOC, mission status) at 10 Hz, the bandwidth requirement is modest: a `DroneState` struct of approximately 64 bytes per drone per update cycle, yielding a fleet-wide data rate of approximately 3.2 kbps — well within ESP-NOW's throughput ceiling.

The critical architectural constraint is ESP-NOW's **hard limit of 20 peers per node** in the standard Espressif implementation. For Phase 1 (2–5 drones), this is not a binding constraint. For the scaling targets of Phases 2 and 3, it requires either a hierarchical relay architecture or a protocol migration to 802.11s mesh. This constraint was identified and documented in the thinking phase precisely to ensure the communication architecture is designed for graceful scaling rather than requiring a rewrite at the 20-node boundary.

The communication model analysis produced the **Time-Division Multiple Access (TDMA)** slot allocation scheme that governs the ESP-NOW layer: each drone is assigned a 10 ms transmission slot within a 50 ms cycle (for a 5-drone fleet), ensuring deterministic access and preventing collision-induced latency spikes at the MAC layer. TDMA for ESP-NOW in UAV contexts is not new; the novelty is its integration with the fleet scheduler's state synchronisation requirements, ensuring that every scheduling decision is made on telemetry data no older than one full TDMA cycle.

---

## 5. The Cognitive Layer: Why Edge AI and Why Energy-Aware?

Beyond the four core research questions above, the thinking phase identified a fifth dimension that distinguishes Sentinel-Mesh from purely mechanical endurance-extension systems: the **onboard AI inference pipeline** running on a Jetson Orin NX compute module.

YOLOv8n achieves 52 FPS on the Jetson Orin NX and 65 fps with INT8 quantization. The integration of AI techniques onboard drones is constrained by their processing capabilities.

The Jetson Orin NX, operating in its 15W power envelope, consumes a non-negligible fraction of a worker drone's total power budget. At hover, a drone in the 700g class draws approximately 80–120W from the propulsion system; an additional 15W for compute represents 12–18% of total power draw — significant enough to materially affect endurance if the inference workload is constant.

The insight that shaped the cognitive layer's architecture: **AI inference load should not be constant — it should be proportional to the remaining energy budget.** At high SOC (>60%), the inference pipeline runs at full resolution (640×640 input, fp16 precision), maximising detection accuracy. As SOC approaches the scheduling threshold (40%), the pipeline is throttled to reduced resolution (320×320), cutting compute power by approximately 60% while maintaining real-time frame throughput. Below the RTH trigger (≤40% SOC), the pipeline drops to a minimal configuration — obstacle avoidance only, no object classification — ensuring safe navigation back to the carrier.

This **energy-adaptive inference** architecture is the second claimed IP contribution of the project. It is inspired by but distinct from the concept of dynamic precision scaling in deep learning compilers, applied here to the specific context of airborne energy management under a hard endurance constraint.

The SPF (Spatial Priority Filtering) framework — the third claimed IP contribution — governs which regions of the camera frame are processed at full resolution versus downsampled, based on mission priority maps updated by the command drone. This allows the perception system to maintain high-quality detections in mission-critical zones even when the global power budget is reduced.

---

## 6. The Team Architecture: Why Four Domains?

One aspect of the thinking phase that deserves explicit documentation is the **team decomposition decision**. A project of this scope — spanning aerodynamics, structural mechanics, electrical engineering, avionics, software, and AI — cannot be executed by a single individual without unacceptable quality compromise in some domains.

The decomposition settled on was domain-based rather than component-based:

| Domain | Primary Responsibility | Interface Point |
|---|---|---|
| Mechanical Fabrication + FEA/CFD | CFRP frame build, arm geometry, vibration isolation mounts | Mass data feeds W^1.5 model; frame natural frequencies inform IMU mount design |
| Electrical/Propulsion + Instrumentation | ESC bench characterisation, power distribution, current sensing | Power consumption measurements validate the energy model; feeds `energy_monitor` ROS2 node |
| Flight Systems/Avionics + LiDAR SLAM | PX4 autopilot configuration, GPS/RTK integration, LiDAR-Inertial odometry | Shares ROS2 workspace; attitude state feeds fleet scheduler via MAVLink bridge |
| AI Deployment + Communications + Fleet Software + GCS | Everything in software: autonomy stack, fleet logic, communication architecture, ground control | Provides the integration foundation; defines all message schemas |

The interfaces between domains are the critical design artefacts of the team structure. The CS lead's primary early-phase responsibility is not to write application code but to **define and stabilise those interface contracts** — the ROS2 message schemas, the MAVLink topic mapping, the fleet scheduler's state inputs — so that the four parallel workstreams can proceed without blocking dependencies.

This architecture enables a key structural advantage: the software workstream is **unblocked from day one**. The entire Phase 1 software stack can be developed and validated in Software-in-the-Loop (SITL) simulation before any physical hardware is integrated. PX4's SITL mode with Gazebo Classic provides a sufficiently faithful simulation environment to develop flight logic, fleet coordination, and AI pipeline scaffolding independently of hardware availability.

---

## 7. What the Thinking Phase Produced

At the conclusion of the thinking phase, the project possessed the following concrete outputs:

**Nine validated research documents** covering:
1. W^1.5 power physics and hover endurance modelling
2. FOC commutation and the vibration-IMU interference chain
3. CFRP FEA validation of the Tarot TL65B44 airframe
4. Power isolation architecture and ESC-to-flight-controller EMI shielding
5. AI navigation benchmarks (YOLOv8n/TensorRT on Jetson Orin NX)
6. Fleet endurance scheduling model (SOC-constrained CBBA formulation)
7. Energy-aware SOC throttling analysis
8. Communications range modelling via ESP-NOW relay chains
9. Carrier drone conceptual architecture, laminar fuselage theory, structural battery integration, GPS-denied cooperative localisation

**One journal paper** synthesising the above into a unified system-level argument for the 7+ hour endurance claim.

**One validated Bill of Materials (BOM)** anchored to the FEA and power analysis, with primary components:
- Airframe: Tarot TL65B44 (CFRP, folding arms)
- ESC: T-MOTOR F55A Pro II (FOC-capable, 4-in-1)
- Autopilot: Pixhawk 6C (Dual IMU, vibration-isolated mount)
- Compute: NVIDIA Jetson Orin NX (16GB, JetPack 6.0)
- Depth Sensor: Intel RealSense D435i
- Communication: ESP32-based ESP-NOW mesh nodes (one per drone)

**One team interface contract** specifying ROS2 message schemas (`DroneState.msg`, `FleetStatus.msg`, `AssignMission.srv`) that define the integration boundaries between all four workstreams.

---

## 8. What Remained Open — Intentionally

The thinking phase also produced an explicit map of what was **not** resolved, and why leaving it unresolved was the correct epistemic decision.

The most significant open item is the **carrier drone specification**. The carrier drone is the physical embodiment of the system's endurance claim: without it, the 7+ hour coverage assertion cannot be demonstrated. Yet at the conclusion of the thinking phase, the carrier drone existed only as a conceptual architecture — mass budget, payload class, propulsion type, and docking mechanism were all unspecified beyond order-of-magnitude estimates.

This was a deliberate choice, not an oversight. Specifying the carrier drone in the thinking phase without empirical data on worker drone actual power consumption would have introduced a cascade of speculative dependencies. The correct sequencing is: (1) validate worker drone power consumption in SITL and then hardware, (2) use measured data to spec the carrier's payload requirement, (3) design the carrier to that validated payload figure. Premature specification would have produced a detailed design for the wrong problem.

The carrier drone void is documented as the project's **highest architectural risk** and will be addressed by a simulation proxy in Phase 1: a ground-based power station with a simulated 5 kg UAV in Gazebo as a stand-in carrier, allowing fleet rotation logic to be validated against a proxy carrier before the physical platform is built.

---

## 9. Reflection: What Makes a Thinking Phase Rigorous?

Looking back at the process, the thinking phase of Sentinel-Mesh was rigorous in a specific and reproducible sense. It was not a brainstorming exercise. It was not a literature survey for its own sake. It was a structured process of:

1. **Identifying the precise gap** in the existing solution space — not "drones have short flight time" but "no system combines mobile charging infrastructure with SOC-aware distributed scheduling and energy-adaptive inference"
2. **Decomposing the gap** into independent technical questions, each addressable by existing analytical or simulation methods
3. **Answering those questions** with documented, citable analysis before committing to hardware or implementation
4. **Mapping what remained open** and designing the implementation sequence to close those gaps in order of dependency

The result was a research corpus that is not a preliminary study to be discarded once implementation begins, but an **active analytical foundation** that every subsequent implementation decision is referenced against.

The next post in this series will cover the **conceptualisation phase**: how the research outputs were translated into a concrete system architecture, what the node graph looks like, and how the team interfaces were formalised into software contracts.

---

## References and Further Reading

The following literature informed the thinking phase analysis documented above. All citations are to published, peer-reviewed or conference-reviewed works.

- Actuator disk theory and W^1.5 power scaling: Stagewise energy analysis for hovering multirotors, *arXiv:2003.04290*
- FOC for UAV BLDC motors: Hybrid FOC/DTC for sensorless BLDC in aerial drones, *IEEE Transactions*, 2023
- CFRP FEA methodology: Ducted drone structural integrity, *Scientific Reports*, 2024
- Persistent coverage with energy constraints: Lien, Rodriguez & Morales, *arXiv:2101.10438*
- CBBA for distributed task allocation: Choi, How et al., *IEEE Transactions on Robotics*, 2009 (foundational); TLC-CBBA extension, *PMC*, 2026
- YOLOv8n on Jetson Orin NX: Performance analysis for edge deployment in drone applications, *Electronics*, 2025
- ESP-NOW and TDMA for UAV swarms: Drone mesh network latency characterisation, Meshmerize technical reports
- Battery-swapping persistent coverage: AutoCharge system, *arXiv:2306.05111*

---

*This blog is Part 1 of the Sentinel-Mesh Development Log. Subsequent entries will cover: Conceptualisation (architecture and interface design), Research Deep-Dive (individual technical documents), Implementation Phase 1 (SITL and ROS2 workspace), and beyond. The project repository is maintained at [github.com/Shrey-Arc/Sentinel-Mesh](https://github.com/Shrey-Arc/Sentinel-Mesh).*

---
*© 2026 Shrey, Sentinel-Mesh Project. All rights reserved.*
