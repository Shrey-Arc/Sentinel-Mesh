# UAV Project Memory File

## Canonical Project Name
Cooperative Persistent UAV Infrastructure Architecture (CPUIA)

## Core Research Pillars
1. Persistent mission infrastructure
2. Resource-decomposed worker UAVs
3. Cooperative UWB/VIO localization
4. Graceful degradation and recovery
5. Fleet-level endurance orchestration

## Frozen Baseline Architecture

Command Node
│
├── Worker UAVs (3–10)
│
└── Dock Network (2–4 docks)

## Worker UAV Baseline
- PX4
- Pixhawk/Cube class FC
- ESP32-S3 companion
- DW3000 UWB
- 4S2P 21700 Li-ion battery
- 7–8 inch propellers
- 850–950 g target mass

## Command Node Baseline
- Jetson Orin NX
- Mapping
- Scheduling
- Localization assistance
- Health monitoring

## Communication Stack
- MAVLink
- WiFi mesh
- UWB ranging
- LoRa fallback

## Research Priorities
Tier 1:
- Endurance
- Dock reliability
- Localization
- Communications

Tier 2:
- Fleet persistence
- Failure recovery

Tier 3:
- Advanced autonomy

## Rejected Concepts
- Massive swarms
- Distributed compute clouds
- Full GPS-denied autonomy as baseline
- Exotic airframes before validation
- Custom flight controller in early phases

## Publication Roadmap
1. Persistent UAV Infrastructure
2. Dock-Assisted Mission Continuity
3. Cooperative Localization
4. Graceful Degradation
5. Integrated Fleet Demonstration

## Long-Term Vision
Persistent cooperative UAV infrastructure for industrial inspection, monitoring, utilities, mining, ports, and campus-scale autonomous operations.
