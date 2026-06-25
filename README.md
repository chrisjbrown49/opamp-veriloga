# Operational Amplifier Verilog-A Macromodel

A behavioural op-amp macromodel written in Verilog-A, compiled with [OpenVAF](https://openvaf.semimod.de/) and simulated with [ngspice](https://ngspice.sourceforge.io/) via the OSDI interface.

Based on the modular macromodel architecture by Mike Brinson (see `docs/Qucs-Brinson Model.pdf`). Default parameters model a typical UA741.

## Quick Start

### Compile

```bash
openvaf model/opamp.va
```

This produces `model/opamp.osdi`.

### Use in an ngspice netlist

```spice
.control
pre_osdi model/opamp.osdi
.endc

.model myamp opamp_module GBP=2e6 AOLDC=100

N1 inp inn out vdd vss myamp
```

Instances use the `N` prefix (OSDI device). Parameters can be overridden on the `.model` card.

### Run the test suite

```bash
cd opamp
ngspice -b test/test_dc_offset.spice
ngspice -b test/test_ac_gain.spice
ngspice -b test/test_transient_slew.spice
ngspice -b test/test_voltage_limit.spice
ngspice -b test/test_unity_gain.spice
```

## Module Interface

```
module opamp_module(inp, inn, out, vdd, vss);
```

| Port  | Description            |
|-------|------------------------|
| `inp` | Non-inverting input    |
| `inn` | Inverting input        |
| `out` | Output                 |
| `vdd` | Positive supply rail   |
| `vss` | Negative supply rail   |

## Key Parameters

| Parameter | Default   | Description                        |
|-----------|-----------|------------------------------------|
| `GBP`    | 1 MHz     | Gain–bandwidth product             |
| `AOLDC`  | 106 dB    | Open-loop DC voltage gain          |
| `FP2`    | 3 MHz     | Second-pole frequency              |
| `RO`     | 75 Ω      | Output resistance (must be > 1 Ω)  |
| `VOFF`   | 0.7 mV    | Input offset voltage               |
| `PSRT`   | 500 kV/s  | Positive slew rate                 |
| `NSRT`   | 500 kV/s  | Negative slew rate                 |
| `ILMAX`  | 35 mA     | Maximum output current             |
| `RPULL`  | 1 MΩ      | Output pull resistor to each rail  |

See `docs/Model-Implementation-Report.md` for the full parameter list and detailed architecture description.

## Floating-Supply / Floating-Ground Operation

The model supports circuits where VDD and VSS float relative to simulator ground (e.g. battery-powered implantable devices with a driven virtual ground).  Three modifications enable this:

1. **Supply stubs** (`I(vdd,vss)` between rails) — the model no longer anchors VDD or VSS to absolute simulator ground.
2. **Output pull resistors** (`RPULL`, default 1 MΩ to each rail) — provide the DC Jacobian path that ngspice needs to find the floating operating point.
3. **Supply-referenced output stage** — drive current flows through VDD and VSS rather than through absolute ground, so the supply rail potentials respond correctly when they are floating and the output is held fixed by feedback.  The Thevenin equivalent at the output is unchanged: `V_th = V(n4)`, `R_th = RO`.

For floating-supply netlists, set `.ic` initial conditions on the supply nodes and the output to assist first-iteration convergence, for example:

```spice
.ic V(vdd)=2.5 V(vss)=-0.5 V(out)=1.0
```

## Known Limitations

**DC sweep in open-loop (comparator) configurations** — the `if/else` non-linearity in the slew rate limiter can still cause the Newton–Raphson solver to become trapped at one supply rail during DC sweeps.

The current limiter and output voltage clamp, which were previously also `if/else` blocks and shared this problem, have been replaced with smooth `sqrt`-based equivalents, so DC sweep convergence is significantly improved in practice.

This does **not** affect:
- Transient analysis (comparator switching works correctly)
- Operating point analysis at a fixed bias
- Any closed-loop configuration (buffers, amplifiers, integrators, etc.)

Fully resolving the remaining slew-rate-limiter non-linearity requires a smooth approximation of that stage as well. See §6 of the implementation report for details.

## Project Structure

```
opamp/
├── model/
│   ├── opamp.va                   Verilog-A source
│   └── opamp.osdi                 Compiled OSDI library (not tracked)
├── test/
│   ├── test_dc_offset.spice       DC offset measurement
│   ├── test_ac_gain.spice         Open-loop gain & bandwidth
│   ├── test_transient_slew.spice  Slew rate measurement
│   ├── test_voltage_limit.spice   Output voltage clamping
│   └── test_unity_gain.spice      Unity-gain buffer DC sweep
├── docs/
│   ├── Model-Implementation-Report.md
│   └── Qucs-Brinson Model.pdf
└── test_opamp_va.spice            Xschem combined testbench
```

## License

The original Brinson model is licensed under the GNU General Public License v2 or later. See the header of `model/opamp.va` for details.
