# 02 · Code Generation (MATLAB Coder)

!!! note "Verification note"
    MATLAB Coder was not available in the environment used to write
    this page. Type inference rules, code generation restrictions, and
    generated-code structure below are documented, version-stable
    MATLAB Coder behaviors, hand-traced against the MathWorks
    documentation rather than run through the actual code generator.

MATLAB Coder translates a restricted subset of MATLAB code into standalone
C/C++ source — code that compiles and runs without MATLAB installed,
suitable for embedding in a larger application or deploying to hardware
that can't run the MATLAB runtime.

## Why not every MATLAB function is generatable

MATLAB is dynamically typed and permits things C fundamentally cannot
represent without extra constraints: variables that change size or type
mid-function, cell arrays of heterogeneous types, unbounded recursion
depth. MATLAB Coder requires **statically determinable types and
sizes** for every variable — the generated C code needs to know at
compile time how much memory a variable needs.

```matlab
function y = badForCodegen(x)
    if x > 0
        y = 5;            % y is a scalar double here
    else
        y = [1 2 3];       % y is a 1x3 double here — TYPE-UNSTABLE, codegen fails
    end
end
```

```matlab
function y = goodForCodegen(x)
    if x > 0
        y = 5;
    else
        y = -5;            % y is always a scalar double — type-stable
    end
end
```

Coder analyzes types and sizes through the whole function; a variable
whose class or dimensions can differ across branches like the first
example fails with a type-mismatch error at code-generation time, even
though the MATLAB function itself runs fine interpretively.

## Generating code: `codegen`

```matlab
function y = movingAverageFilter(x, windowSize) %#codegen
    n = length(x);
    y = zeros(1, n);
    for i = 1:n
        lo = max(1, i - windowSize + 1);
        y(i) = mean(x(lo:i));
    end
end
```

The `%#codegen` pragma is a hint to static analysis tools (and to
readers) that the function is intended for code generation, enabling
extra codegen-specific checks in the MATLAB Editor even before running
`codegen`.

```matlab
codegen movingAverageFilter -args {zeros(1,100), 5} -config:lib
```

`-args` supplies example inputs so Coder can infer concrete types and
sizes (`zeros(1,100)` tells it the first argument is a `1x100 double`;
`5` tells it the second is a scalar double) — Coder cannot infer types
from a bare function signature the way a compiled-language function
declaration would, since MATLAB source carries no type annotations
itself.

`-config:lib` generates a standalone C library (`.c`/`.h` plus a build
folder); other targets include `-config:exe` (standalone executable),
`-config:dll`, and `-config:mex` (a compiled MEX function callable from
MATLAB itself, primarily used to speed up a slow inner loop without
leaving the MATLAB environment).

## Variable-size data

```matlab
function y = flexibleSum(x) %#codegen
    y = sum(x);
end

codegen flexibleSum -args {coder.typeof(0, [1 Inf], [false true])}
```

`coder.typeof(0, [1 Inf], [false true])` declares the input as a `1xN`
double where the second dimension is variable (`Inf` upper bound, `true`
flag) — this lets one generated function handle inputs of different
lengths at runtime, at the cost of dynamic memory allocation in the
generated code (versus a fixed-size input, which Coder can allocate
statically, avoiding `malloc` calls entirely — often preferred for
hard-real-time embedded targets).

## MEX for acceleration inside MATLAB

```matlab
codegen mandelbrotPixel -args {0, 0, 1000} -config:mex
```

A generated MEX function replaces the original `.m` function at the
call site with a compiled binary honoring the same signature — useful
for accelerating one identified performance bottleneck (found via the
Profiler, Module 04) without rewriting the surrounding MATLAB code, at
the cost of losing MATLAB's flexible dynamic typing for that one
function going forward.

```matlab
tic; y1 = mandelbrotPixel(0, 0, 1000); toc;         % interpreted MATLAB
tic; y2 = mandelbrotPixel_mex(0, 0, 1000); toc;     % compiled MEX — often 10-100x faster for tight numeric loops
```

The speed-up is largest for code dominated by scalar loop overhead
(exactly what interpreted MATLAB is slowest at and compiled C is
fastest at) — a function already dominated by calls into optimized
built-ins (matrix multiply, FFT) sees far less benefit, since those
built-ins are already compiled code either way.

## Supported vs. unsupported constructs

Generally supported: numeric arrays, fixed-structure `struct`s,
`classdef` (value classes, with restrictions), most control flow,
most built-in math/matrix functions, fixed-size cell arrays (with
restrictions).

Generally unsupported or restricted: `eval` and other
runtime-code-generation constructs (fundamentally incompatible with
static compilation — there's no way to compile code that doesn't exist
yet), unbounded recursion (must have a statically provable bound or be
converted to iteration), most graphics/plotting functions (no display
exists on an embedded target), cell arrays of mixed, unbounded content,
`containers.Map` in some contexts (varies by target and release).

```matlab
function y = recursiveFactorial(n) %#codegen
    coder.inline('never');   % example of a codegen-specific directive
    if n <= 1
        y = 1;
    else
        y = n * recursiveFactorial(n - 1);
    end
end
```

Coder directives like `coder.inline`, `coder.varsize`, and
`coder.extrinsic` give fine-grained control over generation decisions
that have no equivalent meaning in interpreted MATLAB — they only affect
the code-generation process itself.

## `coder.extrinsic`: calling non-generatable functions

```matlab
function plotDuringSimulation(x, y) %#codegen
    coder.extrinsic('plot');
    if coder.target('MATLAB')
        plot(x, y);   % only actually called when running in MATLAB, not in generated code
    end
end
```

`coder.extrinsic` declares that a call (like `plot`, which has no
meaning without a display) should run in MATLAB itself rather than be
translated to C — useful for keeping diagnostic plotting or logging in
a function during development/testing without blocking code generation
for the rest of the function, though the call is skipped (or requires a
MATLAB runtime callback) in a truly standalone generated deployment.

## Verifying generated code against the original

```matlab
x = randn(1, 200);
yOriginal = movingAverageFilter(x, 5);
yGenerated = movingAverageFilter_mex(x, 5);

maxDiff = max(abs(yOriginal - yGenerated));
disp(maxDiff);   % should be exactly 0 or at floating-point-noise level
```

Bit-for-bit (or near-identical, allowing for legitimate floating-point
association differences) agreement between the interpreted and
generated versions on a battery of test inputs is the standard way to
validate that code generation preserved the intended behavior —
essentially running the Module 09 test suite against both versions and
comparing results.

## Practice

1. Take the type-unstable `badForCodegen` example and fix it two
   different ways (uniform scalar output vs. uniform fixed-size vector
   output), explaining the type-stability requirement in your own
   words.
2. Write a codegen-ready function computing a fixed-size 5-tap FIR
   filter and generate a `-config:mex` version; describe (since you
   can't run it here) what test you would run to confirm the MEX
   version matches the interpreted version numerically.
3. Explain why `coder.typeof` with an `Inf` upper bound produces
   generated code that allocates memory dynamically, and why a hard
   real-time embedded target would generally prefer a fixed maximum
   size instead, even if it wastes some memory in the common case.
4. Identify three MATLAB language features you've used earlier in this
   course (Levels 1-3) that would need to be rewritten or avoided
   before a function using them could pass through MATLAB Coder, and
   explain the underlying static-compilation constraint each one
   violates.
