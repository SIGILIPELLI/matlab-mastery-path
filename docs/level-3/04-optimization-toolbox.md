# 04 · Optimization Toolbox Basics

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Optimization Toolbox function behavior (`fminunc`, `fmincon`,
    `linprog`, `lsqcurvefit`) is hand-traced against documented MATLAB
    semantics and cross-checked against SciPy's `optimize` module (which
    implements comparable algorithms) rather than run in MATLAB itself.

Optimization means finding the input that minimizes (or maximizes) some
function. Level 1's `\` already solved a special case — linear
least-squares — implicitly; this module makes the general problem
explicit and gives you the right tool for each shape of it.

## The vocabulary

- **Objective function**: what you're minimizing, e.g. cost, error,
  negative profit (maximizing `f` is minimizing `-f`).
- **Decision variables**: the inputs you're allowed to adjust.
- **Constraints**: equalities/inequalities the solution must satisfy
  (`x >= 0`, `Ax <= b`).
- **Unconstrained vs. constrained**: whether any constraints apply at
  all — a materially different, generally easier problem when they
  don't.

## Unconstrained: `fminunc`

Minimizing a smooth scalar function of one or more variables with no
constraints:

```matlab
f = @(x) (x(1)-3)^2 + (x(2)+1)^2;   % minimum at (3, -1), value 0
x0 = [0, 0];
opts = optimoptions('fminunc', 'Display', 'off');
[xmin, fval] = fminunc(f, x0, opts);
% xmin ≈ [3.0000, -1.0000], fval ≈ 0
```

`fminunc` uses gradient-based methods (quasi-Newton by default) and
converges quickly on smooth functions, but — like any gradient-based
method — can settle in a local minimum on a non-convex objective, so a
result should always be sanity-checked against a plot or a few different
starting points `x0` when the function's shape isn't already known to be
convex.

`fminsearch` is the derivative-free sibling (Nelder–Mead simplex), useful
when `f` is not differentiable or too expensive/noisy for gradient
estimates, at the cost of typically needing more function evaluations to
converge.

## Constrained: `fmincon`

Real problems usually have limits — a mix must sum to 1, a dimension
can't be negative, a budget can't be exceeded. `fmincon` minimizes
subject to linear and/or nonlinear constraints:

```matlab
f = @(x) x(1)^2 + x(2)^2;    % minimize distance from origin
A = [1 1];  b = 2;            % linear inequality: x1 + x2 <= 2
lb = [0, 0];                   % lower bounds: x1, x2 >= 0
ub = [];                        % no upper bounds

x0 = [1, 1];
opts = optimoptions('fmincon', 'Display', 'off');
[xmin, fval] = fmincon(f, x0, A, b, [], [], lb, ub, [], opts);
% xmin ≈ [0, 0], fval ≈ 0 — the unconstrained minimum already satisfies
% x1+x2<=2, so the constraint isn't binding here
```

The full signature is
`fmincon(f, x0, A, b, Aeq, beq, lb, ub, nonlcon, options)` — linear
inequalities (`A`, `b`), linear equalities (`Aeq`, `beq`), simple bounds
(`lb`, `ub`), and a function handle `nonlcon` returning nonlinear
inequality/equality constraint values for anything the linear forms can't
express. Pass `[]` for any piece that doesn't apply, exactly like
optional positional arguments elsewhere in MATLAB.

## Linear programming: `linprog`

When both the objective and every constraint are linear, `linprog` is
faster and more reliable than `fmincon` because it exploits that
structure directly rather than treating the problem as general nonlinear:

```matlab
f = [-5; -4];          % maximize 5x1 + 4x2  <=>  minimize -5x1 - 4x2
A = [6 4; 1 2];
b = [24; 6];
lb = [0; 0];

x = linprog(f, A, b, [], [], lb);
% x ≈ [3; 1.5], objective 5(3)+4(1.5) = 21
```

`linprog`'s `f` is the cost *vector* for `f'*x` (not a function handle) —
a classic gotcha for anyone moving from `fmincon`'s function-handle
convention. Use `linprog` whenever a problem is genuinely linear;
reaching for `fmincon` on a linear problem still works but gives up
guarantees and speed the linear solver provides for free.

## Curve fitting as optimization: `lsqcurvefit`

Level 2's `polyfit`/`fit` solved *linear-in-parameters* curve fitting.
Fitting a model that is nonlinear in its parameters — an exponential
decay, a sigmoid — needs `lsqcurvefit`, which minimizes the sum of
squared residuals via nonlinear least squares:

```matlab
model = @(p, t) p(1) * exp(-p(2) * t);   % p(1)=amplitude, p(2)=decay rate
t = (0:0.1:5)';
trueP = [10, 0.8];
y = model(trueP, t) + 0.2*randn(size(t));   % noisy synthetic data

p0 = [1, 1];    % initial guess
opts = optimoptions('lsqcurvefit', 'Display', 'off');
pFit = lsqcurvefit(model, p0, t, y, [], [], opts);
% pFit close to [10, 0.8], recovered from the noisy data
```

As with `fminunc`, the initial guess `p0` matters: a poor guess on a
genuinely nonlinear model can converge to the wrong local optimum or fail
to converge at all, so a sanity-check plot of `model(pFit, t)` against
the data is standard practice, not an optional extra.

## Choosing the right solver

| Problem shape | Solver |
|---|---|
| Smooth, unconstrained | `fminunc` (gradient) or `fminsearch` (derivative-free) |
| Smooth, with linear/nonlinear constraints | `fmincon` |
| Fully linear objective and constraints | `linprog` |
| Fit a nonlinear model to data (least squares) | `lsqcurvefit` / `lsqnonlin` |
| Integer or mixed-integer decision variables | `intlinprog` |

Picking the most specific solver that matches a problem's structure
(linear → `linprog`, not `fmincon`) isn't just style — the specialized
solvers are faster, more numerically robust, and give convergence
guarantees the general nonlinear solvers can't.

## Reading solver output honestly

Every solver here returns (or accepts as an extra output) an `exitflag`
and can report `output.iterations`, `output.funcCount`, and a message.
`exitflag > 0` generally means "converged to the algorithm's stopping
criteria," not "definitely the true global optimum" — for non-convex
problems that distinction matters, and multi-start (`fminunc`/`fmincon`
from several `x0` values, keeping the best result) is the standard
mitigation when a wrong local minimum is a real risk.

## Summary

- Optimization problems are characterized by their objective, decision
  variables, and constraints; matching the solver to the problem's actual
  structure (linear vs. nonlinear, constrained vs. not) gives faster and
  more trustworthy results than defaulting to the most general solver.
- `fminunc`/`fminsearch` — unconstrained; `fmincon` — constrained
  nonlinear; `linprog` — fully linear; `lsqcurvefit` — nonlinear curve
  fitting via least squares.
- Initial guesses matter for every gradient-based and nonlinear-fit
  solver; check convergence (`exitflag`, a residual/fit plot) rather than
  trusting a returned answer blindly, and consider multi-start for
  non-convex objectives.
