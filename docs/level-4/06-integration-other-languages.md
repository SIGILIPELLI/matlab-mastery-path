---
description: "Integration with Other Languages — Real systems rarely live entirely in one language. MATLAB provides several bridges to call other languages from MATLAB…"
---

# 06 · Integration with Other Languages

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. Interoperability mechanisms (Python interface, MEX, .NET/Java
    bindings) described below are documented, version-stable MATLAB
    features, hand-traced against the MathWorks documentation rather
    than exercised against real installations.

Real systems rarely live entirely in one language. MATLAB provides
several bridges to call other languages from MATLAB, and to call MATLAB
from other languages — essential when a project needs MATLAB's
numerical strengths alongside libraries or systems that only exist
elsewhere.

## Calling Python from MATLAB

```matlab
np = py.importlib.import_module('numpy');
result = np.array([1, 2, 3, 4]);

pyArray = py.list({1, 2, 3, 4});
pySum = py.sum(pyArray);
disp(double(pySum));   % convert Python numeric type back to a MATLAB double
```

MATLAB's Python interface (built on a Python installation MATLAB is
configured to use via `pyenv`) lets you call arbitrary Python functions
and libraries directly, converting between MATLAB and Python types at
the boundary — Python lists, dicts, and NumPy arrays map to MATLAB
equivalents, though not always automatically; explicit conversion
(`double(...)`, `cell(...)`) is often needed at the boundary.

```matlab
pyenv('Version', '/usr/bin/python3.10');   % point MATLAB at a specific Python install

pd = py.importlib.import_module('pandas');
df = pd.read_csv('data.csv');
```

This is the natural route for reaching a Python-only library (deep
learning frameworks, specialized scientific packages) without leaving
MATLAB for the rest of a workflow that's otherwise MATLAB-based.

## Calling MATLAB from Python

```python
import matlab.engine
eng = matlab.engine.start_matlab()

result = eng.sqrt(16.0)
print(result)  # 4.0

x = matlab.double([1, 2, 3, 4])
y = eng.sum(x)
print(y)

eng.quit()
```

The MATLAB Engine API for Python starts (or connects to) a MATLAB
session from a Python process and calls MATLAB functions as if they
were Python functions — useful when a Python-based application needs to
call into an existing, unported MATLAB codebase (a legacy algorithm, a
validated numerical model) rather than reimplement it.

```python
future = eng.longRunningComputation(data, background=True)
# do other Python work while MATLAB computes
result = future.result()   # blocks until MATLAB finishes, if not already done
```

`background=True` mirrors the asynchronous `parfeval` pattern from
Level 3 Module 05 — useful when the calling Python application shouldn't
block on a slow MATLAB computation.

## MEX functions: calling C/C++ from MATLAB

```c
// example.c — a MEX function computing element-wise square
#include "mex.h"

void mexFunction(int nlhs, mxArray *plhs[], int nrhs, const mxArray *prhs[]) {
    double *input = mxGetPr(prhs[0]);
    mwSize n = mxGetNumberOfElements(prhs[0]);

    plhs[0] = mxCreateDoubleMatrix(1, n, mxREAL);
    double *output = mxGetPr(plhs[0]);

    for (mwSize i = 0; i < n; i++) {
        output[i] = input[i] * input[i];
    }
}
```

```matlab
mex example.c
y = example([1 2 3 4]);   % calls the compiled C function like any MATLAB function
disp(y);                  % [1 4 9 16]
```

A MEX function is compiled C/C++ (or Fortran) code exposing the
`mexFunction` entry point, callable from MATLAB exactly like a normal
`.m` function. This is the manual, hand-written alternative to MATLAB
Coder's automatic `-config:mex` target (Module 02) — appropriate when
you're wrapping *existing* C/C++ code (a legacy library, a
performance-critical routine already written in C) rather than
generating C from MATLAB source.

`mxGetPr`/`mxCreateDoubleMatrix` are the MEX API's functions for reading
MATLAB array data and creating new MATLAB arrays from C — the boundary
layer between MATLAB's array representation and raw C pointers/arrays.

## Calling MATLAB from C/C++: the MATLAB Engine API

```c
#include "engine.h"

Engine *ep = engOpen("");
engEvalString(ep, "x = 1:10; y = x.^2;");

mxArray *result = engGetVariable(ep, "y");
double *data = mxGetPr(result);
// data now points to the computed y values, usable in the C program

engClose(ep);
```

The reverse direction of the Python Engine API — a C/C++ application
launches and drives a MATLAB session, sending it commands as strings
and pulling results back as `mxArray`s. Less common than MEX (which
avoids launching a separate MATLAB process at all) but appropriate when
the calling C++ application specifically wants to run arbitrary MATLAB
scripts, not just one compiled function.

## Java integration

```matlab
javaObj = javaObject('java.util.ArrayList');
javaObj.add('first');
javaObj.add('second');
disp(javaObj.size());   % 2 — calling Java methods directly from MATLAB
```

MATLAB itself runs on the JVM internally, so calling Java classes
(including custom `.jar` files added via `javaaddpath`) is
natively supported without a separate bridge — useful for reaching a
Java library MATLAB has no native equivalent for, or integrating with
an existing enterprise Java codebase.

```matlab
javaaddpath('myLibrary.jar');
import com.mycompany.mylib.Calculator;
calc = Calculator();
result = calc.compute(42);
```

## .NET integration

```matlab
NET.addAssembly('MyLibrary.dll');
obj = MyNamespace.MyClass();
result = obj.Compute(42);
```

Similarly, `.NET` assemblies can be loaded and called directly on
Windows — the natural bridge when integrating with an existing C#/.NET
enterprise system.

## Choosing an integration path

| Need | Mechanism |
|---|---|
| Use a Python-only library from MATLAB | Python interface (`py.*`, `pyenv`) |
| Drive existing MATLAB code from a Python application | MATLAB Engine API for Python |
| Wrap existing C/C++/Fortran code as a callable MATLAB function | Hand-written MEX function |
| Auto-generate C from MATLAB source (no existing C to wrap) | MATLAB Coder (Module 02) instead |
| Drive MATLAB scripts from a C/C++ application | MATLAB Engine API for C/C++ |
| Call Java classes/libraries | Native Java integration (`javaObject`, `javaaddpath`) |
| Call .NET assemblies (Windows) | `.NET` integration (`NET.addAssembly`) |

## A worked scenario: mixed-language pipeline

A realistic production pipeline might: preprocess data in Python
(leveraging `pandas`/data engineering tools), call into MATLAB via the
Python Engine API to run a validated MATLAB signal-processing algorithm
(reusing, say, Level 3 Module 10's `SignalPipeline` unchanged), and
return results to Python for a web API to serve — no piece is rewritten
in another language purely for integration's sake; each language does
the part it's strongest at.

```python
import matlab.engine
eng = matlab.engine.start_matlab()
eng.addpath('matlab_src/')

raw_signal = preprocess_with_pandas(raw_data)   # Python
signal_matlab = matlab.double(raw_signal.tolist())
result = eng.runSignalPipeline(signal_matlab, 1000.0, nargout=1)  # MATLAB
serve_via_api(result)   # Python
```

## How It Actually Works

Calling into other languages from MATLAB uses different mechanisms with
genuinely different performance and data-marshaling implications. A
Python call via MATLAB's Python interface passes data across a
**language-boundary conversion layer**: a MATLAB `double` array is
converted to a NumPy array (or vice versa) by copying its underlying
buffer and translating MATLAB's column-major layout into NumPy's default
row-major (C-order) layout — this conversion is not free, and it's a real,
measurable cost for large arrays crossed frequently between the two
languages, not just an abstract "interop tax"; a workflow that ping-pongs
a large matrix between MATLAB and Python every iteration of a loop pays
this layout-conversion cost every single time.

MEX functions (also covered in Module 08, Level 2, for toolboxes) work
differently — they're compiled C/C++/Fortran linked directly against
MATLAB's C API and MATLAB's own array data structures (`mxArray`), so a
well-written MEX function can, in specific cases, avoid a full data copy
by operating directly on MATLAB's internal array representation, which is
why MEX integration is the standard choice when you need near-zero-copy,
high-frequency interop with native code, while the Python/Java bridges are
better suited to coarser-grained calls (call a Python function once per
batch, not once per loop iteration).

Calling a C library via `calllib`/`clib` (post-R2020b's C++ interface)
still requires MATLAB to marshal arguments into the exact C ABI the
library expects — matching data types precisely (MATLAB `int32` to C
`int`, `double` to C `double`) matters because a mismatch isn't caught by
a friendly MATLAB-level error, it's undefined behavior at the native call
boundary, the same class of risk as any FFI in any language.

*Note: based on documented MATLAB-Python/MEX/C-interop data-marshaling
behavior; these interfaces are unavailable to exercise directly in this
environment.*

## Practice

1. Sketch the MATLAB-side code to call a Python function from a library
   with no MATLAB equivalent (e.g. a specific NLP tokenizer), including
   the type conversions needed to pass a MATLAB string array in and get
   a usable MATLAB cell array of tokens back.
2. Write a minimal MEX function skeleton (in the style of the example
   above) that takes two input arrays and returns their element-wise
   product, and describe what would go wrong if the two inputs have
   different sizes (what check the C code needs and what MATLAB error
   convention it should follow, per Level 1/2's error-handling
   conventions).
3. Explain the tradeoff between using the Python Engine API to call an
   existing MATLAB algorithm from a Python service versus using MATLAB
   Coder (Module 02) to convert that same algorithm to standalone C and
   avoiding MATLAB entirely at runtime — when is each the better
   choice?
4. Describe, for the mixed-language pipeline scenario, what would need
   to change if the MATLAB algorithm needed to run without any MATLAB
   installation available in the production environment at all, tying
   this back to Module 05's compiled-deployment options.

## 🔀 See this in another language

- [C — 07 · Interfacing with Other Languages](https://sigilipelli.github.io/c-mastery-path/level-4/07-interfacing-other-languages/)
