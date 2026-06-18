# Operational Amplifier Verilog-A Macromodel — Implementation Report

## 1. Overview

This document describes the Verilog-A behavioural macromodel for a general-purpose operational amplifier implemented in `model/opamp.va`. The model is based on the modular op-amp macromodel architecture developed by Mike Brinson for the Qucs simulator, adapted here for compilation with **OpenVAF** and simulation with **ngspice** via the OSDI (Open Source Device Interface) mechanism.

The default parameter set models a typical **UA741** operational amplifier. All parameters are exposed and can be overridden per-instance or per-model card in a SPICE netlist, making the model suitable for representing a wide range of voltage-feedback op-amps.

### Toolchain

| Component | Role |
|-----------|------|
| `model/opamp.va` | Verilog-A source |
| OpenVAF | Compiler — produces `opamp.osdi` |
| ngspice (OSDI) | Simulator — loads the compiled `.osdi` at runtime via `pre_osdi` |

### Module Interface

```
module opamp_module(inp, inn, out, vdd, vss);
```

| Port | Direction | Description |
|------|-----------|-------------|
| `inp` | inout | Non-inverting input |
| `inn` | inout | Inverting input |
| `out` | inout | Output |
| `vdd` | inout | Positive supply rail |
| `vss` | inout | Negative supply rail |

The supply pins `vdd` and `vss` carry no significant current (a 1 pS leakage conductance is applied to satisfy the SPICE matrix requirements). Their voltages are used by the output clamp to limit the output swing to the supply rails.

---

## 2. Model Parameters

All parameters carry OpenVAF-compatible attributes (`desc`, `units`) and enforce range constraints.

| Parameter | Default | Units | Description |
|-----------|---------|-------|-------------|
| `GBP` | 1 MHz | Hz | Gain–bandwidth product |
| `AOLDC` | 106.0 | dB | Open-loop DC voltage gain |
| `FP2` | 3 MHz | Hz | Second-pole frequency |
| `RO` | 75 | Ω | Output resistance |
| `CD` | 1 pF | F | Differential input capacitance |
| `RD` | 2 MΩ | Ω | Differential input resistance |
| `IOFF` | 20 nA | A | Input offset current |
| `IB` | 80 nA | A | Input bias current |
| `VOFF` | 0.7 mV | V | Input offset voltage |
| `CMRRDC` | 90.0 | dB | Common-mode rejection ratio (DC) |
| `FCM` | 200 | Hz | CMRR roll-off corner frequency |
| `PSRT` | 500 kV/s | V/s | Positive slew rate |
| `NSRT` | 500 kV/s | V/s | Negative slew rate |
| `ILMAX` | 35 mA | A | Maximum DC output current |
| `CSCALE` | 50 | — | Current limit scaling factor |

---

## 3. Internal Derived Quantities

At the start of every evaluation the model computes a set of intermediate constants from the user-facing parameters:

| Variable | Formula | Purpose |
|----------|---------|---------|
| `MTWOPI` | 2π | Constant |
| `Voffset` | VOFF / 2 | Half the offset voltage, applied symmetrically to each input |
| `Rdiff` | RD / 2 | Half the differential resistance, placed on each input leg |
| `CMRR0` | 10^(CMRRDC/20) | Linear CMRR |
| `CMgain` | 10^6 / CMRR0 | Voltage gain of the common-mode sensing stage |
| `CCM` | 1 / (2π × 10^6 × FCM) | Capacitor in the common-mode filter |
| `RP1` | 10^(AOLDC/20) | Transresistance of the first pole (gain stage) |
| `CP1` | 1 / (2π × GBP) | Capacitor of the first pole |
| `RP2` | 1 Ω | Fixed transresistance of the second pole |
| `CP2` | 1 / (2π × FP2) | Capacitor of the second pole |
| `Slewratepositive` | PSRT / (2π × GBP) | Slew-limiter positive threshold (in volts) |
| `Slewratenegative` | NSRT / (2π × GBP) | Slew-limiter negative threshold (in volts) |

The first-pole transfer function has a DC gain of RP1 (≈ 200 000 for 106 dB) and a time constant of RP1 × CP1, placing the dominant pole at GBP / A_OL. Together with the unity-gain second pole (RP2 = 1 Ω), this produces the classic two-pole open-loop response.

---

## 4. Signal-Path Architecture

The model implements a cascade of functional blocks connected by internal nodes. The signal flow is:

```
            ┌──────────────┐
  inp ──►───┤  Input Stage  ├──► n7 ──┐
            │  (offset,     │         │
  inn ──►───┤   bias, RD,   ├──► n9 ──┤
            │   CD, CMRR)   │         │
            └──────────────┘         │
                                      ▼
                              ┌──────────────┐
                     n8 ◄─────┤  CM midpoint  │
                              └──────────────┘
                                      │
                              n6 → n10 (CM filter)
                                      │
                              ┌──────────────┐
                     n11 ◄────┤  Adder       │ (diff + CM)
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n12 ◄────┤  Slew Limiter │ (if/else clamp)
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n3  ◄────┤  First Pole   │ (gain stage)
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n5  ◄────┤  Second Pole  │
                              └──────────────┘
                                      │
                              ┌──────────────┐
              n2 ──► out  ◄───┤  Output Stage │ (RO + current limiter
                              │               │  + voltage clamp)
                              └──────────────┘
```

Each block is described in detail below.

---

## 5. Functional Blocks

### 5.1 Supply-Pin Bias

```verilog
I(vdd) <+ V(vdd) * 1e-12;
I(vss) <+ V(vss) * 1e-12;
```

A 1 pS conductance on each supply pin ensures that `V(vdd)` and `V(vss)` participate in the SPICE matrix. This prevents floating-node errors when the supplies are driven by ideal voltage sources while drawing negligible current (< 15 fA at 15 V).

### 5.2 Input Stage — Offset, Bias, and Impedance

**Offset voltage** is modelled by inserting VOFF/2 as a voltage source in series with each input, implemented via controlled current sources into 1 Ω branches:

```verilog
I(inp, n7) <+ V(inp, n7);     // 1 Ω series element
I(inp, n7) <+ Voffset;         // +VOFF/2 offset

I(inn, n9) <+ V(inn, n9);     // 1 Ω series element
I(inn, n9) <+ -Voffset;        // -VOFF/2 offset
```

The net effect is that `V(n7) − V(n9) = V(inp) − V(inn) + VOFF`.

**Input bias currents** are modelled as fixed current sinks at each internal input node:

```verilog
I(n7) <+ IB;
I(n9) <+ IB;
```

**Input offset current** adds a differential component:

```verilog
I(n7, n9) <+ IOFF / 2;
```

**Differential input impedance** is modelled with split resistors and a capacitor:

```verilog
I(n7, n8) <+ V(n7, n8) / Rdiff;   // RD/2 from n7 to midpoint n8
I(n9, n8) <+ V(n9, n8) / Rdiff;   // RD/2 from n9 to midpoint n8
I(n7, n9) <+ ddt(CD * V(n7, n9)); // CD across the differential pair
```

Node `n8` is the common-mode midpoint of the input, carrying the signal `V(n8) = (V(n7) + V(n9)) / 2`.

### 5.3 Common-Mode Rejection Stage

The CMRR is implemented as a separate signal path that senses the common-mode voltage at `n8`, attenuates it, and passes it through a single-pole low-pass filter:

```verilog
I(n6) <+ -CMgain * V(n8);          // sense CM voltage, apply attenuation
I(n6) <+ V(n6);                     // 1 Ω termination

I(n6, n10) <+ V(n6, n10) / 1e6;    // 1 MΩ series resistance
I(n6, n10) <+ ddt(CCM * V(n6, n10)); // shunt capacitor → LP filter

I(n10) <+ V(n10);                   // 1 Ω termination
```

The filter pole is at `FCM` Hz. Above this frequency the CM signal is progressively attenuated, modelling the roll-off of CMRR with frequency. The DC CMRR equals the parameter `CMRRDC`.

### 5.4 Signal Adder

The differential and common-mode signal paths are summed at node `n11`:

```verilog
I(n11) <+ -V(n10);        // common-mode contribution
I(n11) <+ -V(n7, n9);     // differential contribution
I(n11) <+ V(n11);          // 1 Ω termination
```

This yields `V(n11) = V(n7, n9) + V(n10)`, combining both signal paths for processing by the subsequent stages.

### 5.5 Slew Rate Limiter

The slew rate limiter uses `if/else` conditionals to hard-clip the drive voltage:

```verilog
if (V(n11) > Slewratepositive)
    I(n12) <+ -Slewratepositive;
else if (V(n11) < -Slewratenegative)
    I(n12) <+ Slewratenegative;
else
    I(n12) <+ -V(n11);
I(n12) <+ V(n12);
```

When `|V(n11)|` is within the slew-rate thresholds, the signal passes through at unity gain. When it exceeds a threshold, the drive is clamped to ±Slewrate. The thresholds are derived from the slew rate parameters as:

```
Slewrate_threshold = SR / (2π × GBP)
```

For the default parameters, `Slewratepositive = Slewratenegative ≈ 0.0796 V`.

### 5.6 First Pole — Gain Stage

The dominant pole is implemented as a transresistance amplifier with a parallel RC:

```verilog
I(n3) <+ -V(n12);              // drive current from slew limiter
I(n3) <+ V(n3) / RP1;          // shunt resistance (sets DC gain)
I(n3) <+ ddt(CP1 * V(n3));     // shunt capacitance (sets pole frequency)
```

The DC gain is `V(n3) / V(n12) = RP1 = 10^(AOLDC/20)` and the pole frequency is `1 / (2π × RP1 × CP1) = GBP / RP1`.

There is no internal voltage clamp on `V(n3)`. In open-loop or comparator configurations the large DC gain (≈ 200 000) can drive `V(n3)` to extreme voltages (thousands of volts). This does not affect circuit accuracy (the output is clamped at the supply rails) but can contribute to convergence difficulties in DC analysis (see §8).

### 5.7 Second Pole

A unity-gain low-pass filter implements the second pole:

```verilog
I(n5) <+ -V(n3);               // drive from first pole
I(n5) <+ V(n5) / RP2;          // RP2 = 1 Ω
I(n5) <+ ddt(CP2 * V(n5));     // pole at FP2
```

The pole frequency is `1 / (2π × RP2 × CP2) = FP2`. This models the parasitic high-frequency roll-off beyond the gain-bandwidth product.

### 5.8 Current Limiter Stage

The output current is limited by a multiplicative feedback mechanism:

```verilog
if (V(n2, out) >= ILMAX)
begin
    I(n4) <+ -V(n5);
    I(n4) <+ CSCALE * V(n5) * (V(n2, out) - ILMAX);
    I(n4) <+ V(n4);
end
else if (V(n2, out) <= -ILMAX)
begin
    I(n4) <+ -V(n5);
    I(n4) <+ -CSCALE * V(n5) * (V(n2, out) + ILMAX);
    I(n4) <+ V(n4);
end
else
begin
    I(n4) <+ -V(n5);
    I(n4) <+ V(n4);
end
```

In all branches, `V(n5)` is passed through to `V(n4)` at unity gain. When `|V(n2,out)|` (the output current flowing through the 1 Ω sense resistor) exceeds `ILMAX`, the additional `CSCALE × V(n5) × excess` term modifies the drive to `n4`, reducing the current delivered to the output.

The `CSCALE` parameter (default 50) controls the stiffness of the current limiting.

### 5.9 Output Resistance

The output resistance `RO` is split into two series elements for current sensing:

```verilog
I(n4, n2) <+ V(n4, n2) / (RO - 1);   // (RO − 1) Ω
I(n2, out) <+ V(n2, out);              // 1 Ω current sense
```

The total output impedance between `n4` and `out` is `(RO − 1) + 1 = RO` ohms. The 1 Ω element between `n2` and `out` serves as the current-sense resistor: `V(n2, out)` equals the output current in amps.

### 5.10 Output Voltage Clamp

The output voltage is clamped to the supply rails using an `if/else` conductance switch:

```verilog
if (V(out) > V(vdd))
begin
    I(out) <+ -10.0 * V(vdd);
    I(out) <+ 10.0 * V(out);
end
else if (V(out) < V(vss))
begin
    I(out) <+ -10.0 * V(vss);
    I(out) <+ 10.0 * V(out);
end
```

When the output exceeds a supply rail, a 10 S conductance activates to pull it back toward the rail. When within the supply range, no clamping current is applied. The clamp limits automatically follow `V(vdd)` and `V(vss)`, so the output swing adapts to any supply configuration without requiring parameter changes.

---

## 6. Known Limitations

### 6.1 DC Sweep Hysteresis in Open-Loop (Comparator) Configurations

The model uses `if/else` conditionals in the slew rate limiter, current limiter, and output voltage clamp. These produce **discontinuous Jacobian derivatives** at the switching boundaries. In closed-loop (negative feedback) configurations this causes no problems because the feedback constrains the operating point to the linear region.

However, in **open-loop (comparator) configurations under DC sweep analysis**, the discontinuous derivatives cause the Newton–Raphson solver to become trapped at whichever supply rail it converges to first. The result is:

- **Forward DC sweep:** output stuck at LOW rail for the entire sweep
- **Reverse DC sweep:** output stuck at HIGH rail for the entire sweep
- **The output never transitions** between rails, regardless of the input voltage crossing the threshold

This is a fundamental convergence issue with the interaction between the hard-clipping non-linearities and the SPICE DC solver. The model produces correct results in:

- **Transient analysis** (including comparator-mode switching), because the time-stepping integrator can track transitions through the clipping boundaries
- **Operating point (OP) analysis** at any fixed bias condition
- **All closed-loop configurations** (buffers, amplifiers, integrators, etc.)

Resolving this limitation requires replacing the `if/else` non-linearities with smooth functions (e.g. `tanh`-based clamps) while preserving correct DC operating point convergence in complex multi-amplifier circuits. This is an area of ongoing work.

---

## 7. Test Suite and Results

The model was verified against five standard test circuits at ±3 V supply (or asymmetric supply where noted), plus a comprehensive Xschem-generated testbench with a 0–3 V single supply.

### 7.1 Test 1 — Input Voltage Offset (DC)

**Circuit:** Non-inverting amplifier, gain = 11 (R1 = 1 kΩ, R2 = 10 kΩ), input grounded, ±3 V supply.

**Method:** DC operating point. The output voltage divided by the gain yields the effective input offset.

| Measurement | Value |
|-------------|-------|
| V(out) | 8.40 mV |
| VOFF (measured) | 0.764 mV |

**Expected:** VOFF parameter = 0.700 mV. The measured 0.764 mV includes the small effect of input bias current flowing through the feedback network, consistent with the model.

### 7.2 Test 2 — Open-Loop Gain and Bandwidth (AC)

**Circuit:** Open-loop configuration with zero offsets (VOFF = 0, IOFF ≈ 0, IB ≈ 0), 1 GΩ load, ±3 V supply.

**Method:** AC analysis, 0.1 Hz to 100 MHz, 20 points per decade.

| Measurement | Value | Expected |
|-------------|-------|----------|
| DC gain | 106.0 dB | 106.0 dB (AOLDC) |
| Unity-gain bandwidth | 957 kHz | ~1 MHz (GBP) |
| Gain at FP2 (3 MHz) | −12.2 dB | ~−10 dB (second pole) |

The open-loop response shows the expected dominant-pole roll-off at 20 dB/decade, transitioning to 40 dB/decade beyond FP2.

### 7.3 Test 3 — Slew Rate (Transient)

**Circuit:** Inverting amplifier, gain = −1 (R1 = R2 = 10 kΩ), driven by a ±1 V square wave at 10 kHz, ±3 V supply.

**Method:** Transient analysis, 300 µs. Slew rate measured from −0.5 V to +0.5 V (rise) and +0.5 V to −0.5 V (fall).

| Measurement | Value | Expected |
|-------------|-------|----------|
| Positive slew rate | 498 kV/s | 500 kV/s (PSRT) |
| Negative slew rate | 498 kV/s | 500 kV/s (NSRT) |

### 7.4 Test 4 — Output Voltage Limiting (Transient)

**Circuit:** Non-inverting amplifier, gain = 11, driven by a 1 V peak 1 kHz sine wave. Asymmetric supplies: VDD = +3 V, VSS = −2 V.

**Method:** Transient analysis, 3 ms. Output clips at both supply rails.

| Measurement | Value | Expected |
|-------------|-------|----------|
| V(out) max | +3.005 V | ≈ VDD = +3 V |
| V(out) min | −2.005 V | ≈ VSS = −2 V |

The output clips at each supply rail with approximately 5 mV of overshoot. The asymmetric supplies confirm that the output clamp correctly tracks each rail independently.

### 7.5 Test 5 — Unity-Gain Buffer (DC Sweep)

**Circuit:** Voltage follower (output fed back to inverting input), 1 MΩ load, 0–3 V single supply.

**Method:** DC sweep of input from 0 to 3 V in 0.01 V steps.

| Measurement | Value | Expected |
|-------------|-------|----------|
| Tracking error (max) | +0.789 mV | ≈ VOFF = 0.7 mV |
| Tracking error (min) | +0.700 mV | ≈ VOFF = 0.7 mV |
| V(out) at mid-supply | 1.5007 V | 1.5 V + VOFF |

The output tracks the input across the full 0–3 V range with a constant offset equal to the input offset voltage. No convergence issues.

### 7.6 Test 6 — Xschem Testbench (Combined)

**Circuit:** The user's Xschem-generated testbench (`test_opamp_va.spice`) instantiates two op-amps on a 0–3 V single supply: one as a unity-gain buffer (XU1, output Y) and one as an open-loop comparator (XU2, output Z).

**Unity-gain buffer (Y):** Tracks the input correctly across the full sweep range with the expected offset voltage.

**Comparator (Z):** Affected by the DC sweep hysteresis issue described in §6.1. The output remains stuck at one rail throughout each DC sweep and does not transition. The comparator operates correctly in transient analysis.

---

## 8. File Organisation

```
opamp/
├── model/
│   ├── opamp.va          Verilog-A source
│   └── opamp.osdi        Compiled OSDI shared library
├── test/
│   ├── test_dc_offset.spice       Test 1: DC offset (±3 V)
│   ├── test_ac_gain.spice         Test 2: AC gain/bandwidth (±3 V)
│   ├── test_transient_slew.spice  Test 3: Slew rate (±3 V)
│   ├── test_voltage_limit.spice   Test 4: Output voltage limiting (+3/−2 V)
│   └── test_unity_gain.spice      Test 5: Unity-gain buffer DC sweep (0–3 V)
├── test_opamp_va.spice            Test 6: Xschem combined testbench (0–3 V)
└── docs/
    └── Model-Implementation-Report.md
```

---

## 9. Usage

### Compilation

```bash
openvaf model/opamp.va
```

This produces `model/opamp.osdi`.

### ngspice Netlist

```spice
.control
pre_osdi model/opamp.osdi
.endc

.model myamp opamp_module GBP=2e6 AOLDC=100

N1 inp inn out vdd vss myamp
```

Instances are created with the `N` prefix (OSDI device). Parameters can be overridden on the `.model` card. The `pre_osdi` directive must appear in a `.control` block before the circuit definition.
