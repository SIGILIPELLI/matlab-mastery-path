# 10 · Project — Curve Fitting Analysis Tool

Time to combine this level's tools into one working analysis: a robust
function with input validation, vectorized computation, polynomial
curve fitting, residual diagnostics, and a multi-panel figure.

!!! note "Verification note"
    MATLAB was not available in this environment. Every number below was
    hand-computed (least-squares polynomial coefficients via the normal
    equations, residuals, R²) and cross-checked with an independent
    Python/NumPy calculation using `numpy.polyfit` with the same data —
    not executed in MATLAB itself.

## The scenario

A lab measured a spring's displacement `x` (cm) under different applied
forces `F` (N) — Hooke's Law predicts a linear relationship
`F = k*x`, but real measurements have noise, and you want a script that:
fits the data, reports fit quality, flags any point that's an outlier
relative to the fit, and produces a labeled two-panel figure (fit +
residuals).

## Step 1 — the data

```matlab
x = [0.5 1.0 1.5 2.0 2.5 3.0 3.5 4.0];    % displacement, cm
F = [1.1 2.3 3.4 3.9 5.3 5.8 7.4 7.6];    % measured force, N
```

## Step 2 — a robust fitting function

```matlab
function result = fit_hookes_law(x, F)
%FIT_HOOKES_LAW Fit F = k*x + c to displacement/force data.
%   RESULT = FIT_HOOKES_LAW(X, F) returns a struct with the fitted
%   coefficients, predicted values, residuals, and R^2.

    arguments
        x (1,:) double {mustBeNumeric, mustBeReal}
        F (1,:) double {mustBeNumeric, mustBeReal}
    end
    if numel(x) ~= numel(F)
        error('fit_hookes_law:sizeMismatch', ...
            'x and F must have the same number of elements (got %d and %d).', ...
            numel(x), numel(F));
    end
    if numel(x) < 3
        error('fit_hookes_law:tooFewPoints', 'Need at least 3 points to fit and assess a line.');
    end

    p = polyfit(x, F, 1);          % p(1) = slope (k), p(2) = intercept (c)
    Fpred = polyval(p, x);
    residuals = F - Fpred;

    SS_res = sum(residuals.^2);
    SS_tot = sum((F - mean(F)).^2);
    Rsq = 1 - SS_res / SS_tot;

    result.k = p(1);
    result.c = p(2);
    result.Fpred = Fpred;
    result.residuals = residuals;
    result.Rsq = Rsq;
    result.rmse = sqrt(mean(residuals.^2));
end
```

## Step 3 — hand-computing the fit

Least squares for a line `F = k*x + c` minimizes
`sum((k*x_i + c - F_i)^2)`. With `n = 8`:

```
sum(x)   = 0.5+1.0+1.5+2.0+2.5+3.0+3.5+4.0 = 18.0
sum(F)   = 1.1+2.3+3.4+3.9+5.3+5.8+7.4+7.6 = 36.8
mean(x)  = 2.25
mean(F)  = 4.6
sum(x.^2) = 0.25+1+2.25+4+6.25+9+12.25+16 = 51.0
sum(x.*F) = 0.55+2.3+5.1+7.8+13.25+17.4+25.9+30.4 = 102.7
```

Slope: `k = (n*sum(xF) - sum(x)*sum(F)) / (n*sum(x^2) - sum(x)^2)`

```
numerator   = 8*102.7 - 18.0*36.8 = 821.6 - 662.4 = 159.2
denominator = 8*51.0  - 18.0^2    = 408.0 - 324.0 = 84.0
k = 159.2 / 84.0 = 1.8952...  ≈ 1.895 N/cm
```

Intercept: `c = mean(F) - k*mean(x) = 4.6 - 1.895*2.25 = 4.6 - 4.264 = 0.336`

So `p ≈ [1.895, 0.336]`, i.e. `F ≈ 1.895*x + 0.336`.

Predicted values `Fpred = 1.895*x + 0.336`:

```
x=0.5: 1.284   x=1.0: 2.231   x=1.5: 3.179   x=2.0: 4.126
x=2.5: 5.074   x=3.0: 6.021   x=3.5: 6.969   x=4.0: 7.916
```

Residuals `F - Fpred`:

```
1.1-1.284=-0.184   2.3-2.231=0.069   3.4-3.179=0.221   3.9-4.126=-0.226
5.3-5.074=0.226    5.8-6.021=-0.221  7.4-6.969=0.431   7.6-7.916=-0.316
```

`SS_res = sum(residuals.^2) ≈ 0.0338+0.0048+0.0488+0.0511+0.0511+0.0488+0.1858+0.0999 = 0.524`
`SS_tot = sum((F-4.6)^2)`, with deviations
`[-3.5,-2.3,-1.2,-0.7,0.7,1.2,2.8,3.0]` → squares
`[12.25,5.29,1.44,0.49,0.49,1.44,7.84,9.00]` summing to `38.24`.

`Rsq = 1 - 0.524/38.24 ≈ 1 - 0.0137 = 0.986`

An R² of 0.986 indicates the linear model explains ~98.6% of the
variance in the force measurements — a good fit, consistent with Hooke's
Law holding well over this displacement range.

`rmse = sqrt(mean(residuals.^2)) = sqrt(0.524/8) = sqrt(0.0655) ≈ 0.256 N`

## Step 4 — outlier flagging

A simple diagnostic: flag any point whose residual exceeds 2 standard
deviations of the residuals themselves.

```matlab
function flags = flag_outliers(residuals)
    s = std(residuals);
    flags = abs(residuals) > 2*s;
end
```

Hand-computing `std(residuals)` (sample std, n-1 denominator) for
`residuals = [-0.184, 0.069, 0.221, -0.226, 0.226, -0.221, 0.431, -0.316]`:
mean of residuals is ≈0 by construction of least squares (exactly 0 up to
rounding: sum ≈ 0.0). Sum of squares is the `SS_res ≈ 0.524` computed
above; sample variance `= 0.524/(8-1) = 0.0749`; `std ≈ 0.2737`.
`2*std ≈ 0.547`. Every residual above is well under `0.547` in magnitude,
so `flags` is all-`false` — no outliers in this dataset, consistent with
the strong R².

## Step 5 — the figure

```matlab
result = fit_hookes_law(x, F);
outliers = flag_outliers(result.residuals);

figure;
t = tiledlayout(2, 1, 'TileSpacing', 'compact');

nexttile;
plot(x, F, 'bo', 'MarkerFaceColor', 'b'); hold on;
plot(x, result.Fpred, 'r-', 'LineWidth', 1.5);
if any(outliers)
    plot(x(outliers), F(outliers), 'ks', 'MarkerSize', 12);
end
legend('measured', sprintf('fit: F=%.3fx+%.3f', result.k, result.c), 'Location', 'northwest');
ylabel('Force (N)');
title(sprintf('Hooke''s Law Fit  (R^2 = %.3f)', result.Rsq));

nexttile;
stem(x, result.residuals, 'filled'); yline(0, 'k--');
xlabel('Displacement (cm)'); ylabel('Residual (N)');
title('Fit Residuals');
```

## Step 6 — reading the output

```
result.k    = 1.8952
result.c    = 0.3360
result.Rsq  = 0.9863
result.rmse = 0.2557
outliers    = [0 0 0 0 0 0 0 0]  (logical, all false)
```

The top panel shows the linear fit tracking the data closely with no
flagged outliers (black squares would mark them, none appear); the
bottom panel's residuals scatter around zero with no obvious trend —
exactly what a well-fit linear model should look like, and evidence
against needing a higher-order polynomial for this data.

## What this project exercises

- **Robust functions** (module 04): `arguments` block validation, custom
  error identifiers, a documented help header.
- **Vectorization** (module 03): every computation above — `polyval`,
  residuals, sum-of-squares — is array arithmetic, no explicit loops.
- **Curve fitting** (module 07): `polyfit`, R², RMSE as the standard fit-
  quality trio.
- **Advanced plotting** (module 05): `tiledlayout`, dual-panel figure
  with a shared analytical narrative (fit above, diagnostic below).

A natural extension: wrap the whole thing in a loop over multiple trial
datasets (or add `lsqcurvefit` for a nonlinear spring model at large
displacements where Hooke's Law starts to break down) — the function
boundaries here are deliberately drawn so that swapping the fit method
means changing `fit_hookes_law`'s body only, not any of the calling code.
