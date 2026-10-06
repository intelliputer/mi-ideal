# Work-plan proposal: IDEAL on 6809/6309

## Purpose

Develop a small, inspectable embodied controller that learns the relationship
between its own experiments and their results.  The immediate target is the
IDEAL Section 1 interaction loop; the long-term target is a constrained
sensorimotor behavior layer for a robot, instrument, or other physical
apparatus.

This is a proposal, not evidence that later phases are complete.  Each phase
must produce a preserved trace or test artifact before the next one begins.

## Design commitments

- The target initiates an experiment, receives its result, records the
  interaction, and updates its anticipation and mood.
- The target is a bounded controller.  It does not attempt to host the .NET
  object model, dynamic dictionaries, or general development environment.
- A modern host builds images, captures traces, compares results, and supports
  offline analysis.  It never bypasses the target to control a physical
  actuator directly.
- Bring-up remains 6809-compatible first.  The 6309 is introduced in
  emulation mode before any native-mode optimization.
- New physical I/O begins with bounded inputs and safe, observable outputs;
  actuator authority is added only after reset and failure behavior is proven.

## Work streams

| Stream | Responsibility | Initial platform |
|---|---|---|
| Reference behavior | Maintain a host-side golden trace from `Existence010`. | `cartheur/ideal` |
| Target runtime | Represent experiments, results, anticipation, mood, and interaction memory in fixed-width target state. | M6x09-I |
| Bench validation | Load images, capture serial output, observe GPIO, and preserve test records. | M6x09-I |
| 6309 platform | Establish monitor, serial upload, emulator coverage, and 6309-compatible execution. | M6x09-II |
| Physical environment | Define isolated inputs, safe output circuitry, reset behavior, and a repeatable fixture. | Expansion hardware |
| Analysis | Compare target traces and retained learning state across experiments. | Linux host |

## Phase 0 — Baseline and evidence capture

**Goal:** establish a reproducible reference before changing hardware or
algorithm behavior.

1. Generate a machine-readable golden trace from `Existence010` for the
   two-experiment environment and boredom threshold of four.
2. Build the existing M6x09-I `ideal010` application, load it at `$0200`, and
   capture its serial output and GPIO observation.
3. Compare the target trace to the golden trace byte for byte; record the
   binary/image checksum, board, CPU, monitor version, load procedure, and
   terminal settings.

**Exit criteria:** a preserved M6x09-I capture matches the expected eleven
cycles and the board can return to its monitor without ROM modification.

## Phase 1 — 6809-compatible seed controller

**Goal:** make the target state and cycle contract explicit and testable.

1. Freeze a compact RAM layout for the seed: current experiment, result,
   prediction/valid entries, mood, satisfaction counter, and trace buffer.
2. Specify the experiment and result interfaces separately.  An action output
   is not assumed to be a representation of the environment; its observed
   consequence is the result.
3. Add test cases for first encounter, correct anticipation, boredom-driven
   experiment change, changed result, invalid result, and reset during a
   cycle.
4. Make reset and any invalid input produce an explicit idle action.

**Exit criteria:** every seed behavior has a traceable test, and memory,
output, and reset semantics are documented independently of an individual SBC.

## Phase 2 — Real but bounded experiment/result boundary

**Goal:** replace the synthetic result function without giving the controller
unsafe authority.

1. Choose one isolated, debounced input on the M6x09-I expansion path as the
   result source.
2. Retain GPIO `$8000` or an equivalent visible, non-hazardous output as the
   experiment indicator.
3. Define sample timing, input encoding, invalid-input handling, and the
   physical fixture's truth table.
4. Demonstrate that changing the fixture changes results and hence the learned
   anticipation, while a reset returns output to idle.

**Exit criteria:** the controller learns from a repeatable physical result
signal; no direct hazardous actuator is connected.

## Phase 3 — M6x09-II monitor and emulator readiness

**Goal:** establish the intended 6309-oriented platform without conflating
firmware errors and board faults.

1. Complete the documented CTS/ACIA transmit investigation and record terminal
   acceptance for M6x09-II.
2. Preserve ASSIST09 as recovery infrastructure and prove a RAM-load smoke
   test before loading IDEAL.
3. Add an IDEAL seed RAM fixture to the M6x09-II emulator, including serial
   trace assertions, table-update tests, and reset-vector behavior.
4. Port the seed image to M6x09-II RAM using its own documented addresses;
   do not inherit M6x09-I I/O constants.

**Exit criteria:** emulator and board both produce the seed trace from a
RAM-loaded image, and monitor recovery remains available.

## Phase 4 — HD63C09 validation and measured optimization

**Goal:** prove that the 6309 is an advantage without compromising the working
6809 baseline.

1. Check the selected HD63C09 part against the board's socket, supply, clock,
   reset, interrupt, and bus wiring.
2. Run the unchanged 6809-compatible image in 6309 emulation mode and compare
   trace and retained state against the baseline.
3. Measure cycle time, memory use, serial behavior, and failure/recovery
   behavior under the real fixture.
4. Identify one measured bottleneck—such as interaction-table scan, framing,
   or arithmetic—and implement a narrowly scoped 6309 routine.
5. Audit every interrupt and monitor assumption before enabling native mode,
   because its extended stack frame differs from the 6809's.

**Exit criteria:** a documented 6309 configuration reproduces behavior without
drift and any native-mode use has a measured benefit plus an audited interrupt
path.

## Phase 5 — Expand the interaction vocabulary

**Goal:** evolve from a two-pair demonstration to useful constrained learning.

1. Add experiments and result categories only when their physical semantics,
   encoding, and safe behavior are documented.
2. Replace the two-entry anticipation store with a bounded table format that
   has known capacity, replacement policy, and integrity checks.
3. Add persistent learned state only after defining versioning, initialization,
   corruption detection, and factory-reset behavior.
4. Keep a host-readable trace format so behavioral regressions can be
   compared across firmware revisions.

**Exit criteria:** the target learns and recalls a documented finite vocabulary
of interactions across repeatable fixture scenarios.

## Phase 6 — Constrained autonomous deployment

**Goal:** use the controller as the behavior layer in a real system while
keeping it observable and recoverable.

1. Select a robot, instrument, or apparatus with a limited action set and an
   explicit safe state.
2. Place hardware interlocks between controller output and every consequential
   actuator; watchdog expiry, reset, cable loss, and invalid state must return
   the apparatus to safe idle.
3. Run supervised experiments first, capturing target traces, host logs, and
   fixture/physical observations together.
4. Treat the host as analysis and update infrastructure, not as a hidden
   replacement controller.

**Exit criteria:** the target autonomously executes only approved experiments,
learns within its bounded vocabulary, records inspectable state, and fails
safe.

## Horizon

The near horizon is a repeatable IDEAL 010 trace on both SBCs with a real,
bounded result input.  The middle horizon is a persistent finite interaction
memory whose behavior can be compared across physical experiments.  The far
horizon is a 6309-native embedded IDEAL kernel serving as a transparent,
deterministic sensorimotor layer, with the Linux host dedicated to building,
inspection, visualization, and offline analysis.

The project should advance only when each horizon is evidenced by artifacts,
not by inferred capability: source revision, image checksum, board and CPU
identity, serial trace, fixture state, and observed output.
