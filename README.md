# FROSTGUARD

**Survival-Value Power and Task Control for Energy-Constrained Lunar Surface Science Nodes**

NASA ORBIT Challenge 2027, ORBIT Space track. Team HD 137010 b (Cal Poly Pomona, RSCL).
Addresses Civil Space Shortfall 1618: survive and operate through the lunar night.

## What this is

A lunar surface science node has little energy and must survive long shadow periods. FROSTGUARD decides which science tasks to run by value per joule. It does this without ever putting the survival reserve (heater energy, thermal floor) at risk.

The Phase 2 deliverable is a bench-scale hardware-in-the-loop (HIL) prototype on two RP2350 microcontrollers. It is compared against three baseline policies across at least 100 seeded runs per scenario. Data are open and the controller design is at TRL 3-4. **No flight qualification is claimed.**

## Architecture

Two separate RP2350 boards on separate supply rails:

| Board | Core | Role |
|---|---|---|
| Mission controller (MC) | Hazard3 RISC-V | Task contracts (v_i, e_i, d_i) and the value-per-joule scheduler. Can only *request* loads. |
| Safety controller (SC) | Arm Cortex-M33 | Sole owner of the load gates L0 (survival heater), L1 (science sensor), L2 (compute/FPGA) and L3 (radio). Grants or vetoes MC requests, enforces E >= E_res(t) and T >= T_min, and watches the MC heartbeat. |

- **Telemetry:** INA228 (V, I, E per domain), TMP117 and NTC (temperatures). The MC gets a read-only copy.
- **Modes:** NOMINAL, CONSERVE, SURVIVE, SAFE-HOLD, with hysteresis. The SC executes SURVIVE and SAFE-HOLD transitions on its own.
- **Policies compared:** A fixed priority, B battery-threshold shutdown, C energy-aware scheduling without switching cost or independent reserve, D FROSTGUARD. Ablations remove the switching cost (E_sw) or merge SC duties into the MC.

## Hypotheses

- **H1 (value):** FROSTGUARD returns at least 25% more validated, priority-weighted observations per joule than fixed-priority scheduling (A), with zero reserve or thermal-floor violations.
- **H2 (switching cost):** Removing measured switching costs (E_sw, t_sw) from the scheduler causes a statistically significant loss of value per joule or reserve margin on traces with frequent illumination changes.
- **H3 (independence):** With the MC crashed, hung or requesting adversarial loads, the SC holds E >= E_res and T >= T_min in 100% of fault-injection trials.

A stop rule applies: if D does not beat C after scheduler overhead, the claim is revised, not hidden.

## Acceptance metrics

| Metric | Acceptance |
|---|---|
| Value per joule | >= 25% over policy A (H1) |
| Survival safety | Zero E < E_res or T < T_min violations in all runs (H3) |
| Switching cost | E_sw, t_sw per mode pair characterized to +/-5% |
| Fault tolerance | SC holds the floor in 100% of fault trials |
| Overhead | MC scheduling energy < 2% of total (reported) |
| Repeatability | >= 100 seeded runs per scenario, 95% CIs, Wilcoxon tests, open CSV + code hash |

## Repository layout

```
firmware/     common/ (protocol, types, CRC only), mc/ (RISC-V), sc/ (Arm), drivers/
tests/        host/ (native unit tests), hil/ (pytest on the real bench, incl. fault injection)
hardware/     kicad/, bom/, thermal-fixture/, safety/
bench/        host-side Python automation (instruments, replay, faults, calibration, runner)
traces/       lunar (LOLA, Diviner) and terrestrial (NSRDB) traces; tools/ regenerates them
data/runs/    one folder per gate/scenario/policy: CSV + metadata.json (open data)
analysis/     notebooks, stats, figures
docs/         proposal, gates (G1-G4), references, decisions (ADRs), customer-discovery
scripts/      build, flash, test helpers
```

## Ground rules

These are summarized here; `AGENTS.md` is authoritative.

1. Only the SC drives the L0-L3 gates. Nothing under `firmware/mc/` touches gate pins.
2. `firmware/mc/` and `firmware/sc/` share no code except `firmware/common/`, which holds protocol, types and CRC only.
3. Task weights are frozen before trials. Changes need a committed decision in `docs/decisions/`.
4. Every run records both firmware git hashes, the seed, the scenario, the policy and the trace ID. Runs without this metadata are invalid.
5. `data/runs/` is append-only. Re-analysis goes in `analysis/`.
6. No powered testing of the power board, battery or thermal fixture until the checklist in `hardware/safety/` is complete and advisor-reviewed.
7. The bench battery stays at >= 0 C.
8. Lunar and terrestrial profiles are reported separately.
9. No unverified claims. Sources go in `docs/references/`.

## Schedule

| Gate | Target | Scope |
|---|---|---|
| G1 | ~Jan 2027 | Safe power board, calibrated sensing, baseline switching |
| G2 | ~Mar 2027 | Task contracts, SC reserve enforcement, full E_sw table |
| G3 | ~May 2027 | Four-policy and ablation results with fault injection |
| G4 | ~Jun 2027 | Terrestrial-profile trials, discovery interviews, BOM |

Dates after Phase 1 follow the official ORBIT schedule and may change. Plans and results per gate go in `docs/gates/`.

## Getting started

- Development is in WSL2 (Ubuntu). Keep the repo on the Linux filesystem, not under `/mnt/c`.
- Host tooling is Python managed with `uv`.
- Firmware targets: Pico SDK 2.x with `rp2350-riscv` (MC, needs a RISC-V GCC) and `rp2350-arm-s` (SC, needs `arm-none-eabi-gcc`).
- Firmware is written mostly in C/C++ with the Pico SDK (see `docs/decisions/0001-firmware-language.md`). Host tooling stays Python.
- USB boards are not visible to WSL2 until attached with `usbipd attach --wsl`. Prefer VISA over TCP for instruments.
- Pure logic (scheduler, reserve, mode state machine) gets native host tests in `tests/host/` before it runs on a board.

Build, flash and test commands will be added here once `scripts/` has content.

## Team

- **Ari Akram** (lead): mission-controller firmware, scheduler, test automation, statistics
- **Ari Baltazar Moreno** (co-lead): power board, safety controller, thermal fixture, trace replay
- **Faculty advisor:** Prof. Mohamed El-Hadedy

## Status

Early scaffolding: folder structure and project rules are in place; no firmware, hardware or results yet.

## License

Not yet chosen. Decide before the open-data release.
