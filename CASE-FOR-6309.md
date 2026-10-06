# The case for an HD63C09 implementation of IDEAL

## Decision

Make the HD63C09 ("6309") the target for a small, fixed-capability runtime of
IDEAL.  Start from 6809-compatible code and board assumptions, then enable
6309-native facilities only after the baseline interaction loop is proven.

This is a case for an *embodied controller*, not for moving the current .NET
program wholesale onto an 8-bit machine.  The host implementation remains the
reference model and a development oracle; the 6309 runs a deliberately bounded
agent connected to real experiments and result signals.

## Why IDEAL fits

The foundational IDEAL implementation is a compact cyclic machine:

```text
select experiment -> anticipate result -> perform experiment -> receive result
    -> record interaction -> update mood -> select next experiment
```

The first implementation has only two experiments, two results, a previous
experiment, a self-satisfaction counter, and a small interaction store.  Its
decisions are local table lookups and comparisons; it has no requirement for a
world model, floating-point arithmetic, an operating system, or a large
runtime.  That is exactly the profile for which a deterministic, memory-mapped
8-bit controller is credible.

The important architectural match is conceptual as well as computational.  In
IDEAL, input is the *result of an experiment*, rather than a general-purpose
description of the world.  On a 6309 board, actuator selection is the
experiment and a sampled sensor/status latch is the result.  Each cycle can
therefore be made visible and testable at the hardware boundary.

## Why the 6309, specifically

The HD63C09 is a software-compatible extension of the 6809 with additional
accumulators (`E` and `F`), combined `W` and `Q` registers, native mode, and
instructions useful for data-oriented work.  These are useful headroom for
later interaction tables, confidence/count fields, scanning, and serial
framing.  They are *not* prerequisites for the first agent.

That gives this project a low-risk sequence:

1. Assemble and validate the seed runtime as conservative 6809 code.
2. Run it in 6809-emulation mode on a socketed 6309 board.
3. Establish cycle traces and table contents against the .NET reference.
4. Profile a real workload.
5. Introduce a small, separately tested 6309-native routine only where it
   provides a measured benefit.

The sequence preserves a working fallback.  It also avoids the classic 6309
hazard: native-mode interrupts save the added `E`/`F` state, changing the stack
frame expected by 6809 interrupt code.  Do not enter native mode until every
interrupt handler, monitor routine, and debugger assumption has been audited.

## Evidence already present in the workspace

| Evidence | What it contributes |
|---|---|
| [`cartheur/ideal`](../../cartheur/ideal/README.md) | Defines IDEAL as an embodied, interaction-first learning algorithm; Section 1 gives the seed algorithm and expected trace. |
| [`Existence010.cs`](../../cartheur/ideal/Existence/Existence010.cs) | Shows the minimal state and step order that the target must preserve. |
| [`M6x09-cross`](../../cartheur/M6x09-cross/README.md) | Provides a local 6809/6309 cross-assembler and a 6309 technical reference. |
| [`M6x09-I-SBC`](../../cartheur/M6x09-I-SBC/README.md) | A physically verified 6809 development board with RAM loading, 19,200-bit/s RS-232, GPIO, keypad monitor, and a stable recovery path. |
| [`M6x09-I Ideal 010`](../../cartheur/M6x09-I-SBC/applications/02-ideal010/README.md) | An existing target-side Section 1 implementation with the expected eleven-cycle trace and GPIO-visible experiments. |
| [`M6x09-II-SBC`](../../cartheur/M6x09-II-SBC/README.md) | A compact 6xC09-socket board with RAM, ROM, ACIA, expansion header, and an associated host emulator. |
| [`M6x09-II emulator`](../../cartheur/M6x09-II-SBC/emulator/README.md) | A repeatable RAM/ROM/ACIA model for firmware tests before electrical board bring-up. |
| [`aiventure-rodney/build/6x09`](../../cartheur/aiventure-rodney/build/6x09/README.md) | Has an existing 6809-first/6309-compatible board strategy, 6309 inventory, support ROM, and staged bring-up plan. |
| [`MEMORY-AND-DECODE-PROPOSAL.md`](../../cartheur/aiventure-rodney/build/6x09/MEMORY-AND-DECODE-PROPOSAL.md) | Provides a simple, documented 64 KiB map and separate environment, action, and learned-memory registers. |
| [`mi-intellivision` serial interface](../mi-intellivision/docs/6309-forth-serial-host-interface.md) | Provides a conservative monitor/serial model: staged RAM loads, checksums, bounded addresses, watchdog-safe outputs, and a host that never directly controls actuators. |

These are assets, not proof that a complete IDEAL port already exists.  The
proof must come from a bounded implementation and a repeatable bench trace.

## Experimentation platforms

The two existing SBCs make the proposal immediately testable, but they should
have different roles.

### M6x09-I: reference trace on physical hardware

Use M6x09-I first for the seed acceptance trace.  It already has the exact
`applications/02-ideal010` experiment: it loads at `$0200` into RAM, writes
`E1`/`E2` to GPIO at `$8000`, emits the IDEAL trace through its 6850 ACIA, and
returns to the monitor with `SWI`.  Its documented serial and RAM-load path is
verified, so it is the shortest route to demonstrating that the interaction
cycle survives assembly, transfer, execution, and physical observation.

The fitted board is documented as a 6809 system.  Treat it as the
6809-compatibility baseline; do not assume a 6309 drop-in experiment until the
socket, supply, clock, reset, and interrupt wiring have been checked against
the particular HD63C09 part being used.  The current seed program has no
interrupts, which makes it an especially good first substitution test once
that electrical review passes.

The next M6x09-I milestone is not another synthetic trace.  Replace the local
`result = experiment` fixture with a debounced, bounded input from the
expansion header or another isolated input circuit, while keeping GPIO output
and serial trace unchanged.  That makes the experiment/result boundary real
without introducing an uncontrolled actuator.

### M6x09-II: 6309-oriented firmware and monitor platform

Use M6x09-II for the 6309-specific track.  Its design explicitly provides a
6xC09 CPU position, 32 KiB RAM, 16 KiB ROM, a 68B50-compatible ACIA, and an
expansion header.  Keep ASSIST09 in ROM and load IDEAL images into RAM; do not
burn a new EPROM for each experiment.

Begin with the host emulator: add an IDEAL RAM fixture and tests for reset,
result-table updates, serial trace bytes, and safe idle output.  The emulator
does not prove the electrical board, but it makes firmware behavior and ACIA
expectations reproducible before a ROM or wiring change.

M6x09-II's current terminal acceptance is still pending the documented CTS/
ACIA transmit investigation.  Until that is closed, it is an emulator and
firmware-preparation target, not the primary physical demonstration platform.
After the monitor banner and RAM-load smoke test pass, repeat the M6x09-I
golden trace on II, then test the 6309 first in emulation mode before enabling
native mode.

## Proposed 6309 seed runtime

Use fixed-width identifiers, not strings or dynamic collections.

| IDEAL concept | 6309 seed representation |
|---|---|
| Experiment | one-byte action/experiment ID |
| Result | one-byte sampled result ID |
| Interaction | packed `(experiment, result)` entry or direct indexed table |
| Anticipation | result byte stored at `anticipation[experiment]`, plus valid bit |
| Previous experiment | one byte |
| Mood | one byte enum |
| Self-satisfaction duration | one byte, saturating counter |
| Boredom threshold | ROM constant, initially `4` to match `Existence010` |

One seed cycle should be structured as follows:

```text
1. If mood is BORED, choose a permitted alternate experiment and clear duration.
2. Read anticipated result for the experiment, if its table entry is valid.
3. Write the experiment to the action latch.
4. Wait for, or sample, the corresponding result latch.
5. Store/update the `(experiment -> result)` table entry.
6. Compare actual result with anticipation; set FRUSTRATED or SELF-SATISFIED.
7. Update the duration; change to BORED at the configured threshold.
8. Emit the cycle record and retain the experiment for the next cycle.
```

The hardware contract can reuse the established names from the Rodney 6x09
proposal: `ENVL`/`ENVH` for result inputs, `ACTL`/`ACTH` for experiments or
actions, and `MMA_L`/`MMA_H`/`MMD` if learned interactions live behind an
indirect learned-memory interface.  Program RAM, support ROM, and learned
memory should remain distinct.  A first version needs neither banking nor a
general object system.

## What success looks like

The first acceptance test should be the exact two-experiment deterministic
environment described by IDEAL, implemented with switches or a fixture rather
than a robot.  With `e1 -> r1`, `e2 -> r2`, and a boredom threshold of four,
the target must produce the reference behavioral sequence:

```text
e1r1 FRUSTRATED
e1r1 SELF-SATISFIED
e1r1 SELF-SATISFIED
e1r1 SELF-SATISFIED
e1r1 BORED
e2r2 FRUSTRATED
...
```

Capture each cycle over the monitor serial port and compare it with a host
golden trace.  Then test at least these adverse cases: an unanticipated result,
a changed result for a known experiment, a stuck result line, reset during a
cycle, and learned-memory read/write failure.  The controller must have a safe
idle action on reset, watchdog timeout, or invalid input.

## Boundaries and objections

This proposal does **not** claim that a 6309 can host the later .NET design
unchanged.  Dynamic dictionaries, string labels, object allocation, tracing
verbosity, and higher branches of IDEAL require a host-side tool or a purpose
designed compact representation.  The port should preserve observable
behavior, not source-level structure.

The 6309 is also not automatically the right choice for every phase.  A modern
host remains better for development, large experiments, visualization, and
offline analysis.  The 6309 earns its place where tight experiment/result
cycles, inspectable state, deterministic I/O, and long-lived standalone
operation matter more than throughput.

Finally, existing local 6x09 work recommends bringing up plain 6809 behavior
before treating the 6309 as an optimization.  This project adopts that advice:
the 6309 is the deployment ceiling and performance option, while
6809-compatible correctness is the bring-up floor.

## Immediate next deliverables

1. Run and capture the existing M6x09-I `ideal010` trace; compare it byte for
   byte with a host-generated `Existence010` golden trace.
2. Define the seed binary table layout and exact I/O addresses for each board;
   do not silently reuse M6x09-I's `$8000` GPIO address on M6x09-II.
3. Complete M6x09-II's CTS/ACIA acceptance, then establish its RAM-load smoke
   test and port the seed loop without changing the behavioral trace.
4. Complete reset, ROM, RAM, action latch, and learned-memory tests before
   connecting an actuator.
5. Check 6309 substitution electrical compatibility, run in emulation mode,
   and only then benchmark a narrowly scoped native-mode optimization.
