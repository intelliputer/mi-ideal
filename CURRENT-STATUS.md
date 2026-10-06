# Current status — 2026-10-06

## Position

`mi-ideal` is a planning and integration repository for running a bounded
IDEAL embodied-interaction loop on a 6809/6309-class target.  The rationale,
target representation, safety boundaries, and staged plan are in
[CASE-FOR-6309.md](CASE-FOR-6309.md).

No new target binary, board modification, or hardware run was performed in
this repository as part of this status update.

## What already exists

| Item | Status | Evidence |
|---|---|---|
| IDEAL reference behavior | Available | [`cartheur/ideal`](../../cartheur/ideal/README.md) defines the Section 1 interaction loop; `Existence010` supplies the reference behavior. |
| Cross-assembly tooling | Available | [`M6x09-cross`](../../cartheur/M6x09-cross/README.md) is a local 6809/6309 assembler project. |
| M6x09-I physical path | Ready for seed-trace execution | Its serial/RAM-load route is recorded as verified, and it has an existing target-side [`ideal010`](../../cartheur/M6x09-I-SBC/applications/02-ideal010/README.md) application. |
| M6x09-I IDEAL seed | Implemented; physical capture still to be recorded for this effort | The program loads at `$0200`, uses GPIO `$8000` for visible experiments, and emits the eleven-cycle Section 1 trace through ACIA. |
| M6x09-II firmware model | Available | [`M6x09-II emulator`](../../cartheur/M6x09-II-SBC/emulator/README.md) models RAM, ROM, reset vector, and basic ACIA behavior. |
| M6x09-II physical serial path | Not yet accepted | The current board record requires the CTS/ACIA transmit measurement before terminal acceptance can be claimed. |
| 6309-native execution | Not started | The next valid step is an electrical compatibility check and an emulation-mode run; native mode remains deferred. |

## Immediate experiment sequence

1. On M6x09-I, build/load `applications/02-ideal010/ideal010.s19`, run at
   `$0200`, and preserve the serial capture plus GPIO observation.
2. Compare that capture with the `Existence010` golden trace.  Investigate any
   mismatch before changing the algorithm or using a 6309.
3. Substitute the synthetic result function with a bounded external input,
   retaining the same trace protocol and a safe idle action.
4. Resolve M6x09-II CTS/ACIA terminal acceptance; then load and run the same
   seed from RAM on that board.
5. Only after repeatable 6809-compatible results, validate the fitted 6309
   electrically and run the seed in emulation mode.  Consider native mode only
   after interrupt/stack assumptions are audited.

## Current blockers

- M6x09-II has an explicit hardware/serial acceptance dependency: its ACIA
  transmit path is awaiting the documented CTS measurement and follow-up.
- Exact external-result input circuitry and the safe-action policy have not
  been selected.  No actuator should be connected until those are specified
  and reset/watchdog behavior is tested.
- A 6309 substitution must not be assumed electrically safe solely from 6809
  success; board wiring and the selected part need a specific check.

## Completion criteria for the seed milestone

- A saved M6x09-I capture matches the Section 1 eleven-cycle trace.
- A real, bounded result input replaces the local deterministic result fixture.
- Reset or invalid input leaves the action output in its documented idle state.
- M6x09-II can reproduce the same RAM-loaded trace after its monitor/serial
  acceptance is complete.
- A 6309 runs the same image in emulation mode without behavioral drift.
