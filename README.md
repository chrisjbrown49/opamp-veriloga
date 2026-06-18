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
| `RO`     | 75 Ω      | Output resistance                  |
| `VOFF`   | 0.7 mV    | Input offset voltage               |
| `PSRT`   | 500 kV/s  | Positive slew rate                 |
| `NSRT`   | 500 kV/s  | Negative slew rate                 |
| `ILMAX`  | 35 mA     | Maximum output current             |

See `docs/Model-Implementation-Report.md` for the full parameter list and detailed architecture description.

## Known Limitations

**DC sweep in open-loop (comparator) configurations** does not work correctly. The `if/else` non-linearities in the slew rate limiter, current limiter, and output voltage clamp cause the Newton–Raphson solver to become trapped at one supply rail during DC sweeps. The comparator output never transitions between rails.

This does **not** affect:
- Transient analysis (comparator switching works correctly)
- Operating point analysis at a fixed bias
- Any closed-loop configuration (buffers, amplifiers, integrators, etc.)

Resolving this requires replacing the `if/else` non-linearities with smooth functions while preserving correct DC operating point convergence in complex multi-amplifier circuits. See §6 of the implementation report for details.

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
