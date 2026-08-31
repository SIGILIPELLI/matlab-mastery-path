# 05 · Parallel Computing in MATLAB

!!! note "Verification note"
    MATLAB and Parallel Computing Toolbox were not available in the
    environment used to write this page. `parfor`, `parpool`, `spmd`,
    and related semantics are documented, version-stable behaviors of
    Parallel Computing Toolbox, hand-traced against the MATLAB
    documentation rather than executed in MATLAB itself. Timing numbers
    given are illustrative orders of magnitude, not measured benchmarks.

Everything so far has run on a single CPU core. Parallel Computing
Toolbox lets MATLAB spread independent work across multiple cores (or
multiple machines, or a GPU) so a workload that would take minutes takes
seconds. This module covers `parfor`, pools, `parfeval` for asynchronous
work, and `spmd` for explicit distributed-memory style programming.

## Why parallelism needs independence

A `for` loop is parallelizable only when iterations don't depend on each
other's results. Compare:

```matlab
% NOT safely parallelizable: iteration i depends on iteration i-1
x = zeros(1, 100);
x(1) = 1;
for i = 2:100
    x(i) = x(i-1) * 1.01;   % running recurrence
end
```

```matlab
% Safely parallelizable: each iteration is independent
results = zeros(1, 1000);
for i = 1:1000
    results(i) = expensiveComputation(i);   % result(i) doesn't touch result(j)
end
```

`parfor` only helps with the second pattern. MATLAB enforces this at
parse time for a `parfor` loop — it rejects loops with unclear
cross-iteration dependencies (a **loop variable classification** error)
rather than silently producing wrong results.

## `parfor`: parallel for-loops

```matlab
tic;
n = 2000;
results = zeros(1, n);
parfor i = 1:n
    results(i) = sum(primes(i * 1000));   % independent, CPU-bound per iteration
end
toc;
```

The first time `parfor` runs, MATLAB starts a **parallel pool** (a set of
worker processes, default equal to the number of physical cores) if one
isn't already open. Workers each get a copy of the workspace variables
they need (via serialization), execute a slice of the iteration range,
and return results, which MATLAB reassembles in order.

```matlab
p = gcp('nocreate');     % get current pool without starting one
if isempty(p)
    p = parpool('local', 4);   % explicit pool of 4 workers
end
```

### Rules `parfor` enforces

- **Loop body must be independent across iterations** — no iteration may
  read a value another iteration wrote in the same run (with the single
  exception of reduction variables, below).
- **Sliced output variables**: `results(i) = ...` is allowed because
  MATLAB can prove each iteration writes a disjoint slice. `results(i) =
  results(i-1) + 1` is rejected.
- **Reduction variables**: an accumulator combined with a commutative,
  associative operator is allowed:

```matlab
total = 0;
parfor i = 1:1000
    total = total + expensiveComputation(i);   % '+' is a recognized reduction
end
```

MATLAB detects the `total = total + ...` pattern and treats it specially
— each worker accumulates a partial sum, and MATLAB combines the
partials afterward. The order of summation is not guaranteed, so this is
only safe for operations where order doesn't affect the (numerical)
result — not exactly true for floating point, but close enough for most
purposes.

- **Broadcast variables**: any variable read but never assigned inside
  the loop (e.g., a lookup table used by every iteration) is copied once
  to every worker.
- **Temporary variables**: a variable assigned fresh every iteration and
  not used afterward is private to each worker.

### What `parfor` cannot do

```matlab
parfor i = 1:10
    figure;              % NOT ALLOWED reliably — graphics are session-bound
end

parfor i = 1:10
    x(i).field = i;      % nested indexing into a sliced variable often rejected
end

parfor i = 2:100
    x(i) = x(i-1) + 1;   % rejected: cross-iteration dependency
end
```

`parfor` bodies also cannot contain `break`, nested `parfor`, or
`global`/`persistent` variable declarations, because there's no
guaranteed sequential control flow to break out of.

## Overhead: when `parfor` is *slower*

Every worker copy of data and every result gather across a process
boundary costs time. For lightweight iterations, that copying overhead
dwarfs the compute saved:

```matlab
% BAD: parfor overhead exceeds the ~microsecond loop body cost
n = 100000;
result = zeros(1, n);
parfor i = 1:n
    result(i) = i^2;      % trivially cheap — plain 'for' would be faster
end
```

Rule of thumb: `parfor` pays off when each iteration does at least tens
of milliseconds of real work, or the iteration count is large enough
that total work is substantial even per-iteration cost is small. Profile
before parallelizing — `tic`/`toc` a serial `for` version first.

## `parfeval`: asynchronous, non-blocking parallel work

`parfor` blocks until all iterations finish. `parfeval` submits one
function call to run on a worker and immediately returns a `parallel.
Future` object, letting the client continue (e.g. update a UI, submit
more work, or check on other jobs) while it computes:

```matlab
p = gcp;
f1 = parfeval(p, @() heavyComputation(1), 1);   % 1 = number of output args
f2 = parfeval(p, @() heavyComputation(2), 1);
f3 = parfeval(p, @() heavyComputation(3), 1);

% do other work here while workers compute...

results = cell(1, 3);
for k = 1:3
    [completedIdx, value] = fetchNext([f1, f2, f3]);  % waits for whichever finishes first
    results(completedIdx) = {value};
end
```

`fetchNext` blocks until *any* pending future completes, returning its
index and value — useful for a pool of independent jobs finishing in
unpredictable order, like handling responses from several servers.

```matlab
function result = heavyComputation(seed)
    rng(seed);
    result = sum(rand(1e6, 1).^2);
end
```

## `spmd`: single program, multiple data

`spmd` (single program, multiple data) runs the *same* block of code
simultaneously on every worker in the pool, each with its own
`labindex` (its worker number) — closer to explicit distributed-memory
programming than `parfor`'s implicit loop splitting.

```matlab
spmd
    labindex               % each worker prints its own index: 1, 2, 3, 4
    localData = rand(1, labindex * 100);   % each worker builds different-sized data
    localSum = sum(localData);
    total = gplus(localSum);   % global sum across all workers (implicit communication)
end

disp(total{1});   % 'total' is replicated on every worker; index into Composite to read
```

Variables assigned inside `spmd` become **Composite** objects on the
client — `total{1}` retrieves worker 1's value, `total{2}` worker 2's,
and so on, since (outside of collectives like `gplus`) each worker may
hold a different value.

`spmd` also supports point-to-point communication between workers:

```matlab
spmd
    if labindex == 1
        labSend(42, 2);            % worker 1 sends 42 to worker 2
    elseif labindex == 2
        received = labReceive(1);  % worker 2 receives it
    end
end
```

## Distributed arrays

For data too large to duplicate on every worker, a **distributed array**
splits storage itself across workers:

```matlab
spmd
    D = rand(1000, 1000, codistributor('1d', 1));  % row-distributed across workers
    localPart = getLocalPart(D);     % this worker's slice only
    localSize = size(localPart);
end
```

Each worker only holds its own rows in memory — useful once a matrix is
too large for one worker's RAM but fits when spread across several.

## Choosing the right tool

| Pattern | Tool |
|---|---|
| Independent loop iterations, want results gathered automatically | `parfor` |
| Fire-and-continue async jobs, results needed later or as-available | `parfeval` + `fetchNext` |
| Same algorithm on every worker with explicit worker-to-worker communication | `spmd` |
| Data too big for one worker's memory | distributed arrays inside `spmd` |
| Vectorizable numeric ops on huge arrays, GPU available | `gpuArray` (see Level 4) |

## Managing the pool

```matlab
parpool('local', 6);       % start a pool of 6 local workers explicitly
p = gcp('nocreate');       % inspect current pool without creating one
p.NumWorkers               % how many workers are active
delete(gcp('nocreate'));   % shut the pool down and free resources
```

Pools consume memory and startup time (several seconds), so a script
that calls a `parfor` once should generally leave pool management to
MATLAB's automatic start-on-first-use rather than opening and closing a
pool per call.

## Practice

1. Write a serial `for` loop that computes `isprime` counts over ranges
   `1:100000`, `100001:200000`, ..., across 8 chunks, timed with
   `tic`/`toc`. Convert it to `parfor` and reason about the expected
   speed-up given a 4-core pool (bounded by `min(4, 8)` — parallel
   speed-up saturates at the worker count).
2. Identify which of these loop bodies `parfor` would reject and why:
   `a(i) = a(i) + 1`, `total = total + a(i)`, `a(i) = b(i-1)`, `c{i} =
   {i, i^2}`.
3. Rewrite a `parfeval`/`fetchNext` job pool so that as soon as a job
   completes, its result is written directly into a pre-allocated
   results array at the original submission index (not appended in
   completion order).
4. Sketch (in words, since MATLAB isn't available here) an `spmd` block
   where 4 workers each sum a quarter of a 10,000-element vector and
   combine via `gplus`; compare its efficiency to `parfor` for the same
   task and explain when each is preferable.
