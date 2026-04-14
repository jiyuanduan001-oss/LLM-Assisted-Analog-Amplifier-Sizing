# 5T OTA Design Flow

## Purpose

Step-by-step sizing procedure for the 5-transistor single-stage OTA.
Invoked after circuit-understanding identifies the topology as 5T OTA.

## References

- Equations: `5t-ota-equation.md`
- Root-cause diagnosis: `5t-ota-root-cause-diagnosis.md`

## Rules

1. Execute steps in order. Do not skip.
2. All computations in Python. No mental arithmetic.
3. After simulation failure, use `5t-ota-root-cause-diagnosis.md`. Do not improvise fixes.

---

## Bias Current Relationships

```
I_tail = ID3 = (M3_fingers / M4_fingers) × I_bias
ID1 = ID2 = I_tail / 2
ID5 = ID6 = ID1
P = (I_tail + I_bias) × VDD
```

---

## Sizing Procedure

### Step 1 — Initial sizing: DIFF_PAIR (M1, M2)

Goal: determine gm, gm/ID, L for the input pair, then derive all
device parameters from LUT.

**1a. Determine gm from GBW spec:**
```
gm1 = 2π × GBW × CL
```

**1b. Choose gm/ID (empirical, based on bandwidth):**

| GBW range   | Recommended gm/ID | Inversion      | Comment                    |
|-------------|-------------------|----------------|----------------------------|
| < 10 MHz    | 14–30 S/A         | Moderate–weak  | Lower power design         |
| 10–100 MHz  | 10–14 S/A         | Moderate       | Balanced across all aspects |
| > 100 MHz   | 5–10 S/A          | Strong         | High speed                 |

**1c. Determine L from gain requirement:**

Sweep available L values in the LUT. For each L, query:
```
gm_gds_M1 = lut_query('nfet', 'gm_gds', L, gm_id_val=(gm/ID)_1)
```
Pick the shortest L where `gm_gds_M1 / 2 ≥ A0_target` (linear, not dB).

If no L satisfies this:
→ Print: "INFEASIBLE: 5T OTA cannot achieve required gain."
→ Ask user to relax gain or switch topology. Do NOT proceed.

**1d. Derive all DIFF_PAIR parameters from LUT:**

With (gm1, L1, (gm/ID)_1) now fixed, derive all parameters from LUT:
```
ID1   = gm1 / (gm/ID)_1
I_tail = 2 × ID1
id_w1 = lut_query('nfet', 'id_w',  L1, gm_id_val=(gm/ID)_1)   # A/m
W1    = ID1 / id_w1                                              # m (meters — display as W1*1e6 for µm)
gds1  = gm1 / lut_query('nfet', 'gm_gds', L1, gm_id_val=(gm/ID)_1)
ft1   = lut_query('nfet', 'ft',    L1, gm_id_val=(gm/ID)_1)    # Hz
Cgs1  = lut_query('nfet', 'cgs_w', L1, gm_id_val=(gm/ID)_1) × W1  # F (no 1e-6, W in m)
Cgd1  = lut_query('nfet', 'cgd_w', L1, gm_id_val=(gm/ID)_1) × W1  # F (no 1e-6, W in m)
Cdb1  = lut_query('nfet', 'cdb_w', L1, gm_id_val=(gm/ID)_1) × W1  # F (drain-bulk junction)
vdsat1  = lut_query('nfet', 'vdsat', L1, gm_id_val=(gm/ID)_1)    # V (BSIM4 |Vds|_sat)
```

### Step 2 — Initial sizing: LOAD (M5, M6)

ID5 = ID1 (already known from Step 1).

**2a. Choose gm/ID for LOAD:**

Use 10–14 S/A (moderate inversion).

**2b. Determine L from gain requirement:**

Step 1c used the approximation `A0 ≈ (gm/gds)_M1 / 2`. Now with the LOAD,
compute the actual gain `A0 = gm1 / (gds1 + gds_eq_LOAD)`.

The LOAD sub-block type was determined during circuit-understanding
(Step 2b there). Compute `gds_eq_LOAD` accordingly — see
`general/knowledge/mirror-load-structures.md`:

- **single**: `gds_eq_LOAD = gds5`
- **cascode / lv_cascode**: `gds_eq_LOAD = (gds5 × gds_cas) / gm_cas`
  (the cascode device is sized in Step 2d below; for this first pass
  use a placeholder `gds_eq_LOAD = gds5 / gm_gds_cas_est` where
  `gm_gds_cas_est ≈ 20` is a reasonable first guess at the cascode's
  intrinsic gain. Refine after Step 2d.)

Sweep L5 candidates to find minimum L5 satisfying the gain:
```
For each L5:
  gds5 = (gm/ID)_5 × ID5 / lut_query('pfet', 'gm_gds', L5, gm_id_val=(gm/ID)_5)
  # If sub_block_type == "single":
  gds_eq = gds5
  # If cascode/lv_cascode:
  gds_eq = gds5 / 20   # placeholder; refine in 2d
  A0 = gm1 / (gds1 + gds_eq)
  If A0 ≥ A0_target: select this L5, BREAK
```

If no L5 satisfies the gain: increase L1 (from Step 1c) and re-derive.

**2c. Derive all LOAD (main) parameters from LUT:**

With (ID5, L5, (gm/ID)_5) now fixed:
```
gm5   = (gm/ID)_5 × ID5
id_w5 = lut_query('pfet', 'id_w',  L5, gm_id_val=(gm/ID)_5)   # A/m
W5    = ID5 / id_w5                                              # m (meters)
gds5  = gm5 / lut_query('pfet', 'gm_gds', L5, gm_id_val=(gm/ID)_5)
ft5   = lut_query('pfet', 'ft',    L5, gm_id_val=(gm/ID)_5)    # Hz
Cgs5  = lut_query('pfet', 'cgs_w', L5, gm_id_val=(gm/ID)_5) × W5  # F (no 1e-6, W in m)
Cgd5  = lut_query('pfet', 'cgd_w', L5, gm_id_val=(gm/ID)_5) × W5  # F (no 1e-6, W in m)
Cdb5  = lut_query('pfet', 'cdb_w', L5, gm_id_val=(gm/ID)_5) × W5  # F (drain-bulk junction)
vdsat5  = lut_query('pfet', 'vdsat', L5, gm_id_val=(gm/ID)_5)    # V (BSIM4 |Vds|_sat)
```

**2d. If LOAD sub_block_type != "single" — Size LOAD_CAS:**

Skip this step for single-transistor loads. For cascode and lv_cascode:

The cascode device carries the same current as M5 (in series), so
`ID_cas = ID5`. Free parameters: `gm/ID_cas` and `L_cas`.

**Parameter choices:**
- `(gm/ID)_cas = 10` S/A (strong-to-moderate — high gm_cas for fast p_int)
- `L_cas = L_min` (shortest available — keeps C_int small)

**Derive all LOAD_CAS parameters from LUT:**
```
ID_cas  = ID5
gm_cas  = (gm/ID)_cas × ID_cas
id_w_c  = lut_query('pfet', 'id_w',  L_cas, gm_id_val=(gm/ID)_cas)   # A/m
W_cas   = ID_cas / id_w_c                                             # m (meters)
gds_cas = gm_cas / lut_query('pfet', 'gm_gds', L_cas, gm_id_val=(gm/ID)_cas)
Cgs_c   = lut_query('pfet', 'cgs_w', L_cas, gm_id_val=(gm/ID)_cas) × W_cas
Cgd_c   = lut_query('pfet', 'cgd_w', L_cas, gm_id_val=(gm/ID)_cas) × W_cas
Cdb_c   = lut_query('pfet', 'cdb_w', L_cas, gm_id_val=(gm/ID)_cas) × W_cas  # F (real cdb from LUT)
vdsat_cas = lut_query('pfet', 'vdsat', L_cas, gm_id_val=(gm/ID)_cas)   # V (BSIM4 |Vds|_sat)
vth_cas   = abs(lut_query('pfet', 'vth', L_cas, gm_id_val=(gm/ID)_cas)) # from LUT
```

**Compute sub-block effective quantities:**
```
gds_eq_LOAD = (gds5 × gds_cas) / gm_cas
C_eq_LOAD   = Cgd_c + Cdb_c                        # at vout
C_int_LOAD  = Cgs_c + Cdb5 + Cgd5                  # at internal node (Csb_c≈0 if shorted to bulk)
p_int_LOAD  = gm_cas / C_int_LOAD                  # rad/s
```

**Verify internal pole vs GBW**:
```
ω_c = 2π × GBW_target
If p_int_LOAD < 3 × ω_c:
  → Reduce L_cas (already at L_min? — then increase (gm/ID)_cas)
  → Or re-derive with stronger inversion (gm/ID = 8)
```

**For lv_cascode — compute required external bias:**
```
# PMOS LV cascode (rail = VDD):
Vbias_cas_p = VDD - (vdsat5 + vdsat_cas + |vth_cas|)

# NMOS LV cascode (rail = VSS):
Vbias_cas_n = vdsat_main + vdsat_cas + vth_cas
```
Record this value — it will be passed to the testbench as the value of
the `Vbias_cas_p` (or `_n`) port at simulation time.

**2e. Refine the first-stage gain with actual cascode values:**

Recompute with the exact LOAD sub-block:
```
A0 = gm1 / (gds1 + gds_eq_LOAD)
```
If `A0 < A0_target`: the design needs more gain. Options:
- Increase L5 (main device longer → smaller gds5 → smaller gds_eq_LOAD)
- Increase L_cas (longer cascode → larger gm_gds_cas → smaller gds_eq_LOAD)
  but verify `p_int_LOAD > 3 × ω_c` still holds.

### Step 3 — Initial sizing: TAIL (M3) and BIAS_REF (M4)

ID3 = I_tail (already known from Step 1).

**3a. Determine multiplier ratio first:**

M4 (BIAS_REF) is the unit cell with M4_M = 1. M3 (TAIL) uses multiple
parallel fingers to set the current ratio:

```
M4_M = 1
M3_M = round(I_tail / I_bias)
```

**3b. Choose gm/ID and L:**

Use (gm/ID)_3 = 10–14 S/A. Initial L3 = 1.0 µm (snap to the nearest value in
`list_available_L('nfet', corner, temp)`).

**3c. Derive single-finger parameters from LUT:**

LUT describes a single transistor. Use the per-finger current
(I_bias = I_tail / M3_M) to derive the unit-cell parameters:

```
ID_finger = I_bias                                                # A per finger
gm_finger = (gm/ID)_3 × ID_finger
id_w3     = lut_query('nfet', 'id_w',  L3, gm_id_val=(gm/ID)_3) # A/m
W3        = ID_finger / id_w3                                     # m (per finger, meters)
gds_finger = gm_finger / lut_query('nfet', 'gm_gds', L3, gm_id_val=(gm/ID)_3)
ft3       = lut_query('nfet', 'ft',    L3, gm_id_val=(gm/ID)_3)  # Hz (same per finger)
Cgs_finger = lut_query('nfet', 'cgs_w', L3, gm_id_val=(gm/ID)_3) × W3  # F (no 1e-6, W in m)
Cgd_finger = lut_query('nfet', 'cgd_w', L3, gm_id_val=(gm/ID)_3) × W3  # F (no 1e-6, W in m)
Cdb_finger = lut_query('nfet', 'cdb_w', L3, gm_id_val=(gm/ID)_3) × W3  # F (drain-bulk)
vdsat3    = lut_query('nfet', 'vdsat', L3, gm_id_val=(gm/ID)_3)  # V (same per finger)
```

**3d. Scale to total M3 device:**

```
gm3  = gm_finger × M3_M
gds3 = gds_finger × M3_M
Cgs3 = Cgs_finger × M3_M
Cgd3 = Cgd_finger × M3_M
Cdb3 = Cdb_finger × M3_M
```

**3e. BIAS_REF (M4):**

M4 shares the same W and L as M3 (single finger = unit cell):
```
L4 = L3
W4 = W3
```

**3f. If TAIL sub_block_type != "single" — Size TAIL_CAS:**

Skip this step for single tail. For cascode / lv_cascode tail
(role `TAIL_CAS`, detected in circuit-understanding Step 2b):

The cascode device carries the full tail current (in series with M3),
so `ID_tcas = I_tail`. Parameter choices:
- `(gm/ID)_tcas = 10` S/A (strong-to-moderate — high gm_tcas)
- `L_tcas = L_min` (shortest available)

**Derive all TAIL_CAS parameters from LUT:**
```
ID_tcas  = I_tail
gm_tcas  = (gm/ID)_tcas × ID_tcas
id_w_tc  = lut_query('nfet', 'id_w',  L_tcas, gm_id_val=(gm/ID)_tcas)
W_tcas   = ID_tcas / id_w_tc                                        # m
gds_tcas = gm_tcas / lut_query('nfet', 'gm_gds', L_tcas, gm_id_val=(gm/ID)_tcas)
Cgs_tcas = lut_query('nfet', 'cgs_w', L_tcas, gm_id_val=(gm/ID)_tcas) × W_tcas
Cgd_tcas = lut_query('nfet', 'cgd_w', L_tcas, gm_id_val=(gm/ID)_tcas) × W_tcas
Cdb_tcas = lut_query('nfet', 'cdb_w', L_tcas, gm_id_val=(gm/ID)_tcas) × W_tcas
vdsat_tcas = lut_query('nfet', 'vdsat', L_tcas, gm_id_val=(gm/ID)_tcas)
vth_tcas   = abs(lut_query('nfet', 'vth',  L_tcas, gm_id_val=(gm/ID)_tcas))
```

**Compute sub-block effective quantities:**
```
gds_eq_TAIL    = (gds3 × gds_tcas) / gm_tcas
V_headroom_TAIL = vdsat3 + vdsat_tcas
```

**For lv_cascode — compute required external bias (NMOS, rail = VSS):**
```
Vbias_cas_n = vdsat3 + vdsat_tcas + vth_tcas
```
Record this value — it will be passed to the testbench as the value of
the `Vbias_cas_n` port at simulation time (emitted as an `extra_ports`
entry when the topology is registered).

### Step 4 — Analytical spec evaluation

All devices are now sized with LUT data. Compute every spec using the
full equations from `5t-ota-equation.md`. **All calculations MUST be
done using Python** — do not compute mentally.

Note: since M1≡M2, `gm2=gm1, gds2=gds1, Cgd2=Cgd1, Cdb2=Cdb1`.
Since M5≡M6, `gm6=gm5, gds6=gds5, Cgs6=Cgs5, Cdb6=Cdb5`.
I_bias is from the spec form.

First, compute the LOAD and TAIL sub-block effective quantities based on
their `sub_block_type` values (detected during circuit-understanding;
see `general/knowledge/mirror-load-structures.md`):
```
# LOAD sub-block:
#   single:     gds_eq_LOAD = gds5
#               C_eq_LOAD   = Cgd5 + Cdb5
#               C_mirror    = Cgs5 + Cgs6 + Cdb6 + Cgd2 + Cdb2
#               p_int_LOAD  = None
#   cascode / lv_cascode:
#               gds_eq_LOAD = (gds5 × gds_cas) / gm_cas
#               C_eq_LOAD   = Cgd_cas + Cdb_cas
#               C_mirror    = Cgs5 + Cgs6 + Cgd_cas + Cdb_cas + Cgd2 + Cdb2
#               C_int_LOAD  = Cgs_cas + Cdb5 + Cgd5
#               p_int_LOAD  = gm_cas / C_int_LOAD

# TAIL sub-block:
#   single:     gds_eq_TAIL     = gds3
#               V_headroom_TAIL = vdsat3
#   cascode / lv_cascode:
#               gds_eq_TAIL     = (gds3 × gds_tcas) / gm_tcas
#               V_headroom_TAIL = vdsat3 + vdsat_tcas
```

Then compute all specs:
```
A0    = gm1 / (gds1 + gds_eq_LOAD)
GBW   = gm1 / (2π × (CL + Cgd1 + Cdb1 + C_eq_LOAD))
fp2   = gm5 / (2π × C_mirror)                               # mirror pole (see LOAD table)
fz2   = 2 × fp2
PM    = 90° - arctan(GBW/fp2) + arctan(GBW/fz2)
# Add cascode internal pole penalty if present:
if p_int_LOAD is not None:
    PM -= arctan(2π·GBW / p_int_LOAD)
SR    = I_tail / CL
# Swing uses V_headroom_LOAD (see mirror-load-structures.md):
#   single:      V_headroom_LOAD = vdsat_M5
#   cascode:     V_headroom_LOAD = vdsat_M5 + |Vgs_cas|
#   lv_cascode:  V_headroom_LOAD = vdsat_main + vdsat_cas
Swing = VDD - vdsat1 - V_headroom_TAIL - V_headroom_LOAD
Rout  = 1/(gds1 + gds_eq_LOAD)
ro_eq_TAIL = 1/gds_eq_TAIL
CMRR  = 2·gm1·gm5·Rout·ro_eq_TAIL
PSRR⁺ ≈ A0
PSRR⁻ ≈ CMRR
V_cm_min = V_headroom_TAIL + Vth_n + vdsat1
P     = (I_tail + I_bias) × VDD
```

Print the results and compare against user spec targets:

```
ANALYTICAL SPEC CHECK
======================
Spec          | Analytical | Target      | Status
A0            | <> dB      | <> dB       | ✅/❌
GBW           | <> MHz     | <> MHz      | ✅/❌
PM            | <>°        | <>°         | ✅/❌
...
[all active spec targets from spec form]
```

**Decision:**
- All specs met → proceed to Step 5 (simulation).
- Any spec failed → invoke `5t-ota-root-cause-diagnosis.md` to identify
  which device parameter to adjust. Apply the fix, re-derive LUT values
  for the affected role, and repeat Step 4.
- After 5 analytical iterations, proceed to Step 5 regardless.

### Step 5 — Submit to simulation

Call `convert_sizing` and `simulate_circuit`:

**⚠️ `id_derived` MUST be the TOTAL current the role carries, not per-finger.**
Step 3 works with per-finger parameters for LUT queries, but `convert_sizing`
needs the total current to compute mirror ratios.  Specifically:
- `TAIL.id_derived = I_tail` (total tail current, e.g. 40 µA), **NOT** `I_bias`
- `BIAS_REF.id_derived = I_bias` (reference current, e.g. 5 µA)
The bridge computes `M3_M = round(I_tail / I_bias)` from the ratio.
Passing `I_bias` for both produces `M3_M = 1` → 5× current deficit.

```python
from tools import convert_sizing, simulate_circuit

result = convert_sizing(
    topology='5t_ota',
    roles_raw={
        "DIFF_PAIR": {"gm_id_target": (gm/ID)_1, "L_guidance_um": L1, "id_derived": ID1},
        "LOAD":      {"gm_id_target": (gm/ID)_5, "L_guidance_um": L5, "id_derived": ID5},
        "TAIL":      {"gm_id_target": (gm/ID)_3, "L_guidance_um": L3, "id_derived": ID3},  # ID3 = I_tail, NOT I_bias
        "BIAS_REF":  {"gm_id_target": 0,          "L_guidance_um": L3, "id_derived": I_bias},
        # Cascode companions (only if sub_block_type != "single" for the parent role):
        # "LOAD_CAS":  {"gm_id_target": (gm/ID)_cas, "L_guidance_um": L_cas, "id_derived": ID5},
        # "TAIL_CAS":  {"gm_id_target": (gm/ID)_tcas, "L_guidance_um": L_tcas, "id_derived": I_tail},
    },
    Ib_a=I_bias,
    l_overrides={"DIFF_PAIR": L1, "LOAD": L5, "TAIL": L3, "BIAS_REF": L3},
)

sim = simulate_circuit(
    result["params"],
    config_path=result["config_path"],
    corner=corner,                       # from validated spec form
    temperature=temperature,             # from validated spec form
    supply_voltage=VDD,                  # from validated spec form
    CL=CL,                              # from validated spec form (Farads)
    # Mismatch is slow (~35 s of Monte Carlo). Honor the spec form:
    #   user's Mismatch field is BLANK → measure_mismatch=False
    #   user provided a numeric Mismatch target → measure_mismatch=True
    measure_mismatch=mismatch_enabled,   # bool from Stage [1] spec form
    # LV-cascode bias overrides — include every time LOAD or TAIL is
    # lv_cascode, recomputed from the CURRENT iteration's sized vdsat/vth:
    # extra_ports={
    #     "Vbias_cas_p": VDD - (vdsat5 + vdsat_cas + abs(vth_cas)),       # PMOS load
    #     "Vbias_cas_n": vdsat3 + vdsat_tcas + abs(vth_tcas),             # NMOS tail
    # },
)
```

**IMPORTANT:** `corner`, `temperature`, `supply_voltage` (VDD), and `CL` MUST
come from the validated spec form (Stage [1]). These are the same values used
for LUT queries and analytical sizing. Omitting them causes the simulator to
fall back to TOML defaults (typically tt/27°C/1.8V/5pF), creating a mismatch
between the LUT-based sizing and the SPICE verification. The `CL` parameter
accepts **Farads** (SI); the bridge converts to picoFarads internally for
CircuitCollector.

**Vbias_cas_p / Vbias_cas_n update rule:** these values depend on the sized
`vdsat` and `vth` of the cascode stack. Every sizing iteration must
recompute them from the LUT-derived `vdsatX` / `vth_cas` of the current
sizing and pass them as `extra_ports={...}` to `simulate_circuit`. Do NOT
rely on the TOML defaults written at `ensure_topology_registered` time —
those are only initial seed values; the live value comes from this call.

→ Proceed to `general/flow/simulation-verification.md` with the results.

