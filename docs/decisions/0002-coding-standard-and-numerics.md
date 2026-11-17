# 0002: Firmware coding rules and numeric representation

Status: Proposed (2026-11-16). Not yet reviewed by the co-lead or advisor.

## Context

ADR 0001 chose C/C++ with the Pico SDK. C/C++ gives no language-level memory safety, and the SC is the component H3 depends on. The MC and SC are built with two different GCC toolchains (RISC-V and Arm), and both consume `firmware/common/`. The Hazard3 cores have no FPU (needs verification against the RP2350 datasheet), so floating-point energy math on the MC is software-emulated. Scheduler overhead is measured and charged (target < 2% of total energy).

## Decision (proposed)

### Language subset

- **SC (`firmware/sc/`)**: C, or C++ restricted to: no exceptions, no RTTI, no dynamic allocation, no STL containers that allocate. All memory is statically allocated. No recursion.
- **MC (`firmware/mc/`)**: same restrictions by default. Exceptions can be argued per case in a new ADR.
- **`firmware/common/`**: C-compatible headers only. Fixed-width integer types. Protocol messages are serialized field by field with explicit byte order. Structs are never sent by `memcpy` or relied on for layout.

### Build and checks

- Compile with `-Wall -Wextra -Werror` on both targets.
- Run a static analyzer (cppcheck or clang-tidy; tool to be chosen) in a script under `scripts/`.
- Pure logic (scheduler, reserve, mode state machine, CRC, protocol encode/decode) compiles natively and is tested in `tests/host/`. The same CRC and protocol test vectors run against both the RISC-V and Arm builds where practical.
- A formal standard (MISRA C, JPL rules, etc.) is **not** adopted at this stage. Revisit if the team wants to cite one in the paper.

### Numerics

- **SC**: reserve and thermal checks use integer or fixed-point arithmetic with documented units (for example millijoules, millikelvin). No floating point in the veto path, so behavior is deterministic and identical across builds.
- **MC scheduler**: starts with fixed-point as well. Floating point is allowed only if a measurement shows fixed-point is inadequate, and the change is recorded in an ADR with the measured scheduler energy.
- Unit conversions happen at module boundaries, and names carry units (`energy_mj`, `temp_mk`).
- INA228 ENERGY (40-bit, rolls over) is widened to a 64-bit running total in the driver. Rollover is handled via delta accumulation, with DIAG_ALRT's energy-overflow flag as a cross-check.

## Consequences

- The SC veto path is simple enough to review line by line and to test exhaustively on the host.
- Fixed-point costs development effort (scaling, overflow analysis) and needs host tests at range extremes.
- Restricted C++ forbids some convenient idioms. That is intended.
- The cross-toolchain tests catch portability bugs between the RISC-V and Arm GCC builds early.

## Open questions

- Which static analyzer, and does CI run it, or only a local script?
- Do we adopt a named standard (MISRA C or JPL) for the SC only?
- The fixed-point formats (Q-format or plain scaled integers), and the unit set.
- Confirm the Hazard3 FPU claim from the RP2350 datasheet and cite it in `docs/references/`.
