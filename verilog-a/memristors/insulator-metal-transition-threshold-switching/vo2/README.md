# VO2 — insulator-metal-transition threshold switching

Verilog-A compact model of a vanadium dioxide (VO2) relaxation-oscillator device.

## Physics class

`insulator-metal-transition-threshold-switching`

VO2 has a volatile, electrically driven insulator-metal transition (IMT). Applying
enough voltage collapses it into a conductive metallic state; removing the bias lets it
revert. It is a **volatile threshold switch**, not a non-volatile memory — nothing is
retained once the bias is gone.

This is a different mechanism from the `electrothermal-threshold-switching` class in
this repo (NbOx), where switching arises from Joule-driven thermal runaway in a
Frenkel-Poole conductor. Both are threshold switches; only the mechanism differs, which
is why they are filed under separate physics classes.

## Model

Maffezzoni-style **ideal two-state resistor**:

- insulating state → resistance `Rins`
- metallic state → resistance `Rmet`
- device voltage rises above `VIMT` → flips metallic
- device voltage falls below `VMIT` → flips insulating

Because `VMIT < VIMT`, turn-on and turn-off occur at different voltages. That difference
is the hysteresis, and it is what allows the device to oscillate when placed in an RC
circuit.

**No temperature, no Joule heating, no phase-transition physics** — deliberately. The
source paper is explicit that real VO2 switching is current- and thermally driven, and
that this abstraction cannot capture cycle-to-cycle threshold drift. It is used because
it is fast and robust enough to simulate whole oscillator networks.

### Governing equations

State-dependent threshold (this single line produces the hysteresis):

    Vth = VMIT + (VIMT - VMIT) * state

Smooth target state (tanh rather than a step, so derivatives exist everywhere):

    target = 0.5 * (1 - tanh((Vvo2 - Vth) / vsm))

State dynamics as a first-order lag:

    tau * d(state)/dt = target - state

Conduction, interpolated in **conductance** (keeps numbers well-scaled; interpolating
resistance would swing across orders of magnitude):

    G = (1 - state)/Rmet + state/Rins
    I = Vvo2 * G

### State-node convention

The internal node `st` is not an electrical quantity. Its "voltage" is the device state:

- `V(st) = 1` → fully insulating
- `V(st) = 0` → fully metallic

matching `Vstate` in the paper's Fig. 2. Probe it in ngspice as `v(n1#st)` — note `#`,
not `.`, for OSDI internal nodes.

## Parameters (Table I of the source paper)

| Symbol | Meaning | Static fit | Dynamic fit | Unit |
|---|---|---|---|---|
| `VMIT` | metal→insulator threshold | 0.5 | 0.4 | V |
| `VIMT` | insulator→metal threshold | 1.65 | 1.2 | V |
| `Rmet` | metallic resistance | 0.8 | 2.8 | kOhm |
| `Rins` | insulating resistance | 50 | 13.5 | kOhm |

Numerical parameters (not from the paper; chosen for this implementation):

| Symbol | Meaning | Default |
|---|---|---|
| `tau` | intrinsic transition time; keep << Rext*Cext | 10 ns |
| `vsm` | threshold smoothing width | 10 mV |
| `Rser` | optional built-in series resistance | 0 (use an external resistor) |

**Two fits, one device.** The same physical device measured statically and while
oscillating gives different parameters — the authors attribute this to self-heating
shifting the thresholds downward during oscillation. The model has no temperature term
and cannot capture that drift, so the paper fits the warmed-up state and recommends the
**dynamic fit for oscillator work**. Module defaults are the dynamic fit.

**Attribution:** all four device parameters are from Debets et al., Table I. The
`tau`/`vsm`/`Rser` values are implementation choices, not published data.

## Files

| File | Location | Purpose |
|---|---|---|
| `VO2_Maffezzoni.va` | here | the compact model |
| `test1_static_iv.cir` | `testbenches/.../vo2/` | static I-V sweep, 100 Ohm and 1 kOhm |
| `test2_oscillator.cir` | `testbenches/.../vo2/` | relaxation oscillator |
| `test3_freq_vs_rext.cir` | `testbenches/.../vo2/` | frequency vs Rext, bias window |
| `RESULTS.md` | `results/.../vo2/` | measured outcomes and figures |
| `VO2_VerilogA_explained.md` | `docs/model-notes/.../` | full walkthrough |

There is **no LTspice version** of this model. LTspice has no native Verilog-A support,
and the `ddt()`-based state dynamics would need an RC-analogue workaround. Verilog-A
only, by design.

## Toolchain

    openvaf.exe VO2_Maffezzoni.va          # -> VO2_Maffezzoni.osdi
    ngspice.exe test2_oscillator.cir       # netlist loads it with pre_osdi

Built and tested with OpenVAF 23.2.0 and ngspice 46 on Windows.

The compiled `.osdi` is a platform-specific build artifact and is **not committed** —
see the repo `.gitignore`.

## Validation status

Compiles cleanly under OpenVAF. All three testbenches run. Five quantitative predictions
from the source paper confirmed, one difference explained, two items open — see
`RESULTS.md` for the numbers.

## Known limitations

- **No temperature.** Cannot reproduce the cycle-to-cycle threshold drift the paper
  observes in real devices; this is inherited from the Maffezzoni formulation.
- **Partial states exist.** Because the state variable is continuous, the device can
  rest in stable half-switched states under some loads. The reference model, an ideal
  comparator, cannot. This is visible at `Rext` = 1 kOhm and documented in `RESULTS.md`.
- **No coupled-pair support tested.** The paper's headline finding concerns coupled
  oscillators; that testbench has not been built yet.

## Reference

R. Debets, M. Galetta, T. Negrini, J. Verest, S. Lahkar, D. F. Falcone, S. Karg,
V. Bragaglia, A. Todri-Sanial, "Investigation of Oscillatory Neural Networks with
Verilog-A Models of VO2 Relaxation Oscillators and Analog RRAM Coupling Elements",
*IEEE JETCAS*, vol. 16, no. 2, June 2026. DOI 10.1109/JETCAS.2026.3705420

Underlying formulation: P. Maffezzoni, L. Daniel, N. Shukla, S. Datta, A. Raychowdhury,
*IEEE TCAS-I* 62(9), 2207 (2015).
