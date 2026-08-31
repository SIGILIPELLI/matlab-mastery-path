# 08 · Advanced Numerical Methods

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. Every numeric example below was hand-traced step by step
    against documented algorithm behavior and cross-checked with
    equivalent NumPy/SciPy computations where feasible, rather than
    executed in MATLAB itself. Solver choices and tolerances follow
    MATLAB's documented defaults.

Level 1 covered basic linear algebra and Level 2 covered curve fitting.
This module goes deeper into MATLAB's numerical solvers: root-finding,
ODE integration, optimization, and numerical integration — the
workhorses behind simulation and engineering computation.

## Root-finding: `fzero`

```matlab
f = @(x) x^3 - 2*x - 5;
root = fzero(f, 2);        % initial guess near 2
disp(root);                % approximately 2.0946
```

`fzero` finds where a scalar function crosses zero, starting from either
a single guess (it searches outward for a sign change) or a bracketing
interval `[a, b]` where `f(a)` and `f(b)` have opposite signs
(guaranteed to find a root between them by bisection-like methods):

```matlab
root = fzero(f, [2, 3]);   % bracket: f(2) = -1, f(3) = 16, sign change guaranteed
```

Hand-trace: `f(2) = 8 - 4 - 5 = -1`, `f(3) = 27 - 6 - 5 = 16` — opposite
signs confirm a root exists in `[2, 3]` by the intermediate value
theorem, and `fzero`'s hybrid bisection/secant/inverse-quadratic
algorithm converges to it reliably.

## Polynomial roots: `roots`

```matlab
p = [1 0 -2 -5];            % represents x^3 + 0*x^2 - 2x - 5
r = roots(p);
```

For polynomials specifically, `roots` uses the companion matrix's
eigenvalues — exact for the algebraic problem (up to floating-point
precision), no initial guess required, and returns *all* roots
(including complex ones) at once, unlike `fzero`'s single real root per
call.

## Numerical integration: `integral`

```matlab
f = @(x) exp(-x.^2);
area = integral(f, 0, 1);
disp(area);                 % approximately 0.7468
```

`integral` uses adaptive quadrature (Gauss-Kronrod by default) —
recursively subdividing the interval, using tighter subdivisions where
the function changes rapidly and coarser ones where it's smooth, until
error estimates fall under a tolerance (default `1e-6` relative,
`1e-10` absolute).

```matlab
area2D = integral2(@(x,y) x.*y, 0, 1, 0, 1);   % double integral over unit square
disp(area2D);   % analytically = 0.25, matches (integral of x*y over [0,1]x[0,1])
```

Hand-check: `∫∫ xy dx dy` over `[0,1]²` separates as `(∫x dx)(∫y dy) =
0.5 × 0.5 = 0.25` — matches.

```matlab
area3D = integral3(@(x,y,z) x + y + z, 0, 1, 0, 1, 0, 1);
% analytically: 3 * (1/2) = 1.5 by symmetry (each term integrates to 0.5 over the unit cube)
```

## Solving ODEs: `ode45` and friends

```matlab
% dy/dt = -2y, y(0) = 1  ->  analytical solution y(t) = e^(-2t)
f = @(t, y) -2*y;
[t, y] = ode45(f, [0 5], 1);

analytical = exp(-2*t);
maxError = max(abs(y - analytical));
disp(maxError);    % very small, e.g. on the order of 1e-6, within ode45's default tolerance
```

`ode45` is a variable-step, 4th/5th-order Runge-Kutta solver — MATLAB's
general-purpose default for non-stiff ODEs, automatically shrinking the
step size where the solution changes quickly and growing it where the
solution is smooth, while controlling local error to a default relative
tolerance of `1e-3`.

```matlab
% system of ODEs: predator-prey (Lotka-Volterra)
function dydt = lotkaVolterra(t, y)
    alpha = 1.1; beta = 0.4; delta = 0.1; gamma = 0.4;
    prey = y(1); predator = y(2);
    dydt = [alpha*prey - beta*prey*predator;
            delta*prey*predator - gamma*predator];
end

[t, y] = ode45(@lotkaVolterra, [0 50], [10; 5]);
plot(t, y(:,1), t, y(:,2));
legend('Prey', 'Predator');
```

Systems of ODEs pass a column vector `y` and return a column vector
`dydt` of the same size — `ode45` handles vector-valued states
transparently.

### Stiff systems: `ode15s`

```matlab
% a stiff system has widely separated time scales (fast transient + slow decay)
function dydt = stiffSystem(t, y)
    dydt = [-1000*y(1) + y(2);
            -y(2)];
end

[t, y] = ode15s(@stiffSystem, [0 5], [1; 1]);   % ode45 would need impractically tiny steps
```

`ode45`'s explicit method must take steps small enough to track the
fastest-decaying component (`-1000*y(1)`) even after it's numerically
negligible, since explicit methods are only *conditionally stable*.
`ode15s` (a variable-order, implicit multistep solver) remains stable
with much larger steps for such systems — a rule of thumb is to try
`ode45` first and switch to `ode15s` if it's extremely slow or the
system is known to have fast and slow dynamics simultaneously.

### Setting tolerances and options

```matlab
opts = odeset('RelTol', 1e-8, 'AbsTol', 1e-10, 'MaxStep', 0.01);
[t, y] = ode45(@lotkaVolterra, [0 50], [10; 5], opts);
```

Tighter tolerances cost more function evaluations but improve accuracy —
appropriate when the analytical comparison above shows error too large
for the application's needs.

## Numerical optimization: `fminsearch` and `fminunc`

```matlab
% minimize the Rosenbrock function: classic optimization test case
rosenbrock = @(x) 100*(x(2) - x(1)^2)^2 + (1 - x(1))^2;
x0 = [-1.2, 1];
xmin = fminsearch(rosenbrock, x0);
disp(xmin);    % converges near [1, 1], the known global minimum where f = 0
```

`fminsearch` uses the derivative-free Nelder-Mead simplex method — no
gradient needed, robust for non-smooth or noisy objectives, but slower
to converge and can stall on high-dimensional problems.

```matlab
opts = optimoptions('fminunc', 'Algorithm', 'quasi-newton', 'Display', 'iter');
xmin2 = fminunc(rosenbrock, x0, opts);
```

`fminunc` (Optimization Toolbox) uses gradient-based methods
(quasi-Newton by default), converging faster when the objective is
smooth — MATLAB estimates the gradient numerically if an analytical
gradient isn't supplied, or you can provide one via `'SpecifyObjectiveGradient'`
for speed and accuracy.

### Constrained optimization: `fmincon`

```matlab
objective = @(x) x(1)^2 + x(2)^2;
A = [1, 1]; b = 2;                  % x(1) + x(2) <= 2
lb = [0, 0];                        % x >= 0
x0 = [1, 1];
xopt = fmincon(objective, x0, A, b, [], [], lb, []);
```

`fmincon` handles linear inequality (`A`,`b`), linear equality
(`Aeq`,`beq`), bounds (`lb`,`ub`), and nonlinear constraint functions —
the general-purpose constrained nonlinear solver.

## Interpolation for numerical work: `interp1`, `griddedInterpolant`

```matlab
x = 0:10;
y = sin(x);
xq = 0:0.1:10;
yq = interp1(x, y, xq, 'spline');   % smooth interpolation between known samples
```

```matlab
F = griddedInterpolant(x, y, 'pchip');   % reusable interpolant object, faster for repeated queries
yq2 = F(xq);
```

`griddedInterpolant` precomputes structure once and is preferred over
repeated `interp1` calls on the same grid inside a loop or solver
callback.

## Choosing a method

| Task | Function |
|---|---|
| Single-variable root, have a guess or bracket | `fzero` |
| All roots of a polynomial | `roots` |
| Definite integral, 1D/2D/3D | `integral`, `integral2`, `integral3` |
| Non-stiff ODE / ODE system | `ode45` |
| Stiff ODE | `ode15s` |
| Unconstrained minimization, no gradient needed | `fminsearch` |
| Unconstrained minimization, smooth, faster convergence | `fminunc` |
| Constrained minimization | `fmincon` |
| Repeated interpolation queries on fixed data | `griddedInterpolant` |

## Practice

1. Use `fzero` to find all three real roots of `x^3 - 6x^2 + 11x - 6`
   (hint: bracket around 1, 2, and 3), and verify with `roots` on the
   coefficient vector `[1 -6 11 -6]`.
2. Solve the damped harmonic oscillator `y'' + 0.5y' + 4y = 0`, `y(0) =
   1`, `y'(0) = 0` by converting to a first-order system and using
   `ode45`; hand-check the early-time behavior against the known
   underdamped analytical solution.
3. Minimize `f(x,y) = (x-3)^2 + (y+1)^2 + 5` with `fminsearch` from
   `x0 = [0, 0]` and confirm the result matches the obvious analytical
   minimum at `(3, -1)`.
4. Explain, using the stiff-system example, why `ode45`'s step size is
   forced small even long after the fast transient has decayed to
   numerical zero, and why `ode15s` doesn't have that limitation.
