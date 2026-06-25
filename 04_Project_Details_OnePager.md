# Project Details
## Modular Autonomous UAV Fleet Ecosystem
**One-Pager for Stakeholders, Accelerators & Hackathon Listings**

---

### The Problem in One Sentence

Every commercial drone today tries to be everything — and fails at most of it. Heavy compute, heavy batteries, heavy sensors, all crammed onto a single airframe that runs out of power in 20–38 minutes.

---

### What We're Building

A **distributed aerial operating system** — not a drone, a *fleet*. Three classes of node, each doing what it's best at:

| Node | Mass | Job | Endurance |
|------|------|-----|-----------|
| **Worker** | 0.5–0.8 kg | Carry mission sensors, execute tasks | 45–70 min |
| **Command** | 1.3–1.5 kg | AI inference, fleet coordination, comms relay | 35–50 min |
| **Carrier** | 3–25 kg | Battery logistics, mobile docking, resupply | 60–120 min |

Workers stay light. Carriers keep them flying. Command nodes keep everything smart.

**System-level endurance:** With five worker drones rotating through a carrier-based dock, effective mission coverage extends to **7+ continuous hours** — without any individual drone exceeding its 45-minute window.

---

### The Physics, Briefly

Adding 470 g of payload to a 1.5 kg drone doesn't increase power by 31%. It increases it by **50.5%** — because hover power scales as mass^1.5 (actuator disc theory, independently verified). Our architecture avoids this trap by keeping workers at their lightest possible configuration and offloading everything else.

---

### Key Verified Innovations

**1. Field Oriented Control (FOC) Propulsion**
- 13 dB noise reduction at operating RPM (~50% quieter perceived)
- 11× lower vibration amplitude → directly improves AI navigation accuracy by 44.7%
- Verified in anechoic chamber; at 5 m distance: ≈41 dB SPL (below urban ambient)

**2. CFRP Airframe**
- Safety factor 4.83 vs 2.08 for aluminium — at 59% less weight for the arm set
- First resonant mode (142 Hz) is 70% above motor excitation (83 Hz) — no coupling
- Predicted fatigue life: 2,400 flight-hours vs 820 for aluminium
- 10–15× better vibration damping → cleaner IMU → better SOC estimation → longer useful endurance

**3. Autonomous Docking**
- Flush-mounted magnetic spine on drone underbelly: ±5 cm misalignment tolerance
- Pogo-pin power transfer (10 A continuous), I²C + USB data
- 99% docking success rate in trials
- <2% drag penalty when flying (flush integration)

**4. Near-Human AI Navigation**
- SPF (See, Point, Fly) framework: 92.7% real-world success rate
- Human operator baseline: 95.2% — gap of only 2.5 percentage points
- 67 ms inference latency, viable at flight speeds up to 4.8 m/s
- Architecture: LLM as intent parser → structured plan → classical flight control (never LLM in the control loop)

**5. Fleet Endurance Orchestration**
- Energy-aware SOC scheduling: AI resolution throttles at 40% SOC → +11.4% mission endurance
- EKF battery state estimation: <1.5% RMS error → 3-minute pre-brownout warning
- Isolated 5V/4A compute rail: eliminated 12% flight-controller brownout rate at full payload

---

### Software-Defined Missions

The platform ships with a **modular mission marketplace** — think app store for drone capabilities. Every mission is a software package. Hardware stays fixed; capabilities are licensed and downloaded. Examples: precision agriculture survey, perimeter security, search & rescue, aerial relay comms, environmental sensing.

---

### MVP Scope (Phase 1, 0–6 months)

- 2× Worker drones
- 1× Command node
- 1× Docking station (fixed ground pad)
- 1× GCS interface (mission planning + telemetry)
- **Target cost: ~$6,450 total**

**Phase 1 success criteria:** Stable autonomous flight, hot-swap module detection, dynamic mission mode changes, safe RTH, zero brownouts.

---

### Roadmap

| Phase | Timeline | Milestone |
|-------|----------|-----------|
| Phase 1 — MVP | Months 0–6 | 2 workers + 1 command + dock flying together |
| Phase 2 — Fleet | Months 6–12 | 5-drone swarm, carrier prototype, air-docking |
| Phase 3 — Platform | Months 12–18 | Mission marketplace, SDK, 20+ drone operations |
| Phase 4 — Scale | Months 18–24 | Commercial deployment, regulatory certification |

---

### Who It's For

- **Search & Rescue** teams needing multi-hour persistent aerial coverage
- **Infrastructure inspection** operators (power lines, pipelines, towers)
- **Agriculture** operations requiring wide-area, multi-session surveys
- **Defence / border monitoring** requiring long-endurance, low-noise platforms
- **Smart city / logistics** players building aerial mesh network infrastructure

---

### Why Now

- FOC ESC hardware has reached consumer price points ($65–72/unit)
- Jetson Orin NX hits 100 TOPS at 25 W — AI viable at drone scale
- LiPo energy density plateauing → system architecture is the only remaining lever
- Solid-state batteries (500 Wh/kg, 2027–28 target) will double endurance once they arrive — our fleet architecture will compound that gain further

---

### IP Position

Strongest patent angles: flush docking spine integration, energy-aware AI scheduling (specific SOC thresholds + throttle logic), structural battery arm architecture, fleet-level endurance orchestration protocol. Broad "modular drone fleet" claims are too generic. The value is in the mechanisms.

---

### Team

**Shrey Kumar** — Computing Technologies, SRM IST Chennai
Research lead across propulsion electronics, structural mechanics, AI navigation benchmarking, and energy physics. Published methodology framework covering FOC, CFRP FEA, autonomous navigation, and regulatory compliance (April 2026).

---

### Contact / Links

📧 shreykumarsks@gmail.com
🔗 GitHub: github.com/uav-systems-lab/advanced-uav-research-2026
📄 Technical paper: *Advanced Autonomous Unmanned Aerial Systems* — Confidential Draft, April 2026

---

*All performance figures independently verified against actuator disc theory, FEA stress analysis, and published navigation benchmarks. May 2026.*
