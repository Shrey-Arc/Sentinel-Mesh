# UAV Project Memory File

## Project Identity
Distributed Autonomous UAV Infrastructure Architecture

## Core Principles
- Fleet-level endurance
- Distributed compute
- Lightweight workers
- Hierarchical autonomy
- Persistent aerial operations
- Modular payload ecosystem

## Key Architectural Decisions

### Localization
Hybrid Hierarchical Localization Architecture (HHLA)

Workers:
- IMU
- Optical flow
- VIO-lite

Command node:
- Drift correction
- Global reference
- Map fusion

### Communications
- ESP-NOW TDMA
- LoRa failsafe
- WiFi high-bandwidth layer
- Deterministic scheduling

### Docking
- Funnel-guided capture
- Magnetic locking
- Battery swap preferred

### Flight Stack
Prototype:
- Betaflight

Production:
- PX4

### Worker Drone Goals
Mass:
0.6–0.8 kg

Endurance:
45–60 min

Power budget:
<8 W avionics

### Command Node Goals
Mass:
2.5–4 kg

Functions:
- Localization fusion
- AI tracking
- Coordination

### Manufacturing Strategy
Custom:
- Integration board
- Docking system
- Orchestration software

Off-the-shelf:
- FC
- ESC
- Motors
- IMUs

## Research Priorities
1. Cooperative localization validation
2. TDMA swarm synchronization
3. Autonomous docking
4. Thermal architecture
5. Rotor optimization
6. Scalability testing
7. Security architecture

## Long-Term Objective
Deployable distributed aerial infrastructure ecosystem rather than isolated autonomous drones.
