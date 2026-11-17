# 0001: Firmware language is C/C++ with the Pico SDK

Status: Accepted (2026-11-15)

## Context

`AGENTS.md` requires the firmware language to be recorded here before any firmware is written. The default was C/C++ with the Pico SDK. The `.gitignore` also carries Rust entries.

The firmware runs on two RP2350 boards built with different toolchains and ISAs: the MC on Hazard3 RISC-V, the SC on Arm Cortex-M33. The project relies on that diversity. The scheduler, reserve computation and mode state machine must also be testable natively on the host before they run on a board.

## Decision

Firmware under `firmware/` is written mostly in C and C++ using Pico SDK 2.x.

- MC builds with the `rp2350-riscv` platform and a RISC-V GCC.
- SC builds with the `rp2350-arm-s` platform and `arm-none-eabi-gcc`.
- `firmware/common/` stays C-compatible (protocol, types, CRC only) so both toolchains consume it identically.
- Pure logic is written so it also compiles natively for `tests/host/`, with no SDK or hardware dependencies.
- Host-side tooling (`bench/`, `analysis/`, `scripts/`) remains Python managed with `uv`. This decision covers firmware only.

Rust is not used for firmware for now. The Rust entries in `.gitignore` can be removed or kept; they are harmless.

## Consequences

- One SDK and one build system serve both boards, with a clear route to each target platform.
- Native host tests of the scheduler, reserve and state machine are straightforward.
- C/C++ gives no memory-safety guarantees from the language. Safety-critical SC code needs review, tests and fault-injection coverage (H3) to compensate. Coding rules (for example a restricted C++ subset or a standard such as MISRA) are not yet chosen and should be decided separately.
- Revisiting this later would mean a new ADR superseding this one.
