# 01 · Advanced Simulink & Model-Based Design

!!! note "Verification note"
    Simulink was not available in the environment used to write this
    page. Block semantics, solver behavior, and model-based design
    workflow described below are documented, version-stable Simulink
    concepts, hand-traced against the MathWorks documentation rather
    than built or simulated in Simulink itself.

Simulink extends MATLAB from text-based numerical computing into
block-diagram modeling of dynamic systems — signal flow drawn as
connected blocks rather than written as code, particularly suited to
control systems, signal processing chains, and physical system
simulation where the block diagram *is* the natural representation of
the system.

## Why model-based design

Model-based design (MBD) treats the Simulink model itself as the
executable specification of a system, used across the whole development
lifecycle: design and simulate the controller in Simulink, generate
C/C++ code from the *same* model (Module 02 covers code generation in
depth), and deploy that code to embedded hardware — eliminating the gap
between a design document and the code that implements it, since they're
the same artifact.

## Core block categories

- **Sources**: `Step`, `Sine Wave`, `Ramp`, `Constant`, `From
  Workspace` (inject a MATLAB variable as a time series).
- **Sinks**: `Scope` (view signals during simulation), `To Workspace`
  (export simulated signals back to MATLAB for analysis), `Display`.
- **Continuous**: `Integrator`, `Derivative`, `Transfer Fcn`,
  `State-Space` — these introduce dynamics governed by differential
  equations.
- **Discrete**: `Unit Delay`, `Discrete Transfer Fcn`, `Discrete
  State-Space` — the sampled-time equivalents.
- **Math Operations**: `Gain`, `Sum`, `Product`, `Abs`, `Saturation`.
- **Logic and Bit Operations**: `Switch`, `Relational Operator`, `Logical
  Operator`.
- **Signal Routing**: `Mux`/`Demux` (bundle/unbundle signals),
  `Bus Creator`/`Bus Selector` (structured multi-signal groups).

## A simple closed-loop example (described structurally)

A PID-controlled first-order plant, block by block:

```
Step (setpoint) --> Sum (+) --> PID Controller --> Transfer Fcn (plant) --> Scope
                       ^ (-)                              |
                       |----------------------------------|
```

The `Sum` block computes `error = setpoint - measured_output`; the `PID
Controller` block (a pre-built Simulink block, not hand-derived
difference equations) computes the control signal from proportional,
integral, and derivative terms; the `Transfer Fcn` block, parameterized
by numerator/denominator coefficients (e.g. `[1]` over `[1 2 1]` for a
critically-damped second-order plant), represents the physical system's
dynamics; the feedback line closes the loop back into the `Sum` block.

## Solvers: fixed-step vs. variable-step

Simulink's solver settings mirror `ode45`/`ode15s` from Module 08
(Level 3) but chosen through model configuration rather than a function
call:

```
Simulation > Model Configuration Parameters > Solver
  Type: Variable-step   |  Fixed-step
  Solver: ode45 (Dormand-Prince)  |  ode15s (stiff)  |  ode4 (fixed-step RK4)
```

- **Variable-step** solvers (like `ode45`) adapt step size to control
  local error — appropriate for design and analysis simulations where
  you want the most accurate result for the least computation.
- **Fixed-step** solvers are *required* for code generation and
  real-time / hardware-in-the-loop simulation, because generated
  embedded code runs on a fixed control-loop period (e.g. a 10ms task on
  a microcontroller) — there's no notion of "shrink the step size" once
  code is deployed to real hardware ticking on a hardware timer.

Choosing a fixed step too large for a stiff or fast-dynamics model
produces the Simulink equivalent of the `ode45`-on-a-stiff-system
problem from Level 3 Module 08: numerical instability, not just
inaccuracy — the simulation can diverge to `Inf`/`NaN` outright.

## Subsystems for structure

Just as functions decompose a MATLAB script, **subsystems** decompose a
large Simulink model into named, reusable blocks with defined
input/output ports:

```
[Sensor Noise Model] --> [Kalman Filter Subsystem] --> [PID Controller Subsystem] --> [Plant Subsystem]
```

A **masked subsystem** additionally exposes a custom parameter dialog
(e.g. "Filter cutoff frequency (Hz)") so the subsystem behaves like a
configurable, reusable library block rather than exposing its internal
block structure to every user of the model — the Simulink analogue of a
function's parameter list hiding its implementation.

## Buses and data dictionaries

For models with many related signals (e.g. a vehicle model's speed,
RPM, throttle, brake all traveling together), a **Bus** groups them into
one structured signal, mirroring a MATLAB `struct`:

```matlab
busInfo = Simulink.Bus();
busInfo.Elements(1) = Simulink.BusElement;
busInfo.Elements(1).Name = 'Speed';
busInfo.Elements(2) = Simulink.BusElement;
busInfo.Elements(2).Name = 'RPM';
```

A **Data Dictionary** (`.sldd` file) centralizes shared parameter and
type definitions across multiple models — analogous to a shared header
file in C, ensuring every model referencing "Kp" or "vehicle mass" uses
the same authoritative value rather than each model hardcoding its own
copy.

## Simulating from and returning to MATLAB

```matlab
simOut = sim('myControlModel', 'StopTime', '10');

t = simOut.tout;
y = simOut.get('logsout').get('PlantOutput').Values.Data;

plot(t, y);
xlabel('Time (s)'); ylabel('Output');
```

`sim` runs a model programmatically (rather than clicking Run in the
GUI), returning a `Simulink.SimulationOutput` object — essential for
parameter sweeps or batch simulation runs driven from a MATLAB script:

```matlab
Kp_values = [1, 2, 5, 10];
results = cell(size(Kp_values));
for i = 1:numel(Kp_values)
    set_param('myControlModel/PID Controller', 'P', num2str(Kp_values(i)));
    simOut = sim('myControlModel', 'StopTime', '10');
    results{i} = simOut.get('logsout');
end
```

`set_param` programmatically changes a block parameter before each
simulation run — this pattern (loop over parameter values, run, collect
results) is how Simulink models integrate into a larger MATLAB-driven
design-space exploration or tuning workflow.

## Model verification: Model Advisor and requirements traceability

Before trusting a model for code generation, the **Model Advisor**
checks it against a configurable set of modeling-standard rules
(algebraic loops, unconnected ports, non-standard naming, solver
settings inconsistent with fixed-step code generation):

```matlab
ModelAdvisor.run('myControlModel');
```

An **algebraic loop** — a feedback path with no delay or integrator
anywhere in the loop, so the solver can't determine an output without
already knowing it — is a common structural defect the Advisor flags;
resolving it typically means inserting a `Unit Delay` or restructuring
the feedback path so the loop has a well-defined evaluation order.

## Practice

1. Sketch (in block-diagram-as-text form, as in the PID example above) a
   model for a thermostat: a `Step` setpoint, a `Sum` computing error
   against a `From Workspace` measured temperature, a `Relay` block
   turning a heater on/off with hysteresis, and a first-order `Transfer
   Fcn` representing room thermal dynamics.
2. Explain why a model destined for C code generation and deployment to
   a microcontroller must use a fixed-step solver, connecting this to
   the earlier explanation of variable-step solvers adapting their step
   size.
3. Describe how you'd structure a masked subsystem for a reusable
   "low-pass filter" block exposing only a cutoff-frequency parameter,
   and why hiding its internal `Transfer Fcn` block from the top-level
   model view matters for someone reusing it in a larger system.
4. Given the parameter-sweep script above, extend it to also record the
   settling time of `PlantOutput` for each `Kp` value (using `stepinfo`
   if available, or a manual threshold-crossing search), and describe
   how you would present the tradeoff between response speed and
   overshoot across the swept values.
