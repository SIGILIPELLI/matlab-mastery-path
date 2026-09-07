# 08 · Working with Toolboxes

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Toolbox names, licensing model, and representative function names
    reflect MathWorks' published product documentation, hand-traced
    rather than run against a live MATLAB toolbox installation.

MATLAB ships a relatively small core language; almost all
domain-specific capability (signal processing, statistics, optimization,
control design, deep learning...) lives in separately licensed
**toolboxes**. This module is about navigating that ecosystem: figuring
out what's installed, what a toolbox actually adds, and how to write code
that degrades gracefully when a toolbox isn't available.

## Checking what's installed

```matlab
ver                     % lists every installed toolbox and its version
license('test', 'Signal_Toolbox')     % returns 1 if a Signal Processing Toolbox license exists
license('test', 'Optimization_Toolbox')
```

`ver` alone is the quickest sanity check when reading someone else's
script that calls an unfamiliar function — if the function isn't in a
toolbox `ver` lists, it's either a core MATLAB function, a custom
function on the path, or genuinely missing on this machine.

`which functionname` reports the file path a function resolves to, which
also reveals whether it's a toolbox function, a user file shadowing a
toolbox function of the same name (a classic and confusing bug), or
undefined:

```matlab
which fft
% /MATLAB/toolbox/signal/signal/fft.m   -- or the core MATLAB one, depending on install
which -all fft
% lists every fft.m on the path, in shadowing-priority order — useful when
% two toolboxes (or a user script) define the same function name
```

## Major toolboxes at a glance

| Toolbox | Adds |
|---|---|
| **Signal Processing Toolbox** | Filter design (`butter`, `fir1`), spectral analysis (`pwelch`, `spectrogram`), windowing |
| **Statistics and Machine Learning Toolbox** | Distributions (`normpdf`, `fitdist`), hypothesis tests (`ttest`), classifiers (`fitcsvm`, `fitctree`) |
| **Optimization Toolbox** | `fmincon`, `linprog`, `lsqcurvefit`, `intlinprog` |
| **Control System Toolbox** | Transfer functions (`tf`), state-space models (`ss`), `bode`, `step`, `pid` |
| **Curve Fitting Toolbox** | `fit`, `cftool` GUI, spline/smoothing-spline models |
| **Image Processing Toolbox** | `imread`/`imshow` extensions, filtering, segmentation, morphology |
| **Parallel Computing Toolbox** | `parfor`, `gpuArray`, `spmd`, distributed arrays |
| **Deep Learning Toolbox** | `trainNetwork`, layer objects, pretrained networks |
| **Simulink** (product, not strictly a toolbox) | Block-diagram modeling, covered in module 06 |

Base MATLAB alone (no toolboxes) already includes linear algebra
(`\`, `eig`, `svd`), basic numerics (`ode45`, `fzero`, `integral`),
and 2D/3D plotting — the toolboxes above are additive specializations,
not replacements for core functionality.

## Same problem, with and without a toolbox

A concrete illustration: fitting a distribution to data.

**Without Statistics Toolbox** (manual, core MATLAB only):

```matlab
data = [4.2 5.1 3.9 4.8 5.5 4.0 4.6];
mu = mean(data);
sigma = std(data);
pdf_manual = @(x) exp(-(x-mu).^2 / (2*sigma^2)) / (sigma*sqrt(2*pi));
```

**With Statistics and Machine Learning Toolbox:**

```matlab
pd = fitdist(data', 'Normal');    % returns a probability distribution object
x = linspace(min(data), max(data), 100);
y = pdf(pd, x);                    % pd.mu, pd.sigma also directly accessible
[h, p] = kstest((data-pd.mu)/pd.sigma);   % goodness-of-fit test, not available manually without more code
```

The toolbox version isn't doing fundamentally different math for the
simple normal-fit case — it's providing goodness-of-fit tests,
distribution objects that generalize to dozens of other distributions
(`'Weibull'`, `'Gamma'`, `'Poisson'`, ...) via the same `fitdist`/`pdf`
API, and a consistent object model that scales to problems the manual
version doesn't cover at all (censored data, multivariate distributions).

## Writing toolbox-optional code

A function that has a toolbox-accelerated path but a working core-MATLAB
fallback is more portable across teams/machines with different licenses:

```matlab
function y = smooth_signal(x, windowSize)
    if license('test', 'Signal_Toolbox') && exist('movmean', 'file') == 0
        % (illustrative: movmean is actually core MATLAB since R2016a;
        % shown here as the general pattern for toolbox-guarded code)
    end

    if exist('sgolayfilt', 'file') == 2   % Signal Processing Toolbox function
        y = sgolayfilt(x, 3, 11);         % Savitzky-Golay: better edge behavior
    else
        y = movmean(x, 5);                % core-MATLAB fallback: simpler, still reasonable
    end
end
```

`exist(name, 'file')` returning `2` (an M-file exists) or `3`
(a MEX-file exists) rather than `0` (nothing found) is the standard guard
for "is this specific function available," which is more precise than
checking the license alone — a license can be present but the toolbox's
functions not on the current path, or vice versa in some managed/HPC
environments.

## Toolbox function naming conventions worth knowing

Many toolboxes follow the `fit<Something>` / `predict` object pattern
(especially Statistics and Machine Learning, Deep Learning):

```matlab
mdl = fitlm(X, y);              % fit a linear model, returns an object
ypred = predict(mdl, Xnew);      % use the object to predict on new data
mdl.Coefficients                 % inspect fitted coefficients as a table
plot(mdl);                       % many model objects support direct plotting
```

Once you recognize `fit___` returns an object with `.predict()`,
`.plot()`, and property access, that pattern transfers directly to
`fitcsvm`, `fitctree`, `fitglm`, and dozens of other Statistics/ML
Toolbox functions — a big part of "learning MATLAB toolboxes" is
recognizing these repeated object-API shapes rather than memorizing every
function individually.

## Add-On Explorer and installing toolboxes

Toolboxes are typically installed through MATLAB's **Add-On Explorer**
(Home tab → Add-Ons → Get Add-Ons) or bundled at install time by a
system administrator for a site license. Free community "Add-Ons" (as
distinct from licensed MathWorks toolboxes) are also distributed this way
— useful utility functions shared by other users, installed the same way
but without a license check, and worth checking `exist`/`which` on the
same as any other function since naming collisions with your own code are
possible.

## How It Actually Works

A MATLAB "toolbox" is, mechanically, a licensed collection of `.m`
functions, `.mex` compiled binaries (native machine code callable from
MATLAB via a defined C ABI), and class definitions installed into
MATLAB's search path, gated by a license-checkout mechanism — calling a
toolbox function triggers a license check against the toolbox's license
token *before* the function body runs; if no license is available (all
seats in use, or the toolbox isn't installed/licensed at all), MATLAB
throws an error rather than silently falling back to a base-MATLAB
implementation. This is why the same script can work on one machine and
fail with an "Undefined function" or licensing error on another — the
function genuinely doesn't exist (or isn't licensed) in that installation,
distinct from a typo or missing file.

MEX functions matter for performance-sensitive toolbox code specifically
because they bypass the interpreter entirely: a `.mex` file is compiled
C/C++/Fortran linked against MATLAB's C API, invoked from MATLAB script
code exactly like an ordinary function call but executing as native
machine code with no per-statement interpretation overhead — many
toolbox functions that need to be fast (image processing kernels,
optimization inner loops) are implemented this way rather than as plain
`.m` files, which is part of why they can outperform an equivalent
hand-written MATLAB loop even beyond what vectorization alone would
achieve.

`ver` and `license('test', 'toolbox_name')` work by reading MATLAB's
installed-products registry and cross-referencing it against the local
license file, respectively — two genuinely different checks, since a
toolbox can be installed but not currently licensed (e.g., an expired
network license), or licensed but not installed.

*Note: based on MathWorks' documented toolbox/MEX/licensing architecture;
not executed in a real MATLAB installation, which is unavailable here.*

## Summary

- `ver` and `license('test', 'ToolboxName')` tell you what's actually
  available on a given machine; `which -all` diagnoses which specific
  file a function name resolves to.
- Toolboxes generally *extend* rather than *replace* core MATLAB — most
  can be worked around manually with more code, at the cost of losing the
  toolbox's tested edge cases, performance, and consistent object API.
- Recognize the common `fit___()` → object → `.predict()`/`.plot()`
  pattern; it repeats across Statistics, Machine Learning, and Curve
  Fitting toolboxes and generalizes your intuition across all of them.
- Guard toolbox-dependent code paths with `exist(name, 'file')` so
  scripts fail with a clear message (or gracefully fall back) rather than
  an opaque "undefined function" error on a machine without the license.
