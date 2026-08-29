# 09 · Basic Numerical Methods

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Every numeric result below was hand-traced through the algorithm's
    exact steps and cross-checked with an independent Python implementation
    of the same arithmetic — not executed in MATLAB itself. The underlying
    formulas (Newton's method, bisection, the trapezoidal rule) are standard
    and match MATLAB's documented behavior for these built-ins.

MATLAB's real strength is numerical computing — much of what a dedicated
numerical methods course covers by hand, MATLAB provides as a single
function call. This module covers both: the built-in function to reach for,
and the underlying method, so a MATLAB-less environment or an unusual case
doesn't leave you stuck.

## Root finding with `fzero`

`fzero` finds a value of `x` where a function crosses zero, starting from
an initial guess:

```matlab
f = @(x) x^2 - 2;          % looking for sqrt(2), where x^2 - 2 = 0
root = fzero(f, 1)
```

```matlab
root =

    1.4142
```

## Newton's method, by hand

`fzero` uses a more robust hybrid algorithm internally, but the classic
method it's related to is **Newton's method**: starting from a guess `x0`,
repeatedly step to `x - f(x)/f'(x)`, using the function's slope to jump
toward the root:

```matlab
x = 1;                 % initial guess
for i = 1:6
    fx  = x^2 - 2;
    fpx = 2*x;          % derivative of x^2 - 2 is 2x
    x = x - fx/fpx;
    fprintf('Iteration %d: x = %.10f\n', i, x)
end
```

```
Iteration 1: x = 1.5000000000
Iteration 2: x = 1.4166666667
Iteration 3: x = 1.4142156863
Iteration 4: x = 1.4142135624
Iteration 5: x = 1.4142135624
Iteration 6: x = 1.4142135624
```

Notice how fast this converges — from `1.5` to matching `sqrt(2)` to 10
decimal places in just 4 iterations. This rapid ("quadratic") convergence
is Newton's method's signature strength, but it requires knowing the
derivative `f'(x)` analytically, and can fail to converge (or converge to
the wrong root) from a poor starting guess — `fzero` is generally the safer
default in practice.

## Bisection method — slower but guaranteed

Bisection needs no derivative and is guaranteed to converge as long as the
function changes sign across the starting interval `[a, b]` — trading speed
for robustness:

```matlab
f = @(x) x^3 - x - 2;      % root known to lie between 1 and 2
a = 1; b = 2;
for i = 1:10
    c = (a + b) / 2;        % midpoint
    if f(a) * f(c) < 0
        b = c;               % root is in the left half
    else
        a = c;               % root is in the right half
    end
    fprintf('Iteration %d: c = %.6f, f(c) = %.6f\n', i, c, f(c))
end
```

```
Iteration 1: c = 1.500000, f(c) = -0.125000
Iteration 2: c = 1.750000, f(c) = 1.609375
Iteration 3: c = 1.625000, f(c) = 0.666016
Iteration 4: c = 1.562500, f(c) = 0.252197
Iteration 5: c = 1.531250, f(c) = 0.059113
Iteration 6: c = 1.515625, f(c) = -0.034054
Iteration 7: c = 1.523438, f(c) = 0.012250
Iteration 8: c = 1.519531, f(c) = -0.010971
Iteration 9: c = 1.521484, f(c) = 0.000622
Iteration 10: c = 1.520508, f(c) = -0.005179
```

Each iteration halves the search interval, so `c` converges toward the true
root (≈ `1.5214`) roughly one bit of precision at a time — much slower per
step than Newton's method, but it can't diverge or jump away from the root
the way Newton's method occasionally can from a bad starting guess.

## Numerical integration with `trapz`

`trapz` approximates the area under a curve using the trapezoidal rule —
given sampled `x` and `y` points, it sums the areas of the trapezoids
between consecutive samples:

```matlab
x = linspace(0, pi, 6);   % 6 points -> 5 intervals across [0, pi]
y = sin(x);
area = trapz(x, y)
```

```matlab
area =

    1.9338
```

The exact value of ∫₀^π sin(x) dx is `2`, so this 5-interval approximation
is off by about `0.066` — the trapezoidal rule slightly *underestimates*
the area here because straight line segments connecting points on a
concave-down curve (like the top half of a sine wave) cut inside the true
curve. More sample points close this gap:

```matlab
x = linspace(0, pi, 1001);    % 1000 intervals -- much finer
y = sin(x);
area = trapz(x, y)
```

```matlab
area =

    2.0000
```

## Function handles as arguments — the pattern behind `fzero`, `integral`, `fminbnd`

All of MATLAB's numerical solver functions (`fzero`, `integral`,
`fminbnd`, `ode45`, and more) share one calling convention: you pass a
**function handle** (`@functionName` or an anonymous `@(x) ...`), and the
solver calls it internally as many times as its algorithm needs:

```matlab
integral(@(x) x.^2, 0, 3)      % integrate x^2 from 0 to 3 analytically-accurate
```

```matlab
ans =

     9
```

(`∫₀³ x² dx = [x³/3]₀³ = 27/3 = 9` — exact, since `integral` uses adaptive
quadrature, not a fixed-step approximation like `trapz`.) Note the `.^`
inside the anonymous function: `x` will be a vector of sample points the
solver chooses internally, so the function handle must use element-wise
operations to evaluate correctly across all of them at once.

## Numerical methods cheat sheet

| Task | Function | Needs derivative? |
|------|----------|---------------------|
| Find a root near a guess | `fzero(f, x0)` | No |
| Find a root in an interval (by hand) | Bisection | No |
| Find a root fast (by hand) | Newton's method | Yes |
| Integrate sampled data | `trapz(x, y)` | — |
| Integrate a known function accurately | `integral(f, a, b)` | — |
| Minimize a function | `fminbnd(f, a, b)` | No |

## Exercise

Use `fzero` to find the root of `f(x) = cos(x) - x` starting from a guess
of `x0 = 0.5` (the answer should be near `0.7391`). Then implement bisection
by hand on the interval `[0, 1]` for the same function, running 8
iterations, and confirm your final midpoint is within `0.01` of `fzero`'s
answer. Finally, use `trapz` to approximate ∫₀² x² dx with only 3 sample
points (`linspace(0, 2, 3)`), compare it to the exact value (8/3 ≈ 2.667),
then repeat with 100 sample points and observe the approximation improve.
