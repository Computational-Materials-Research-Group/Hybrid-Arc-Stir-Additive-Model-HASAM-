# HASAM — Hybrid Arc-Stir Additive Model

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Steel-WAAM%20Deposit-silver?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/3D-Coupled%20Thermal%20%2B%20Mechanical-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Interlayer-Friction%20Stir%20Processing-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Goldak-Double%20Ellipsoid%20Arc%20Source-darkgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A fully 3D <b>element-activation</b> simulation of <b>Wire Arc Additive
  Manufacturing (WAAM) hybridized with interlayer Friction Stir Processing
  (FSP)</b> — the <b>Hybrid Arc-Stir Additive Model (HASAM)</b> — implemented
  in FreeFEM++. A vertical steel wall is built layer-by-layer on a base plate
  using a moving <b>Goldak double-ellipsoid arc source</b>; after each layer
  cools, a slow-moving <b>friction-stir tool pass</b> re-heats and stirs that
  same layer through solid-state frictional/plastic dissipation, refining the
  as-deposited microstructure and relieving part of the accumulated residual
  strain before the next layer is added.
</p>

<img width="1008" height="772" alt="WAAM HYBRID" src="https://github.com/user-attachments/assets/755e893f-cfd3-4360-9f1d-760f67c29b31" />

---

## Concept

HASAM couples two physically distinct heat sources on the same evolving,
progressively-activated mesh:

| Aspect | Description |
|---|---|
| Deposition source | Goldak double-ellipsoid moving arc (WAAM) |
| Stirring source | Schmidt–Hattel sticking-friction moving heat "puck" (FSP) |
| Mesh strategy | Quiet-element activation — all 26 layers pre-meshed, activated progressively as the arc torch passes over them |
| Process sequence | Weld layer → cool → **stir layer (FSP)** → cool → next layer |
| Mechanical proxy | Linear-elastic thermal-strain accumulation, with partial strain relief inside FSP-stirred volume |
| Coupling | One-way thermal → mechanical (no melt-pool flow, no CEL); FSP pass reuses the same weak forms as the arc pass with a different `Tsource` |

This is the FE analogue of the published **UAMFSP** (Unified
Additive-deformation Manufacturing with Friction Stir Processing) strategy
reported for FSP-assisted aluminium WAAM walls — grain refinement and
hardness gains via interlayer stirring — adapted here to a steel wall and
implemented from scratch as a coupled arc + friction-stir thermal model.

---

## Geometry

```
        z
        |  Layer 26  <- last deposited (z = 88 mm)
        |  Layer 25
        |  ...
        |  Layer  2  (z = 13 mm)
 HzBase |  Layer  1  <- first deposited ON base plate (z = 10-13 mm)
        |  BASE PLATE (substrate)
      0 +---------------------------------------------- x
               arc torch AND FSP tool both traverse in +X, per layer

  Base plate (substrate):  200 x 100 x 10 mm   (pre-existing)
  WAAM deposit:            200 x   8 x 80 mm   (26 layers x 3 mm)
  Deposit centred at y = LyBase/2 = 50 mm

  FSP tool:
    Shoulder radius:  Rshoulder = 9 mm
    Pin radius:       Rpin      = 3 mm  (reference only, not separately meshed)
    Pin penetration:  hPin      = 3.6 mm  (1.2 x layer thickness)
    Speed:            900 RPM
```

---

## Physics

Three coupled thermal regimes share one transient heat-transfer solve, plus a
final linear-elastic mechanical stage:

### 1. Arc pass — Goldak double-ellipsoid moving source

```
  ρ Cp ∂T/∂t = div(k ∇T) + Q_arc

  Q_arc(x,y,z,t) = [Q0f * exp(-3(x-xT)²/af²) * frontFrac
                   + Q0r * exp(-3(x-xT)²/ar²) * rearFrac]
                   * exp(-3(y-yT)²/bG²) * exp(-3(z-zT)²/cG²) * ψ_active

  Q0f, Q0r from arc power Pweld = I * U * η_arc,  split front/rear ff/fr = 0.6/1.4
```

### 2. FSP pass — Schmidt–Hattel sticking-friction source (NEW)

```
  Q_shoulder = (2/3) * μ * F_axial * R_shoulder * ω      [W]   (total, sticking condition)

  Tsource_FSP(x,y,z,t) = Q0fsp
        * exp(-0.5((x-xTool)/σxy)²)
        * exp(-0.5((y-yDepCentre)/σxy)²)
        * exp(-0.5((z-zFSPmid)/σz)²)
        * ψ_active

  σxy = Rshoulder/1.7,   σz = hPin/1.7      (Gaussian "puck" under the tool shoulder)
```

FSP is solid-state: the model does not force melting, and prints a warning if
predicted peak temperature exceeds `Tsol − 200 °C` rather than silently
accepting an unphysical result.

### 3. Element activation (quiet-element method, unchanged from arc-only model)

```
  Each layer starts QUIET:  k* = 1e-4 * k_real,  Cp* = 1e-4 * Cp_real
  Activated progressively as the arc torch's x-position passes each node
  FSP heat source only ever acts on already-ACTIVE (already deposited) material
```

### 4. Mechanical stage — thermal-strain accumulation with FSP relief

```
  ε_th += α * ΔT                                    every thermal step (arc, FSP, cooling)
  ε_th *= (1 − strainReliefFactor)   inside stir zone, during the FSP pass

  Elastic solve:  -div(σ) = 0,  σ = λ(T) tr(ε) I + 2μ(T) ε,  driven by ε_th
```

The strain-relief term is an engineering proxy for dynamic recrystallization
and severe plastic deformation homogenizing the microstructure — it is **not**
a plasticity model. See *Key Physics Insights* below for an important caveat
on the resulting stress magnitudes.

---

## Material Properties — Structural Steel

| Property | Symbol | Value | Unit |
|---|---|---|---|
| Density | ρ | 7850 | kg/m³ |
| Solidus temperature | T_sol | 1450 | °C |
| Liquidus temperature | T_liq | 1520 | °C |
| Ambient temperature | T_amb | 25 | °C |
| Thermal conductivity | k(T) | 60 → 27 (piecewise, T-dependent) | W/m/K |
| Specific heat | Cp(T) | 440 → 800 (piecewise, peaks ~5000 near 720 °C) | J/kg/K |
| Convective coefficient (sides) | h_conv | 15 | W/m²/K |
| Convective coefficient (base/anvil) | h_bot | 50 | W/m²/K |
| Thermal expansion proxy | α | 2.1 × 10⁻⁵ | 1/K |
| Poisson's ratio | ν | 0.30 | — |
| Elastic modulus (ambient → near-solidus) | E(T) | 210 → 80 | GPa |

---

## WAAM Arc Process Parameters

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Weld current | I_weld | 200 | A |
| Weld voltage | U_weld | 24 | V |
| Arc efficiency | η_arc | 0.8 | — |
| Arc power | P_weld | 3840 | W |
| Traverse speed | v_scan | 10 | mm/s |
| Time per layer pass | t_pass | 20 | s |
| Inter-layer cooling wait | t_wait | 60 | s |
| Layers | N_layers | 26 | — |
| Layer height | h_bead | 3 | mm |
| Goldak front/rear semi-axes | af, ar | 8 / 20 | mm |
| Goldak width/depth semi-axes | bG, cG | 7 / 4 | mm |
| Goldak front/rear fractions | ff, fr | 0.6 / 1.4 | — |

---

## FSP (Interlayer Friction Stir) Process Parameters

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Apply FSP | applyFSP | true | — |
| Stir every N layers | FSPeveryN | 1 (every layer) | — |
| Tool rotation speed | ω | 900 | RPM |
| FSP traverse speed | v_FSP | 3 | mm/s |
| Tool shoulder radius | R_shoulder | 9 | mm |
| Tool pin radius | R_pin | 3 | mm |
| Pin penetration depth | h_pin | 3.6 (1.2 × h_bead) | mm |
| Axial (plunge/forge) force | F_axial | 8000 | N |
| Effective friction coefficient | μ | 0.40 | — |
| Frictional-work-to-heat efficiency | η_FSP | 0.90 | — |
| Total FSP heat generation | Q_shoulder | ≈ 1629 (steady-state, per layer) | W |
| Post-FSP cooling wait | t_waitFSP | 30 | s |
| Solid-state temperature ceiling (warn only) | T_FSPmax_allow | 1250 (= T_sol − 200) | °C |
| Strain-relief factor in stir zone | — | 0.5 | — |

---

## Mesh

| Component | Nx | Ny | Nz | Notes |
|---|---|---|---|---|
| Base plate (substrate) | 30 | 20 | 8 | 200 × 100 × 10 mm |
| Deposit zone (all 26 layers) | 30 | 6 | 26 | 200 × 8 × 78 mm, 1 element/layer in Z |

`cube()` + linear `movemesh3` for both blocks, combined via `Th = ThBase + ThDep`.

**Boundary labels:**

```
  Label 1 = z = 0    substrate bottom   →  h_bot  = 50 W/m²/K  (anvil)
  Label 2-6           all other faces   →  h_conv = 15 W/m²/K  (free convection)
```

---

## Timestepping

| Phase | dt | Notes |
|---|---|---|
| Arc weld pass | 0.333 s | v_scan·dt ≈ 3.3 mm ≈ 0.5 × mesh spacing |
| Inter-layer cooling | 10 s | after weld, before FSP |
| FSP pass | 0.5 s | v_FSP·dt ≈ 1.5 mm |
| Post-FSP cooling | 10 s | after stir, before next layer |
| Final cooling | 120 s × 20 steps | after last layer |

---

## Process Sequence Per Layer

```
  [Layer iL]
     WELDING       — Goldak arc traverses x = 0 -> Lx, progressive activation
     INTERPASS     — 60 s cooling, no source
     FSP_PASS      — friction-stir tool traverses x = 0 -> Lx over the
                      just-deposited layer, stir zone flagged + strain relieved
     POST_FSP_COOL — 30 s cooling, tool retracted
  [Layer iL+1] ...
```

---

## Output Files — `D:\freefem++\arc_waam_fsp_simulation\`

| File | Description |
|---|---|
| `waam.pvd` | Master animation — open this in ParaView |
| `T_NNNN.vtu` | Per-frame: TempVis (masked temperature), Activation, StirZone |
| `waam_mech.vtu` | Single static file: final displacement + von Mises stress + StirZone |
| `waam_data.csv` | Step-by-step diagnostics |

### VTU Data Fields

| Field | Description | Unit |
|---|---|---|
| `TempC` (`TempVis`) | Temperature, −1 sentinel for not-yet-deposited nodes | °C |
| `Activation` | Quiet-element activation flag, 0 (quiet) → 1 (fully active) | — |
| `StirZone` | 1 where FSP has stirred the material, else 0 | — |
| `Ux`, `Uy`, `Uz` | Displacement components (final mechanical stage only) | m |
| `DispMag` | Displacement magnitude | m |
| `SigMises` | Von Mises stress (final mechanical stage only) | Pa |

### CSV Columns

| Column | Description | Unit |
|---|---|---|
| `time_s` | Physical simulation time | s |
| `layer` | Current layer index | — |
| `phase` | WELDING / INTERPASS / FSP_PASS / POST_FSP_COOL | — |
| `xTool_mm` | Torch or FSP tool x-position | mm |
| `zRef_mm` | Reference z (Goldak mid-plane or FSP pin mid-depth) | mm |
| `Tmax_C` | Maximum temperature in domain | °C |
| `TinterPass_C` | Temperature snapshot at inter-pass / cooling points | °C |
| `epsTh_max` | Maximum accumulated thermal-strain proxy | — |

---

## Repository Structure

```
waam_fsp_hybrid.edp                  # Main FreeFEM++ simulation script
README.md                            # This file

D:\freefem++\arc_waam_fsp_simulation\
├── waam.pvd                         # Master animation (open first)
├── waam_data.csv                    # Step-by-step diagnostics
├── T_0000.vtu                       # Initial state
├── T_0001.vtu                       # After first saved step
├── ...
├── T_NNNN.vtu                       # Final state (1746 frames in reference run)
└── waam_mech.vtu                    # Final distortion + residual stress
```

---

## How to Run

### Requirements
- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org

### Step 1 — Run the simulation

```bash
FreeFem++ waam_fsp_hybrid.edp
```

The script will:
1. Build the substrate + full 26-layer deposit-zone mesh using `cube()` + `movemesh3`
2. Initialise the substrate as fully active, all deposit layers quiet
3. For each of 26 layers:
   - Run the Goldak arc weld pass, activating elements progressively
   - Run inter-layer cooling (60 s)
   - Run the FSP pass (if `applyFSP`), flagging the stir zone and relieving strain there
   - Run post-FSP cooling (30 s)
4. Run final cooling (20 steps × 120 s)
5. Solve the linear-elastic mechanical stage using the accumulated thermal-strain field
6. Write `waam.pvd`, `T_NNNN.vtu`, `waam_mech.vtu`, and `waam_data.csv`

Reference console output (26-layer run, FSP every layer):
```
==========================================
 T-JOINT AS HYBRID WAAM + INTERLAYER FSP (UAMFSP)
 Substrate: 200x100x10mm
 Deposit: 26 layers x 3mm = 78mm total
 ARC:  P=3840W  v=10mm/s
 FSP:  Q_total=1628.6W  omega=900rpm  v=3mm/s  Rshoulder=9mm  applyFSP=1
==========================================
...
  L26 ws=60  x=199.8mm  Tmax=1834.27C
  -> Inter-pass cool: Tmax=758.166C  t=4496.67s
  >> FSP pass over layer 26  (omega=900rpm, v=3mm/s, Q=1628.6W)
  << FSP pass + cooldown complete. Tmax=738.076C  t=4593.33s
 === FINAL COOLING ===
COOL t=6513.33s  Tmax=65.9181C
 === MECHANICAL STAGE ===
==========================================
 DONE. 1746 thermal frames.
 Max distortion: 0.355753 mm
 Max von Mises:  8391.16 MPa   (see caveat below — elastic-only, not capped by yield)
 OPEN: D:\freefem++\arc_waam_fsp_simulation\waam.pvd
==========================================
```

### Step 2 — Open in ParaView

1. `File > Open` → navigate to `D:\freefem++\arc_waam_fsp_simulation\`
2. Select `waam.pvd` → OK → Apply

---

## Visualisation in ParaView

### Option A — Watch the part grow (Threshold on Activation)

```
Filters > Threshold
  Scalars:  Activation
  Range:    0.5 to 1.0
-> Hides un-deposited nodes so the wall visibly grows layer by layer
   instead of sitting there as a static block
```

### Option B — Temperature field

```
Color by:     TempVis (on the thresholded output)
Colormap:     Inferno or Rainbow
Range:        0 to ~1550 °C
Press Play -> hot spot climbs with each arc pass, then a second,
             cooler, broader hot spot sweeps back for the FSP pass
```

### Option C — Stir zone

```
Duplicate the thresholded pipeline item
Color by:     StirZone
-> Binary field showing exactly which volume was friction-stirred,
   layer by layer, as the animation plays
```

### Option D — Final distortion and residual stress

```
Open waam_mech.vtu separately (single static file, not part of the .pvd series)
Color by:        SigMises
Filters > Warp By Vector -> (Ux, Uy, Uz), scale factor ~20-50x
-> Exaggerated view of build distortion, colored by residual stress
```

---

## Key Physics Insights

### Schmidt–Hattel friction heat model
`Q_shoulder = (2/3)·μ·F_axial·R_shoulder·ω` is the standard full-sticking-condition
estimate for shoulder-interface frictional heat generation in FSP, derived from
torque `M = (2π/3)·μ·P·R³` with average contact pressure `P = F/(πR²)`. It is an
engineering estimate, not a contact-mechanics solve — tune `μ` against measured
plunge force/torque if matching a specific tool and alloy.

### Why FSP is modelled as a Gaussian "puck," not a rotating rigid body
Unlike a full CEL/Stokes model of the tool (see the companion U-FSSW project),
HASAM treats FSP purely as a **moving heat source with a stirring-zone strain
proxy** — there is no material flow solve. This keeps the model in the same
element-activation thermal framework as the arc pass, at the cost of not
capturing nugget-scale flow, mixing, or void closure directly.

### Interlayer strain relief and its limits
The `strainReliefFactor = 0.5` reset of `epsTh` inside the stir zone is a proxy
for dynamic recrystallization and severe plastic deformation redistributing
locked-in thermal strain — it is calibrated by inspection, not against
measured residual-stress data. Treat the resulting stress *pattern* (lower in
stirred regions) as more meaningful than the stress *magnitude*.

### ⚠️ Elastic-only mechanical stage overestimates stress magnitude
The reference run's `Max von Mises: 8391 MPa` is **not physical** — real steel
yields plastically in the 250–900 MPa range depending on grade, well before
this. The mechanical stage has no plasticity or stress-relaxation model beyond
the FSP strain-relief proxy, so `epsTh` accumulates elastically over all 26
layers with nothing capping the resulting stress. Use this model to compare
*relative* distortion/stress trends (e.g. arc-only vs HASAM, or stirred vs
unstirred regions) rather than as an absolute residual-stress prediction.
Adding an elastic-perfectly-plastic (or isotropic hardening) return-mapping
capped at the material's yield stress is the natural next extension.

### Quiet-element activation
Because all 26 layers are pre-meshed at `t = 0` with suppressed conductivity
and heat capacity (`k*, Cp* = 1e-4 × real`), FSP's frictional heat source must
be explicitly masked by `ψ_active` — it can only heat material that has
already been "deposited" by the arc pass, never material still waiting to be
built.

---

## HASAM vs Arc-only WAAM — Expected Differences

| Quantity | Arc-only WAAM | HASAM (WAAM + interlayer FSP) | Change |
|---|---|---|---|
| Peak layer temperature | higher, accumulates layer-to-layer | lower, redistributed by stirring | reduced |
| Grain structure | coarse, columnar/dendritic | refined, more equiaxed (not directly meshed here — proxy via StirZone) | qualitative only |
| Residual strain in stirred zone | full accumulation | partially relieved (`strainReliefFactor`) | reduced |
| Von Mises stress in stirred zone | baseline (unbounded elastic) | lower, proportional to strain relief | reduced |
| Build time per layer | t_pass + t_wait | + t_FSPpass + t_waitFSP | longer |
| Process complexity | single heat source | two coupled heat sources, two tool passes | higher |

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `syntax error ... before token _` at a `real` declaration | FreeFEM identifiers with underscores (e.g. `Qfsp_total`) | Rename to camelCase without underscores (e.g. `QfspTotal`) — matches the convention used throughout this script |
| `Tsol does not exist` on the FSP parameter line | FSP block declared before the material constants block | Move `Tsol`, `Tliq`, `Tamb`, `rhoS`, `hConv`, `hBot` above the FSP parameters section |
| `Tmax` stays near ambient during FSP pass | `Tsource` not multiplied by `psiActive`, or tool path outside `[0, LxW]` | Confirm the FSP Gaussian source includes the `* psiActive` factor and the `xTool` bound check |
| Von Mises stress in the thousands of MPa | Expected — elastic-only model, no plasticity cap | See *Key Physics Insights* above; treat as relative, not absolute |
| `waam_mech.vtu` looks identical to arc-only run | `applyFSP = false`, or `strainReliefFactor = 0` | Confirm `applyFSP = true` and `strainReliefFactor > 0` |
| `movemesh3` parser error | `sin/cos/pi` used in `transfo` | Use only linear `scalar*variable` expressions (already the case here) |

---

## Extending the Model

| Extension | What to change |
|---|---|
| Aluminium instead of steel | Replace `kSteel`/`CpSteel` with Al-alloy curves; update `Tsol`, `Tliq`, `rhoS`, and rescale `Faxial`/`ω`/`μ` for the softer, lower-melting alloy |
| Stir only every other layer | Set `FSPeveryN = 2` |
| Post-build FSP instead of interlayer | Move the FSP block outside the layer loop, run once after all 26 layers with a deeper `hPin` |
| Plasticity-capped stress | Add elastic-perfectly-plastic return-mapping to the mechanical stage, capped at material yield stress |
| Full CEL/Stokes stirring flow | See the companion U-FSSW-CEL-3D-Simulation project for a coupled Stokes + VOF treatment of the tool-material interaction |
| Calibrate strain relief | Replace the fixed `strainReliefFactor = 0.5` with a function of local peak temperature / stir-zone residence time |
| Multi-pass FSP (double stir) | Loop the FSP pass block `k` times before the post-FSP cooling wait |

---

## Citation

```
Mishra, A. (2026). Hybrid Arc-Stir Additive Model (HASAM) [Computer software]. Zenodo. https://doi.org/10.5281/zenodo.21431876
```

```bibtex
@software{mishra2026hasam,
  author       = {Mishra, Akshansh},
  title        = {Hybrid Arc-Stir Additive Model (HASAM)},
  year         = {2026},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.21431876},
  url          = {https://doi.org/10.5281/zenodo.21431876}
}
```

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0"
  src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:
- **Share** — copy and redistribute in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:
- **Attribution** — give appropriate credit and link to this repository
- **NonCommercial** — not for commercial use without permission

Copyright 2026. All rights reserved for commercial use.
