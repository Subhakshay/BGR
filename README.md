# Bandgap Reference (BGR) Circuit — Self-Biased Current-Mirror (Widlar-Type) Design

---

## 1. Overview

This project implements a **Bandgap Reference (BGR)** circuit using a ** current-mirror topology**, designed and simulated entirely in **LTspice**. Unlike the classical op-amp-based bandgap, this design has **no dedicated error amplifier** — the loop biases itself through a positive-feedback current-mirror arrangement that settles to a stable, well-defined operating current.

The goal of the circuit is to produce an output voltage that is **independent of temperature and supply voltage**, referenced to the silicon bandgap voltage of ~1.12 eV, extrapolated to ~1.2 V at 0 K.

---

## 2. Why We Need a Bandgap Reference

Almost every analog/mixed-signal IC (ADCs, DACs, LDOs, PLLs, sensors) needs a **stable DC voltage** that does not drift with:

- **Temperature** — because transistor parameters (VBE, mobility, threshold voltage) are all thermally dependent.
- **Supply voltage** — because the reference must not "follow" noisy or varying VDD rails.
- **Process variation** — because absolute resistor/transistor values vary chip to chip, but *ratios* of matched devices are far more reliable.

A bandgap reference solves this by combining two voltages that have **equal and opposite temperature dependence**, so that their weighted sum is (to first order) temperature-invariant.

---

## 3. Basic Building Blocks

Every bandgap reference is built from two complementary quantities:

| Quantity | Behavior with Temperature | Physical Origin |
|---|---|---|
| **CTAT** (Complementary To Absolute Temperature) | Decreases with T | Base-emitter voltage VBE of a forward-biased BJT (or diode) |
| **PTAT** (Proportional To Absolute Temperature) | Increases with T | Difference of two VBE's operated at unequal current densities (ΔVBE) |

By adding a scaled version of the PTAT voltage to the CTAT voltage, the negative slope of VBE is cancelled by the positive slope of the PTAT term, producing a nearly flat (zero-TC) output around 1.2 V.

---

## 4. CTAT Voltage — The Base-Emitter Voltage

For a bipolar transistor operating in forward active region, the collector current is:

```
IC = IS · exp(VBE / VT)
```

where:
- `IS` = reverse saturation current (strongly temperature dependent, ∝ T^n · exp(−VG0/VT), n ≈ 3–4)
- `VT = kT/q` = thermal voltage (k = Boltzmann constant, q = electron charge, T = absolute temperature in Kelvin)

Rearranging for VBE:

```
VBE = VT · ln(IC / IS)
```

Because `IS` increases much faster with T than the `ln` term compensates for, **VBE decreases as temperature rises**. Empirically:

```
dVBE/dT ≈ −1.5 mV/K to −2.2 mV/K   (typical range, device dependent)
```

This is the **CTAT** component. At room temperature (300 K), a typical VBE ≈ 0.65–0.75 V.

**Full first-order expression** (derived from the IS temperature dependence):

```
VBE(T) = VG0 − (VG0 − VBE(T0))·(T/T0) − (η − n)·VT·ln(T/T0)
```

where:
- `VG0` ≈ 1.205 V = extrapolated silicon bandgap voltage at 0 K
- `T0` = reference temperature (usually 300 K / 27 °C)
- `η` = temperature exponent of mobility/carrier diffusion (~4 for BJTs)
- `n` = temperature exponent assumed for bias current (n = 1 for constant/PTAT-independent current, higher if IC itself varies with T)

The last term, `(η−n)·VT·ln(T/T0)`, is the **non-linear "curvature" term** — discussed further in Section 12.

---

## 5. PTAT Voltage — Generating ΔVBE

If two identical BJTs (or a BJT and an N-times-larger BJT) are biased at **different current densities**, their VBE's differ by an amount that is directly proportional to absolute temperature.

Consider two BJTs Q1 and Q2, where Q2 has emitter area **N times** larger than Q1, and both carry currents I1 and I2 respectively:

```
VBE1 = VT · ln(I1 / IS1)
VBE2 = VT · ln(I2 / (N · IS1))
```

If the mirror forces **I1 = I2 = I**, then:

```
ΔVBE = VBE1 − VBE2 = VT · ln(N)
```

This is the key result:

```
ΔVBE = (kT/q) · ln(N)        ← PTAT, purely proportional to T
```

**Proportionality:**
```
ΔVBE ∝ T          (linear, no curvature — this term is "clean" PTAT)
ΔVBE ∝ ln(N)       (logarithmic in area/current-density ratio)
```

This ΔVBE appears across a resistor **R1** placed in series with Q2's emitter, generating a PTAT current:

```
IPTAT = ΔVBE / R1 = (VT · ln(N)) / R1
```

---

## 6. The Bandgap Principle — Adding CTAT + PTAT

The final reference voltage is built by summing the CTAT voltage (VBE of a reference transistor) with a **scaled** version of the PTAT voltage:

```
VREF = VBE + K · VT · ln(N)
```

where `K = R2/R1` is a resistor ratio (set by design) that scales the PTAT term until its positive slope exactly cancels the CTAT's negative slope at the temperature of interest.

**Condition for zero first-order TC** (at reference temp T0):

```
dVBE/dT + K · ln(N) · d(VT)/dT = 0

⇒ K = |dVBE/dT| / (ln(N) · (k/q))
```

Since `d(VT)/dT = k/q ≈ 0.0862 mV/K`, and `dVBE/dT ≈ −1.5 to −2 mV/K`, typical designs require:

```
K · ln(N) ≈ 17 – 23   (dimensionless multiplier on VT)
```

so that:

```
VREF ≈ VG0 ≈ 1.2 V    at T = T0
```

This is why every bandgap reference — regardless of topology — converges to ~1.2 V: it is literally built to extrapolate to the silicon bandgap energy.

---

## 7. Current-Mirror Topology (No Op-Amp)

Unlike the classical **op-amp-based BGR**, where an amplifier forces two branch voltages to be equal, this design uses a **self-biased loop**:

- A **PMOS current mirror** (M1, M2) on the top forces the two branch currents to be equal (assuming 1:1 mirror ratio).
- Each branch contains a diode-connected BJT: one **unit-size BJT (Q1)** and one **N-times-larger BJT (Q2)** in series with resistor **R1**.
- Because the mirror forces `I1 = I2`, the ΔVBE(= VT ln N) appears entirely across R1, self-consistently setting the branch current to `IPTAT = VT ln(N)/R1`.
- This is a **positive feedback loop**: more current → higher gate-source drive → more current. It has **two stable operating points**:
  1. **Zero-current (degenerate) state** — all currents = 0, a stable but useless equilibrium.
  2. **Desired high-current PTAT state** — the one we actually want.

A **startup circuit** is mandatory to kick the loop out of the zero-current state every time the supply is powered on.


---


## 8. Startup Circuit

### Why It Is Needed
The self-biased core has **two equilibria**: (I = 0) and (I = IPTAT). Without an external kick, parasitic capacitances and zero initial conditions mean the circuit could power up and simply sit at I = 0 forever — a "dead" bandgap output at 0 V.

### How It Works (typical implementation)
1. At power-up, a small PMOS (or resistor) senses that the core's bias node voltage is low (indicating zero current) and turns on a startup transistor.
2. This transistor injects a small current into the core's high-impedance node, forcing the mirror to begin conducting.
3. As current builds up in the core, the core's own bias node rises/falls to a value that turns the startup device **off**, so it draws **zero current** in normal operation (crucial — otherwise it adds a constant error current and burns extra power).
4. A common structure uses a diode-connected sensing transistor + a switch transistor whose gate is driven by the core's internal node, guaranteeing self-disconnection once the desired operating point is reached.

### Design Check
Always verify in LTspice with **`.ic` (initial condition) directives set to zero** on all node voltages, or a **`.tran` startup simulation from 0 V supply ramp**, to confirm the circuit does NOT latch into the zero-current state.

---

## 9. Design Equations and Component Sizing

**Step 1 — Choose emitter area ratio N** (common choices: 8, 16, 24; larger N gives larger ΔVBE for a given R1, relaxing resistor matching requirements, but consumes more area).

**Step 2 — PTAT current:**
```
IPTAT = (VT · ln N) / R1
```

**Step 3 — Choose R2/R1 ratio for zero TC** (from Section 6):
```
R2/R1 = |dVBE/dT| / (k/q · ln N)
```

**Step 4 — Output reference voltage:**
```
VREF = VBE3 + (R2/R1) · VT · ln N ≈ 1.2 V
```

**Step 5 — Mirror sizing:** M1 = M2 (unit ratio) in the core for correct ΔVBE forcing; the output-branch mirror ratio (M3:M1) sets the extra scaling if the output branch current differs from the core current.

**Step 6 — Minimum supply voltage:**
```
VDD,min ≈ VSD(M1) + VBE(Q2) + IPTAT·R1
```

This determines the minimum operating headroom — important for low-voltage designs.

---

## 10. Temperature Coefficient (TC) Analysis

The **Temperature Coefficient** of the reference is usually reported in ppm/°C:

```
TC (ppm/°C) = [ (VREF_max − VREF_min) / (VREF_avg · ΔT) ] × 10^6
```

where `VREF_max`, `VREF_min` are the reference voltage extremes over the operating temperature range `ΔT` (e.g., −40 °C to 125 °C), and `VREF_avg` is the nominal/typical value.

**First-order cancellation** (as derived above) zeroes the *linear* slope at one design temperature `T0`, but VBE's higher-order curvature (Section 12) means the reference still bows slightly at temperatures away from T0 — this residual is the dominant TC error in simple (uncorrected) designs, typically limiting TC to **~20–50 ppm/°C** without curvature correction.

---

## 11. Curvature Correction

Because `VBE(T)` contains the non-linear term `(η−n)·VT·ln(T/T0)` (Section 4), the simple PTAT+CTAT sum is not perfectly flat — it exhibits a **downward-bowing (concave) curvature** across temperature, worst at the extremes of the range.

### Common Curvature-Correction Techniques
1. **Non-linear (PTAT²) current injection** — Add a small current proportional to T² (generated by multiplying a PTAT current by another PTAT voltage) into the output resistor, whose shape approximately cancels the `T·ln T` term over the operating range.
2. **VBE temperature-dependent resistor (TC-tailored resistors)** — Use resistors with a designed temperature coefficient (e.g., combining polysilicon and diffusion resistors of opposite TC) to introduce a compensating non-linear term.
3. **Piecewise-linear compensation** — Switch in extra correction current only above/below a threshold temperature, "kinking" the curve back flat at the corners.
4. **Exponential curvature compensation** — Use a translinear loop to generate a term that mimics `exp` behavior, subtracting it near temperature extremes.

**Effect:** A well-implemented curvature correction can reduce TC from ~30–50 ppm/°C down to **~5–15 ppm/°C**.

---

## 12. PSRR (Power Supply Rejection Ratio)

PSRR quantifies how much supply noise/ripple couples through to VREF:

```
PSRR (dB) = 20 · log10( ΔVDD / ΔVREF )
```

or equivalently expressed as a rejection transfer function `VREF(s)/VDD(s)`.

### Why Self-Biased Topologies Need Care Here
Without an amplifier actively regulating the branch currents against VDD variation, the self-biased core's current depends on the **finite output impedance of the PMOS mirror** (`ro`), which varies with `VDS`, meaning:

```
ΔIPTAT/ΔVDD ≈ 1/ro(M1,M2)  (finite, non-zero)
```

Any change in this current couples through `R1`/`R2` into `VREF`.

### Improving PSRR
- **Cascoded current mirrors** — increases `ro` by (gm·ro) of the cascode device, drastically reducing supply-coupled current variation.
- **Adding a simple pre-regulator / supply-independent bias branch** ahead of the core.
- **Decoupling capacitor** at the VREF output node to shunt high-frequency ripple.
- Low-frequency PSRR is set mainly by mirror output impedance; high-frequency PSRR is dominated by parasitic capacitive feedthrough (gate-drain overlap caps of the mirror devices) — keep these paths symmetric and minimize routing capacitance to VDD.

**Typical benchmarks:** Uncascoded self-biased core: **−40 to −50 dB** at DC/low frequency. With cascoding: **−60 to −70 dB**.

---

## 13. Loop Stability and Compensation

Although there is no dedicated amplifier, the self-biased loop still has **loop gain** (from the positive-feedback current-mirror path) and **parasitic poles** (at the mirror gate nodes, BJT base nodes, and output node), so stability must still be checked:

- **Dominant pole** is usually at the high-impedance mirror gate/diode-connected node.
- A **compensation capacitor** (`Ccomp`) is often added from VREF (or from the core bias node) to ground, or in a Miller configuration across the mirror, to push non-dominant poles safely beyond the loop's unity-gain-like crossover, ensuring the loop does not ring or oscillate — especially important during the transient startup event.
- Verify with a **`.tran` startup transient** (checking for ringing) and, if your LTspice technique allows, a loop-gain **`.ac`** simulation by breaking the loop with a large inductor/capacitor injection pair.

---
