# 07 · Curve Fitting & Interpolation

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    `polyfit`/`polyval`/`interp1`/`fit` behavior is documented, and the
    worked numeric examples below were hand-computed / cross-checked
    against equivalent NumPy (`numpy.polyfit`, `numpy.interp`) results
    rather than run in MATLAB itself.

Interpolation and curve fitting solve related but distinct problems:
**interpolation** finds a function that passes *exactly* through given
data points, useful for estimating values between measurements.
**Curve fitting** finds a function (usually simpler than the data,
e.g. a low-order polynomial) that fits the data *approximately*,
useful when the data is noisy and you want the underlying trend rather
than the noise itself.

## Polynomial fitting with `polyfit`/`polyval`

```matlab
x = [0 1 2 3 4 5];
y = [1.1 2.9 9.1 18.2 31.8 49.7];   % ~ noisy quadratic

p = polyfit(x, y, 2);    % fit degree-2 polynomial, least squares
% p = [coefficients highest-degree first]: roughly [2.0 -0.1 1.0]

yfit = polyval(p, x);
```

`polyfit(x, y, n)` solves the least-squares problem of finding the
degree-`n` polynomial coefficients `p` minimizing
`sum((polyval(p,x) - y).^2)`. Under the hood this is a linear least
squares solve on the Vandermonde matrix
`V = [x.^n, x.^(n-1), ..., x, ones(size(x))]`, i.e. `p = V \ y` — the
same backslash-operator machinery from Level 2's linear algebra module,
just applied to a specially constructed design matrix.

```matlab
xx = linspace(0, 5, 100);
plot(x, y, 'o', xx, polyval(p, xx), '-');
legend('data', 'fit');
```

### Choosing the polynomial degree

A degree too low **underfits** (misses real curvature); a degree too
high **overfits** (chases noise, oscillates wildly between data points —
Runge's phenomenon). A quick diagnostic: plot the residuals
`y - polyval(p, x)` — a good fit's residuals look like unstructured
noise; visible remaining curvature in the residuals means the degree is
too low.

```matlab
residuals = y - yfit;
figure; plot(x, residuals, 'o-'); yline(0);
```

`polyfit` also accepts an output for fit statistics:

```matlab
[p, S] = polyfit(x, y, 2);
[yfit, delta] = polyval(p, x, S);   % delta: estimated std error of the prediction
```

`delta` gives a rough per-point prediction uncertainty, useful for error
bars on the fitted curve without needing the full Curve Fitting Toolbox.

## `interp1`: 1D interpolation

```matlab
xdata = [0 1 2 3 4];
ydata = [0 0.84 0.91 0.14 -0.76];   % samples of sin(x)

xq = 0:0.1:4;                        % query points
ylin = interp1(xdata, ydata, xq);          % linear (default)
ycub = interp1(xdata, ydata, xq, 'spline'); % cubic spline
ynear = interp1(xdata, ydata, xq, 'nearest');
```

| Method | Behavior | When to use |
|---|---|---|
| `'nearest'` | Step function, snaps to closest known point | Categorical or genuinely discontinuous data |
| `'linear'` (default) | Straight line between consecutive points | Fast, safe default; matches data exactly, no overshoot |
| `'spline'` | Smooth cubic spline through all points | Data is known to come from a smooth underlying function |
| `'pchip'` | Shape-preserving piecewise cubic | Like spline but avoids overshoot near sharp local changes |

`'spline'` **can overshoot** between points if the data has a sharp
kink — because a global cubic spline enforces smooth first *and* second
derivatives everywhere, a kink forces nearby oscillation to compensate.
`'pchip'` sacrifices second-derivative smoothness to guarantee it never
overshoots the local data range — the safer default for physical data
that shouldn't produce impossible values (e.g. a fitted curve dipping
negative between two positive measurements).

```matlab
% Extrapolation: querying outside [min(xdata), max(xdata)]
yextrap = interp1(xdata, ydata, 5, 'linear', 'extrap');
% Without 'extrap', interp1 would return NaN for out-of-range queries —
% a deliberate default, since extrapolation is often not meaningful.
```

## 2D interpolation: `interp2`

```matlab
[X, Y] = meshgrid(1:5, 1:5);
Z = X.^2 + Y.^2;             % known values on a grid

[Xq, Yq] = meshgrid(1:0.25:5, 1:0.25:5);
Zq = interp2(X, Y, Z, Xq, Yq, 'spline');
```

Same method-name conventions as `interp1` (`'linear'`, `'spline'`,
`'nearest'`); the practical use case is upsampling a coarse measurement
grid (e.g. a sparse sensor array) to a fine grid for smooth contour or
surface plotting.

## Nonlinear curve fitting: beyond polynomials

Not every relationship is polynomial — exponential decay, saturating
growth, and sinusoids need `lsqcurvefit` (Optimization Toolbox) or the
Curve Fitting Toolbox's `fit` function, which fit an arbitrary
user-specified model by nonlinear least squares:

```matlab
% Exponential decay model: y = a*exp(-b*x) + c
modelfun = @(p, x) p(1)*exp(-p(2)*x) + p(3);

x = [0 1 2 3 4 5];
y = [10.2 6.4 4.1 2.8 2.1 1.9];

p0 = [10, 0.5, 1];   % initial guess: crucial for nonlinear fits to converge
p = lsqcurvefit(modelfun, p0, x, y);
```

Unlike `polyfit` (a *linear* least-squares problem with a closed-form
solution), `lsqcurvefit` iterates from an initial guess `p0` using a
nonlinear solver (Levenberg-Marquardt/trust-region by default). A poor
initial guess can converge to the wrong local minimum or fail to
converge at all — for the exponential-decay example above, guessing
`p0` roughly from the data (`a ≈ y(1)`, `c ≈ min(y)`, `b` from how fast
`y` drops) makes convergence far more reliable than guessing `[1 1 1]`
blindly.

If the Curve Fitting Toolbox is available, `fit` provides a higher-level
interface with named library models:

```matlab
ftype = fittype('a*exp(-b*x)+c');
[fitresult, gof] = fit(x', y', ftype, 'StartPoint', [10 0.5 1]);
% gof.rsquare, gof.rmse give fit-quality statistics directly
plot(fitresult, x, y);   % built-in fit-vs-data plot
```

## Evaluating fit quality

**R² (coefficient of determination)** — fraction of variance explained:

```matlab
SS_res = sum((y - yfit).^2);
SS_tot = sum((y - mean(y)).^2);
Rsq = 1 - SS_res/SS_tot;
```

`Rsq` close to 1 means the model explains most of the variance in `y`;
close to 0 means the model is no better than predicting the mean. Note
R² alone doesn't reveal overfitting — a degree-(n-1) polynomial through n
points always gives `Rsq = 1` while being a terrible predictive model
(interpolating noise exactly). Cross-validation (fit on a subset,
evaluate error on held-out points) is the correct check for that, not R²
on the training data alone.

**RMSE (root-mean-square error)** in the same units as `y`, more
interpretable than raw sum-of-squares:

```matlab
rmse = sqrt(mean((y - yfit).^2));
```

## Interpolation vs. fitting: a worked contrast

Given noisy data, `interp1` (exact pass-through) chases every noise
spike; `polyfit` (approximate) smooths through it:

```matlab
x = 0:0.5:10;
y_true = sin(x);
y_noisy = y_true + 0.15*sin(37*x);   % deterministic "noise" for reproducibility here

y_interp = interp1(x, y_noisy, x, 'spline');   % passes through every noisy point
p = polyfit(x, y_noisy, 5);                     % smooths across noise
y_fit = polyval(p, x);
```

Plotting all three (`y_noisy`, `y_interp`, `y_fit`) against `y_true`
makes the distinction visible: `y_interp` reproduces every wiggle in
`y_noisy` exactly (since interpolation must pass through every point by
definition), while `y_fit` averages the wiggles out and tracks `y_true`
more closely — the correct choice depends entirely on whether the
"noise" in your real data is actual measurement error (fit) or real
signal you need to preserve exactly (interpolate).

## How It Actually Works

`polyfit` solves a **linear least-squares problem** even though the fitted
curve is nonlinear in `x` — the trick is that a polynomial `a0 + a1*x +
a2*x^2 + ...` is *linear in its coefficients* `a0, a1, a2, ...`. `polyfit`
builds a Vandermonde matrix `V` whose columns are `1, x, x^2, ..., x^n`,
then solves `V \ y` for the coefficient vector — the same backslash
dispatch covered in Module 01, which for this rectangular, overdetermined
system resolves to a QR-based least-squares solve rather than an exact
solve. High-degree polynomial fits are numerically fragile for exactly
this reason: as the degree grows, columns of the Vandermonde matrix
(`x^5`, `x^6`, `x^7`, ...) become nearly linearly dependent for `x` values
confined to a small range, driving the condition number of `V` up sharply
and amplifying rounding error in the fitted coefficients — the practical
symptom is wild oscillation near the data boundaries (Runge's phenomenon),
not a bug in `polyfit` but a direct consequence of an ill-conditioned
linear system.

`interp1`'s linear mode is piecewise: for a query point `xq`, MATLAB
performs a **binary search** over the sorted breakpoints to locate the
bracketing interval in `O(log n)` time, then evaluates the line segment
between those two neighboring points directly — no global fit is
computed, which is why interpolation always passes exactly through every
data point while a degree-`n` polynomial fit generally does not (fitting
minimizes total squared *residual* across all points, not zero-error at
each one). Spline interpolation (`'spline'`) instead solves a global
tridiagonal linear system for the piecewise-cubic segment coefficients
that enforce matching first and second derivatives at each breakpoint —
smoother, but meaning a change to one data point can, in principle,
perturb the spline's shape across the whole domain, unlike the strictly
local `'linear'` mode.

*Note: cross-checked conceptually against NumPy/SciPy's equivalent
`polyfit`/`interp1d` implementations, which share the same
Vandermonde/LAPACK least-squares approach; not executed in MATLAB itself.*

## Summary

| Task | Function |
|---|---|
| Fit a polynomial by least squares | `polyfit` / `polyval` |
| Interpolate exactly through 1D data | `interp1` |
| Interpolate exactly through 2D grid data | `interp2` |
| Fit an arbitrary nonlinear model | `lsqcurvefit` (Optimization Toolbox) or `fit` (Curve Fitting Toolbox) |
| Judge fit quality | R², RMSE, and visual residual inspection |

The recurring judgment call is degree/method choice: too little
flexibility underfits, too much interpolates noise as if it were signal
— always look at residuals and, where possible, evaluate on data the fit
didn't see.
