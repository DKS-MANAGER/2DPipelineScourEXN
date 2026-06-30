# 2DPipelineScourEXN — sedExnerFoam

Comparative CFD case for 2D submarine pipeline scour using **sedExnerFoam**  
(single-phase RANS + moving-mesh Exner morphodynamics).

---

## Solver Comparison

| Property | Baseline `sedFoam` | This case `sedExnerFoam` |
|---|---|---|
| Repo | [2DPipelineScour](https://github.com/DKS-MANAGER/2DPipelineScour) | [2DPipelineScourEXN](https://github.com/DKS-MANAGER/2DPipelineScourEXN) |
| Solver | `sedFoam_rbgh` Euler-Euler two-phase | `sedExnerFoam` single-phase + Exner |
| Bed treatment | Volume fraction `alpha.a` in fluid | Sharp moving boundary |
| Turbulence | k-ω SST (phase b) | k-ω SST (identical) |
| End time | 5 s | 45 s |
| Write interval | 0.1 s | 0.5 s |
| Morphological accel. | — | morphFac = 10 |

---

## Domain (Extracted from Baseline)

```
y=0.205 ┌─────────────────────────────────────────┐ surface (symmetry)
        │         upper fluid  578×20 cells        │
y=0.08  ├─────────────────────────────────────────┤
        │    ○ cylinder       main flow 578×35     │
y=0.0   ├─────────────────────────────────────────┤ ← initial bed / faMesh
y=-0.025│         sub-bed     578×25 cells         │
y=-0.1  └─────────────────────────────────────────┘ bottom (wall)
      x=-0.75                                   x=1.0
```

- Cylinder: D = 0.05 m, centre (0, 0.0255, 0.0005), gap **e/D = 0.02**
- 2D extrusion: z = [0, 0.001], 1 cell (empty patches: lateralfront/back)
- Total cells: 578 × 80 ≈ 46,240 background + snappy refinement

---

## Physical Parameters (All Extracted from Baseline)

| Parameter | Value | Source |
|---|---|---|
| ρ_fluid | 1000 kg/m³ | `constant/transportProperties` phaseb |
| ν_fluid | 1×10⁻⁶ m²/s | `constant/transportProperties` phaseb |
| ρ_sediment | 2650 kg/m³ | `constant/transportProperties` phasea |
| d₅₀ | 0.36 mm | `constant/transportProperties` phasea |
| s = ρs/ρf | 2.65 | derived |
| u* | 0.04318 m/s | `0_org/U.b` codedFixedValue |
| κ (von Kármán) | 0.41 | `0_org/U.b` codedFixedValue |
| z₀ (roughness) | 3.0×10⁻⁵ m | `0_org/U.b` codedFixedValue |
| U₀ (depth-avg) | ~0.35 m/s | derived from log-law |
| θ_cr (Soulsby, D*=9.15) | 0.0473 | derived |
| Porosity n | 0.4 | standard for medium sand |

---

## Morphodynamics

**Exner equation** (solved on faMesh `bottom` patch):

```
(1 - n) ∂z/∂t + ∂qs/∂x = 0   →   dz/dt = -1/(1-n) · ∂qs/∂x
```

**Nielsen (1992) bedload**:
```
θ   = τ_b / [ρ(s-1)g d₅₀]
qb* = 12 √θ · (θ - θ_cr)        [θ > θ_cr]
qb  = qb* · √[(s-1)g d₅₀³]
    = qb* · 1.48×10⁻⁵ m²/s
```

**Slope correction (Bagnold / van Rijn 1984)**:
```
qb,corr = qb · (cos β - sin β / tan φ_r)
φ_r = 32°  (angle of repose)
```

**Moving mesh**: `displacementLaplacian` with `inverseDist` diffusivity  
→ deformation concentrated near `bottom`, upper mesh nearly rigid

---

## How to Run

```bash
# Option A: generate fresh mesh
bash Allrun

# Option B: reuse baseline polyMesh (recommended for exact comparison)
cp -r /path/to/2DPipelineScour/constant/polyMesh constant/polyMesh
makeFaMesh
cp -r 0_org/. 0/
sedExnerFoam 2>&1 | tee log.sedExnerFoam
```

---

## Post-Processing: Scour Depth Comparison

```bash
# Extract minimum y-coordinate of 'bottom' patch at each timestep
postProcess -func 'patchExpression(bottom, Cf.y())' -latestTime
```

Compare against sedFoam scour depth extracted from `alpha.a` isosurface at 0.5.
