# 02 · Advanced Simulink Modeling

!!! note "Verification note"
    MATLAB/Simulink was not available in the environment used to write
    this page. Block semantics, signal routing, and configuration options
    are documented, version-stable Simulink features, hand-traced against
    the Simulink documentation rather than built and simulated live.

Level 2 introduced Simulink as a block-diagram way to build the same kind
of system a script would compute step by step. This module goes past a
single flat diagram into the structuring tools real models need:
subsystems, buses, state machines, and the settings that make a model
scale beyond a toy example.

## Subsystems: giving a diagram structure

A model with fifty blocks wired directly on one canvas is unreadable.
**Subsystems** group a cluster of blocks behind a single block with its
own inputs/outputs, the Simulink equivalent of factoring code into a
function:

```
[Sum]->[Subsystem: PID Controller]->[Plant]->[Scope]
              |    ^
          (In1) (Out1)   -- ports inside map to the block's edges outside
```

Right-click a selection → **Create Subsystem from Selection** replaces
the selected blocks with one masked block; double-clicking it opens the
grouped diagram, with `In1`/`Out1` port blocks marking where the outside
wires attach. This is purely organizational — simulation results are
identical before and after — but it is what makes a controls model with
hundreds of blocks navigable and reusable (the same "PID Controller"
subsystem can be copy-pasted or saved as a library block and dropped into
another model).

**Masking** a subsystem adds a custom dialog (parameters like `Kp`, `Ki`,
`Kd` exposed as fields) and a custom icon, so a reusable subsystem behaves
like a first-class block rather than exposing its internal wiring to
every user of the model.

## Buses: bundling related signals

Passing a dozen separate wires between subsystems (temperature, pressure,
flow rate, three status flags...) is as unwieldy as fifty scalar function
arguments would be in code. A **Bus** groups related signals into one
line, using a **Bus Creator** to pack and a **Bus Selector** to unpack:

```
[Temp]  --\
[Pressure]--[Bus Creator]---[single bus wire]---[Bus Selector]--[Temp]-->[Scope]
[Flow]   --/                                                  \-[Pressure]->[...]
```

A `Simulink.Bus` object (defined once, e.g. in a model's `PreLoadFcn` or a
project setup script) formally names and types each element, the way a
`struct` with named fields formally documents what's inside it instead of
an unlabeled cell array of values in some agreed-upon order:

```matlab
elems(1) = Simulink.BusElement;
elems(1).Name = 'Temp';
elems(1).DataType = 'double';
elems(2) = Simulink.BusElement;
elems(2).Name = 'Pressure';
elems(2).DataType = 'double';
sensorBus = Simulink.Bus;
sensorBus.Elements = elems;
```

## Stateflow-style logic: the Switch and enable/trigger ports

Not every system is purely continuous math — many need mode-dependent
behavior ("run the controller only while `Enable` is true," "reset the
integrator when `Reset` pulses"). Two block-level tools cover the common
cases without a full state chart:

- **Enabled subsystem**: runs its contents only while its `Enable` input
  is nonzero; while disabled it holds (or optionally resets to zero, per
  block setting) its last output — the block-diagram version of an `if`
  guard wrapped around a whole chunk of a model.
- **Triggered subsystem**: executes exactly once per rising (or falling,
  or either) edge of a `Trigger` input, useful for sample-and-hold style
  behavior driven by an external event rather than every simulation step.

```
[EnableSignal] --> (enable port)
                        |
[Input] --> [Enabled Subsystem: Controller] --> [Output]
```

For genuinely mode-based systems (idle → ramping → running → fault, each
with different logic and edges between them triggered by conditions),
**Stateflow** provides a proper state-chart editor with states,
transitions, and guard conditions — the block-diagram analog of a `switch`
over an enum plus explicit transition rules, and the standard tool once
"enabled subsystem" checks start nesting more than one or two levels deep.

## Signal attributes and the model's data dictionary

Every wire in Simulink carries a **data type**, **dimension**, and
**sample time**, usually inferred automatically but explicitly
overridable via a **Signal Specification** block or a bus element's
`DataType` field. This matters most when a model will later be compiled
with MATLAB Coder or deployed to fixed-point hardware (Level 4 covers
both): pinning `int16` on a sensor signal early catches a type mismatch
in simulation, in the same design environment, rather than after
generating C code for an embedded target.

A **Data Dictionary** (`.sldd` file) centralizes these definitions —
signal types, bus objects, calibratable parameters — outside any single
model, so multiple models in one project reference the same shared
definitions instead of each redefining `sensorBus` independently and
risking drift between copies.

## Solver configuration for accuracy vs. speed

Level 2 used the default variable-step solver without discussing it.
Simulink's **Model Configuration Parameters** dialog exposes the
trade-off explicitly:

| Solver | Behavior | When to use |
|---|---|---|
| `ode45` (variable-step) | Adapts step size to the signal's rate of change | Default for smooth, continuous dynamics — usually the right first choice |
| `ode23` | Lower-order, cheaper per step | Mildly stiff or when `ode45` is unnecessarily fine-grained |
| `ode23s`/`ode15s` | Implicit, designed for stiff systems | Dynamics with widely separated time constants (fast electrical transient + slow thermal drift) |
| Fixed-step (`ode4`, discrete) | Constant `Δt` every step | Required before generating code for real-time or embedded targets, where the target hardware runs on a fixed clock |

Switching from variable-step to fixed-step is a required step, not an
optional tune, before Level 4's code-generation workflow — real-time
targets execute one tick per hardware clock cycle and cannot honor a
solver that shrinks or grows `Δt` based on signal behavior.

## Model referencing: composing models from models

A **Model block** embeds one whole `.slx` model inside another, the
system-level equivalent of one script calling another as a function
rather than being pasted inline. Large projects (a full vehicle model
built from separately-owned `Engine.slx`, `Transmission.slx`,
`Brakes.slx` models) use this to let teams work on subsystems
independently, version them separately, and simulate the top-level model
either against the full sub-model or a fast simplified "surrogate"
version swapped in during rapid top-level iteration.

## Summary

- **Subsystems** group blocks for readability and reuse, purely
  organizational; **masking** turns a subsystem into a reusable block
  with its own parameter dialog.
- **Buses** bundle related signals under one wire and one named type
  (`Simulink.Bus`), avoiding dozens of individually-routed scalar wires.
- **Enabled**/**triggered subsystems** add lightweight mode-dependent
  execution; **Stateflow** is the full state-chart tool once mode logic
  gets genuinely complex.
- Signal types/dimensions and a shared **Data Dictionary** keep large
  models consistent and are prerequisites for later code generation.
- Solver choice (variable-step `ode45`/`ode15s` vs. fixed-step) trades
  simulation accuracy/speed and, for fixed-step, is mandatory before
  targeting real-time or embedded hardware.
- **Model referencing** composes large models from independently-owned
  sub-models, mirroring modular software architecture at the block-diagram
  level.
