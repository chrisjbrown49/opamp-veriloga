# Operational Amplifier Verilog-A Macromodel — Implementation Report

## 1. Overview

This document describes the Verilog-A behavioural macromodel for a general-purpose operational amplifier implemented in `model/opamp.va`. The model is based on the modular op-amp macromodel architecture developed by Mike Brinson for the Qucs simulator, adapted here for compilation with **OpenVAF** and simulation with **ngspice** via the OSDI (Open Source Device Interface) mechanism.

The default parameter set models a typical **UA741** operational amplifier. All parameters are exposed and can be overridden per-instance or per-model card in a SPICE netlist, making the model suitable for representing a wide range of voltage-feedback op-amps.

All non-linearities in the model use smooth `tanh`-based functions — there are no `if/else` conditionals in the signal path. This ensures continuous Jacobian derivatives throughout the operating range, which is essential for robust Newton–Raphson convergence in DC sweep and transient analysis.

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
| `CSCALE` | 50 | S | Output current clamp conductance |
| `VILIMSF` | 1000 | — | Gain-stage clamp scaling factor (see §5.6) |

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
| `Slewavg` | (Slewratepositive + Slewratenegative) / 2 | Average slew-rate threshold for the tanh saturation |
| `Vilim` | VILIMSF × max(Slewratepositive, Slewratenegative) | Gain-stage voltage clamp limit |

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
                     n12 ◄────┤  Slew Limiter │ (tanh saturation)
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n3  ◄────┤  First Pole   │ (gain stage + Vilim clamp)
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n5  ◄────┤  Second Pole  │
                              └──────────────┘
                                      │
                              ┌──────────────┐
                     n4  ◄────┤  Pass-through │
                              └──────────────┘
                                      │
                              ┌──────────────┐
              n2 ──► out  ◄───┤  Output Stage │ (RO + current clamp
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

The slew rate limiter uses a `tanh` saturation function to smoothly limit the drive voltage to the gain stage:

```verilog
Slewavg = (Slewratepositive + Slewratenegative) / 2.0;
I(n12) <+ -Slewavg * tanh(V(n11) / Slewavg);
I(n12) <+ V(n12);
```

This produces the transfer characteristic `V(n12) = Slewavg × tanh(V(n11) / Slewavg)`, which has three key properties:

1. **Unity gain for small signals:** When `|V(n11)| ≪ Slewavg`, `tanh(x) ≈ x`, so `V(n12) ≈ V(n11)`. The slew limiter is transparent to normal-amplitude signals.

2. **Smooth limiting for large signals:** When `|V(n11)| ≫ Slewavg`, `tanh → ±1` and `V(n12) → ±Slewavg`. The drive to the gain stage is bounded, limiting the current available to charge `CP1` and constraining the output slew rate.

3. **Bounded Jacobian:** The self-admittance at `n12` is always 1 S (from the `V(n12)` term). The cross-admittance `∂I(n12)/∂V(n11)` ranges smoothly from −1 to 0. There are no stiffness spikes or Jacobian discontinuities.

The threshold voltages are derived from the slew rate parameters as:

```
Slewrate_threshold = SR / (2π × GBP)
```

For the default parameters, `Slewavg ≈ 0.0796 V`. The `tanh` function begins to compress the signal at about 50% of this value and is effectively saturated above 3× the threshold.

When `PSRT ≠ NSRT`, the limiter uses the average of the two thresholds. For the default symmetric case (`PSRT = NSRT`), this is exact.

### 5.6 First Pole — Gain Stage with Voltage Clamp

The dominant pole is implemented as a transresistance amplifier with a parallel RC:

```verilog
I(n3) <+ -V(n12);              // drive current from slew limiter
I(n3) <+ V(n3) / RP1;          // shunt resistance (sets DC gain)
I(n3) <+ ddt(CP1 * V(n3));     // shunt capacitance (sets pole frequency)
```

The DC gain is `V(n3) / V(n12) = RP1 = 10^(AOLDC/20)` and the pole frequency is `1 / (2π × RP1 × CP1) = GBP / RP1`.

**Internal voltage clamp (Vilim):** In open-loop or comparator configurations the large DC gain (≈ 200 000) can drive `V(n3)` to extreme voltages. A smooth `tanh`-based conductance clamp bounds `V(n3)` to the range ±Vilim:

```verilog
Vilim = VILIMSF * max(Slewratepositive, Slewratenegative);

Vn3hi = V(n3) - Vilim;
Vn3lo = (-Vilim) - V(n3);
I(n3) <+ 0.5 * 1.0 * (1.0 + tanh(50.0 * Vn3hi)) * Vn3hi;
I(n3) <+ 0.5 * 1.0 * (1.0 + tanh(50.0 * Vn3lo)) * (-Vn3lo);
```

The clamp limit `Vilim` is derived from the slew-rate threshold rather than from the supply voltages, keeping it purely parameter-dependent. This avoids introducing Jacobian coupling between the gain-stage node `n3` and the supply nodes.

With the default `VILIMSF = 1000` and default slew parameters, `Vilim ≈ 80 V`. This provides sufficient headroom for the gain stage during normal operation (where `V(n3)` tracks the output voltage) while preventing the extreme internal voltages (thousands of volts) that cause convergence failures in open-loop configurations.

**Anti-windup effect:** When the output is clamped at a supply rail (e.g. in the voltage-limit test), the gain-stage integrator would otherwise "wind up" — accumulating voltage on `V(n3)` far beyond the output range. With `Vilim = 80 V`, the maximum windup is bounded. When the input reverses, `V(n3)` can slew back through this range in approximately `2 × Vilim / (Slewavg / CP1) ≈ 320 µs`, which is fast enough for typical operating frequencies. At the previous default of `VILIMSF = 10000` (`Vilim ≈ 800 V`), recovery could take over 3 ms, exceeding a full cycle of a 1 kHz signal and preventing the output from reaching the opposite rail.

Each clamp term implements a soft-switched 1 S conductance that activates when `V(n3)` exceeds the clamp boundary. The `tanh` function provides a smooth transition (approximately 40 mV wide with K = 50), ensuring continuous derivatives in the Jacobian matrix.

In closed-loop operation, `V(n3)` is proportional to the output voltage and is always well below `Vilim`, so the clamp is completely inactive and does not affect circuit accuracy.

### 5.7 Second Pole

A unity-gain low-pass filter implements the second pole:

```verilog
I(n5) <+ -V(n3);               // drive from first pole
I(n5) <+ V(n5) / RP2;          // RP2 = 1 Ω
I(n5) <+ ddt(CP2 * V(n5));     // pole at FP2
```

The pole frequency is `1 / (2π × RP2 × CP2) = FP2`. This models the parasitic high-frequency roll-off beyond the gain-bandwidth product.

### 5.8 Signal Pass-Through

The signal from the second pole is passed through to the output driver at unity gain:

```verilog
I(n4) <+ -V(n5);
I(n4) <+ V(n4);
```

This gives `V(n4) = V(n5)` at DC. Node `n4` is the voltage that drives the output resistance network. Critically, this is a simple pass-through with no multiplicative dependence on `V(n5)`, which ensures that `V(n4)` always has the same sign as `V(n5)`. This prevents the reversed-drive pathology that can arise when a feedback-based current limiter multiplies by `V(n5)` (see §5.10).

### 5.9 Output Resistance

The output resistance `RO` is split into two series elements for current sensing:

```verilog
I(n4, n2) <+ V(n4, n2) / (RO - 1);   // (RO − 1) Ω
I(n2, out) <+ V(n2, out);              // 1 Ω current sense
```

The total output impedance between `n4` and `out` is `(RO − 1) + 1 = RO` ohms. The 1 Ω element between `n2` and `out` serves as the current-sense resistor: `V(n2, out)` equals the output current in amps.

### 5.10 Output Current Limiter

The output current is limited by a `tanh`-based clamp applied at the current-sense node `n2`. When the magnitude of `V(n2, out)` (which represents the output current through the 1 Ω sense resistor) exceeds `ILMAX`, the clamp absorbs excess current at `n2`, preventing it from reaching the output:

```verilog
Iohi = V(n2, out) - ILMAX;
Iolo = (-ILMAX) - V(n2, out);
I(n2) <+ 0.5 * CSCALE * (1.0 + tanh(50.0 * Iohi)) * Iohi;
I(n2) <+ 0.5 * CSCALE * (1.0 + tanh(50.0 * Iolo)) * (-Iolo);
```

When `V(n2, out)` is within ±ILMAX, both `tanh` factors are ≈ 0 and the clamp contributes no current. When `V(n2, out)` exceeds `ILMAX`, the positive clamp term activates, draining current from `n2` to ground. This lowers `V(n2)`, reducing the voltage across the 1 Ω sense resistor and thereby limiting the output current. The negative limit works symmetrically.

The `CSCALE` parameter (default 50 S) controls how stiffly the current is clamped. Larger values enforce the limit more tightly.

This formulation is **independent of `V(n5)`**, which is essential for robustness. An earlier multiplicative design (`CSCALE × V(n5) × ΔI`) suffered from a sign-flip pathology: when `|V(n5)| > RO / CSCALE` (= 1.5 V with defaults), the feedback term overwhelmed the signal pass-through, inverting V(n4) and driving current backwards through the output stage. This created a stable parasitic DC equilibrium where the output was pinned at a supply rail regardless of input. The decoupled `tanh` clamp eliminates this failure mode entirely.

### 5.11 Output Voltage Clamp

The output voltage is clamped to the supply rails using a pair of smooth `tanh`-based conductance clamps:

```verilog
Vhi = V(out) - V(vdd);
Vlo = V(vss) - V(out);
I(out) <+ 0.5 * 10.0 * (1.0 + tanh(50.0 * Vhi)) * Vhi;
I(out) <+ 0.5 * 10.0 * (1.0 + tanh(50.0 * Vlo)) * (-Vlo);
```

Each term implements a soft-switched 10 S conductance that activates when the output exceeds the respective supply rail. The `tanh` function with K = 50 gives a smooth transition approximately 40 mV wide around each rail, ensuring:

- **Continuity:** The clamp current and its first derivative are continuous everywhere, which is essential for Newton–Raphson convergence during DC sweep and transient analysis.
- **Rail tracking:** The clamp limits automatically follow `V(vdd)` and `V(vss)`, so the output swing adapts to any supply configuration without requiring parameter changes.
- **Low interference:** Below the supply rails, the `tanh` factor evaluates to ≈ 0 and the clamp contributes no current, preserving the accuracy of the linear-region behaviour.

The 10 S conductance (vs. 1 S for the internal gain-stage clamp) provides a stiffer clamp at the output, appropriate for the lower-impedance output node.

---

## 6. Summary of Smooth Non-Linearities

All non-linearities in the model use smooth, continuously differentiable functions. There are no `if/else` conditionals in the signal path. Four subsystems use `tanh`-based mechanisms:

| Subsystem | Location | Type | Key Parameters | Boundary |
|-----------|----------|------|----------------|----------|
| Slew rate limiter | n12 | tanh saturation | Slewavg | ±Slewavg |
| Gain-stage voltage clamp | n3 | tanh conductance clamp | G=1 S, K=50 | ±Vilim |
| Output current limiter | n2 | tanh conductance clamp | G=CSCALE (50 S), K=50 | ±ILMAX |
| Output voltage clamp | out | tanh conductance clamp | G=10 S, K=50 | V(vdd), V(vss) |

The **slew rate limiter** uses a direct `tanh` saturation function, `Slewavg × tanh(V(n11) / Slewavg)`, which inherently provides unity gain for small signals and saturates at ±Slewavg. Its Jacobian is always bounded between 0 and 1.

The remaining three subsystems use the conductance clamp form:

```
I_clamp = 0.5 × G × (1 + tanh(K × ΔV)) × ΔV
```

where:
- **G** is the maximum clamp conductance
- **K** controls the transition sharpness (K = 50 gives ≈ 40 mV transition width)
- **ΔV** is the excursion beyond the clamp boundary

When ΔV ≪ 0, the factor `(1 + tanh(K × ΔV))` → 0 and no clamp current flows. When ΔV ≫ 0, the factor → 2 and the clamp behaves as a linear conductance `G × ΔV`. The smooth transition ensures that the Jacobian matrix has no discontinuities.

---

## 7. Simulator Options for Transient Analysis

Transient simulations that drive the output into clipping at the supply rails can encounter "timestep too small" convergence failures with ngspice's default settings. This occurs because the interaction between the output voltage clamp, the gain-stage integrator, and the slew rate limiter creates a stiff system at the clipping boundary.

The following `.option` settings resolve this and are recommended for any transient simulation where the output is expected to clip:

```spice
.option method=gear reltol=5e-3 itl4=500 trtol=7
```

| Option | Value | Purpose |
|--------|-------|---------|
| `method=gear` | — | Use Gear (BDF) integration instead of the default trapezoidal rule. Gear is L-stable, making it more robust for stiff systems. |
| `reltol=5e-3` | 0.5% | Relax the relative tolerance from the default 0.1% to 0.5%. This allows the solver to accept slightly less precise solutions at the clipping boundary without triggering timestep reduction. |
| `itl4=500` | — | Increase the transient iteration limit from the default 10 to 500, giving the Newton–Raphson solver more attempts to converge at difficult operating points. |
| `trtol=7` | — | Increase the transient truncation error tolerance from the default 7 (ngspice's default is already 7, but setting it explicitly ensures it is not overridden). This controls how aggressively the simulator reduces the timestep to control local truncation error. |

These settings do not affect DC or AC analysis. They can safely be included in all testbenches. Simulations that do not clip at the rails will converge with or without these options.

---

## 8. Test Suite and Results

The model was verified against five standard test circuits at ±3 V supply (or asymmetric supply where noted), plus a comprehensive Xschem-generated testbench with a 0–3 V single supply. All tests produce `.dat` output files in the `test/` folder.

### 8.1 Test 1 — Input Voltage Offset (DC)

**Circuit:** Non-inverting amplifier, gain = 11 (R1 = 1 kΩ, R2 = 10 kΩ), input grounded, ±3 V supply.

**Method:** DC operating point. The output voltage divided by the gain yields the effective input offset.

| Measurement | Value |
|-------------|-------|
| V(out) | 8.40 mV |
| VOFF (measured) | 0.764 mV |

**Expected:** VOFF parameter = 0.700 mV. The measured 0.764 mV includes the small effect of input bias current flowing through the feedback network, consistent with the model.

### 8.2 Test 2 — Open-Loop Gain and Bandwidth (AC)

**Circuit:** Open-loop configuration with zero offsets (VOFF = 0, IOFF ≈ 0, IB ≈ 0), 1 GΩ load, ±3 V supply.

**Method:** AC analysis, 0.1 Hz to 100 MHz, 20 points per decade.

| Measurement | Value | Expected |
|-------------|-------|----------|
| DC gain | 106.0 dB | 106.0 dB (AOLDC) |
| Unity-gain bandwidth | 957 kHz | ~1 MHz (GBP) |
| Gain at FP2 (3 MHz) | −12.2 dB | ~−10 dB (second pole) |

The open-loop response shows the expected dominant-pole roll-off at 20 dB/decade, transitioning to 40 dB/decade beyond FP2.

### 8.3 Test 3 — Slew Rate (Transient)

**Circuit:** Inverting amplifier, gain = −1 (R1 = R2 = 10 kΩ), driven by a ±1 V square wave at 10 kHz, ±3 V supply.

**Method:** Transient analysis, 300 µs. Slew rate measured from −0.5 V to +0.5 V (rise) and +0.5 V to −0.5 V (fall).

| Measurement | Value | Expected |
|-------------|-------|----------|
| Positive slew rate | 511 kV/s | 500 kV/s (PSRT) |
| Negative slew rate | 511 kV/s | 500 kV/s (NSRT) |

The measured slew rate is 2.3% above the target, which reflects the soft roll-off inherent in the `tanh` saturation function. The `tanh` begins compressing the signal slightly before the threshold, resulting in a marginally higher effective slew rate than the hard-clipped ideal. This is well within acceptable tolerance.

### 8.4 Test 4 — Output Voltage Limiting (Transient)

**Circuit:** Non-inverting amplifier, gain = 11, driven by a 1 V peak 1 kHz sine wave. Asymmetric supplies: VDD = +3 V, VSS = −2 V.

**Method:** Transient analysis, 3 ms. Output clips at both supply rails. This test requires the simulator options described in §7 to avoid timestep convergence failures.

| Measurement | Value | Expected |
|-------------|-------|----------|
| V(out) max | +3.008 V | ≈ VDD = +3 V |
| V(out) min | −2.008 V | ≈ VSS = −2 V |

The output clips at each supply rail with approximately 8 mV of overshoot due to the smooth tanh transition. The asymmetric supplies confirm that the output clamp correctly tracks each rail independently.

### 8.5 Test 5 — Unity-Gain Buffer (DC Sweep)

**Circuit:** Voltage follower (output fed back to inverting input), 1 MΩ load, 0–3 V single supply.

**Method:** DC sweep of input from 0 to 3 V in 0.01 V steps.

| Measurement | Value | Expected |
|-------------|-------|----------|
| Tracking error (max) | +0.789 mV | ≈ VOFF = 0.7 mV |
| Tracking error (min) | +0.703 mV | ≈ VOFF = 0.7 mV |
| V(out) at mid-supply | 1.5007 V | 1.5 V + VOFF |

The output tracks the input across the full 0–3 V range with a constant offset equal to the input offset voltage. No convergence issues or parasitic equilibria.

### 8.6 Test 6 — Xschem Testbench (Combined)

**Circuit:** The user's Xschem-generated testbench (`test_opamp_va.spice`) instantiates two op-amps on a 0–3 V single supply: one as a unity-gain buffer (XU1, output Y fed back to inverting input) and one as an open-loop comparator (XU2, output Z, with reference voltage VB on the inverting input). A DC sweep of V3 from 0 to 3 V (0.01 V steps) is performed at six values of VB (1.0 to 2.0 V in 0.2 V steps), using `reset` and `alterparam` to iterate.

**Result:** All seven analyses completed successfully (1 OP + 6 DC sweeps × 301 data points each, no convergence failures).

**Unity-gain buffer (Y) — correct at all VB values:**

The buffer tracks the input across the full 0–3 V sweep with a constant offset of 0.70–0.79 mV (= VOFF) at every VB value. No tracking failures or parasitic equilibria.

**Comparator (Z) — correct at all VB values:**

| VB (V) | Switch Point (V) | V(Z) Low (mV) | V(Z) High (V) |
|--------|-------------------|----------------|----------------|
| 1.0 | 1.005 | −8 | 3.008 |
| 1.2 | 1.205 | −8 | 3.008 |
| 1.4 | 1.405 | −8 | 3.008 |
| 1.6 | 1.605 | −8 | 3.008 |
| 1.8 | 1.805 | −8 | 3.008 |
| 2.0 | 2.005 | −8 | 3.008 |

The comparator switches cleanly at each threshold with no hysteresis. The output swings from ≈ 0 V (VSS) to ≈ 3 V (VDD). The switch-point offset of +5 mV from VB is consistent with the model's input offset voltage.

---

## 9. File Organisation

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
│   ├── test_unity_gain.spice      Test 5: Unity-gain buffer DC sweep (0–3 V)
│   ├── test_ac_gain.dat           AC gain/phase output
│   ├── test_transient_slew.dat    Slew rate waveform output
│   ├── test_unity_gain.dat        Unity-gain DC sweep output
│   └── test_voltage_limit.dat     Voltage limit waveform output
├── test_opamp_va.spice            Test 6: Xschem combined testbench (0–3 V)
└── docs/
    └── Model-Implementation-Report.md
```

---

## 10. Usage

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

For transient simulations where the output may clip at the supply rails, include the convergence options:

```spice
.option method=gear reltol=5e-3 itl4=500 trtol=7
```

See §7 for details.

### Parameter Tuning Notes

**VILIMSF** controls the internal gain-stage voltage clamp as a multiple of the slew-rate threshold:

```
Vilim = VILIMSF × max(Slewratepositive, Slewratenegative)
```

With the default parameters, `Slewratepositive ≈ 0.08 V`, giving `Vilim ≈ 80 V` at `VILIMSF = 1000`. The clamp is inactive during normal closed-loop operation and only activates in open-loop or comparator configurations to bound internal node voltages for convergence. If the model is used at higher supply voltages (e.g. ±15 V), `VILIMSF` may need to be increased to provide adequate headroom above the supply range.

**CSCALE** controls the stiffness of the output current clamp (in siemens). The default value of 50 provides soft current limiting that prevents the output current from significantly exceeding ILMAX while maintaining smooth derivatives.
