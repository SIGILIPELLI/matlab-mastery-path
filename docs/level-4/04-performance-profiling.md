# 04 · Performance Profiling & Optimization

!!! note "Verification note"
    MATLAB was not available in the environment used to write this
    page. Profiler output structure and optimization techniques below
    are documented, version-stable MATLAB behaviors, hand-traced
    against the MATLAB documentation and general numerical-computing
    performance principles rather than measured in MATLAB itself; any
    timing numbers are illustrative orders of magnitude, not
    benchmarks.

Guessing where code is slow is unreliable — the actual bottleneck is
often not where intuition points. This module covers MATLAB's Profiler
for finding real bottlenecks, and the concrete optimization techniques
(vectorization, preallocation, avoiding unnecessary copies) that
typically fix what the Profiler finds.

## The Profiler

```matlab
profile on;
result = expensiveAnalysis(data);
profile viewer;    % opens the interactive report
```

The Profiler report ranks functions by **self time** (time spent in the
function's own code, excluding calls to other functions) and **total
time** (self time plus everything called underneath it) — self time is
usually the more actionable number, since it points to the specific
lines doing the actual work, rather than a wrapper function that merely
calls something slow.

```matlab
p = profile('info');    % programmatic access instead of the GUI
[~, idx] = sort([p.FunctionTable.TotalTime], 'descend');
for i = 1:5
    fprintf('%s: %.3f s\n', p.FunctionTable(idx(i)).FunctionName, p.FunctionTable(idx(i)).TotalTime);
end
```

## `tic`/`toc` for targeted measurement

```matlab
tic;
resultA = approachA(data);
fprintf('Approach A: %.4f s\n', toc);

tic;
resultB = approachB(data);
fprintf('Approach B: %.4f s\n', toc);
```

For comparing two specific implementations of the same task,
`tic`/`toc` around each is faster to set up than the full Profiler and
sufficient when you already suspect which function is the bottleneck —
use the Profiler first to *find* the bottleneck across a whole program,
then `tic`/`toc` to compare candidate fixes for that specific
bottleneck.

```matlab
% average over several runs — single-run timing is noisy (OS scheduling, JIT warm-up)
n = 100;
times = zeros(1, n);
for i = 1:n
    tic;
    approachA(data);
    times(i) = toc;
end
fprintf('Median: %.4f s, std: %.4f s\n', median(times), std(times));
```

The first call to a function is often slower than subsequent calls
(MATLAB's JIT compiler optimizes hot code paths after repeated
execution) — always discard or account for a "warm-up" run when
comparing timings.

## Preallocation: the single biggest common fix

```matlab
% BAD: array grows by one element every iteration
result = [];
for i = 1:10000
    result(end+1) = i^2;   % MATLAB reallocates and copies the WHOLE array, every iteration
end
```

```matlab
% GOOD: allocate the final size up front
result = zeros(1, 10000);
for i = 1:10000
    result(i) = i^2;       % writes into existing memory, no reallocation
end
```

Growing an array one element at a time is quadratic overall (each growth
step copies everything so far), even though each individual step
"looks" cheap — for 10,000 elements this is the difference between
roughly 10,000 reallocation-and-copy operations totaling tens of
millions of element copies versus 10,000 direct writes into
pre-existing memory. MATLAB's Code Analyzer (the editor's built-in
linter) flags this pattern with an explicit "array is growing on every
loop iteration" warning precisely because it's such a common,
significant, and easy-to-fix cost.

The same principle applies to growing cell arrays, struct arrays, and
appending rows to a table — preallocate whenever the final size is
knowable before the loop starts.

## Vectorization

```matlab
% loop version
n = 1000000;
x = rand(1, n);
y = zeros(1, n);
for i = 1:n
    y(i) = sin(x(i)) * cos(x(i));
end
```

```matlab
% vectorized version — same result, operates on the whole array at once
y = sin(x) .* cos(x);
```

Vectorized code avoids MATLAB interpreter loop overhead (bytecode
dispatch on every iteration) by delegating the actual work to
MATLAB's underlying compiled array-operation routines, which process
whole arrays with much less per-element overhead. For simple
elementwise math like this, vectorization is typically the single
largest speed-up available without leaving MATLAB entirely (larger than
`parfor`'s multi-core speed-up, and compatible with combining both).

```matlab
% logical indexing instead of a loop with an if-check
x = randn(1, 100000);
% BAD:
result = zeros(size(x));
for i = 1:numel(x)
    if x(i) > 0
        result(i) = x(i)^2;
    end
end
% GOOD:
result = zeros(size(x));
positive = x > 0;
result(positive) = x(positive).^2;
```

## Avoiding unnecessary copies

MATLAB uses copy-on-write for arrays — assigning `b = a` doesn't
actually copy data until either `a` or `b` is modified, at which point
MATLAB makes a private copy for whichever one is mutated. This is
usually invisible and beneficial, but a few patterns defeat it:

```matlab
function modifyInPlace(x)
    x(1) = 0;      % triggers a private copy inside this function — caller's array is untouched
end

a = [1 2 3];
modifyInPlace(a);
disp(a);           % still [1 2 3] — the function's local copy was modified, not the caller's
```

Passing a large array to a function that modifies it always triggers a
copy — if a function needs to mutate large data in place across many
calls (e.g. an iterative solver updating a big matrix each step), a
`handle` class (Module 01, Level 3) avoids the repeated copy, since
handle objects are passed by reference.

## Function handles vs. `eval` / string-based dispatch

```matlab
% SLOW and fragile: constructing and evaluating code as a string
funcName = 'sin';
y = eval([funcName '(x)']);
```

```matlab
% FAST and safe: function handles
f = str2func('sin');
y = f(x);
```

`eval` forces MATLAB to re-parse and interpret a string as code on every
call — far slower than a function handle, which is resolved once. Aside
from performance, `eval` on any string derived from external input is
also a code-injection risk; `str2func`/direct handles (`@sin`) are both
faster and safer wherever dynamic function dispatch is needed at all.

## Preallocating and reusing figure/plot handles

```matlab
% BAD: recreating a figure every iteration inside a live-update loop
for i = 1:100
    figure;
    plot(data(:,i));
    pause(0.05);
end
```

```matlab
% GOOD: create once, update data in place
h = plot(data(:,1));
for i = 1:100
    set(h, 'YData', data(:,i));
    drawnow limitrate;    % throttles rendering, avoids redrawing faster than the screen refresh
end
```

`drawnow limitrate` matters for animation loops specifically — without
it, MATLAB may attempt to render every single frame regardless of how
fast the loop produces them, wasting time on frames the display can't
even show.

## Combining techniques: a realistic before/after

```matlab
% BEFORE: unvectorized, growing array, eval-based dispatch
function result = processBad(data, opName)
    result = [];
    for i = 1:length(data)
        result(end+1) = eval([opName '(data(i))']);
    end
end
```

```matlab
% AFTER: preallocated, vectorized, function handle
function result = processGood(data, opFunc)
    result = opFunc(data);   % opFunc = @sin, or any vectorized function handle
end
```

The "after" version isn't just faster — it delegates the entire
computation to a single vectorized call, sidesteps `eval` entirely, and
removes the preallocation concern altogether by not looping at the
MATLAB level at all.

## Practice

1. Profile (conceptually — describe what you'd expect to see) a
   function that loads a large CSV inside a loop that runs 1000 times
   versus loading it once outside the loop; explain which line the
   Profiler's self-time ranking would surface as the bottleneck.
2. Rewrite a hypothetical function that builds a growing `results{end+1}
   = ...` cell array inside a 50,000-iteration loop to preallocate the
   cell array first, and explain the asymptotic difference in total
   copy operations.
3. Given `y(i) = x(i)^2 + 3*x(i) - 1` inside a `for` loop over a
   200,000-element vector, write the fully vectorized equivalent and
   explain why it avoids interpreter dispatch overhead per element.
4. Explain a scenario, using the copy-on-write behavior described above,
   where switching a large-data-mutating function's class from a value
   class to a `handle` class would measurably reduce memory churn, and
   what caution that introduces (referencing Level 3 Module 01's
   discussion of shared mutable state).
