---
title: "MATLAB Interview Questions and Answers (179 Questions)"
description: "179 MATLAB interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the MATLAB Mastery Path."
---

# MATLAB Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free MATLAB Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is MATLAB?](level-1/01-what-is-matlab.md)

**Q1.** What does MATLAB stand for?

**Answer:** MATrix LABoratory

Everything is an array.

**Q2.** What is the Command Window used for?

**Answer:** Typing commands and seeing immediate results

Interactive.

**Q3.** What's a script vs a function file?

**Answer:** A script runs commands in the workspace; a function has its own workspace and arguments

Functions are reusable.

**Q4.** What does a semicolon at the end of a line do?

**Answer:** Suppresses displaying the result

Use it for quiet statements.

### [02 · Variables &amp; Basic Data Types](level-1/02-variables-types.md)

**Q5.** What's MATLAB's default numeric type?

**Answer:** double (64-bit floating point)

Even scalars are 1x1 arrays.

**Q6.** What's the difference between char and string?

**Answer:** char is a character array; string is a text container (double quotes)

"text" vs 'text'.

**Q7.** What does `whos` show?

**Answer:** Workspace variables with size and class

Inspect types.

**Q8.** What does `x = int8(200)` give?

**Answer:** 127 (integer types saturate)

No wraparound.

### [03 · Vectors &amp; Matrix Operations](level-1/03-vectors-matrices.md)

**Q9.** What is the sum of the 2x2 matrix [1 2; 3 4] elementwise vs the matrix product A*A? (A.*A sum, A*A)

```python
A=[[1,2],[3,4]]
elem=sum(x*x for r in A for x in r)
prod=[[sum(A[i][k]*A[k][j] for k in range(2)) for j in range(2)] for i in range(2)]
print(elem, prod)
```

**Answer:** 30 [[7, 10], [15, 22]]

A.*A is elementwise; A*A is the matrix product. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q10.** How do you create 1 to 10 in steps of 2?

**Answer:** 1:2:10

start:step:stop.

**Q11.** What does A(2,:) return?

**Answer:** The entire second row

Colon means all.

**Q12.** What does the apostrophe do to a matrix?

**Answer:** Transposes it (conjugate transpose for complex)

.' is plain transpose.

### [04 · Control Flow](level-1/04-control-flow.md)

**Q13.** Which loop iterates over a vector's elements?

**Answer:** for v = vec ... end

Columns of the iterator.

**Q14.** What does `break` do?

**Answer:** Exits the innermost loop

continue skips.

**Q15.** Which operator is logical AND for scalars with short circuit?

**Answer:** &amp;&amp;

&amp; is elementwise.

**Q16.** What does `switch` compare strings with?

**Answer:** Case labels like case 'abc'

Works with text.

### [05 · Functions in MATLAB](level-1/05-functions.md)

**Q17.** How are functions declared in a file?

**Answer:** function out = name(in) ... end, saved as name.m

File name matches.

**Q18.** What's an anonymous function?

**Answer:** A one-line function handle like f = @(x) x.^2

Closures.

**Q19.** What does nargin give?

**Answer:** The number of input arguments passed

Default handling.

**Q20.** What's variable scope in functions?

**Answer:** Local workspace separate from the caller

Pass data explicitly.

### [06 · Plotting &amp; Visualization Basics](level-1/06-plotting-basics.md)

**Q21.** What does `plot(x, y)` draw?

**Answer:** A 2-D line plot of y versus x

Basic.

**Q22.** What does `hold on` do?

**Answer:** Adds new plots to the same axes

Overlays.

**Q23.** How label axes?

**Answer:** xlabel, ylabel and title

Readable plots.

**Q24.** What saves a figure to a file?

**Answer:** exportgraphics or saveas

Formats.

### [07 · Working with Data Files](level-1/07-data-files.md)

**Q25.** Which function reads a CSV into a table?

**Answer:** readtable

Mixed types.

**Q26.** What does `load` do?

**Answer:** Reads a .mat file's variables into the workspace

save writes.

**Q27.** What does `writematrix` do?

**Answer:** Writes a numeric matrix to a file

Companion to readmatrix.

**Q28.** Why check for missing data after reading?

**Answer:** Files often contain NaN/blank fields affecting statistics

ismissing.

### [08 · String &amp; Text Processing](level-1/08-string-text-processing.md)

**Q29.** How concatenate strings?

**Answer:** "a" + "b" (or strcat/[...] for char)

String arrays.

**Q30.** What does `split(s, ",")` do?

**Answer:** Splits a string at the delimiter into pieces

strsplit for char.

**Q31.** What does `regexp` do?

**Answer:** Finds patterns with regular expressions

Text extraction.

**Q32.** What does `sprintf` return?

**Answer:** A formatted char/string without printing

fprintf prints.

### [09 · Basic Numerical Methods](level-1/09-numerical-methods.md)

**Q33.** What does the trapezoid rule give for the integral of x^2 on [0,1] with 4 intervals?

```python
n=4;h=1/n
ys=[(i*h)**2 for i in range(n+1)]
print(round(h*(ys[0]/2+sum(ys[1:-1])+ys[-1]/2),5))
```

**Answer:** 0.34375

trapz approximates integrals; the exact value is 1/3. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q34.** How does Newton's method iterate to solve x^2 = 2 from x0 = 1 (three steps)?

```python
x=1.0
for _ in range(3): x=x-(x*x-2)/(2*x)
print(round(x,6))
```

**Answer:** 1.414216

Quadratic convergence. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q35.** What does `fzero` find?

**Answer:** A root of a function

Bracketing.

**Q36.** What does `\` (mldivide) solve?

**Answer:** Linear systems A\b (least squares if overdetermined)

Prefer to inv(A)*b.

### [10 · Project — Data Analysis &amp; Plotting Script](level-1/10-project-data-analysis.md)

**Q37.** What does the data analysis project combine?

**Answer:** Reading data, cleaning, statistics, plots and reporting in a script

Level 1 project.

**Q38.** How handle missing values?

**Answer:** Detect with isnan/ismissing and remove or fill deliberately

Document it.

**Q39.** What summarizes a variable?

**Answer:** mean, median, std and a histogram

Descriptive stats.

**Q40.** How make the script reproducible?

**Answer:** Set paths and random seeds explicitly and save outputs

Repeatable.

## Level 2 · Intermediate

### [01 · Advanced Matrix Operations &amp; Linear Algebra](level-2/01-advanced-matrix-linear-algebra.md)

**Q41.** What is the determinant of [[2,1],[1,3]]?

```python
print(2*3-1*1)
```

**Answer:** 5

det(A) = ad - bc. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q42.** What do eig and svd compute?

**Answer:** Eigenvalues/vectors and singular value decomposition

Linear algebra.

**Q43.** Why avoid inv(A)*b?

**Answer:** It's slower and less accurate than A\b

Numerical stability.

**Q44.** What does `rank` tell you?

**Answer:** The number of linearly independent rows/columns

Singularity.

### [02 · Cell Arrays &amp; Structs](level-2/02-cell-arrays-structs.md)

**Q45.** What does a cell array hold?

**Answer:** Different types/sizes in each cell, indexed with {}

c{2}.

**Q46.** What's a struct?

**Answer:** Named fields grouping related data (s.name)

Records.

**Q47.** What's the difference between c(2) and c{2}?

**Answer:** Parentheses return a cell; braces return the contents

Common bug.

**Q48.** When prefer a table?

**Answer:** Tabular data with named columns of mixed types

Analysis.

### [03 · Vectorization vs Loops](level-2/03-vectorization-vs-loops.md)

**Q49.** Computing the sum of squares of 1..5 with a loop vs vectorized gives?

```python
print(sum(i*i for i in range(1,6)))
```

**Answer:** 55

sum((1:5).^2). (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q50.** Why vectorize?

**Answer:** Built-in array operations are faster and clearer than loops

JIT helps loops too.

**Q51.** What does preallocating do?

**Answer:** Avoids growing arrays inside loops (zeros(n,1))

Performance.

**Q52.** What's implicit expansion?

**Answer:** Operations broadcasting arrays of compatible sizes

A + row vector.

### [04 · Writing Robust Functions](level-2/04-robust-functions.md)

**Q53.** What does `validateattributes` or `arguments` block do?

**Answer:** Checks inputs' type, size and values

Robust functions.

**Q54.** How report errors?

**Answer:** error('id:sub', 'message %d', n)

Identifiers.

**Q55.** What does try/catch do?

**Answer:** Handles runtime errors gracefully

ME.message.

**Q56.** Why default arguments?

**Answer:** Optional behavior without breaking callers

nargin/arguments.

### [05 · Advanced Plotting &amp; Figures](level-2/05-advanced-plotting-figures.md)

**Q57.** What does `subplot` or tiledlayout do?

**Answer:** Arranges multiple axes in one figure

tiledlayout is newer.

**Q58.** What does `surf` draw?

**Answer:** A 3-D surface from a grid

meshgrid.

**Q59.** How customize appearance programmatically?

**Answer:** Set properties of graphics objects (handles)

set/get.

**Q60.** What does `drawnow` do?

**Answer:** Updates figures immediately

Animations.

### [06 · Intro to Simulink Concepts](level-2/06-intro-simulink.md)

**Q61.** What is Simulink?

**Answer:** A block-diagram environment for modeling and simulating dynamic systems

Model-based.

**Q62.** What are blocks and signals?

**Answer:** Blocks transform signals connected by lines

Dataflow.

**Q63.** What does a solver do?

**Answer:** Integrates the model's equations over time

Fixed/variable step.

**Q64.** Where do results go?

**Answer:** Scopes, To Workspace blocks or logged signals

Inspect.

### [07 · Curve Fitting &amp; Interpolation](level-2/07-curve-fitting-interpolation.md)

**Q65.** Fit a line y = a + b x to points (0,1),(1,3),(2,5). What are a and b?

```python
import statistics as s
x=[0,1,2];y=[1,3,5]
mx,my=s.mean(x),s.mean(y)
b=sum((a-mx)*(c-my) for a,c in zip(x,y))/sum((a-mx)**2 for a in x)
print(my-b*mx,b)
```

**Answer:** 1.0 2.0

polyfit(x,y,1) returns [2 1]. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q66.** What does `interp1` do?

**Answer:** Interpolates values between known data points

Methods: linear, spline.

**Q67.** What does polyfit return?

**Answer:** Polynomial coefficients, highest power first

polyval evaluates.

**Q68.** What's overfitting a polynomial?

**Answer:** High degree fits noise and generalizes poorly

Validate.

### [08 · Working with Toolboxes](level-2/08-toolboxes-overview.md)

**Q69.** What are toolboxes?

**Answer:** Add-on libraries for domains like signal processing or optimization

Licensed.

**Q70.** How check what's installed?

**Answer:** ver or the Add-On Explorer

Dependencies.

**Q71.** What does Signal Processing Toolbox provide?

**Answer:** Filtering, spectral analysis and signal generation

fft, filter.

**Q72.** Why note toolbox dependencies?

**Answer:** Code may fail for users without the licenses

Portability.

### [09 · Debugging in MATLAB](level-2/09-debugging.md)

**Q73.** What is a breakpoint?

**Answer:** A line where execution pauses for inspection

Debugging.

**Q74.** What does `dbstop if error` do?

**Answer:** Pauses at the line that throws

Post-mortem.

**Q75.** What does the Workspace browser show?

**Answer:** Current variables and values while debugging

Inspect state.

**Q76.** What's a good debugging approach?

**Answer:** Reproduce with a small input and inspect intermediate values

Bisect.

### [10 · Project — Curve Fitting Analysis Tool](level-2/10-project-curve-fitting-tool.md)

**Q77.** What does the curve fitting tool project combine?

**Answer:** Data import, fitting, residual analysis and plotting

Level 2 project.

**Q78.** How judge fit quality?

**Answer:** Residuals, R^2 and validation on held-out data

Evidence.

**Q79.** What makes it robust?

**Answer:** Input validation and clear error messages

User-friendly.

**Q80.** How present results?

**Answer:** Annotated plots with fit equations and confidence

Communicate.

## Level 3 · Advanced

### [01 · Object-Oriented Programming in MATLAB](level-3/01-oop-matlab.md)

**Q81.** What does a classdef file define?

**Answer:** A class with properties and methods

OOP in MATLAB.

**Q82.** What's a handle class?

**Answer:** Objects with reference semantics shared across variables

&lt; handle.

**Q83.** Value vs handle class?

**Answer:** Value classes copy on assignment; handle classes share the same object

Know which you need.

**Q84.** What do properties' attributes control?

**Answer:** Access, constness and dependencies (e.g. Access = private)

Encapsulation.

### [02 · Advanced Simulink Modeling](level-3/02-advanced-simulink-modeling.md)

**Q85.** What do subsystems do in Simulink?

**Answer:** Group blocks into reusable hierarchical units

Organization.

**Q86.** What are model references?

**Answer:** Including one model inside another for reuse

Big projects.

**Q87.** What's a bus?

**Answer:** A grouped set of signals passed together

Structured data.

**Q88.** Why use data dictionaries?

**Answer:** Manage parameters and types centrally

Consistency.

### [03 · Signal Processing Basics](level-3/03-signal-processing-basics.md)

**Q89.** What's the FFT bin spacing for 1000 samples at 1000 Hz?

```python
print(1000/1000)
```

**Answer:** 1.0

fs/N Hz per bin. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q90.** What does the Nyquist rate require?

**Answer:** Sampling at more than twice the highest signal frequency

Avoid aliasing.

**Q91.** What does `filter(b,a,x)` do?

**Answer:** Applies a digital filter to data

IIR/FIR.

**Q92.** What is aliasing?

**Answer:** High frequencies appearing as lower ones when undersampled

Anti-alias filter.

### [04 · Optimization Toolbox Basics](level-3/04-optimization-toolbox.md)

**Q93.** What does `fminsearch` do?

**Answer:** Minimizes a function without gradients

Nelder-Mead.

**Q94.** What does `linprog` solve?

**Answer:** Linear programs with linear constraints

Optimization Toolbox.

**Q95.** Why give good initial guesses?

**Answer:** Local solvers can converge to poor minima

Multi-start.

**Q96.** What defines a convex problem benefit?

**Answer:** Any local minimum is global

Guarantees.

### [05 · Parallel Computing in MATLAB](level-3/05-parallel-computing.md)

**Q97.** What does `parfor` do?

**Answer:** Runs loop iterations in parallel on workers

Parallel Computing Toolbox.

**Q98.** What must parfor iterations be?

**Answer:** Independent of each other

No dependencies.

**Q99.** What's a parallel pool?

**Answer:** A set of workers started with parpool

Overhead exists.

**Q100.** When is parallel not worth it?

**Answer:** Small tasks where overhead dominates

Measure.

### [06 · App Designer Basics](level-3/06-app-designer-basics.md)

**Q101.** What is App Designer?

**Answer:** A tool to build apps with a GUI and code

UI components.

**Q102.** What are callbacks?

**Answer:** Functions run when a user interacts with a component

Events.

**Q103.** How share an app?

**Answer:** Package as a standalone app or MATLAB App

Deployment.

**Q104.** Why separate logic from UI?

**Answer:** Testability and reuse

Layered.

### [07 · Working with Large Datasets](level-3/07-large-datasets.md)

**Q105.** How process data too big for memory?

**Answer:** Use datastore, tall arrays or chunked reads

Out-of-memory.

**Q106.** What are tall arrays?

**Answer:** Arrays evaluated lazily over data larger than memory

gather.

**Q107.** What does `matfile` allow?

**Answer:** Partial loading/saving of parts of a MAT file

Memory.

**Q108.** What file format is efficient?

**Answer:** Parquet/MAT v7.3 for large data

Columnar.

### [08 · Advanced Numerical Methods](level-3/08-advanced-numerical-methods.md)

**Q109.** What does the RK2 (midpoint) method give for y'=y, y(0)=1, h=0.5 after 2 steps?

```python
y=1.0;h=0.5
for _ in range(2):
    k1=y;k2=y+h/2*k1
    y=y+h*k2
print(round(y,5))
```

**Answer:** 2.64062

Higher order than Euler. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q110.** What does ode45 solve?

**Answer:** Non-stiff ordinary differential equations

Adaptive steps.

**Q111.** When use ode15s?

**Answer:** For stiff problems

Implicit.

**Q112.** What's step size trade-off?

**Answer:** Smaller steps are more accurate but slower

Tolerances.

### [09 · Testing MATLAB Code](level-3/09-testing-matlab-code.md)

**Q113.** What does the unit testing framework provide?

**Answer:** matlab.unittest with test classes, assertions and runners

runtests.

**Q114.** What's a test fixture?

**Answer:** Shared setup/teardown for tests

Fresh state.

**Q115.** How compare floating-point values?

**Answer:** With tolerances (AbsTol/RelTol)

Round-off.

**Q116.** What does code coverage report?

**Answer:** Which lines tests executed

Not assertion quality.

### [10 · Project — Signal Processing Pipeline](level-3/10-project-signal-processing-pipeline.md)

**Q117.** What does the signal pipeline project combine?

**Answer:** Filtering, spectral analysis, tests and reporting

Level 3 project.

**Q118.** How validate a filter?

**Answer:** Compare frequency response and test known signals

Evidence.

**Q119.** How organize the code?

**Answer:** Functions/classes per stage with tests

Modular.

**Q120.** How present results?

**Answer:** Plots with labeled axes and units

Clarity.

## Level 4 · Master

### [01 · Advanced Simulink &amp; Model-Based Design](level-4/01-advanced-simulink-model-based-design.md)

**Q121.** What is model-based design?

**Answer:** Developing systems from executable models through simulation to code

Workflow.

**Q122.** What does model verification involve?

**Answer:** Requirements-based tests and coverage on the model

Simulink Test.

**Q123.** Why use fixed-step solvers for codegen?

**Answer:** Embedded targets need deterministic timing

Real time.

**Q124.** What are model advisor checks?

**Answer:** Automated guideline checks (naming, safety standards)

Quality.

### [02 · Code Generation (MATLAB Coder)](level-4/02-code-generation-matlab-coder.md)

**Q125.** What does MATLAB Coder generate?

**Answer:** C/C++ code from MATLAB functions

Embedded.

**Q126.** What limits codegen?

**Answer:** Unsupported functions and dynamic features

Check support.

**Q127.** What helps codegen?

**Answer:** Fixed-size types and explicit input specs

Static.

**Q128.** How verify generated code?

**Answer:** Compare outputs with MATLAB (SIL testing)

Equivalence.

### [03 · Advanced Optimization &amp; Machine Learning Toolbox](level-4/03-optimization-ml-toolbox.md)

**Q129.** What does Statistics and ML Toolbox provide?

**Answer:** Models for classification, regression and clustering

fitc*, fitr*.

**Q130.** How avoid overfitting?

**Answer:** Cross-validation and regularization

Generalization.

**Q131.** What does hyperparameter tuning do?

**Answer:** Searches settings to improve validation performance

OptimizeHyperparameters.

**Q132.** What's a baseline model?

**Answer:** A simple benchmark any model must beat

Reference.

### [04 · Performance Profiling &amp; Optimization](level-4/04-performance-profiling.md)

**Q133.** What's the first step in performance work?

**Answer:** Profile with the Profiler to find hotspots

profile on.

**Q134.** What do tic/toc measure?

**Answer:** Elapsed wall-clock time

timeit is more reliable.

**Q135.** What's a common slowdown?

**Answer:** Growing arrays in loops and unnecessary copies

Preallocate.

**Q136.** Why use timeit?

**Answer:** Repeated runs and warm-up for stable timings

Benchmarks.

### [05 · Building Production MATLAB Applications](level-4/05-production-applications.md)

**Q137.** How deploy MATLAB code to users without MATLAB?

**Answer:** Compile with MATLAB Compiler and the MATLAB Runtime

Standalone.

**Q138.** What do production apps need?

**Answer:** Logging, error handling, configuration and tests

Robustness.

**Q139.** What's MATLAB Production Server?

**Answer:** Serving MATLAB functions as web services

RESTful.

**Q140.** How manage dependencies?

**Answer:** Projects and dependency analysis

Reproducible.

### [06 · Integration with Other Languages](level-4/06-integration-other-languages.md)

**Q141.** How call Python from MATLAB?

**Answer:** py. prefix with a configured pyenv

Interop.

**Q142.** How call MATLAB from Python?

**Answer:** The MATLAB Engine API for Python

matlab.engine.

**Q143.** What converts data between languages?

**Answer:** Type mappings (numpy arrays, matlab.double)

Careful.

**Q144.** What else interoperates?

**Answer:** C/C++ (MEX, loadlibrary), Java and .NET

Integration.

### [07 · Advanced Parallel &amp; GPU Computing](level-4/07-parallel-gpu-computing.md)

**Q145.** What does gpuArray do?

**Answer:** Moves data to the GPU so supported functions run there

Parallel Computing Toolbox.

**Q146.** When does the GPU help?

**Answer:** Large data-parallel computations

Transfer cost.

**Q147.** What's gather?

**Answer:** Brings GPU data back to the CPU

Results.

**Q148.** Why measure with gputimeit?

**Answer:** GPU calls are asynchronous; naive timing is wrong

Accurate timing.

### [08 · CI/CD for MATLAB Projects](level-4/08-cicd-matlab.md)

**Q149.** What does CI for MATLAB run?

**Answer:** Tests via runtests/buildtool and code analysis (checkcode)

Automation.

**Q150.** What's buildtool?

**Answer:** MATLAB's build tool for defining tasks and dependencies

Pipelines.

**Q151.** Which CI services support MATLAB?

**Answer:** GitHub Actions (matlab-actions), Jenkins and others

Official actions.

**Q152.** What should fail the build?

**Answer:** Failing tests or analyzer errors

Gate.

### [09 · MATLAB in Industry](level-4/09-matlab-in-industry.md)

**Q153.** Where is MATLAB used in industry?

**Answer:** Engineering, finance, automotive, aerospace and research

Numerical computing.

**Q154.** What differentiates it?

**Answer:** Domain toolboxes, Simulink and mature numerics

Ecosystem.

**Q155.** What's a common alternative?

**Answer:** Python (NumPy/SciPy), Julia and Octave

Trade-offs.

**Q156.** How choose?

**Answer:** By team skills, licensing and existing code

Pragmatic.

### [10 · Capstone — Full Numerical Analysis Application](level-4/10-capstone-numerical-analysis-app.md)

**Q157.** What does the capstone combine?

**Answer:** A numerical analysis app with tests, performance work and deployment

Level 4 project.

**Q158.** How prove it works?

**Answer:** Validation against known solutions and tests

Verification.

**Q159.** What documentation helps?

**Answer:** Help text, examples and a README

Usability.

**Q160.** What sustains it?

**Answer:** Tests, CI and version control

Maintenance.

## Mixed questions across all levels

**Q161.** What does MATLAB stand for and what is its core data type?

**Answer:** MATrix LABoratory; everything is an array (even scalars are 1x1)

Array-first design.

**Q162.** What are the elementwise square sum and the matrix product of A = [1 2; 3 4]?

```python
A=[[1,2],[3,4]]
print(sum(x*x for r in A for x in r), [[sum(A[i][k]*A[k][j] for k in range(2)) for j in range(2)] for i in range(2)])
```

**Answer:** 30 [[7, 10], [15, 22]]

A.*A is elementwise; A*A is the matrix product. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q163.** How do you solve A x = b in MATLAB?

**Answer:** x = A\b (not inv(A)*b)

More accurate and faster.

**Q164.** What's the difference between c(2) and c{2} for a cell array?

**Answer:** Parentheses return a cell; braces return its contents

Common bug.

**Q165.** What does int8(200) give?

**Answer:** 127: integer types saturate

No wraparound.

**Q166.** Why preallocate arrays in loops?

**Answer:** Growing arrays each iteration copies memory and is slow

zeros(n,1).

**Q167.** A loop computing sum((1:n).^2) is slow. What's the idiom?

**Answer:** Vectorize with array operations

Built-in array ops.

**Q168.** What does Newton's method give for sqrt(2) after 3 steps from x0 = 1?

```python
x=1.0
for _ in range(3): x=x-(x*x-2)/(2*x)
print(round(x,6))
```

**Answer:** 1.414216

Quadratic convergence. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q169.** What's the trapezoid estimate of the integral of x^2 on [0,1] with 4 intervals?

```python
n=4;h=1/n
ys=[(i*h)**2 for i in range(n+1)]
print(round(h*(ys[0]/2+sum(ys[1:-1])+ys[-1]/2),5))
```

**Answer:** 0.34375

Exact value is 1/3; trapz overestimates convex functions. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q170.** Fit a line to (0,1),(1,3),(2,5). What are intercept and slope?

```python
import statistics as s
x=[0,1,2];y=[1,3,5]
mx,my=s.mean(x),s.mean(y)
b=sum((a-mx)*(c-my) for a,c in zip(x,y))/sum((a-mx)**2 for a in x)
print(my-b*mx,b)
```

**Answer:** 1.0 2.0

polyfit(x,y,1) returns [2 1]. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q171.** Which solver handles stiff ODEs?

**Answer:** ode15s (ode45 is for non-stiff)

Implicit methods.

**Q172.** An A-matrix is nearly singular and A\b warns. What does it mean?

**Answer:** The system is ill-conditioned: results are sensitive to small errors

Check cond(A).

**Q173.** What does the Nyquist criterion require?

**Answer:** Sampling above twice the highest signal frequency to avoid aliasing

Anti-alias filter.

**Q174.** What is the FFT bin spacing for 2000 samples at 1000 Hz?

```python
print(1000/2000)
```

**Answer:** 0.5

fs/N Hz per bin. (Numbers checked with an equivalent Python computation; MATLAB itself was not available.) Verified by running it.

**Q175.** A parfor loop gives wrong results. What's the likely cause?

**Answer:** Iterations depend on each other (parfor needs independent iterations)

Parallel Computing Toolbox.

**Q176.** Data is bigger than memory. What do you use?

**Answer:** datastore and tall arrays (or chunked reads)

Out-of-memory analysis.

**Q177.** What does a handle class give you?

**Answer:** Reference semantics: copies of the variable refer to the same object

Value classes copy on assignment.

**Q178.** How do you compare floating-point values in unit tests?

**Answer:** Within a tolerance (AbsTol/RelTol), not ==

Round-off.

**Q179.** What do you use to deploy to users without a MATLAB license?

**Answer:** MATLAB Compiler with the free MATLAB Runtime

Standalone apps.

## Practical coding questions

### Root finding and integration

Implement newton, bisect (with a bracketing check) and a composite trapz, then compare their behaviour on x^2 - 2 and on integrals of x^2 and sin. Python mirrors the MATLAB functions fzero/trapz.

Starter code:

```
import math

def newton(f, df, x0, tol=1e-10, max_iter=50):
    # iterate x -= f(x)/df(x); return (x, iterations) once |step| < tol; RuntimeError if it never converges
    return x0, 0

def bisect(f, a, b, tol=1e-8):
    # halve [a, b] keeping the sign change; ValueError("root not bracketed") if f(a)*f(b) > 0;
    # return (midpoint, number of halvings) once the half-width <= tol
    return a, 0

def trapz(f, a, b, n):
    # composite trapezoid rule with n intervals
    return 0.0

f = lambda x: x * x - 2
r, it = newton(f, lambda x: 2 * x, 1.0)
print("newton:", round(r, 8), "iterations", it)
r, it = bisect(f, 0, 2)
print("bisect:", round(r, 6), "iterations", it)
try:
    bisect(f, 2, 3)
except ValueError as e:
    print("error:", e)
print("trapz n=4:", round(trapz(lambda x: x * x, 0, 1, 4), 5), " n=100:", round(trapz(lambda x: x * x, 0, 1, 100), 6))
print("sin integral:", round(trapz(math.sin, 0, math.pi, 200), 5))
```

Expected output:

```text
newton: 1.41421356 iterations 5
bisect: 1.414214 iterations 27
error: root not bracketed
trapz n=4: 0.34375  n=100: 0.33335
sin integral: 1.99996
```

??? success "Show a solution"

    ```
    import math

    def newton(f, df, x0, tol=1e-10, max_iter=50):
        x = x0
        for i in range(1, max_iter + 1):
            step = f(x) / df(x)
            x -= step
            if abs(step) < tol:
                return x, i
        raise RuntimeError("did not converge")

    def bisect(f, a, b, tol=1e-8):
        if f(a) * f(b) > 0:
            raise ValueError("root not bracketed")
        n = 0
        while (b - a) / 2 > tol:
            m = (a + b) / 2
            if f(a) * f(m) <= 0:
                b = m
            else:
                a = m
            n += 1
        return (a + b) / 2, n

    def trapz(f, a, b, n):
        h = (b - a) / n
        return h * (f(a) / 2 + sum(f(a + i * h) for i in range(1, n)) + f(b) / 2)

    f = lambda x: x * x - 2
    r, it = newton(f, lambda x: 2 * x, 1.0)
    print("newton:", round(r, 8), "iterations", it)
    r, it = bisect(f, 0, 2)
    print("bisect:", round(r, 6), "iterations", it)
    try:
        bisect(f, 2, 3)
    except ValueError as e:
        print("error:", e)
    print("trapz n=4:", round(trapz(lambda x: x * x, 0, 1, 4), 5), " n=100:", round(trapz(lambda x: x * x, 0, 1, 100), 6))
    print("sin integral:", round(trapz(math.sin, 0, math.pi, 200), 5))
    ```

    Newton converges in a handful of iterations when started near the root, bisection needs about 27 halvings for 1e-8 but always works when the root is bracketed, and the trapezoid rule's error shrinks as the interval count grows. Verified by running the solution in Python (an equivalent of the MATLAB approach).

### Linear solve and least-squares fit

Implement Gaussian elimination with partial pivoting (what A\b does for square systems) and a least-squares polyfit using the normal equations. Detect singular matrices, fit a noisy line and compute R^2.

Starter code:

```
def solve(A, b):
    # Gaussian elimination with partial pivoting; ValueError("matrix is singular") when a pivot is ~0
    return []

def polyfit(xs, ys, deg):
    # least squares via the normal equations (X'X)c = X'y (use solve); coefficients highest power first
    return []

print([round(v, 6) for v in solve([[2, 1, -1], [-3, -1, 2], [-2, 1, 2]], [8, -11, -3])])
try:
    solve([[1, 2], [2, 4]], [1, 2])
except ValueError as e:
    print("error:", e)
xs = [0, 1, 2, 3, 4]
ys = [1.1, 2.9, 5.2, 6.8, 9.1]
c = polyfit(xs, ys, 1)
print("line coefficients:", [round(v, 4) for v in c])
pred = [c[0] * x + c[1] for x in xs]
ss_res = sum((y - p) ** 2 for y, p in zip(ys, pred))
mean = sum(ys) / len(ys)
print("R^2:", round(1 - ss_res / sum((y - mean) ** 2 for y in ys), 4))
print("quadratic exact:", [round(v, 6) + 0.0 for v in polyfit([0, 1, 2, 3], [1, 2, 5, 10], 2)])
```

Expected output:

```text
[2.0, 3.0, -1.0]
error: matrix is singular
line coefficients: [1.99, 1.04]
R^2: 0.9973
quadratic exact: [1.0, 0.0, 1.0]
```

??? success "Show a solution"

    ```
    def solve(A, b):
        """Gaussian elimination with partial pivoting (what A\\b does for a square system)."""
        n = len(A)
        M = [row[:] + [b[i]] for i, row in enumerate(A)]
        for c in range(n):
            p = max(range(c, n), key=lambda r: abs(M[r][c]))
            if abs(M[p][c]) < 1e-12:
                raise ValueError("matrix is singular")
            M[c], M[p] = M[p], M[c]
            for r in range(c + 1, n):
                f = M[r][c] / M[c][c]
                for k in range(c, n + 1):
                    M[r][k] -= f * M[c][k]
        x = [0.0] * n
        for i in range(n - 1, -1, -1):
            x[i] = (M[i][n] - sum(M[i][j] * x[j] for j in range(i + 1, n))) / M[i][i]
        return x

    def polyfit(xs, ys, deg):
        """Least squares via the normal equations (X'X)c = X'y; coefficients highest power first."""
        X = [[x ** p for p in range(deg, -1, -1)] for x in xs]
        XtX = [[sum(X[k][i] * X[k][j] for k in range(len(xs))) for j in range(deg + 1)] for i in range(deg + 1)]
        Xty = [sum(X[k][i] * ys[k] for k in range(len(xs))) for i in range(deg + 1)]
        return solve(XtX, Xty)

    print([round(v, 6) for v in solve([[2, 1, -1], [-3, -1, 2], [-2, 1, 2]], [8, -11, -3])])
    try:
        solve([[1, 2], [2, 4]], [1, 2])
    except ValueError as e:
        print("error:", e)
    xs = [0, 1, 2, 3, 4]
    ys = [1.1, 2.9, 5.2, 6.8, 9.1]
    c = polyfit(xs, ys, 1)
    print("line coefficients:", [round(v, 4) for v in c])
    pred = [c[0] * x + c[1] for x in xs]
    ss_res = sum((y - p) ** 2 for y, p in zip(ys, pred))
    mean = sum(ys) / len(ys)
    print("R^2:", round(1 - ss_res / sum((y - mean) ** 2 for y in ys), 4))
    print("quadratic exact:", [round(v, 6) + 0.0 for v in polyfit([0, 1, 2, 3], [1, 2, 5, 10], 2)])
    ```

    A\b is elimination with pivoting; polyfit minimizes squared residuals, and R^2 compares residuals with the variance of the data. Production code prefers QR for numerical stability. Verified by running the solution in Python (an equivalent of the MATLAB approach).

---

More free practice: the [MATLAB Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
