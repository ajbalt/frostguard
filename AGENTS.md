# FROSTGUARD

NASA ORBIT Challenge 2027, ORBIT Space track, Team HD 137010 b (Cal Poly Pomona, RSCL).
Survival-value power and task control for energy-constrained lunar surface science nodes.
Addresses Civil Space Shortfall 1618: survive and operate through the lunar night.

Phase 2 deliverable: a bench-scale hardware-in-the-loop (HIL) prototype on two RP2350 microcontrollers, compared against three baseline policies across at least 100 seeded runs per scenario, with open data and a controller design at TRL 3-4. No flight qualification is claimed.

Team: Ari Akram (lead: mission-controller firmware, scheduler, test automation, statistics), Ari Baltazar Moreno (co-lead: power board, safety controller, thermal fixture, trace replay). Faculty advisor: Prof. Mohamed El-Hadedy.

## Hypotheses (what the code must be able to test)

- **H1 (value)**: FROSTGUARD returns >= 25% more validated, priority-weighted observations per joule than fixed-priority scheduling (policy A), with zero reserve or thermal-floor violations.
- **H2 (switching cost)**: removing measured switching costs (E_sw, t_sw) from the scheduler (ablation) causes a statistically significant loss of value per joule or reserve margin on traces with frequent illumination changes.
- **H3 (independence)**: with the mission controller crashed, hung or requesting adversarial loads, the safety controller holds E >= E_res and T >= T_min in 100% of fault-injection trials.

A stop rule exists: if policy D (FROSTGUARD) does not beat policy C after scheduler overhead, the claim is revised, not hidden.

## Architecture

Two separate RP2350 boards on separate supply rails:

- **Mission controller (MC)**: Hazard3 RISC-V cores. Holds task contracts (v_i, e_i, d_i) and the value-per-joule scheduler. Can only *request* loads.
- **Safety controller (SC)**: Arm Cortex-M33 cores. Sole owner of the load gates L0 (survival heater), L1 (science sensor), L2 (compute/FPGA), L3 (radio). Grants or vetoes MC requests, enforces E >= E_res(t) and T >= T_min, runs a watchdog on the MC heartbeat.
- **Telemetry**: INA228 (V, I, E per domain), TMP117 and NTC (temperatures). The MC receives a read-only copy.
- **Modes**: NOMINAL, CONSERVE, SURVIVE, SAFE-HOLD with hysteresis m2 > m1 > m0 >= 0. The SC executes the SURVIVE and SAFE-HOLD transitions on its own.
- **Policies compared**: A fixed priority, B battery-threshold shutdown, C energy-aware scheduling without switching cost or independent reserve, D FROSTGUARD. Ablations remove E_sw (H2) or merge SC duties into the MC.

## Hard rules (never violate)

1. **The SC is the only code that drives the L0-L3 gates.** Nothing under `firmware/mc/` may touch gate pins, directly or indirectly. The MC sends requests over the protocol; the SC decides.
2. **No code shared between `mc/` and `sc/` except `firmware/common/`**, and `common/` contains only protocol definitions, types and CRC. No scheduling or safety logic in `common/`. This preserves the toolchain and ISA diversity argument (RISC-V vs Arm).
3. **Task values and constraints are frozen before trials.** `firmware/mc/config/task_weights.yaml` is changed only by an explicit, committed decision recorded in `docs/decisions/`. Never tune weights to results. Sensitivity to +/-50% weights is reported separately.
4. **Every experimental run records**: the firmware git hash (both boards), the random seed, the scenario and policy, and the trace file identifier. Runs without this metadata are invalid and must not enter `data/runs/`.
5. **Never overwrite or hand-edit files in `data/runs/`.** Raw run data is append-only. Re-analysis goes in `analysis/`.
6. **No powered testing of the power board, battery or thermal fixture** until the safety checklist in `hardware/safety/` is complete and has been reviewed by the faculty advisor. Do not write scripts that energize hardware without the E-stop and current limits documented there.
7. **Bench battery stays at >= 0 C.** The Peltier replays environment temperature scaled to bench range; the battery is outside the cold zone.
8. **Lunar and terrestrial profiles are reported separately.**
9. **No unverified claims in the paper or docs.** Mark anything unsupported as needing verification. Cite sources in `docs/references/`.

## Repository layout

```
firmware/
  common/       protocol (MC<->SC messages, heartbeat), types (task contract, modes), crc. No logic.
  mc/           mission controller: scheduler, tasks, switchcost, config. RISC-V target.
  sc/           safety controller: modes, reserve, gates, watchdog. Arm Cortex-M33 target.
  drivers/      INA228, TMP117, NTC ADC, microSD/UART logger
tests/
  host/         native unit tests of scheduler, reserve, state machine (no hardware)
  hil/          pytest tests driving the real bench, including fault injection
hardware/
  kicad/        power board schematic and PCB
  bom/          bill of materials
  thermal-fixture/  Al thermal mass, Peltier, heater, calibration notes
  safety/       fusing, E-stop, current limits, pre-power review checklist
bench/          host-side automation (Python): instruments, replay, faults, calibration, runner
traces/         lunar (LOLA, Diviner), terrestrial (NSRDB); tools/ regenerates them from public sources
data/runs/      one folder per gate/scenario/policy: CSV + metadata.json (open-data release)
analysis/       notebooks, stats (95% CIs, Wilcoxon, weight sensitivity), figures
docs/           proposal, gates (G1-G4), references, decisions (short ADRs), customer-discovery
scripts/        build, flash, test helpers
```

Empty folders hold a `.gitkeep`; delete it when the folder gets real content.

## Schedule and gates

- **G1** (~Jan 2027): safe power board, calibrated sensing, baseline switching.
- **G2** (~Mar): task contracts, SC reserve enforcement, full E_sw table.
- **G3** (~May): four-policy and ablation results with fault injection.
- **G4** (~Jun): terrestrial-profile trials, discovery interviews, BOM.

Dates after Phase 1 follow the official ORBIT schedule and will be adjusted. Plans and results per gate go in `docs/gates/`.

## Scheduling model (reference)

Over a receding horizon H, choose x_i(t) in {0,1} to maximize sum(v_i x_i(t)) subject to:

- E(t+1) = E(t) + eta*G_hat(t)*dt - P_base*dt - sum(e_i x_i) - sum(E_sw) - P_heat(t)*dt
- E(t) >= E_res(t) and T(t) >= T_min
- task deadlines met, or the task is dropped

G_hat is a lower-confidence generation forecast, so uncertainty consumes margin rather than reserve. E_res(t) is the energy the heater and SC need to hold T >= T_min until the end of the current shadow, from the lumped thermal model C dT/dt = P_heat + P_diss - (T - T_env)/R, with R and C calibrated on the bench. Expected task count <= 16, so exact enumeration or greedy value-per-joule fits; the scheduler's own execution cost is measured and charged.

## Metrics and acceptance (Table 1 of the proposal)

| Metric | Acceptance |
|---|---|
| Value per joule | >= 25% over policy A (H1) |
| Survival safety | zero E < E_res or T < T_min violations, all runs (H3) |
| Switching cost | E_sw, t_sw per mode pair characterized to +/-5% |
| Fault tolerance | SC holds the floor in 100% of fault trials |
| Overhead | MC scheduling energy < 2% of total (reported) |
| Repeatability | >= 100 seeded runs per scenario, 95% CIs, Wilcoxon tests, open CSV + code hash |

## Environment

- Development is in WSL2 (Ubuntu). The repo lives on the Linux filesystem (`/home/ajb/frostguard`); do not move it under `/mnt/c`.
- Targets: RP2350 Hazard3 RISC-V (MC) and Arm Cortex-M33 (SC). Pico SDK 2.x selects the platform (`rp2350-riscv` vs `rp2350-arm-s`); the Arm build needs `arm-none-eabi-gcc` and the RISC-V build needs a RISC-V GCC toolchain.
- Firmware language is not yet decided (C/C++ with the Pico SDK is the default; the `.gitignore` also carries Rust entries). Record the decision in `docs/decisions/` before writing firmware.
- Host-side tooling is Python, managed with `uv`.
- USB boards and instruments are not visible to WSL2 until attached with `usbipd attach --wsl`. Prefer LAN (VISA over TCP) for instruments where supported.
- Large trace files stay out of git; commit the regeneration scripts in `traces/tools/` instead (or use Git LFS/DVC if the team decides to).

## Working conventions

- Pure logic (scheduler, reserve computation, mode state machine) must be hardware-independent and covered by native host tests in `tests/host/` before it runs on a board.
- Prefer small, reviewable commits. Do not commit build output, local instrument credentials or personal data.
- Record non-obvious design decisions as short ADRs in `docs/decisions/` (context, decision, consequences).
- When something in the proposal is ambiguous or a requirement conflicts with a safety rule above, ask rather than guess.
- Source documents: `docs/proposal/` (Phase 1 paper, advisor letter). The team's general EE reference wiki lives outside this repo at `C:\Users\balta\OneDrive\LLM_WIKI\wiki\` (WSL: `/mnt/c/Users/balta/OneDrive/LLM_WIKI/wiki/`); read it for background only, never edit it from here.
