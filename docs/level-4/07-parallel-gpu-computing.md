# 07 · Advanced Parallel & GPU Computing

!!! note "Verification note"
    MATLAB, Parallel Computing Toolbox, and a CUDA-capable GPU were not
    available in the environment used to write this page. `gpuArray`
    semantics, cluster/job submission patterns, and performance
    characteristics below are documented, version-stable behaviors,
    hand-traced against the MathWorks documentation rather than
    executed against real hardware. Any speed-up figures are
    illustrative orders of magnitude, not measurements.

Level 3 Module 05 covered multi-core CPU parallelism (`parfor`,
`parfeval`, `spmd`). This module extends to GPU acceleration
(`gpuArray`) and scaling parallel work beyond a single machine onto a
compute cluster.

## GPU computing with `gpuArray`

```matlab
x = rand(5000, 5000);
gx = gpuArray(x);          % transfer data to GPU memory

tic;
gy = gx * gx;               % matrix multiply executes ON the GPU
y = gather(gy);              % transfer result back to CPU/host memory
toc;
```

`gpuArray` moves data to GPU memory; MATLAB functions that have a
GPU-aware overload (a large and growing list — most elementwise math,
`fft`, matrix multiplication, many linear algebra and image processing
functions) then automatically execute on the GPU instead of the CPU
when their input is a `gpuArray`, with no separate GPU-specific syntax
needed for those functions. `gather` moves the result back to ordinary
CPU memory (a `gpuArray` doesn't display, index into non-GPU-aware
functions, or interoperate with the rest of MATLAB directly without
`gather`).

### When GPU acceleration actually helps

```matlab
% GOOD candidate: large, simple, highly parallel elementwise/matrix operation
n = 10000;
x = gpuArray(rand(n));
y = sin(x) .* cos(x) + x.^2;   % thousands of independent elementwise operations

% POOR candidate: small data, or an inherently sequential algorithm
x = gpuArray([1 2 3]);   % transfer overhead dwarfs any compute saved on 3 elements
y = x + 1;
```

GPUs have thousands of simple cores optimized for the same operation
applied to massive amounts of data simultaneously (SIMD-style
parallelism), and a fixed, non-trivial overhead to transfer data
across the PCIe bus to and from GPU memory. Small arrays, or
sequential algorithms with heavy branching and data-dependent control
flow, gain little or actively lose performance versus CPU — GPU
acceleration pays off specifically for large-scale, uniform, and
massively parallel numeric workloads (large matrix operations, FFTs,
image/signal processing over big arrays, and deep learning training).

```matlab
gpuDevice()          % inspect GPU properties: memory, compute capability
reset(gpuDevice());  % clear GPU memory if it fills up across a long session
```

### Custom GPU kernels: `arrayfun` and `CUDAKernel`

```matlab
function y = customCompute(x)
    y = x^3 - 2*x + 1;    % scalar-in, scalar-out function
end

x = gpuArray(rand(1, 1000000));
y = arrayfun(@customCompute, x);   % applied element-by-element, executed on the GPU
```

`arrayfun` on a `gpuArray` compiles the scalar function into a GPU
kernel automatically, applying it to every element in parallel — the
closest thing to writing custom GPU code without touching CUDA
directly.

```matlab
kernel = parallel.gpu.CUDAKernel('myKernel.ptx', 'myKernel.cu');
kernel.ThreadBlockSize = [256, 1, 1];
output = feval(kernel, inputGpuArray);
```

For maximum control (and for reusing existing hand-written CUDA code),
`CUDAKernel` loads a precompiled `.ptx` CUDA kernel directly — this is
the escape hatch when `gpuArray`'s built-in overloads and `arrayfun`
aren't expressive enough for a specialized algorithm.

## Scaling beyond one machine: clusters

Everything in Level 3 Module 05 ran on a **local** pool (workers on the
same machine as the client). For workloads exceeding one machine's
cores, Parallel Computing Toolbox integrates with cluster resource
managers:

```matlab
c = parcluster('MyClusterProfile');   % a profile configured for e.g. a Slurm/PBS cluster
job = createJob(c);

createTask(job, @heavyComputation, 1, {inputData1});
createTask(job, @heavyComputation, 1, {inputData2});
createTask(job, @heavyComputation, 1, {inputData3});

submit(job);
wait(job);
results = fetchOutputs(job);
```

A **job** is a collection of independent **tasks**, submitted to a
cluster scheduler which distributes them across available compute
nodes — this is the batch-processing analogue of `parfeval`, scaled to
many machines rather than many cores on one machine, appropriate when a
single workstation's core count is the actual bottleneck.

```matlab
% cluster-based parfor: same syntax, different pool backend
c = parcluster('MyClusterProfile');
parpool(c, 32);    % request 32 workers from the cluster scheduler

parfor i = 1:1000
    results(i) = heavyComputation(i);
end
```

The key point: `parfor` code itself doesn't change moving from a local
pool to a cluster pool — only the pool's origin changes (`parpool('local',
4)` vs. `parpool(c, 32)`), which is precisely why writing
correctly-parallelizable `parfor` loops (Level 3 Module 05's rules)
pays off — that same code scales from a laptop to a cluster without a
rewrite.

## Combining GPU and multi-worker parallelism

```matlab
parfor i = 1:4
    g = gpuDevice(i);          % each worker claims a distinct GPU (if 4 GPUs are available)
    x = gpuArray(dataChunks{i});
    results{i} = gather(processOnGPU(x));
end
```

On a multi-GPU machine, combining `parfor` (one worker per GPU) with
`gpuArray` inside each worker lets independent chunks of work run on
separate GPUs simultaneously — the two parallelism mechanisms compose,
provided each worker is pinned to its own GPU device to avoid
contention.

## Profiling GPU code

```matlab
x = gpuArray(rand(5000));
tic;
y = x * x;
wait(gpuDevice());   % GPU operations are asynchronous — wait() ensures timing includes actual completion
elapsed = toc;
```

GPU operations queue asynchronously and return control to MATLAB
immediately — `tic`/`toc` around a GPU call without `wait(gpuDevice())`
measures only how long it took to *launch* the operation, not how long
it took to *complete*, a common measurement mistake when benchmarking
GPU code.

## Choosing the right scale

| Situation | Approach |
|---|---|
| Independent loop iterations, moderate per-iteration cost, one machine | `parfor` (Level 3, Module 05) |
| Massively parallel, uniform numeric operation over large arrays | `gpuArray` |
| Need custom low-level GPU kernel logic | `arrayfun` on `gpuArray`, or `CUDAKernel` |
| Workload exceeds one machine's cores entirely | cluster `parpool`/jobs via `parcluster` |
| Multiple GPUs available | `parfor` + per-worker `gpuDevice` pinning |

## Practice

1. Explain, referencing the fixed overhead of PCIe data transfer, why
   `gpuArray` applied to a 3-element vector is likely to be slower than
   plain CPU computation, while the same operation on a 10-million
   element vector is likely much faster on GPU.
2. Sketch (in words) a `parfor` loop over 500 independent Monte Carlo
   simulation replications, and describe what would need to change
   (only in the pool creation, per the discussion above) to run it on a
   32-node cluster instead of a 4-core laptop.
3. Identify the bug in this snippet and explain why it produces a
   misleadingly fast-looking timing result:
   ```matlab
   tic;
   y = gpuArray(x) * gpuArray(x);
   t = toc;
   ```
4. Describe a workload that would benefit from combining `parfor`
   (across workers) and `gpuArray` (within each worker) simultaneously,
   and what precaution is needed on a machine with fewer GPUs than
   workers.
