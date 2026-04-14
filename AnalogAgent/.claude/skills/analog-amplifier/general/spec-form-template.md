# Design Spec Form

## Required (sizing will not proceed without these)
VDD          : 1.8         # Supply voltage (V)
CL           : 5e-12       # Load capacitance (F)
Gain         : 70          # DC gain target (dB)
GBW          : 50e6        # Gain-bandwidth product (Hz)
PM           : 60          # Phase margin (degrees)

## Environment (recommended — defaults applied if blank)
Temperature  : 20          # °C  (default: 27)
Corner       : tt          # tt, ff, ss, fs, sf  (default: tt)

## Optional (leave blank to skip — will not be optimized)
Power        : 500e-6      # Max power (W)
SR+          : 10           # Positive slew rate (V/µs)
SR-          :             # Negative slew rate (V/µs)
CMRR         : 60          # (dB)
PSRR+        : 60          # Positive PSRR (dB)
PSRR-        : 60          # Negative PSRR (dB)
IRN          : 30e-6       # Integrated input-referred noise (V rms)
ORN          :             # Integrated output-referred noise (V rms)
Output_swing :             # (V)
I_bias       : 10e-6       # External bias current (A)

## Mismatch (leave blank to skip — saves significant runtime)
#
# Mismatch simulation uses Monte Carlo (50 runs) and is much slower than
# other specs. When blank, mismatch is completely skipped: no Monte Carlo
# simulation is run, no mismatch data is reported, and mismatch is excluded
# from the iteration loop and optimization constraints.
#
# When a number is provided, mismatch becomes an active design target.
# The sizing flow will run Monte Carlo each iteration, check the 3σ offset
# against the target, and include it in root-cause diagnosis if it fails.
# Diagnosis focuses on two fixes: (1) increase W×L (transistor area),
# (2) reduce |Vdsat| (push toward weaker inversion).
#
Mismatch     :        # 3σ mismatch offset (V) — Monte Carlo, 50 runs
                            #   Leave BLANK to skip mismatch entirely

## Post-Sizing Options
Extreme_PVT  : yes          # yes/no — run additional sims at extreme corners
                            #   after sizing converges (SS/85°C + FF/−40°C)
Optimize     : yes          # yes/no — run numerical optimization after sizing
                            #   converges. After the LLM sizing stage, the system
                            #   will ask which metric to prioritize: Power, Gain,
                            #   or GBW. The selected metric receives a higher
                            #   weight; the other two are still improved but with
                            #   lower priority. All other specs are kept above
                            #   user targets as constraints.
