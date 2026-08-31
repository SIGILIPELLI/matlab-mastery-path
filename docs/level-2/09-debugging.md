# 09 · Debugging in MATLAB

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Debugger command names (`dbstop`, `dbstep`, `dbcont`...) and their
    behavior are documented MATLAB features, hand-traced against the
    MATLAB debugging documentation rather than exercised in a live
    debugger session.

Reading error messages carefully and knowing MATLAB's interactive
debugger turns "my script crashed" from a guessing game into a
systematic process. This module covers both.

## Reading a MATLAB error, top to bottom

```
Error using  *
Inner matrix dimensions must agree.

Error in computeForces (line 14)
    F = M * a;

Error in simulate (line 27)
    forces = computeForces(mass, accel);
```

Read the **stack from bottom to top** to trace how execution got there,
but read the **top error message first** — it names the actual failure
(`*` with mismatched dimensions). `Error in computeForces (line 14)`
means line 14 of `computeForces.m` is where the failing operation lives;
`Error in simulate (line 27)` is the calling line one level up. For a
deep stack, the *first* frame (closest to the actual `Error using ...`
line) is almost always where the real bug is — outer frames just show
how you got there.

A very common mistake is to start debugging at the *outermost* frame
(`simulate`) instead of the innermost (`computeForces`, line 14) — check
`size(M)` and `size(a)` at that innermost line first.

## `dbstop`: breakpoints from the command line

Setting a breakpoint in the Editor (clicking the margin) is one way;
`dbstop` is the command-line equivalent, useful in scripts, from a remote
session, or for conditional breaks the Editor UI doesn't expose as
easily:

```matlab
dbstop in computeForces at 14              % break at a specific line
dbstop in computeForces at 14 if any(isnan(a))   % conditional breakpoint
dbstop if error                             % break automatically at any uncaught error
dbstop if naninf                            % break the instant a NaN/Inf is produced
```

`dbstop if error` is one of the most useful blanket debugging commands —
rather than reading a stack trace after the fact, it drops you into the
debugger *at the moment of the crash*, with every local variable still in
scope to inspect, before MATLAB unwinds the stack.

Once stopped, the prompt changes to `K>>` and normal commands operate in
the paused function's own workspace:

```matlab
K>> a          % inspect the local variable directly
K>> size(M)
K>> M(1,:)
```

## Stepping commands

| Command | Shortcut | Effect |
|---|---|---|
| `dbstep` | F10 | Execute the current line, stop at the next |
| `dbstep in` | F11 | Step *into* a function call on the current line |
| `dbstep out` | Shift+F11 | Run until the current function returns |
| `dbcont` | F5 | Resume running until the next breakpoint (or completion) |
| `dbquit` | Shift+F5 | Abort debugging entirely, return to base workspace |

`dbstep` vs `dbstep in` is the distinction that matters most: if the
current line calls a function you already trust (e.g. a well-tested
utility), use `dbstep` to skip over it as one unit; if you suspect the
bug is *inside* that call, use `dbstep in` to follow execution into it.

## `dbstack` and `dbup`/`dbdown`

While paused, `dbstack` prints the current call stack; `dbup`/`dbdown`
move your inspection context one frame up or down the stack *without*
resuming execution — useful to check a caller's variables while still
stopped inside a deeply nested function:

```matlab
K>> dbstack
> In computeForces at 14
  In simulate at 27
K>> dbup            % now inspecting simulate's workspace
K>> mass            % see the value simulate passed in
K>> dbdown           % back to computeForces's workspace
```

## Removing breakpoints

```matlab
dbclear all          % remove every breakpoint everywhere
dbclear in computeForces          % remove all breakpoints in one file
dbstatus              % list every currently set breakpoint
```

Forgetting to `dbclear` a conditional breakpoint before handing off a
script is a common source of "why does this randomly stop for my
teammate" — `dbstatus` at the end of a debugging session is a good habit
before saving/committing code.

## The Editor's visual debugger

Everything above has an Editor equivalent: click the margin next to a
line number to toggle a breakpoint (a red dot appears); run the script
normally and execution pauses there with the line highlighted; the
toolbar's step/step-in/step-out/continue buttons mirror
`dbstep`/`dbstep in`/`dbstep out`/`dbcont`. The **Workspace** panel while
paused shows every variable in the current stopped scope, which is often
faster for casual inspection than typing variable names at `K>>`.

## Common bug categories and how to localize them

**Dimension mismatches** — the single most frequent MATLAB runtime
error. `dbstop if error`, then at the point of failure:

```matlab
K>> size(A), size(B)
```

almost always immediately reveals whether a transpose (`'`) is missing,
a loop index went one too far, or a function returned a different shape
than expected.

**Off-by-one indexing** — MATLAB is 1-indexed; a loop written with
0-indexed habits from another language either errors (`Index exceeds
array bounds`, when using index `0`) or silently reads/writes the wrong
element (`length(x)+1`, going one past the end and *growing* the array
instead of erroring — a much sneakier bug since MATLAB happily
auto-extends arrays on assignment).

```matlab
x = [10 20 30];
x(4) = 99;     % no error! silently extends x to [10 20 30 99]
x(0)           % this one *does* error: "Index must be a positive integer"
```

Print `size(x)` before and after suspicious loop bodies when values seem
to silently grow.

**Wrong function shadowing another** — from module 08's `which -all`:
if a variable is accidentally named the same as a function (`sum = 5;`
followed later by `sum(x)`), MATLAB resolves the *variable* first, and
subsequent calls to `sum(x)` error with `Index exceeds array bounds` or
similar, since `sum` is no longer callable as a function in that scope.
`which sum` (or `clear sum`) diagnoses this immediately.

**NaN propagation** — using `dbstop if naninf` catches the exact line
where a NaN or Inf is first produced, rather than debugging its
downstream symptoms (a garbage plot, a `polyfit` warning) several
function calls later.

## A worked debugging session

```matlab
function out = process(data)
    normalized = (data - mean(data)) / std(data);
    out = normalized.^2;
end

x = process([]);   % empty input by mistake somewhere upstream
```

Calling `process([])` produces `mean([])` = `NaN` (mean of empty is
NaN by MATLAB convention), silently propagating to `out` as all-NaN with
*no error at all* — a classic silent-failure bug that a stack trace won't
catch since nothing actually throws. `dbstop if naninf` set *before*
calling `process` would break the instant `mean([])` produces `NaN`,
right at the true source, rather than downstream where the all-NaN
result eventually causes a confusing plot or fit failure.

## Summary

- Read the error's **top line** for what failed, the **stack bottom-up**
  for how you got there; start inspecting at the innermost frame.
- `dbstop if error` (break at the crash site) and `dbstop if naninf`
  (break the instant a NaN/Inf appears) are the two highest-value blanket
  breakpoints for most numerical bugs.
- `dbstep` vs `dbstep in` vs `dbstep out` — skip, follow, or exit a call,
  respectively — and `dbup`/`dbdown` to inspect other frames without
  resuming.
- Dimension mismatches, off-by-one/auto-extending array indexing, and
  silent NaN propagation account for the large majority of real-world
  MATLAB bugs; each has a specific debugger habit (`size`, watch array
  length, `dbstop if naninf`) that finds it fast.
