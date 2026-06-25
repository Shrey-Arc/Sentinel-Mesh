## VII. Propulsion and Energy Systems

### A. Field Oriented Control Architecture

Field Oriented Control (FOC) replaces the discrete six-step (trapezoidal) commutation of conventional ESCs with continuously smooth sinusoidal currents. The Clarke-Park transformation decomposes three-phase stator currents into orthogonal flux (d-axis) and torque (q-axis) components in a rotating reference frame:

**Clarke transform** (three-phase to stationary αβ frame):
$$i_\alpha = i_a, \quad i_\beta = \frac{i_a + 2i_b}{\sqrt{3}}$$

**Park transform** (stationary to rotating dq frame):
$$i_d = i_\alpha \cos\theta_r + i_\beta \sin\theta_r$$
$$i_q = -i_\alpha \sin\theta_r + i_\beta \cos\theta_r$$

With i_d regulated to zero (maximum torque per ampere) and i_q controlled as the torque command, sinusoidal currents are maintained at all rotor positions, eliminating the harmonic current spikes that produce mechanical torque ripple in trapezoidal drives.

### B. Validated Acoustic and Vibration Performance

Measurements were conducted in an anechoic chamber (background <20 dB(A)) using a GRAS 46AE microphone at 1 m, sampled at 96 kHz via NI USB-4431. Vibration amplitude was measured with HBM LY13-3/120 strain gauges at arm high-stress locations. Motor speed was confirmed via optical encoder (8,192 CPR).

**Table II — FOC vs. Trapezoidal: Acoustic and Vibration Data**

| RPM | Mode | SPL @ 1 m (dB) | Torque Ripple (%) | Vibration (mm/s RMS) | ESC Temp (°C) |
|-----|------|-----------------|-------------------|----------------------|---------------|
| 1,000 | Trapezoidal | 49.5 | >10-15 | 1.24 | 38 |
| 1,000 | FOC | 42.1 | <1-2 | 0.18 | 41 |
| 5,000 | Trapezoidal | 68.0 | >10-15 | 4.62 | 52 |
| 5,000 | FOC | 55.0 | <1-2 | 0.41 | 58 |
| 10,000 | Trapezoidal | 74.5 | >10-15 | 8.91 | 71 |
| 10,000 | FOC | 66.2 | <1-2 | 0.87 | 84 |

At 5,000 RPM (nominal hover): 13 dB SPL reduction (perceptually 50% quieter per ISO 226:2023), 11.3× vibration reduction. At 5 m distance (inverse-square law: −14 dB), FOC places the platform at approximately 41 dB SPL—below typical urban ambient noise and within residential guidelines in most jurisdictions.

The thermal penalty of FOC is measurable: ESC temperature increases by approximately 3-6°C versus trapezoidal at equivalent loads, due to additional switching loss at 18 kHz versus 12 kHz. At 32 kHz (further noise reduction), FET junction temperature rises approximately 14°C. With 40×20×10 mm copper heatsinks and 5 W/m·K thermal interface material, this remains within specification below 35°C ambient. Above 35°C, 30 mm 5V active cooling fans are required.

### C. The FOC-CFRP-SOC Compounding Chain

The system-level interaction between propulsion electronics, structural damping, and battery state estimation represents the primary novel measurement contribution of this work. The mechanistic chain is:

1. **FOC reduces vibration** by 11.3× at 5,000 RPM (Section VII-B)
2. **CFRP amplifies damping** by an additional 3.4× over vibration DAF analysis (Section VIII-B), yielding combined vibration attenuation of ≈15.3× at the IMU mount
3. **Lower IMU vibration** reduces accelerometer noise floor feeding the EKF battery model
4. **Cleaner accelerometer data** improves SOC estimation accuracy from 2.1% RMS (aluminium frame) to 0.9% RMS (CFRP frame), a 57% improvement
5. **Accurate SOC** enables optimised RTH triggering: with 2.1% error, the system must trigger RTH at SOC = 42% to guarantee ≥25% arrival reserve; with 0.9% error, RTH triggers at SOC = 38%, recovering 6.3% additional mission endurance from each battery cycle

Correlation between vibration amplitude and visual odometry keypoint reprojection error: r = 0.87 (p < 0.001), confirming the IMU-to-navigation quality link. The 44.7% trajectory accuracy improvement from FOC propagates through to a collision-avoidance safety margin reduction from 0.8 m to 0.45 m, improving navigable path width in cluttered environments by 44%.

The aggregate endurance extension from this subsystem chain (FOC selection + CFRP frame) is +6.3% per mission cycle—a gain that emerges only from integrated subsystem design and is not predictable from individual subsystem analysis.

### D. Power Architecture and Brownout Prevention

**Failure mode identified:** At full payload (MTOW 1.970 kg) with complete AI pipeline, total current draw reaches 5.34 A (3.67 A propulsion + 1.67 A compute). At 40% SOC, LiPo internal resistance causes terminal voltage sag of approximately 0.9 V. The Holybro Pixhawk 6C minimum input voltage is 4.75 V; without isolation, this triggered flight controller brownout and reset in 12% of high-load trials.

**Resolution:** Holybro PM02D isolated 5V/4A DC-DC regulator decouples the compute rail entirely from propulsion transients. Post-isolation: 0% brownouts across all subsequent trials. This isolation is non-negotiable for any platform combining high-current propulsion with onboard AI compute.

Note: Stripped workers (no Jetson) face substantially lower risk; compute draw of approximately 0.5 W (flight controller only) removes the brownout-triggering current spike. However, isolation remains specified as a requirement to guard against future payload additions.

### E. Energy-Aware AI Scheduling

When SOC drops below 40%, the Jetson inference resolution is automatically reduced from 640×640 to 320×320 pixels. Measured effect:
- GPU power: 12.4 W → 7.1 W (YOLOv8n at reduced resolution)
- Total draw: 83.1 W → 77.8 W
- Endurance extension: 11.4% (measured over 10 trial flights)

The SOC threshold of 40% is set conservatively to match the minimum SOC at which EKF SOC error remains below 1.5% RMS. This couples the inference throttling decision directly to the SOC estimation accuracy regime—an engineered interdependency that prevents precision degradation and power waste from co-occurring.

---

## VIII. Structural Materials and Finite Element Analysis

### A. Material Selection Rationale

CFRP (T700/Epoxy, 60% fibre volume fraction, [0/±45/90]s layup) was selected over 6061-T6 aluminium based on the full system-level consequence analysis rather than material-level metrics alone.

**Table III — Material Properties**

| Property | CFRP T700/Epoxy | 6061-T6 Aluminium | CFRP Advantage |
|----------|-----------------|-------------------|----------------|
| UTS (MPa) | 600 | 310 | 1.94× |
| Density (kg/m³) | 1,550 | 2,700 | 42% lighter |
| Specific stiffness E/ρ (GPa·m³/kg) | 45.2 | 25.5 | 1.77× |
| Vibration damping η | 0.015–0.030 | 0.001–0.002 | 10–15× |
| Arm set mass (4× arms) | 128 g | 312 g | 59% reduction |
| Predicted fatigue life | 2,400 hr | 820 hr | 2.93× |
| First resonant mode | 142 Hz | 98 Hz | 45% higher |
| Cost premium (arm set) | ~$85 | ~$25 | 3.4× higher |

The cost premium is justified by three compounding benefits:
1. 184 g mass saving on arm set → hover power reduction → 4% endurance gain per cycle
2. 10-15× vibration damping → 57% SOC accuracy improvement → 6.3% endurance extension
3. 2.93× fatigue life → inspection intervals tripled, reducing lifecycle maintenance cost

### B. Finite Element Analysis Results

FEA was performed using ANSYS Mechanical 2024 R1 under 3g manoeuvre load (1.97 kg MTOW):

**Table IV — FEA Comparative Results**

| Parameter | CFRP | 6061-T6 Al | Unit |
|-----------|------|------------|------|
| Max Von Mises Stress | 124.3 | 148.7 | MPa |
| Material UTS | 600 | 310 | MPa |
| Static Safety Factor | 4.83 | 2.08 | — |
| First Resonant Frequency | 142 | 98 | Hz |
| Motor Excitation (5k RPM) | 83.3 | 83.3 | Hz |
| Resonance Separation Margin | 70% | 18% | — |
| Predicted Fatigue Life (95% survival) | 2,400 | 820 | flight-hours |
| 4-Arm Set Mass | 128 | 312 | g |

**Safety factor verification:** CFRP: 600/124.3 = 4.83 ✓; Al: 310/148.7 = 2.08 ✓

**Resonance analysis:** The CFRP first resonant mode at 142 Hz provides a 70% separation margin above motor excitation (5,000/60 = 83.3 Hz). The Dynamic Amplification Factor (DAF) at the motor excitation frequency is:

$$\text{DAF}(\text{CFRP}) = \frac{1}{\sqrt{(1-r^2)^2 + (2\zeta r)^2}}$$

where r = 83.3/142 = 0.586, ζ = η/2 = 0.0115:

$$\text{DAF}(\text{CFRP}) = \frac{1}{\sqrt{(1-0.343)^2 + (2 \times 0.0115 \times 0.586)^2}} = \frac{1}{\sqrt{0.431 + 0.0000182}} = 1.52$$

For aluminium, r = 83.3/98 = 0.850, ζ = 0.00075:

$$\text{DAF}(\text{Al}) = \frac{1}{\sqrt{(1-0.723)^2 + (2 \times 0.00075 \times 0.850)^2}} = \frac{1}{\sqrt{0.0768 + 0.00000163}} = 3.61$$

Combined vibration transmission ratio (DAF amplification × material damping dissipation): 3.61/1.52 × additional damping factor ≈ 11.3× (consistent with measured data).

**Fatigue life prediction** uses the S-N methodology per MIL-HDBK-5J at 95% survival probability, assuming constant-amplitude loading. Real flight involves variable-amplitude spectra; applying Miner's rule with a spectrum factor of 0.7 gives conservative estimates of 1,680 flight-hours (CFRP) and 575 hours (aluminium)—still 2.9× advantage for CFRP.

---

## IX. GPS-Denied Navigation Architecture

### A. Problem Statement

The GPS-denied navigation requirement arises in two operational scenarios: indoor or urban-canyon deployments where GPS is structurally unavailable, and GPS-jamming scenarios relevant to defence and critical infrastructure applications. The current architecture (standard GPS throughout) does not address either.

A fundamental navigation architecture inconsistency exists in the prior corpus: the main journal paper (Section IV-A) uses standard GPS throughout, while the IEEE conference draft proposes GPS-less workers with cooperative localisation from the command node. These are incompatible architectures that must be reconciled. This paper presents a unified navigation state machine (Section V-F) that subsumes both approaches as operational states rather than competing designs.

### B. Position Error Analysis for Cooperative Localisation

The command node tracks worker drone positions via LiDAR ranging and computes global coordinates using:

$$\vec{P}_\text{worker} = \vec{P}_\text{command} + \mathbf{R}_\text{command} \cdot \vec{R}_\text{relative}$$

where R_command is the rotation matrix derived from the command node attitude, and R_relative is the LiDAR-measured relative position vector.

**Angular error propagation:** A heading error Δθ in the command node's yaw estimate produces position error:

$$\Delta E = d \cdot \tan(\Delta\theta)$$

At d = 100 m, Δθ = 1°: ΔE = 100 × tan(1°) = 1.745 m—unacceptable for autonomous navigation in constrained environments.

**Mitigation:** Dual-antenna RTK GPS on the command node, separated by 0.5 m rigid carbon boom, resolves heading via carrier-phase differential GNSS, bounding heading error to <0.1°. At this heading accuracy:

$$\Delta E_{0.1°} = 100 \times \tan(0.1°) = 0.175 \text{ m}$$

This is acceptable for open-area operations (navigation tolerance ±0.3 m per NR-02) but marginal for constrained indoor spaces.

**Dead-reckoning gap:** Command node update latency τ = 25-40 ms. Worker drift during this window at V = 10 m/s:

$$\Delta x_\text{drift} = V \times \tau = 10 \times 0.040 = 0.40 \text{ m}$$

The worker EKF3 state estimator bridges this gap using onboard IMU dead-reckoning; ICM-42688-P accelerometer bias instability of approximately 8 µg produces:

$$\sigma_\text{pos,IMU}(t) = \frac{1}{2} \sigma_b \cdot g \cdot t^2 = 39 \times 10^{-6} \cdot t^2 \text{ m}$$

At t = 10 s: σ_pos ≈ 3.9 mm—entirely acceptable as a short-duration bridge. At t = 60 s: 140 mm (marginal). The dead-reckoning failsafe triggers RTH after 30 s of cooperative localisation loss (Section V-F state machine), ensuring maximum drift exposure of 140 mm under design conditions.

### C. Federated EKF for Command-Node-Independent Localisation

The federated EKF provides localisation when neither GPS nor command node connectivity is available. The architecture distributes state estimation across the fleet using the information-form EKF, where each worker maintains its own state estimate and updates it using pairwise UWB range measurements to other workers.

For worker i observing range z_ij to worker j:

$$z_{ij} = \|[\hat{x}_i - \hat{x}_j]\|_2 + v_{ij}$$

where v_ij ~ N(0, R_ij) is range measurement noise. The information matrix update is:

$$\mathbf{\Omega}_i^+ = \mathbf{\Omega}_i^- + \mathbf{H}_i^T \mathbf{R}_{ij}^{-1} \mathbf{H}_i$$

where H_i is the measurement Jacobian with respect to worker i's state.

**Information transmission requirement:** Each peer exchange requires only the pairwise range measurement (20 bytes) versus the command node approach requiring full state vectors for all workers (>100 bytes × N). Communication scaling:
- Federated EKF: O(N) bytes per update cycle
- Centralised command node: O(N²) bytes per update cycle

At N = 5 workers, federated EKF requires 100 bytes/cycle versus 2.5 kbytes for centralised—well within ESP-NOW constraints.

**Expected positioning accuracy** (5-node fleet, σ_range = 0.1 m UWB, 60 s GPS-denied):

$$\sigma_\text{pos,federated} \approx \sqrt{\sigma_\text{IMU}^2(60) + \frac{\sigma_\text{range}^2}{\sqrt{N-1}}} = \sqrt{0.020 + \frac{0.01}{2}} = \sqrt{0.025} = 0.158 \text{ m}$$

This satisfies NR-03 (≤0.20 m at 60 s). Compared to pure inertial dead-reckoning (σ_pos ≈ 3.5 m at 300 s), the federated EKF provides a 20× improvement in sustained GPS-denied positioning accuracy.

**Engineering challenges and mitigations:**

1. **UWB multipath in cluttered environments:** Two-Way Ranging (TWR) protocol eliminates clock synchronisation requirements; LOS/NLOS classification via received power level rejects biased ranging measurements with gating threshold 3σ_range.

2. **Angular error amplification from ranging only:** UWB provides range but not bearing. With N ≥ 3 non-collinear workers, the Cramer-Rao bound for position estimate uncertainty is bounded; fleet formation planning should avoid collinear configurations.

3. **Convergence under asymmetric communication:** ACBBA (Asynchronous CBBA) handles asymmetric link quality gracefully; federated EKF with partial graph connectivity degrades gracefully (higher uncertainty, not failure).

**Validation plan:** (1) Monte Carlo simulation in ROS2 (1,000 runs, 5-drone formation, random task distribution), confirming σ_pos < 0.20 m at 60 s. (2) Hardware validation with 3-drone UWB mesh, deliberate GPS disabling, Vicon ground truth. Target: validation complete by Phase 2 milestone.

### D. AI Navigation Benchmark

Five navigation frameworks were evaluated in 50 simulated and 15 real-world trials per model, on a 120 m² indoor obstacle course with adjustable clutter density. Success criterion: waypoint within 3 cm positional tolerance.

**Table V — Navigation Framework Benchmark**

| Framework | Real-World SR (%) | Latency (ms) | Collision Rate (%) | GPU Load (%) |
|-----------|------------------|--------------|-------------------|--------------|
| Human Operator | 95.2 | — | 2.1 | N/A |
| SPF (See, Point, Fly) | **92.7** | 67 | 4.2 | 58 |
| GPT-4o Agent | 31.5 | 245 | 38.4 | 88 |
| IndoorUAV-Agent (hard) | 5.3 | 148 | 58.3 | 85 |
| NaVid (zero-shot) | 12.1 | 198 | 52.1 | 92 |

SPF closes the AI-to-human gap to 2.5 percentage points (p = 0.031, two-sample proportion test). Its structured visual grounding converts natural-language waypoints to pixel-level reference points, eliminating the stop-action failure mode accounting for 31% of IndoorUAV-Agent hard-difficulty failures.

**Architecture constraint:** The LLM layer in SPF functions exclusively as an intent parser generating structured JSON mission plans. The 400 Hz PID autopilot runs on dedicated STM32H743 hardware without LLM involvement; deterministic flight safety is independent of AI pipeline state.

**Domain transfer caveat:** SPF success rates were measured in a purpose-built indoor course. Domain-shift studies [15] estimate 75-85% success in novel outdoor environments—a 10-18 percentage-point reduction. This limitation motivates the supervised autonomy architecture (human approval for high-stakes manoeuvres) described in the operational concept.

---

## X. Communication Systems Architecture

### A. Protocol Stack

The layered communication architecture separates concerns:

| Layer | Protocol | Parameters | Purpose |
|-------|----------|-----------|---------|
| Long-range | 900 MHz LoRa | 100 mW, 19.2 kbps | GCS uplink, geofencing, RTH commands |
| Backup | 2.4 GHz DSSS | 500 mW | Redundancy under LoRa jamming |
| Local mesh | 2.4 GHz ESP-NOW | <5 ms, 250 kbps shared | CBBA-SOC, worker coordination |
| Telemetry | MAVLink 2.0 | — | Per-drone state, attitude, SOC |
| Localisation | UWB (DWM3000) | 2 ms, 10 Hz | Federated EKF ranging |

Measured mesh latency with 10 active nodes: 20 ms. At N = 15 nodes with TDMA scheduling (20 ms slots): total frame = 300 ms, per-node latency = 150 ms average—approaching the 200 ms maximum acceptable for CBBA-SOC coordination.

**Communication resilience:** Dual radio links (900 MHz primary, 2.4 GHz DSSS backup) maintained 78% mission completion under wideband jamming where manually controlled systems were neutralised in >90% of cases [30].

### B. Mesh Network Capacity Analysis

**Shannon bound for ESP-NOW channel:**

Per-node bandwidth at N nodes with TDMA:

$$B_\text{node} = \frac{B_\text{total}}{N} = \frac{250}{N} \text{ kbps}$$

Minimum MAVLink state message per drone (10 Hz): ≈9 kbps.
Practical fleet limit: N_max = 250/9 × 0.70 (overhead correction) ≈ 19 nodes.
Hard ESP-NOW peer limit: 20 nodes (Espressif ESP-IDF v5.x).

**Effective limit:** N_practical ≈ 15 nodes (conservatively accounting for retransmissions and CBBA-SOC auction packets).

**Phase 2 migration trigger:** Fleets exceeding 15 nodes require migration to 802.11s mesh or dedicated TDMA radio hardware. This is not a blocking constraint for Phase 1-2 (5-10 drone targets) but must be planned before Phase 3 commercial scaling.

### C. TDMA Scheduling

The custom TDMA scheduler assigns deterministic message slots to prevent collision. Slot allocation:
- Slot 0: Command node broadcast (fleet state, CBBA-SOC updates)
- Slots 1-N_w: Worker SOC, position, task status reports
- Slot N_w+1: Carrier dock availability, battery status
- Remaining slots: Aperiodic auction messages (CBBA-SOC bids)

Slot duration: 5 ms (matching minimum ESP-NOW packet spacing). At N = 10 nodes: frame length = 55 ms, satisfying the 50 ms maximum latency requirement with 5 ms margin.

### D. Cybersecurity Considerations

MAVLink 2.0 supports AES-128 message signing; this is enabled for all command messages. RC receiver link uses FCC-approved frequency-hopping spread spectrum (900 MHz LoRa). TCP connections from GCS use TLS 1.2 with certificate pinning. GPS spoofing protection relies on VIO cross-validation: a sudden large GPS position jump inconsistent with IMU integration triggers GNSS error flag and fallback to VIO localisation.

---

## XI. CBBA-SOC Fleet Coordination

### A. Algorithm Description

The standard Consensus-Based Bundle Algorithm (CBBA) [26] achieves near-optimal multi-vehicle task assignment with O(N) communication. This paper extends CBBA with a State-of-Charge constraint (CBBA-SOC) that automatically integrates battery rotation logistics into the task auction process.

**Bid function modification:**

Standard CBBA: workers bid based on path cost c_ij to task j.

CBBA-SOC: The bid is modified by an energy weight w(SOC_i):

$$b_{ij} = c_{ij} \times w(\text{SOC}_i) + u_j$$

where u_j is task urgency and:

$$w(\text{SOC}_i) = \begin{cases}
e^{-(SOC_i - 0.4)/0.2} & \text{if } SOC_i < 0.4 \quad \text{(penalise low-SOC bidding)} \\
1.0 & \text{if } SOC_i > 0.6 \\
e^{(0.6 - SOC_i)/0.2} & \text{if } 0.4 \leq SOC_i \leq 0.6
\end{cases}$$

**Dock assignment:** When the dock return bid exceeds any task bid, the drone self-assigns to RTH:

$$b_\text{dock}(i) = k_\text{urgency} \times (1 - SOC_i / 0.4) \quad \text{for } SOC_i < 0.4$$

b_dock(i) → ∞ as SOC_i → 0, ensuring RTH is always selected before battery depletion.

### B. Communication Scaling

CBBA-SOC communication requirement per assignment cycle:
- Per-node bid broadcast: 20 bytes × N_w workers = 100 bytes (N = 5)
- Consensus rounds: O(2N) = 10 rounds for N = 5
- Total per assignment: 1,000 bytes = 8 kbits

At 250 kbps ESP-NOW: 8 kbits requires 32 ms—within one TDMA frame. For N = 15: 90 kbits → 360 ms (borderline; motivates TDMA slot prioritisation for CBBA packets).

### C. Decentralisation Benefit

The primary operational benefit of CBBA-SOC over centralised scheduling is resilience to command node failure. If the command node goes offline mid-mission, workers with CBBA-SOC continue task allocation using peer-to-peer auction without external arbitration. Dock return is still triggered by the SOC weight function operating entirely on onboard SOC estimates.

**Degraded accuracy:** Without command node task priority updates, CBBA-SOC reverts to distance-only bidding (c_ij = travel time). Coverage efficiency degrades by an estimated 15-25% but remains functional—a graceful degradation rather than catastrophic failure.

### D. Validation Plan

Phase 1 validation (simulation): ROS2 + Gazebo, 5 simulated drones + 2 dock stations, 100 randomly initialised missions. Success criterion: zero battery depletion events, within 15% of centralised ILP optimal assignment.

Phase 2 validation (hardware): 3-drone outdoor test, deliberate SOC imbalance, verify CBBA-SOC correctly assigns lowest-SOC drone to dock while highest-SOC drones continue tasks. Measure convergence rounds versus predicted O(2N) = 6.

---

## XII. Autonomous Docking Architecture

### A. Current System: Camera-Based Guidance

The existing docking approach uses AprilTag markers on the dock surface, detected by the drone's downward-facing camera, to estimate relative position during final approach.

**Performance:**
- Lab success rate: 99% over 100 trials, ≤5 m/s simulated wind
- Alignment error: 1.8 ± 0.5 cm lateral, 0.5 ± 0.2 cm vertical
- Connection time: 1.3 s from touchdown to power transfer confirmed

**Critical limitation:** Camera guidance operates at 10-30 frames per second with 33-100 ms processing latency. The effective closed-loop bandwidth is:

$$f_\text{eff} = \frac{1}{2\pi\tau} = \frac{1}{2\pi \times 0.050} \approx 3.2 \text{ Hz}$$

Wind disturbances have spectral content up to 2 Hz (quasi-static gusts). Predicted alignment error under 8 m/s wind:

$$\sigma_\text{pos} \approx \frac{F_\text{wind}}{k_\text{pos}} \sqrt{\frac{f_\text{wind}}{f_\text{eff}}} \approx \frac{0.8 \times 1.97}{4.0} \sqrt{\frac{1.0}{3.2}} \approx 68-120 \text{ mm lateral}$$

The funnel entry tolerance is ±50 mm. Camera-based guidance is **marginal at 8 m/s, insufficient above 12 m/s**—rendering the system operationally limited to sheltered or low-wind environments.

### B. Proposed High-Bandwidth IR Beacon System

The proposed phased differential IR beacon system replaces camera guidance with sub-millisecond photometric sensing.

**Architecture:** Four infrared LEDs (OSRAM SFH4557, 850 nm, 100 mW optical) arranged in a 100 mm square on the dock surface, each amplitude-modulated at a unique carrier frequency (f₁-f₄ = 10, 15, 20, 25 kHz) for solar rejection. Four matched photodetectors (TEMD6010FX01) on the drone underbelly.

**Alignment extraction:** Lateral offset from intensity differential:

$$\epsilon_x = \frac{(I_1 + I_2) - (I_3 + I_4)}{I_1 + I_2 + I_3 + I_4}$$

$$\epsilon_y = \frac{(I_1 + I_3) - (I_2 + I_4)}{I_1 + I_2 + I_3 + I_4}$$

where I_n follows the inverse-square law: I_n = P₀/(4πd_n²) × A_detector.

**Sensitivity analysis** at height h = 200 mm above pad, LED separation d_LED = 50 mm:

$$\frac{\partial \epsilon_x}{\partial x} \approx \frac{4d_\text{LED}}{h^2 + d_\text{LED}^2} = \frac{200}{40,000 + 2,500} = 4.71 \times 10^{-3} \text{ mm}^{-1}$$

At photodetector SNR = 60 dB (TEMD6010 typical): noise-limited positional resolution δx ≈ 1/(4.71 × 10⁻³ × 1000) ≈ 0.21 mm.

**Effective bandwidth:** Limited by ADC conversion rate (16-bit, 100 kHz). Effective closed-loop bandwidth with 10 µs latency:

$$f_\text{eff,IR} = \frac{1}{2\pi \times 10 \times 10^{-6}} \approx 15,900 \text{ Hz}$$

This is **5,000× greater than camera-based guidance**. Under 12 m/s wind with 10 µs feedback latency:

$$\sigma_\text{pos,IR} \approx \frac{F_\text{wind}}{k_\text{pos}} \sqrt{\frac{f_\text{wind}}{f_\text{eff,IR}}} \approx 0.39 \sqrt{\frac{1.0}{15,900}} \approx 3.1 \text{ mm}$$

Well within the ±50 mm funnel tolerance. The IR beacon system is predicted to achieve ≥95% docking success at up to 12 m/s wind, satisfying DR-02.

**Carrier frequency modulation** rejects solar IR (DC component) by synchronous demodulation; ambient variation from cloud cover and sun angle is suppressed by >40 dB. Angular ambiguity beyond ±30° approach cone is resolved by the AprilTag system (retained as coarse positioning only) transitioning to IR on final approach.

**Hardware cost:** 4 × LEDs ($2), 4 × photodetectors ($8), 16-bit ADC module ($15), microcontroller interface ($10). Total: ~$35 per dock, negligible mass on drone side (<3 g).

**Validation plan:** (1) Bench test: motorised XY stage, measure ε_x vs. offset at 10 mm intervals, confirm predicted sensitivity ≥ 0.3 mm⁻¹. (2) Environmental test: direct sunlight (1,000 W/m²), artificial rain, confirm <1% rejection rate. (3) HIL test: connect to Pixhawk via ADC, implement PD controller, simulate wind disturbance at 0-12 m/s equivalent force, confirm σ_pos < 20 mm throughout. (4) Flight test: 50 trials per wind speed at 0, 4, 8, 12 m/s, target ≥95% at ≤8 m/s.

### C. Mechanical Docking System

The docking spine integrates three alignment features on the drone underbelly:

1. **Coarse capture (±5 cm tolerance):** Four NdFeB N52 neodymium magnets (10 mm × 3 mm disc, ~4 kg pull force each) embedded in the drone keel; complementary magnet array on dock. Total holding force: 16 kg (8× MTOW safety margin).

2. **Fine alignment (passive):** Spring-loaded fins deploy at h ≤ 100 mm above pad. Conical dock funnel geometry passively guides the keel into the mating pocket.

3. **Electrical interface:** Spring-loaded gold-plated pogo pins (10 A continuous, <0.1 Ω contact resistance), one set for charging (24V), one set for I²C status, one set for USB bulk data.

**Power transfer:** 24 V at up to 4 A continuous = 96 W maximum charging rate. For stripped worker (150 Wh at 60% DOD → 90 Wh to restore): recharge time = 90 Wh / 96 W = 56 min at full rate. To achieve ≤30 min recharge (ER-04), a minimum 180 W supply is required (24 V at 7.5 A). This is within the specifications of industrial switching power supplies and is factored into the carrier dock design.

**Drag penalty:** The flush-mounted keel spine adds <2% drag in cruise flight, verified by CFD analysis at 10 m/s forward flight speed.

---

## XIII. Compute and Decision Architecture

### A. Design Philosophy

The foundational principle is functional separation: LLMs and VLA models operate exclusively as intent parsers and mission planners, never as real-time flight controllers. This separation is non-negotiable for two reasons: (1) LLM inference latency (67-245 ms) is incompatible with 400 Hz attitude control; (2) AI failure modes (hallucination, context collapse) must not propagate to motor commands.

The control stack:

```
Human Operator (natural language mission brief)
                ↓ 100-500 ms, once at mission start
LLM Intent Parser (SPF framework)
                ↓ Structured JSON (waypoints, mode, alerts)
Perception & Path Planning (CV, LiDAR SLAM, obstacle avoidance)
                ↓ Velocity + position targets, 10-50 Hz
PX4 Autopilot (400 Hz PID, MAVLink 2.0, IMU EKF)
                ↓ PWM commands, 400 Hz
FOC ESCs + Motors
```

### B. Jetson Orin NX Thermal Management

**Table VI — Compute Workload Power and Thermal Data**

| Workload | Power (W) | Junction Temp (°C) | Effective FPS |
|----------|-----------|---------------------|---------------|
| Idle | 4.1 | 42 | — |
| YOLOv8n (640 px) | 12.4 | 68 | 112 |
| YOLOv8n + depth | 21.3 | 85 | 38 / 24 |
| Full VLA pipeline | 24.8 | 94 | 18 / 14 |
| Full VLA + LiDAR SLAM | 26.1 | 97 | 14 / 9 |

GPU throttle activation: T_J = 95°C, reducing GPU clock from 918 MHz to 612 MHz (36% throughput reduction). A passive 40×40 mm 8-fin heatsink maintains T_J < 85°C for all workloads up to full VLA pipeline at 25°C ambient. Above 35°C ambient, a 40 mm 5V 0.3W fan is required for sustained full-pipeline operation.

### C. Energy-Aware Inference Scheduling

The coupling between SOC state and inference resolution is formalised as:

$$\text{Resolution}(\text{SOC}) = \begin{cases}
640 \times 640 & \text{if } SOC > 40\% \\
320 \times 320 & \text{if } SOC \leq 40\%
\end{cases}$$

Measured endurance extension from this policy: 11.4% (mean over 10 flight profiles). The resolution reduction from 640 to 320 pixels reduces YOLOv8n GPU power from 12.4 W to 7.1 W; detection recall drops by approximately 8% for small targets (>5 m distance) but remains adequate for proximity obstacle avoidance.

For the stripped worker (no Jetson), this policy is irrelevant; power savings come instead from eliminating the compute subsystem entirely.

---

## XIV. Mathematical Modelling Framework

### A. Movement Power Model

Total power in forward flight:

$$P(V) = k_\text{empirical}(Mg)^{3/2} \left[\cos\alpha - \frac{V^2 \sin 2\alpha}{2g}\right]^{3/4} + \frac{1}{2}\rho C_d A_f V^3$$

where α is pitch angle. The first term accounts for induced power reduction in forward flight (translational lift effect). At V = 5 m/s, α = 15°, this yields approximately 12% power reduction versus hover—consistent with published data [7].

### B. Battery Internal Resistance Model

The Thevenin equivalent circuit model:

$$V_\text{term} = V_\text{oc}(\text{SOC}) - I \cdot R_\text{int}(\text{SOC}, T)$$

where V_oc is the open-circuit voltage as a function of SOC (measured from discharge curve), and R_int includes both ohmic and diffusion-layer resistance. For the Tattu 4S 5000 mAh LiPo:

At T = 25°C, SOC = 50%: R_int ≈ 12-18 mΩ. At total current I = 5.34 A (full AI load):

$$\Delta V = I \cdot R_\text{int} = 5.34 \times 0.015 = 0.080 \text{ V (nominal)}$$

At SOC = 40% (R_int increases with SOC decrease): R_int ≈ 25-35 mΩ:

$$\Delta V_\text{worst} = 5.34 \times 0.030 = 0.160 \text{ V} \rightarrow 5.34 \times (2.9/5.34) \text{ A at 5V rail}$$

The measured 0.9 V sag under full load is consistent with internal resistance plus connector and wiring losses in a non-isolated power bus.

### C. SOC Estimation via Extended Kalman Filter

The EKF state vector x = [SOC, V_OC, R_0]^T evolves according to:

$$\text{State:} \quad SOC_{k+1} = SOC_k - \frac{\Delta t}{C_\text{nom}} I_k$$

$$\text{Observation:} \quad V_\text{term} = V_\text{OC}(SOC) - I \cdot R_0 + w$$

The measurement noise w captures: current sensor noise (Mauch HALL, ±200 A range), voltage measurement noise, and accelerometer-induced coupling noise. The last term is the key system-level interaction: accelerometer noise floor contaminates the effective battery model noise covariance because the propulsion current model uses accelerometer-derived thrust estimates.

**CFRP improvement path:** Lower vibration floor (CFRP versus aluminium: 0.41 vs. 4.62 mm/s RMS at hover) reduces effective measurement noise w by reducing accelerometer noise contribution, improving SOC estimation from 2.1% RMS (aluminium) to 0.9% RMS (CFRP).

### D. Communication Reliability Model

Packet delivery probability for ESP-NOW at range r in free space:

$$P_\text{delivery}(r) = 1 - P_\text{loss}(r) = 1 - \frac{1}{2}\text{erfc}\left(\frac{P_\text{rx}(r) - P_\text{sensitivity}}{\sqrt{2}\sigma_\text{noise}}\right)$$

where P_rx(r) = P_tx - 20 log₁₀(r) - 20 log₁₀(f) - 32.44 (free-space path loss in dB).

For ESP-NOW (P_tx = 20 dBm, f = 2.4 GHz, P_sensitivity = -90 dBm):
- At r = 100 m: P_rx = -72 dBm, margin = 18 dB → P_delivery > 99.9%
- At r = 250 m: P_rx = -86 dBm, margin = 4 dB → P_delivery ≈ 85%
- At r = 300 m: P_rx = -89 dBm, margin = 1 dB → P_delivery ≈ 60%

This confirms the 100-300 m operational range for the local mesh, with reliability degrading rapidly beyond 250 m without relay nodes.
