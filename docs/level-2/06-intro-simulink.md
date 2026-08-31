# 06 · Intro to Simulink Concepts

!!! note "Verification note"
    MATLAB/Simulink was not available in the environment used to write
    this page. Block names, port semantics, and solver behavior described
    here reflect Simulink's documented block-diagram semantics, hand-
    traced against the Simulink documentation rather than built and run
    in Simulink itself. No `.slx` file is included since it cannot be
    verified to open correctly without Simulink installed.

Everything so far has been *textual* MATLAB — scripts and functions.
Simulink is MATLAB's companion product for **block-diagram modeling** of
dynamic systems: instead of writing `x(k+1) = f(x(k))` as code, you wire
together visual blocks (integrators, gains, sums) that represent the same
mathematics, and Simulink handles simulating the system over time. This
module is a conceptual introduction — enough to read an existing model,
understand what each block family does, and know when block-diagram
modeling is the right tool versus plain MATLAB code.

## Why block diagrams instead of code?

Physical and control systems are naturally described as signal flow: a
sensor signal enters a filter block, feeds a controller block, drives an
actuator block, which affects a physical plant block, whose output feeds
back into the sensor. Simulink's diagram *is* the system's structure —
engineers who design controllers, not software, can read and modify it
without reading MATLAB syntax. It also comes with:

- A numerical **solver** (fixed-step or variable-step ODE solvers) built
  in, so you don't hand-write your own Euler/Runge-Kutta integration loop.
- Automatic handling of **continuous time** (differential equations) and
  **discrete time** (sample-and-hold systems) in the same model, with
  Simulink managing how they interact.
- Direct paths to **code generation** (Simulink Coder) and **hardware
  deployment**, which is why it's standard in automotive, aerospace, and
  industrial control engineering.

## The core block families

| Family | Examples | Purpose |
|---|---|---|
| Sources | `Constant`, `Sine Wave`, `Step`, `Ramp` | Generate an input signal with no upstream input |
| Sinks | `Scope`, `To Workspace`, `Display` | Consume a signal for visualization/logging, no downstream output |
| Continuous | `Integrator`, `Transfer Fcn`, `State-Space` | Represent differential-equation dynamics |
| Discrete | `Unit Delay`, `Discrete Transfer Fcn`, `Zero-Order Hold` | Represent difference-equation / sampled dynamics |
| Math Operations | `Gain`, `Sum`, `Product`, `Abs` | Elementary signal-processing operations |
| Signal Routing | `Mux`, `Demux`, `Switch`, `From`/`Goto` | Combine, split, or conditionally route signals |
| Logic and Bit Ops | `Relational Operator`, `Logical Operator`, `Compare To Zero` | Boolean/conditional logic on signals |

A block's **ports** are its inputs (left, filled triangle) and outputs
(right, filled triangle); a **signal line** is a directed wire connecting
one block's output port to another's input port. Signal lines carry
values continuously during simulation, analogous to a variable being
overwritten every timestep in code.

## A minimal first-order system by hand

Consider the classic first-order ODE `dy/dt = -a*y + u(t)`, a simple
decaying/driven system. In plain MATLAB you'd integrate this with
`ode45`:

```matlab
a = 2;
u = @(t) 1;                      % step input
dydt = @(t, y) -a*y + u(t);
[t, y] = ode45(dydt, [0 5], 0);
plot(t, y);
```

The equivalent Simulink block diagram wires:

```
[Step] ---> (+ Sum -) ---> [Gain: 1] ---> [Integrator] ---+---> [Scope]
              ^                                            |
              |______________ [Gain: -a] <_________________|
```

Reading this diagram: the `Integrator` block's *output* is `y(t)`
(integrating its input over time); that output feeds back through a
`Gain` block of `-a` into a `Sum` block that also receives the `Step`
input `u(t)`; the sum, `u(t) - a*y(t)`, is exactly `dy/dt`, which is fed
into the `Integrator`'s input. The `Integrator` block is the direct
visual analog of `ode45` integrating `dydt` — Simulink is doing the same
numerical integration, just expressed as a diagram with feedback instead
of an anonymous function.

The `Scope` block is the visual analog of `plot(t, y)`, but live-updating
during simulation rather than plotted after the fact.

## Solvers: fixed-step vs. variable-step

Simulink's **Configuration Parameters** (Simulation → Model Configuration
Parameters) choose how time advances:

- **Variable-step** solvers (e.g. `ode45`, `ode23`) shrink or grow the
  timestep automatically to hit an error tolerance — the same algorithm
  family as MATLAB's own `ode45` function, and the right default choice
  for most continuous-time simulation and analysis.
- **Fixed-step** solvers (e.g. `ode4` a.k.a. RK4, or a simple
  discrete/no-continuous-states solver) advance in constant time
  increments — required for real-time simulation, code generation to
  embedded hardware, and any model with only discrete-time blocks (where
  "variable step" has no meaning).

Choosing fixed-step with too large a step size is the most common source
of a Simulink model that "doesn't match" a MATLAB `ode45` result of the
same equations — the fixed step is under-resolving fast dynamics that
the variable-step solver would have caught and refined automatically.

## Discrete-time blocks and sample time

A `Unit Delay` block outputs its input from the *previous* sample step —
the block-diagram equivalent of `y(k) = x(k-1)` in a discrete-time
difference equation, and the fundamental building block for digital
filters and discrete controllers in Simulink. Every block in a
discrete-time model has a **sample time** (e.g. 0.01s) governing how
often it updates; mixing sample times (a `Unit Delay` at 0.01s feeding
into a block at 0.1s) requires an explicit `Rate Transition` block to be
correct, otherwise Simulink issues a warning about an implicit and
possibly unintended rate transition.

## `Mux`/`Demux` and vector signals

Simulink signals can be vectors, not just scalars — a `Mux` block
combines several scalar (or vector) signals into one vector signal;
`Demux` splits a vector signal back into its scalar (or sub-vector)
components:

```
[Sine Wave] --\
               [Mux] ---> [Scope]   (Scope shows both traces overlaid)
[Step]      --/
```

This is the block-diagram equivalent of `plot(t, [y1, y2])` plotting two
signals on one axes, or of building `[y1; y2]` as a 2-row matrix in code.

## Subsystems: the Simulink equivalent of a function

A **Subsystem** block groups a cluster of blocks behind one box with its
own input/output ports — directly analogous to extracting repeated code
into a MATLAB function. Subsystems can be:

- **Virtual** (the default) — purely a visual grouping; Simulink flattens
  it during simulation, equivalent to inlining a function call.
- **Atomic** — forces the subsystem to execute as one unit with its own
  scheduling, closer to a true function call boundary, and required for
  code generation of that subsystem as a standalone unit.

## Running a model from MATLAB code

Models can be simulated programmatically rather than by pressing the
"Run" button, which matters for parameter sweeps and automated testing:

```matlab
simOut = sim('my_first_order_model', 'StopTime', '10');
t = simOut.tout;
y = simOut.yout{1}.Values.Data;
plot(t, y);
```

`sim` returns a `Simulink.SimulationOutput` object; logged signals (via
`To Workspace` blocks, or "Log Signal" enabled on a wire) are retrievable
from it by name, letting a MATLAB script drive many simulation runs (e.g.
varying `a` in the earlier example) the same way it would sweep a
parameter in an `ode45` loop.

## When to reach for Simulink vs. plain MATLAB code

| Use Simulink when... | Use plain MATLAB code when... |
|---|---|
| The system is naturally block/signal-flow structured (control systems, signal chains) | The algorithm is naturally sequential/algorithmic (data analysis, optimization) |
| Non-programmers on the team need to read/modify the model | Only programmers will maintain it |
| You need automatic code generation for an embedded target | Deployment target is a MATLAB/desktop environment |
| The system mixes continuous and discrete dynamics with nontrivial interaction | Time-stepping is simple enough to hand-code clearly |

Many real projects use both: MATLAB scripts to set up parameters, call
`sim`, and post-process results; Simulink to hold the actual dynamic
model. That combination — not "Simulink instead of MATLAB" — is the
typical professional workflow.

## Summary

- A Simulink block diagram represents the same mathematics as a MATLAB
  script, using visual blocks and signal lines instead of statements and
  variables — an `Integrator` block corresponds to `ode45`-style
  integration, a `Unit Delay` to `y(k) = x(k-1)`.
- Solvers, sample time, and Mux/Demux vector signals are the concepts
  that don't have a direct one-line MATLAB-code analog and are worth
  understanding explicitly before opening the Simulink editor.
- `sim()` lets MATLAB code drive a Simulink model programmatically —
  the bridge that makes the two tools work together rather than as
  alternatives.
