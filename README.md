# VSD Bandgap Reference Design Workshop
### Sky130 PDK | ngspice | Magic | Netgen

> A complete analog IP design walkthrough — from circuit theory to pre-layout simulation, physical layout, and LVS verification — using open-source tools.

---

## Objective

To design and simulate a temperature-stable Bandgap Reference circuit using SkyWater 130nm PDK, covering the complete flow from SPICE-level pre-layout simulation through physical layout and LVS verification in open-source EDA tools.

---

## Introduction

A Bandgap Reference (BGR) is a fundamental analog circuit block that generates 
a stable DC voltage independent of temperature, supply voltage, and process 
variations. It is a critical component in virtually every mixed-signal SoC — 
powering LDOs, ADCs, DACs, and DC-DC converters. This workshop walks through 
the end-to-end design of a BGR using the open-source SkyWater 130nm PDK, 
from understanding the underlying CTAT/PTAT physics to running pre-layout 
simulations in ngspice, drawing the physical layout in Magic, and verifying 
it with Netgen LVS.

---

## Table of Contents

1. [Why a Bandgap Reference?](#1-why-a-bandgap-reference)
2. [Bandgap Reference — Theory](#2-bandgap-reference--theory)
   - 2.1 [The BGR Principle](#21-the-bgr-principle)
   - 2.2 [CTAT Voltage Generation](#22-ctat-voltage-generation)
   - 2.3 [PTAT Voltage Generation](#23-ptat-voltage-generation)
   - 2.4 [BGR Types](#24-bgr-types)
3. [Self-Biased Current Mirror Based BGR](#3-self-biased-current-mirror-based-bgr)
   - 3.1 [Self-Biased Current Mirror](#31-self-biased-current-mirror)
   - 3.2 [Reference Branch Circuit](#32-reference-branch-circuit)
   - 3.3 [Start-up Circuit](#33-start-up-circuit)
   - 3.4 [Complete BGR Circuit](#34-complete-bgr-circuit)
4. [Design Specification and Device Datasheet](#4-design-specification-and-device-datasheet)
   - 4.1 [Design Specifications](#41-design-specifications)
   - 4.2 [Device Datasheet](#42-device-datasheet)
   - 4.3 [Circuit Sizing](#43-circuit-sizing)
5. [Tools and PDK Setup](#5-tools-and-pdk-setup)
   - 5.1 [Tool Setup](#51-tool-setup)
   - 5.2 [PDK Setup](#52-pdk-setup)
6. [Writing a SPICE Netlist](#6-writing-a-spice-netlist)
   - 6.1 [Netlist Structure](#61-netlist-structure)
   - 6.2 [Key Rules](#62-key-rules)
   - 6.3 [Sky130 Device Instantiation](#63-sky130-device-instantiation)
   - 6.4 [Simulation Commands](#64-simulation-commands)
   - 6.5 [Example Netlists](#65-example-netlists)
7. [Pre-Layout Simulation](#7-pre-layout-simulation)
   - 7.1 [CTAT Simulations](#71-ctat-simulations)
   - 7.2 [PTAT Simulations](#72-ptat-simulations)
   - 7.3 [Resistor Tempco](#73-resistor-tempco)
   - 7.4 [FET Tempco](#74-fet-tempco)
   - 7.5 [Full BGR Simulations](#75-full-bgr-simulations)
8. [Layout Design](#8-layout-design)
   - 8.1 [Leaf Cell Layouts](#81-leaf-cell-layouts)
   - 8.2 [Block-Level Layouts](#82-block-level-layouts)
   - 8.3 [Top-Level Layout](#83-top-level-layout)
9. [LVS Verification](#9-lvs-verification)
10. [Results Summary](#10-results-summary)
11. [References](#11-references)

---

## 1. Why a Bandgap Reference?

Every analog, mixed-signal, and digital SoC needs a stable DC reference voltage — one that doesn't change when the chip heats up, when the supply droops, or when the process shifts between fabrication runs.

<p align="center">
  <img src="Circuits/BGR1.png" alt="BGR Overview">
</p>

The naive approaches all fail:

| Reference Source | Problem |
|:---|:---|
| Battery | Voltage decays over time |
| Resistor divider from VDD | Supply sensitivity = 1 (100%), unusable |
| Forward-biased diode from VDD | Better supply rejection but −2.2 mV/°C tempco (~3142 ppm/°C) |
| Buried Zener diode IC | Needs extra components, high-frequency filtering; low-voltage Zeners unavailable in sub-micron CMOS |

**The solution** is a **Bandgap Reference (BGR)** — an all-silicon circuit that can be integrated into bulk CMOS, BiCMOS, or Bipolar processes with no external components.

A BGR exploits the opposing temperature behaviour of two on-chip voltages to produce a reference that is stable across the range of **−40 °C to +125 °C**, with:
- Typical tempco: **10–50 ppm/°C**
- Typical PSRR: **40–60 dB** (10–1 mV/V supply sensitivity)
- Output voltage: **≈ 1.2 V** (close to silicon's bandgap energy extrapolated to 0 K)

### Applications of BGR

A BGR sits at the heart of nearly every precision analog block in a SoC:

| Block | Role of BGR |
|:---|:---|
| LDO Regulator | Sets output regulation voltage |
| DC-DC Buck Converter | PWM controller reference |
| ADC | Sets full-scale conversion reference |
| DAC | Sets full-scale output reference |

---

## 2. Bandgap Reference — Theory

### 2.1 The BGR Principle

The fundamental insight behind a BGR is **cancellation of opposite temperature slopes**:

- A **CTAT** (Complementary-to-Absolute-Temperature) voltage decreases with temperature — roughly **−2.0 mV/°C**.
- A **PTAT** (Proportional-to-Absolute-Temperature) voltage increases with temperature — the thermal voltage V_T = kT/q rises at **+0.085 mV/°C**.

<p align="center">
  <img src="Circuits/BGR_Principle.png" alt="BGR Principle — CTAT + PTAT cancellation">
</p>

Scaling the PTAT slope up by a constant **K ≈ 26** and adding it to the CTAT gives a flat output:

$$V_{REF} = V_{EB} + K \cdot V_T$$

The two opposing slopes cancel, yielding a residual tempco of **10–50 ppm/°C** — ideal for a precision reference.

> **Why 1.2 V?**  
> At 0 K, silicon's bandgap energy is ~1.12 eV. The required K times V_T at room temperature pushes V_REF to ≈ 1.2 V, which is why all BGRs converge to this output regardless of supply or process.

---

### 2.2 CTAT Voltage Generation

<p align="center">
  <img src="Circuits/CTAT.png" alt="CTAT Voltage Generation Circuit">
</p>


A forward-biased PNP BJT (or diode-connected transistor) biased by a constant current I₀ gives:

$$V_D = V_t \ln\left(\frac{I_0}{I_S}\right), \quad V_t = \frac{kT}{q}$$

The saturation current I_S depends strongly on temperature:

$$I_S = A T^{(4+m)} e^{\left[\frac{-E_g}{kT}\right]}, \quad \mu \propto \mu_0 T^m,\ m = -\frac{3}{2}$$

Differentiating V_D with respect to temperature yields:

$$\frac{dV}{dT} = \frac{V_D - (4+m)V_t - \frac{E_g}{q}}{T}$$

Plugging in room-temperature numbers (T = 300 K, V_D ≈ 0.7 V, m = −3/2):

$$\frac{dV}{dT} = \frac{0.7 - (4 - 1.5)(0.026) - 1.2}{300} \approx -1.88\ \text{mV/°C}$$

This negative slope is the CTAT behaviour. The exact slope can be tuned by:
- **Multiplying the BJT count (N):** More parallel BJTs in branch 1 vs branch 2 steepens the slope (more negative tempco).
- **Changing bias current:** Lower I₀ reduces V_BE, increasing the magnitude of the slope.

The CTAT voltage generation can use a simple diode or a diode-connected PNP BJT (as used in SkyWater 130nm). Compared to a plain resistor divider (supply sensitivity = 1), the diode circuit reduces supply sensitivity to 1/ln(I/I_S) < 1.

---

### 2.3 PTAT Voltage Generation

<p align="center">
  <img src="Circuits/PTAT.png" alt="PTAT Voltage Generation Circuit">
</p>

To generate a purely PTAT voltage, consider two identical BJTs Q1 and Q2 biased at equal currents I, but with Q2 having **N times the emitter area** of Q1:

$$V_{Q1} = V_t \ln\frac{I}{I_S}, \quad V_{Q2} = V_t \ln\frac{I/N}{I_S}$$

$$\Delta V_{BE} = V_{Q1} - V_{Q2} = V_t \ln(N)$$

Since V_T = kT/q is inherently proportional to absolute temperature and ln(N) is a fixed constant:

$$\frac{d(\Delta V_{BE})}{dT} = \frac{k}{q} \ln(N) \approx 85\ \mu\text{V/°C per unit of ln(N)}$$

The resistor R1 in the PTAT branch carries this voltage difference, so the voltage across it is PTAT:

$$V_{R1} = V_t \ln(N)$$

Key design parameter — the PTAT slope is small (+85 µV/°C × ln(N)), so N is chosen to be large (e.g., N = 8) to maximise slope for better cancellation, and R1 is calculated as:

$$R1 = \frac{V_t \ln(N)}{I} = \frac{26\ \text{mV} \times \ln(8)}{10\ \mu\text{A}} \approx 5\ \text{k}\Omega$$

The PTAT circuit requires a **current mirror or op-amp** to force equal currents in both branches. In the self-biased topology this is handled by the current mirror itself.

---

### 2.4 BGR Types

**Architecture-wise:**

| Architecture | Advantages | Limitations |
|:---|:---|:---|
| Self-biased current mirror | Simplest, always stable (loop gain < 1), easy to design | Lower PSRR; needs start-up circuit; voltage headroom issues |
| Operational amplifier based | Higher PSRR | Stability concerns; more complex |

**Application-wise classification:**
- Low-voltage BGR
- Low-power BGR
- High-PSRR / low-noise BGR
- Curvature-compensated BGR (for < 10 ppm/°C)

This workshop implements the **self-biased current mirror** architecture for its simplicity, stability, and ease of layout matching.

---

## 3. Self-Biased Current Mirror Based BGR

The full BGR is assembled from five sub-circuits:

| Sub-circuit | Function |
|:---|:---|
| CTAT voltage generator | Provides the falling V_BE voltage |
| PTAT voltage generator | Provides the rising ΔV_BE voltage |
| Self-biased current mirror (SBCM) | Forces equal, supply-independent currents in all branches |
| Reference branch | Sums CTAT + PTAT to produce V_REF |
| Start-up circuit | Kicks the SBCM out of the zero-current degenerate state |

---

### 3.1 Self-Biased Current Mirror

A conventional current mirror referenced to a resistor from VDD is sensitive to supply — any change in VDD directly changes I_REF. The self-biased approach breaks this dependence:

**Key idea:** If I_out should be independent of VDD, make I_REF a *replica* of I_out (bootstrapped). PMOS transistors MP1 and MP2 copy I_out back to define I_REF. Since each diode-connected device is fed by a current source rather than from a resistor to VDD, both I_out and I_REF become largely independent of VDD.

With a degeneration resistor R_S in the NMOS source, the operating current is uniquely set:

$$I_{out} = \frac{2}{\mu_n C_{ox} (W/L)_N} \cdot \frac{1}{R_S^2} \left(1 - \frac{1}{\sqrt{K}}\right)^2$$

**Properties of the SBCM:**
- Loop gain is always less than 1 — inherently stable with no compensation needed.
- I_out and I_REF are weakly dependent on VDD.
- Requires a **start-up circuit** because zero current (I = 0) is also a valid solution.

<p align="center">
  <img src="Circuits/currentmirror.png" alt="Self-Biased Current Mirror Circuit">
</p>

---

### 3.2 Reference Branch Circuit

A third PMOS MP3 mirrors the same current I₃ ≈ I₁ ≈ I₂ into a reference branch consisting of **R2** in series with a PNP BJT Q3.

The voltage across Q3 is CTAT, and the voltage across R2 (= R2 × I₃) is PTAT since I₃ is PTAT-proportional. Their sum is:

$$V_{REF} = V_{Q3} + V_{R2} = \underbrace{V_{EB}}_{\text{CTAT}} + \underbrace{I_3 \cdot R2}_{\text{PTAT}}$$

For zero tempco of V_REF:

$$\frac{dV_{R2}}{dT} + \frac{dV_{Q3}}{dT} = 0 \implies \alpha \ln(N) \frac{dV_t}{dT} + \frac{dV_{Q3}}{dT} = 0$$

With dV_Q3/dT ≈ −1.6 mV/°C and dV_t/dT = 85 µV/°C:

$$\alpha = \frac{1.6\ \text{mV/°C}}{85\ \mu\text{V/°C} \times \ln(8)} \approx 6.4 \implies R2 = \alpha \cdot R1 \approx 33\ \text{k}\Omega$$

<p align="center">
  <img src="Circuits/refbranch1.png" alt="Reference Branch Circuit">
</p>

---

### 3.3 Start-up Circuit

The SBCM has two stable operating points:
1. **I_in = I_out = 0 A** (degenerate / undesired)
2. The desired bias current

At power-on, the circuit is at the zero-current point. The start-up circuit must:
- **Detect** the zero-current state and force the circuit out of it.
- **Disengage** once the correct operating point is reached — otherwise it will corrupt the bias.

**Operation:**
- Initially, all branch currents = 0 → net2 follows VDD.
- When net2 voltage exceeds net6 by one V_T, current flows through MP5 → net1 rises → MN1/MN2 turn on → circuit reaches the desired operating point.
- Once stable, the start-up circuit is self-defeating: net2 drops to the correct bias level, MP5 turns off, and the start-up path is isolated.

<p align="center">
  <img src="Circuits/startup.png" alt="Start-up Circuit">
</p>

---

### 3.4 Complete BGR Circuit

The final circuit integrates all five blocks into a single self-contained reference generator.

<p align="center">
  <img src="Circuits/fullbgr.png" alt="Complete BGR Circuit">
</p>

| Block | Devices |
|:---|:---|
| Startup | MP4, MP5, MN3 |
| SBCM | MP1, MP2, MN1, MN2 |
| CTAT branch | Q1 (1× BJT) |
| PTAT branch | R1 (5 kΩ), Q2 (8× BJT parallel) |
| Reference branch | MP3, R2 (33 kΩ), Q3 (1× BJT) |

---

## 4. Design Specification and Device Datasheet

### 4.1 Design Specifications

| Parameter | Target |
|:---|:---|
| Supply Voltage | 1.8 V |
| Operating Temperature | −40 °C to +125 °C |
| Power Consumption | < 60 µW |
| Off Current | < 2 µA |
| Start-up Time | < 2 µs |
| Tempco of V_REF | < 50 ppm/°C |

---

### 4.2 Device Datasheet

**MOSFET (LVT, 1.8V)**

| Parameter | NFET | PFET |
|:---|:---|:---|
| Type | LVT | LVT |
| Voltage | 1.8 V | 1.8 V |
| Threshold Voltage | ~0.4 V | ~−0.6 V |
| Sky130 Model | `sky130_fd_pr__nfet_01v8_lvt` | `sky130_fd_pr__pfet_01v8_lvt` |

**PNP BJT**

| Parameter | Value |
|:---|:---|
| Current Rating | 1 µA – 10 µA / µm² |
| Beta (β) | ~12 |
| Emitter Area | 11.56 µm² (3.40 × 3.40 µm) |
| Sky130 Model | `sky130_fd_pr__pnp_05v5_W3p40L3p40` |

**Resistor (RPOLYH)**

| Parameter | Value |
|:---|:---|
| Sheet Resistance | ~350 Ω/□ |
| Tempco | 2.5 Ω/°C |
| Available Bin Widths | 0.35 µm, 0.69 µm, 1.41 µm, 2.85 µm, 5.73 µm |
| Sky130 Model | `sky130_fd_pr__res_high_po` |

---

### 4.3 Circuit Design

**Step 1 — Current Calculation**

Max power = 60 µW at VDD = 1.8 V → max total current = 33.33 µA.  
With 3 branches: **10 µA/branch** (3 × 10 = 30 µA, leaving headroom for start-up).

**Step 2 — BJT ratio N**

A moderate N = 8 BJTs in parallel in branch 2 gives:
- Good layout matching (common-centroid array)
- Moderate R1 value (not too large, keeping area reasonable)

**Step 3 — R1 calculation**

$$R1 = \frac{V_t \ln(N)}{I} = \frac{26\ \text{mV} \times \ln(8)}{10.7\ \mu\text{A}} \approx 5\ \text{k}\Omega$$

Implemented as: W = 1.41 µm, L = 7.8 µm, unit = 2 kΩ → 2 series + 2 parallel (2+2+(2‖2))

**Step 4 — R2 calculation**

Slope of V_R2 = (R2/R1) × ln(8) × 115 µV/°C  
Slope of V_Q3 = −1.6 mV/°C  
Setting sum to zero → **R2 ≈ 33 kΩ**

Implemented as: 16 in series + 2 in parallel (2+2+...+2+(2‖2))

**Step 5 — PMOS sizing (SBCM)**

MP1, MP2: Both in saturation. Long channel (L = 2 µm) to reduce channel length modulation.  
**Final size: L = 2 µm, W = 5 µm, M = 4**

**Step 6 — NMOS sizing (SBCM)**

MN1, MN2: Operated in deep sub-threshold to achieve low current with compact area.  
**Final size: L = 1 µm, W = 5 µm, M = 8**

**Final Sized Circuit:**

<p align="center">
  <img src="Circuits/finalbgr.png" alt="Final BGR Circuit with Component Values">
</p>

---

## 5. Tools and PDK Setup

### 5.1 Tool Setup

The complete design flow uses three open-source EDA tools:

| Tool | Version | Role | PDK Artifact Used |
|:---|:---|:---|:---|
| **Ngspice** | 34.0 | Pre- and post-layout SPICE simulation | Sky130 model file |
| **Magic** | 8.3.178 | Layout design, DRC, parasitic extraction (PEX) | Sky130 tech file, magic tech file |
| **Netgen** | 1.5.185 | LVS (Layout vs Schematic) | Sky130 Netgen rule file |

#### Ngspice

Open-source SPICE simulator for electrical circuit simulation. Used to perform DC sweep, transient, and temperature sweeps.

```bash
sudo apt-get install ngspice
```

#### Magic

Berkeley VLSI layout editor — used for drawing layout, running DRC, and extracting parasitics.

```bash
wget http://opencircuitdesign.com/magic/archive/magic-8.3.32.tgz
tar xvfz magic-8.3.32.tgz
cd magic-8.3.32
./configure
sudo make
sudo make install
```

#### Netgen

LVS tool — compares the extracted SPICE netlist from Magic against the pre-layout schematic netlist.

```bash
git clone git://opencircuitdesign.com/netgen
cd netgen
./configure
sudo make
sudo make install
```

---

### 5.2 PDK Setup

This project uses **Google SkyWater Sky130** — a 130 nm open-source process design kit.

```bash
# Clone primitive device models
git clone https://github.com/google/skywater-pdk-libs-sky130_fd_pr.git

# Clone EDA technology files (Magic tech, Netgen rules, etc.)
git clone https://github.com/silicon-vlsi-org/eda-technology.git
```

**Design Flow Overview:**

```
Schematic Design   ──[Ngspice + Sky130 models]──►  Pre-Layout Simulation
       │
       ▼
 Layout Design     ──[Magic + Sky130 tech file]──►  DRC + PEX
       │
       ▼
      LVS          ──[Netgen + Sky130 rule file]──►  Schematic vs Layout
       │
       ▼
Post-Layout Sim    ──[Ngspice + extracted netlist]──►  Final Verification
```

---

## 6. Writing a SPICE Netlist

A SPICE netlist is a text file that describes a circuit — its components, connections, and simulation commands — in a format that ngspice can read and simulate. Every `.sp` file in this project follows the same structure.

---

### 6.1 Netlist Structure

```spice
*  Title / Comment line (must be the first line)

*  ----------------------------
*  1. Include PDK model files
*  ----------------------------
.lib "/path/to/sky130.lib.spice" tt

*  ----------------------------
*  2. Global supply nodes
*  ----------------------------
.global gnd vdd
vdd vdd gnd 1.8

*  ----------------------------
*  3. Circuit components
*  ----------------------------
* Syntax: ComponentName  Node+ Node-  ModelName  Parameters

* BJT (PNP)
xqp1  vdd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* MOSFET (PFET)
xmp1  net1  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4

* MOSFET (NFET)
xmn1  net2  net3  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8

* Resistor
xra1  net3  net4  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=1

*  ----------------------------
*  4. Simulation commands
*  ----------------------------
.dc temp -40 125 5        * DC sweep: temperature from -40 to 125°C in steps of 5

.control
  run
  plot v(vref)            * Plot the reference voltage node
.endc

.end                      * Every netlist must end with .end
```

---

### 6.2 Key Rules

| Rule | Explanation |
|:---|:---|
| First line is always a comment | ngspice treats line 1 as the title, never as a component |
| Component names are case-insensitive | `XMP1` and `xmp1` are the same |
| `X` prefix = subcircuit call | Sky130 devices are subcircuits, so always prefix with `X` |
| Node `0` or `gnd` = ground | All voltages are measured relative to this |
| `.end` is mandatory | Netlist is ignored without it |

---

### 6.3 Sky130 Device Instantiation

Sky130 devices are defined as subcircuits in the PDK model files, so they are called using `X` (subcircuit instance) syntax:

#### PNP BJT

```spice
* xInstanceName  Collector  Base  Emitter  ModelName  m=<multiplier>
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1
```

#### PFET

```spice
* xInstanceName  Drain  Gate  Source  Bulk  ModelName  l=<> w=<> m=<>
xmp1  net1  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4
```

#### NFET

```spice
* xInstanceName  Drain  Gate  Source  Bulk  ModelName  l=<> w=<> m=<>
xmn1  net2  net3  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8
```

#### RPOLYH Resistor

```spice
* xInstanceName  Node+  Node-  ModelName  l=<> w=<> m=<>
xra1  net3  net4  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=1
```

---

### 6.4 Simulation Commands

| Command | Purpose |
|:---|:---|
| `.dc temp -40 125 5` | Sweep temperature from −40°C to +125°C |
| `.dc vdd 1.62 1.98 0.01` | Sweep supply voltage ±10% |
| `.op` | Single operating point analysis |
| `.control / .endc` | Interactive block for run and plot commands |
| `.lib` | Include PDK model library for a specific corner (tt / ff / ss) |
| `.param` | Define parameters for swept simulations |

---

### 6.5 Example Netlists

#### CTAT Single BJT (`ctat_voltage_gen.sp`)

```spice
* CTAT Voltage Generation — Single BJT
* Temperature sweep to observe negative tempco of V_BE

.lib "/home/vsduser/Desktop/Work/eda-technology/sky130/libs/sky130.lib.spice" tt

.global gnd vdd
vdd vdd gnd 1.8

* Constant current source (10uA)
isrc  vdd  net1  dc  10u

* Diode-connected PNP BJT
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

.dc temp -40 125 5

.control
  run
  plot v(net1)
.endc

.end
```

#### BGR with SBCM (`bgr_lvt_rpolyh_3p40.sp`)

```spice
* BGR — Self-Biased Current Mirror (TT corner)

.lib "/home/vsduser/Desktop/Work/eda-technology/sky130/libs/sky130.lib.spice" tt

.global gnd vdd
vdd vdd gnd 1.8

* ---- PMOS Current Mirror ----
xmp1  net1  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4
xmp2  net2  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4
xmp3  vref  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4

* ---- NMOS (SBCM) ----
xmn1  net1  net1  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8
xmn2  net2  net1  net3  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8

* ---- CTAT Branch (Q1 x1) ----
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* ---- PTAT Branch (R1 + Q2 x8) ----
xra1  net3  net4  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=2
xqp2  gnd  net4  net4  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=8

* ---- Reference Branch (R2 + Q3 x1) ----
xrb1  vref  net5  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=16
xqp3  gnd  net5  net5  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* ---- Start-up Circuit ----
xmp4  net6  net6  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=1
xmp5  net7  net6  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=1
xmn3  net7  net7  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=7 w=1 m=1
xmn4  net6  net7  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=7 w=1 m=1

.dc temp -40 125 5

.control
  run
  plot v(vref)
.endc

.end
```

#### Changing Corner

To run FF or SS corners, change only the `.lib` tag — everything else stays the same:

```spice
.lib "/path/to/sky130.lib.spice" ff   * Fast-Fast
.lib "/path/to/sky130.lib.spice" ss   * Slow-Slow
```

---

## 7. Pre-Layout Simulation

All netlists are simulated with Ngspice. Launch any simulation from the `prelayout/` directory.

### 7.1 CTAT Simulations

#### Single BJT CTAT

```bash
ngspice ctat_voltage_gen.sp
```

The base-emitter voltage of a single PNP BJT biased at a constant current decreases linearly with temperature at approximately −2 mV/°C — classic CTAT behaviour.

<p align="center">
  <img src="prelayout_results/ctat_voltage_gen.png" alt="CTAT Single BJT Simulation Result">
</p>

---

#### Multiple BJT CTAT

```bash
ngspice ctat_voltage_gen_mul_bjt.sp
```

With 8 BJTs in parallel in branch 2 vs 1 BJT in branch 1, the CTAT slope of the branch 1 node becomes steeper (more negative), as expected from the BJT area ratio.

<p align="center">
  <img src="prelayout_results/ctat_voltage_gen_mul_bjt.png" alt="CTAT Multiple BJT Simulation Result">
</p>

---

#### CTAT with Varying Current

```bash
ngspice ctat_voltage_gen_var_current.sp
```

Reducing bias current lowers V_BE and increases the magnitude of the negative slope — confirming the I₀ dependence in the CTAT equation.

<p align="center">
  <img src="prelayout_results/ctat_voltage_gen_var_current.png" alt="CTAT Varying Current Simulation Result">
</p>

---

### 7.2 PTAT Simulations

#### PTAT with Ideal Voltage Source (VCVS)

```bash
ngspice ptat_voltage_gen.sp
```

Using a VCVS to force equal voltages at the branch nodes, the voltage across R1 is proportional to V_T ln(N) — confirming the PTAT slope of +0.085 mV/°C per unit of ln(N).

<p align="center">
  <img src="prelayout_results/ptat_voltage_gen.png" alt="PTAT with VCVS Simulation Result">
</p>

---

#### PTAT with Ideal Current Source

```bash
ngspice ptat_voltage_gen_ideal_current_source.sp
```

<p align="center">
  <img src="prelayout_results/ptat_voltage_gen_ideal_current_source.png" alt="PTAT with Ideal Current Source Simulation Result">
</p>

---

### 7.3 Resistor Tempco

```bash
ngspice res_tempco.sp
```

This simulation characterises the temperature behaviour of the RPOLYH resistor (~2.5 Ω/°C). Since R1 and R2 both use the same poly resistor type, their tempcos track each other — this self-cancelling ratio (R2/R1) means resistor tempco has minimal impact on V_REF stability.

<p align="center">
  <img src="prelayout_results/res_tempco.png" alt="Resistor Temperature Coefficient Simulation">
</p>

#### Resistor Tempco with Varying Current

```bash
ngspice res_tempco_var_current.sp
```

<p align="center">
  <img src="prelayout_results/res_tempco_var_current.png" alt="Resistor Tempco with Varying Current Simulation">
</p>

---

### 7.4 FET Tempco

```bash
ngspice fet_tempco.sp
```

This simulation characterises the temperature behaviour of the LVT NFET and PFET devices used in the SBCM. FET tempco directly affects how mirror current shifts with temperature, which in turn contributes to V_REF deviation — understanding this is essential for predicting corner performance.

<p align="center">
  <img src="prelayout_results/fet_tempco.png" alt="FET Temperature Coefficient Simulation">
</p>

---

### 7.5 Full BGR Simulations

#### BGR with Ideal Op-Amp

```bash
ngspice bgr_using_ideal_opamp.sp
```

Before committing to the self-biased current mirror, the full BGR loop is verified using a VCVS as an ideal op-amp. The output should be an umbrella-shaped curve centred near 1.2 V — confirming that the CTAT + PTAT cancellation is working correctly.

<p align="center">
  <img src="prelayout_results/bgr_using_ideal_opamp.png" alt="BGR with Ideal Op-Amp Simulation Result">
</p>

#### Transient Analysis

<p align="center">
  <img src="prelayout_results/bgr_using_ideal_opamp_1_trans.png" alt="BGR with Ideal Op-Amp Simulation Result">
</p>

---

#### BGR with SBCM — Typical (TT corner)

```bash
ngspice bgr_lvt_rpolyh_3p40.sp
```

The self-biased current mirror replaces the ideal op-amp. Expected result:
- V_REF ≈ 1.2 V, flat across −40 °C to +125 °C
- Tempco ≈ **21.7 ppm/°C**

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40.png" alt="BGR SBCM TT Corner Simulation Result">
</p>
<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_V_ref.png" alt="BGR SBCM V ref Result">
</p>

#### Transient Analysis

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_1__trans.png" alt="BGR SBCM V ref Result">
</p
---

#### BGR — Fast-Fast (FF) Corner

```bash
ngspice bgr_lvt_rpolyh_3p40_ff.sp
```

Tempco ≈ **10 ppm/°C** (best case — process corners tend to improve tempco here)

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ff.png" alt="BGR FF Corner Simulation Result">
</p>
<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ff_V_ref.png" alt="BGR FF Corner Simulation Result">
</p>

#### Transient Analysis

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ff_1_trans.png" alt="BGR FF Corner Simulation Result">
</p>

---

#### BGR — Slow-Slow (SS) Corner

```bash
ngspice bgr_lvt_rpolyh_3p40_ss.sp
```

Tempco ≈ **45 ppm/°C** (worst case — still within the 50 ppm/°C specification)

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ss.png" alt="BGR SS Corner Simulation Result">
</p>
<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ss_V_ref.png" alt="BGR SS Corner Simulation Result">
</p>

#### Transient Analysis

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_ss_1_trans.png" alt="BGR SS Corner Simulation Result">
</p>

---

#### BGR — Supply Variation

```bash
ngspice bgr_lvt_rpolyh_3p40_var_supply.sp
```

VDD swept from 1.62 V to 1.98 V (±10%). V_REF should remain stable, validating the PSRR of the SBCM topology.

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_var_supply.png" alt="BGR Supply Variation Simulation Result">
</p>
<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_var_supply_V_ref.png" alt="BGR Supply Variation Simulation Result">
</p>

#### Transient Analysis

<p align="center">
  <img src="prelayout_results/bgr_lvt_rpolyh_3p40_var_supply_1_trans.png" alt="BGR Supply Variation Simulation Result">
</p>

---

## 8. Layout Design

All layout is done in **Magic** with the Sky130A technology file.

```bash
# Launch Magic with Sky130 tech
magic -T /home/vsduser/Desktop/Work/eda-technology/sky130/tech/magic/sky130A.tech
```

The layout is built hierarchically — starting from individual leaf cells, assembled into matched blocks, then placed and routed at the top level. All blocks use **common-centroid placement** and **guard rings** for noise isolation and mismatch reduction.

---

### 8.1 Leaf Cell Layouts

These are the primitive device layouts that form the building blocks of the complete BGR.

#### NFET — W=5µm, L=1µm (`nfet.mag`)

```bash
magic -T sky130A.tech nfet.mag
```

LVT NFET used in the SBCM (MN1, MN2). Operated in deep sub-threshold. Guard ring included for latch-up prevention.

<p align="center">
  <img src="Layout_results/nfet.png" alt="NFET Layout — W=5µm L=1µm">
</p>

---

#### NFET — W=1µm, L=7µm (`nfet1.mag`)

```bash
magic -T sky130A.tech nfet1.mag
```

Long-channel LVT NFET used in the start-up circuit (MN3, MN4). Long L ensures very low leakage in the off-state, preventing the start-up path from disturbing the bias after the circuit has settled.

<p align="center">
  <img src="Layout_results/nfet1.png" alt="NFET Layout — W=1µm L=7µm">
</p>

---

#### PFET — W=5µm, L=2µm (`pfet.mag`)

```bash
magic -T sky130A.tech pfet.mag
```

LVT PFET used for the current mirror (MP1–MP3). Long channel (L = 2 µm) reduces channel-length modulation, improving current mirror accuracy across VDD variation.

<p align="center">
  <img src="Layout_results/pfet.png" alt="PFET Layout — W=5µm L=2µm">
</p>

---

#### PNP BJT Unit Cell (`pnpt1.mag`)

```bash
magic -T sky130A.tech pnpt1.mag
```

Single PNP BJT with emitter area 3.40 × 3.40 µm (11.56 µm²). This is the unit cell replicated to build all three BJT instances: Q1 (1×), Q2 (8×), and Q3 (1×).

<p align="center">
  <img src="Layout_results/pnpt1.png" alt="PNP BJT Unit Cell Layout">
</p>

---

#### Resistor Unit Cell (`res1p41.mag`)

```bash
magic -T sky130A.tech res1p41.mag
```

Single RPOLYH unit with W = 1.41 µm, L = 7.8 µm giving approximately 2 kΩ per unit. Multiple units are connected in series and parallel to achieve R1 = 5 kΩ and R2 = 33 kΩ.

<p align="center">
  <img src="Layout_results/res1p41.png" alt="Resistor Unit Cell Layout">
</p>

---

### 8.2 Block-Level Layouts

Leaf cells are grouped into functionally matched blocks. All blocks use common-centroid interdigitation and guard rings.

#### NFET Bank (`nfets.mag`)

```bash
magic -T sky130A.tech nfets.mag
```

MN1 and MN2 placed together in a common-centroid arrangement with a shared guard ring. Both carry equal currents — tight physical matching minimises systematic current offset between branches.

<p align="center">
  <img src="Layout_results/nfets.png" alt="NFET Bank Layout">
</p>

---

#### PFET Bank (`pfets.mag`)

```bash
magic -T sky130A.tech pfets.mag
```

MP1–MP5 placed together with matched interdigitation and a p-well guard ring. Current mirror accuracy — which directly sets branch currents and hence V_REF — depends critically on PFET matching.

<p align="center">
  <img src="Layout_results/pfets.png" alt="PFET Bank Layout">
</p>

---

#### Resistor Bank (`resbank.mag`)

```bash
magic -T sky130A.tech resbank.mag
```

All R1 and R2 unit resistors are placed with dummy resistors at the ends of each row to ensure uniform etching environments. A guard ring surrounds the entire block. Series and parallel connections implement R1 = 5 kΩ and R2 = 33 kΩ with the ratio R2/R1 matching as the critical design parameter.

<p align="center">
  <img src="Layout_results/resbank.png" alt="Resistor Bank Layout">
</p>

---

#### BJT Array (`pnp10.mag`)

```bash
magic -T sky130A.tech pnp10.mag
```

10-BJT common-centroid array containing Q2×8, Q1×1, and Q3×1. Dummy BJTs surround all active devices to ensure uniform implant density. Common-centroid placement minimises gradient-induced mismatch between Q1 and Q2 — the primary source of PTAT inaccuracy.

<p align="center">
  <img src="Layout_results/pnp10.png" alt="BJT Array Layout — 10 PNPs common-centroid">
</p>

---

#### Starter NFET (`starternfet.mag`)

```bash
magic -T sky130A.tech starternfet.mag
```

Two W=1µm, L=7µm NFETs (MN3 and MN4) placed together with a shared guard ring. These form the sensing and pull-up devices in the start-up circuit.

<p align="center">
  <img src="Layout_results/starternfet.png" alt="Starter NFET Layout">
</p>

---

### 8.3 Top-Level Layout

All blocks — NFET bank, PFET bank, resistor bank, BJT array, and starter NFET — are placed and routed together to form the complete BGR.

```bash
magic -T sky130A.tech top.mag
```

| File | Description |
|:---|:---|
| `top.mag` | Full BGR top-level layout with all blocks placed and routed |

<p align="center">
  <img src="Layout_results/top.png" alt="Top-Level BGR Layout">
</p>

**Parasitic Extraction sequence (Magic Tcl console):**

```tcl
extract all
ext2sim label on
ext2sim
ext2spice scale off
ext2spice hierarchy off
ext2spice
```

This produces `top.spice` — the extracted netlist including all parasitic capacitances — which is then fed into Ngspice for post-layout simulation and into Netgen for LVS.

---

## 9. LVS Verification

After extraction, the layout netlist is compared against the schematic netlist using Netgen.

```bash
netgen
```

Inside netgen console window enter below command for LVS verification.

```bash
lvs "top.spice top" "bgr_lvt_rpolyh_3p40.sp top" /home/vsduser/Desktop/Work/eda-technology/sky130/tech/netgen//sky130A_setup.tcl
```

A clean LVS result confirms that:
- All devices match in type, size, and connectivity.
- No shorts or opens were introduced during layout.

---

## 11. References

- [VSDOpen — Bandgap Reference Workshop](https://www.vlsisystemdesign.com/)
- [SkyWater Sky130 PDK Documentation](https://skywater-pdk.readthedocs.io/en/latest/)
- [Google SkyWater PDK — Primitive Devices](https://github.com/google/skywater-pdk-libs-sky130_fd_pr)
- [EDA Technology Files (Magic/Netgen for Sky130)](https://github.com/silicon-vlsi-org/eda-technology)
- [Ngspice — Open Source SPICE Simulator](http://ngspice.sourceforge.net)
- [Magic VLSI Layout Tool](http://opencircuitdesign.com/magic/)
- [Netgen LVS Tool](http://opencircuitdesign.com/netgen/)

---
