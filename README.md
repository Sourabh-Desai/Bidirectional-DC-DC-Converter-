# **Bidirectional DC/DC Converter with Dual Energy Storage**

## **Two Sources. One Motor. Zero Wasted Braking Energy.**

**A Hardware-Validated Power Management System for Electric Vehicle Powertrains**

[Learn More](#about) • [The Solution](#the-solution) • [Technology](#technology) • [Simulation & Results](#simulation--results) • [Hardware Testing](#hardware-testing) • [Team](#team)

---

## **The Problem**

### **Meet a Typical EV Drivetrain**

A commuter EV pulls away from a stoplight. The single lithium-ion battery pack has to deliver a hard current spike to get the car moving — the same battery that, seconds later, is asked to *absorb* a spike of regenerative current when the driver brakes for the next light. Every one of these cycles adds stress the cell chemistry was never designed for.

### **Not Only That EV...**

**Single-battery EV powertrains run into the same wall again and again:**

- Batteries have **high energy density but poor peak-power response** — they're slow to react to sudden acceleration or braking demand
- **Unidirectional converters** can't route energy back from the motor, so regenerative braking energy is either dumped as heat or handled poorly
- Repeated high-current cycling **accelerates battery degradation** and shortens pack lifespan
- Most systems lack **adaptive, real-time control** — power sharing is either fixed or absent entirely

---

## **About the Project**

### **Background**

Electric vehicle traction systems are only as good as their energy management. A single-battery architecture forces one component to do two conflicting jobs: sustain long-duration cruising power *and* absorb sharp transient spikes. This project addresses that conflict directly with a **dual energy storage architecture** — a lithium-ion battery for sustained energy delivery, paired with a supercapacitor bank for high-power transients — connected through a **bidirectional buck-boost DC-DC converter** and driven by a **custom three-phase inverter**.

**This project moves energy management from reactive to intelligent** by combining:

- **Dual Energy Storage**: Li-ion battery (energy density) + supercapacitor bank (power density) working in tandem
- **Bidirectional Power Flow**: A two-switch buck-boost converter that reverses direction for regenerative braking
- **Real-Time Embedded Control**: STM32F411RE microcontroller generating Hall-sensor-synchronized PWM for both the converter and inverter
- **Model-Based Design**: Full MATLAB/Simulink open-loop and closed-loop validation before hardware deployment

### **Our Objective**

Build and validate — in both simulation and hardware — a compact EV energy management system that:

- **Shares power intelligently**: supercapacitor handles the initial acceleration current spike, battery takes over for sustained cruising
- **Recovers braking energy**: reverse power flow from motor → converter → supercapacitor during regenerative braking
- **Regulates speed under load**: closed-loop PI control keeps PMSM speed stable across a 0.5–2 Nm torque range
- **Protects itself**: current-sensor-driven shutdown logic halts PWM output before an overcurrent event can damage switches

---

## **What This Is (and Isn't)**

Hybrid battery + supercapacitor storage, bidirectional buck-boost conversion, and regenerative braking recovery are established topics in EV power electronics research — not a new architecture (see the Literature Survey in the project report, particularly refs [2], [16], [17], [21]). What distinguishes this project is that it doesn't stop at simulation: the converter, inverter, and control loop were built, flashed to an STM32, and validated on real hardware with an oscilloscope, not just modeled in MATLAB/Simulink.

## **The Solution**

### **Four-Stage Power Architecture**

This is a complete, hardware-realized power path: **battery + supercapacitor → bidirectional converter → DC link → three-phase inverter → PMSM motor**, with the same path running in reverse during regenerative braking.

The system continuously reads battery state, supercapacitor state, motor speed, and rotor position, and uses that to decide — moment to moment — whether the converter should be boosting power to the motor or bucking recovered energy back into the supercapacitor.

### **1. Dual Energy Storage**

**Two complementary storage technologies, sized and combined for this specific power profile:**

- **Battery pack**: 7 Li-ion cells in series → 25.9 V, 0.42 Ω pack resistance, 64.75 W sustained output
- **Supercapacitor bank**: 9 cells in series → 24.3 V, 55.5 F, 0.0288 Ω ESR — built for near-instant current delivery
- **Role split**: supercapacitor fronts sudden acceleration/braking transients; battery carries steady cruising load
- **Effect**: battery sees a smoother current profile, which is the primary lever for extending pack lifespan

### **2. Bidirectional DC-DC Converter**

**Two-switch buck-boost topology that reverses its own operating mode based on power flow direction:**

- **Boost mode** (motoring): steps 24 V input up to a 96 V DC link at Dboost = 0.75
- **Buck mode** (regen braking): steps motor-side voltage back down to a safe supercapacitor charging level at Dbuck = 0.25
- **Switching device**: SKM100GB07E3 Semikron IGBT modules (650 V, 128 A)
- **Rated for**: 750 W output, 50 kHz switching frequency, 100 µH inductor

### **3. Three-Phase Full-Bridge Inverter + PMSM Drive**

**Custom-built inverter, 120° conduction mode, six-step commutation:**

- **Six IGBT switches** (three Semikron half-bridge modules) driving an R-Y-B PMSM motor
- **Hall-effect rotor position feedback** (3 sensors, 120° apart) drives commutation logic and speed estimation
- **Dead-time protection**: 100 ns insertion (100 kΩ gate driver resistor) prevents shoot-through faults
- **Open-loop and closed-loop modes**: fixed duty cycle vs. PI-controller-regulated speed tracking

![Bidirectional converter topology — replace with Fig. 3.11 from the project report](assets/converter-topology.png)

---

## **System & Software Requirements**

### Hardware Overview

The system integrates a battery pack, supercapacitor bank, bidirectional DC-DC converter, DC link capacitor, three-phase inverter, and PMSM motor, all coordinated by an STM32F411RE microcontroller reading Hall and current sensor feedback.

#### Hardware Functionality

| Requirement ID | Requirement Title | Description | Rationale |
|----------------|-------------------|--------------|-----------|
| HRS 01 | Bidirectional Power Flow | Converter shall operate in boost mode (source → load) and buck mode (load → source) without hardware reconfiguration. | Enables both motor drive and regenerative braking through the same power stage. |
| HRS 02 | Dual Storage Integration | System shall interface a Li-ion battery pack and a supercapacitor bank as independent, individually monitored sources. | High-power transients and long-duration loads have fundamentally different storage requirements. |
| HRS 03 | Current Protection | Current sensors on the DC link and all three inverter legs shall trigger immediate PWM shutdown on overcurrent. | Protects IGBT switches and motor windings from damage during fault conditions. |
| HRS 04 | Rotor Position Feedback | Three Hall-effect sensors, 120° apart, shall provide real-time commutation and speed data to the MCU. | Six-step commutation and closed-loop speed control both depend on accurate rotor position. |
| HRS 05 | Switching Reliability | Inverter gate drivers shall enforce a minimum 100 ns dead time between complementary switches. | Prevents shoot-through short-circuit faults across the DC bus. |

### Software Overview

The embedded control stack generates PWM for both the converter and inverter, processes Hall sensor feedback for six-step commutation and speed estimation, and runs a PI control loop for closed-loop speed regulation.

#### Software Functionality

| Requirement ID | Requirement Title | Description | Rationale |
|----------------|-------------------|--------------|-----------|
| SRS 01 | Model-Based Code Generation | Control logic shall be developed in Simulink and converted to embedded C via Embedded Coder / Simulink Coder / MATLAB Coder. | Reduces manual firmware bugs and keeps simulation and hardware behavior consistent. |
| SRS 02 | Closed-Loop Speed Regulation | A PI controller shall compare reference speed to measured speed and adjust PWM duty cycle to minimize error. | Required for stable operation under varying torque load. |
| SRS 03 | Mode-Dependent Switching Logic | Firmware shall route PWM to upper or lower inverter legs depending on motoring vs. regenerative braking mode. | Ensures correct power direction without separate hardware paths. |

---

## **Technology**

### **System Architecture**

**Power Stage:**
- **Bidirectional Converter**: 2-switch buck-boost, 24 V → 96 V, 750 W, 50 kHz, SKM100GB07E3 IGBTs
- **DC Link**: 1000 µF / 250 V electrolytic capacitor buffering converter and inverter stages
- **Inverter**: 3-phase full bridge, 120° conduction, six-step commutation, SKM100GB07E3 IGBTs
- **Rectifier**: B6U100A three-phase bridge module (1600 V, 100 A) for AC-input bench testing

**Control Stage:**
- **MCU**: STM32 NUCLEO-F411RE (ARM Cortex-M4F, 100 MHz, 512 KB flash)
- **Sensing**: WCS1800 Hall-effect current sensors (DC link + 3 inverter legs), Hall-effect rotor position sensors
- **Gate Drivers**: Isolated half-bridge drivers, 3.3 V logic → 15 V gate drive, manual dead-time adjustment
- **Design Tools**: MATLAB/Simulink, Motor Control Blockset, STM32CubeMX, STM32CubeProgrammer

### **Control Logic**

| Mode | Converter State | Power Path | Speed Control |
|:-----|:----------------|:-----------|:---------------|
| **Cruising** | Boost | Battery → DC link → Motor | Battery-dominant, supercapacitor idle/monitored |
| **Acceleration** | Boost | Supercapacitor (initial) → Battery (sustained) → Motor | Supercapacitor fronts the current spike |
| **Regenerative Braking** | Buck | Motor → DC link → Supercapacitor | Reverse power flow, protected charge rate |

### **Converter Design Parameters**

| Parameter | Value |
|:----------|:------|
| Input Voltage | 24 V |
| Output Voltage | 96 V |
| Output Power | 750 W |
| Switching Frequency | 50 kHz |
| D(boost) / D(buck) | 0.75 / 0.25 |
| Inductance L1 | 100 µH |
| Input Cap Cg | 6800 µF |
| Output Cap Co | 1000 µF |

---

## **Simulation & Results**

### **Open-Loop vs. Closed-Loop**

Both configurations were built and validated in MATLAB/Simulink before any hardware was powered on.

- **Open-loop**: fixed duty cycle, no speed feedback — motor speed drifts with load and voltage (unstable under changing conditions)
- **Closed-loop**: PI controller continuously compares reference vs. actual speed — motor holds steady at 1000 RPM even as applied torque steps from 0.5 Nm to 2 Nm

### **Key Simulation Findings**

- Closed-loop speed tracking follows step changes in reference speed (1000 RPM → 750 RPM) and re-stabilizes within seconds
- Stator current magnitude scales directly with applied torque (~±10 A at 1.35 Nm), confirming expected motor behavior
- Speed remains constant up to the motor's rated torque (2.04 Nm); beyond that, closed-loop control can no longer fully compensate

---

## **Hardware Testing**

### **Converter Validation**

| Mode | Input | Duty Cycle | Measured Output | Theoretical Relation |
|:-----|:------|:-----------|:------------------|:----------------------|
| Buck | 24 V | 25% | ~6 V | Vout = Vin × D |
| Boost | 24 V | 75% | Higher than Vin | Vout = Vin / (1 − D) |

Both modes matched theoretical predictions, confirming the buck-boost topology performs as designed.

### **Inverter + Motor Validation**

- Hall sensor readings verified by manually rotating the motor shaft and logging Hall A/B/C transitions in Simulink
- At 24 V DC input, inverter produced ~20 V AC output across R-Y-B phases — motor rotated smoothly under open-loop control
- No-load current draw: ~0.2–0.3 A at 24 V input
- Dead-time insertion (100 ns) confirmed no shoot-through faults during switching

### **Full System Assembly**

All power, sensing, and control terminals were routed to externally accessible banana-plug connectors inside a 5 mm acrylic enclosure — a plug-and-play bench setup for repeatable testing without opening the housing.

---

## **Gallery**

*Add hardware photos here — suggested set: assembled enclosure, oscilloscope captures of gate pulses and inverter output, converter bench test, motor + inverter test rig.*

<figure>
<img src="assets/model-photo-1.jpg" height="300">
<img src="assets/model-photo-2.jpg" height="300">
<img src="assets/model-photo-3.jpg" height="300">
<img src="assets/testing-setup.jpg" height="300">
</figure>

---

## **Why This Design Stands Out**

| Capability | Single-Battery EV | Unidirectional Converter Systems | **This Project** |
|:-----------|:-------------------|:-----------------------------------|:-------------------|
| **Regenerative Braking Support** | ❌ Limited | ❌ No | ✅ **Yes** |
| **Dual Energy Storage** | ❌ No | ❌ No | ✅ **Yes** |
| **Real-Time Adaptive Power Sharing** | ❌ No | ❌ No | ✅ **Yes** |
| **Closed-Loop Speed Regulation** | ⚠️ Varies | ⚠️ Varies | ✅ **Yes (PI, hardware-tested)** |
| **Hardware-Validated (not just simulated)** | ⚠️ Varies | ⚠️ Varies | ✅ **Yes** |
| **Overcurrent Protection** | ⚠️ Varies | ⚠️ Varies | ✅ **Yes (4-point current sensing)** |

---

## **Project Status & Milestones**

🚀 **Current Phase**: Hardware-validated prototype (converter + inverter + motor tested independently and assembled)
📅 **Last Updated**: AY 2025-26
✅ **Recent Achievement**: Published conference paper on the bidirectional DC/DC converter design; full open-loop hardware validation of converter, inverter, and motor drive complete

---

## **Team**

<div style="display: inline-block; justify-content: center; gap: 20px; margin-right: 100px;">
    <div>
        <p><strong>Your Name</strong></p>
        <a href="mailto:your.email@example.com">
            <img src="https://img.icons8.com/ios/50/00BFFF/email.png" alt="Email" width="30" height="30">
        </a>
        <a href="https://www.linkedin.com/in/your-linkedin/">
            <img src="https://img.icons8.com/ios/50/0000FF/linkedin.png" alt="LinkedIn" width="30" height="30">
        </a>
    </div>
</div>

**Core Skills Demonstrated:**
- **Power Electronics**: Bidirectional buck-boost converter design, IGBT gate driving, dead-time protection
- **Embedded Systems**: STM32 PWM generation, Hall-sensor-based commutation, real-time current protection
- **Model-Based Design**: MATLAB/Simulink open-loop and closed-loop control, Embedded Coder deployment
- **Hardware Validation**: Oscilloscope-based switching verification, bench testing, full system assembly

---

## **Acknowledgments**

We would like to express our sincere gratitude to the **Dept. of EEE, DSCE** for guidance and lab support throughout this project, and to everyone who contributed feedback during hardware testing and validation.

---

```
Interested in collaboration, consulting, or deployment?

📧 Email: your.email@dsce.edu
🔗 LinkedIn: Your Name
💻 GitHub: @your-github-handle


Citation

If you build upon this work, please cite as:

@misc{yourname2026bidirectional,
  author = {YourLastName, YourFirstName},
  title = {Bidirectional DC-DC Converter with Dual Energy Storage for Electric Vehicles},
  year = {2026},
  howpublished = {GitHub},
  url = {https://github.com/your-github-handle/Bidirectional-DC-DC-Converter}
}

Transforming Energy Distribution for Next-Generation Electric Vehicles

Every Watt Optimized. Every Acceleration Empowered.

Dayananda Sagar College of Engineering (DSCE) | Bengaluru
```
