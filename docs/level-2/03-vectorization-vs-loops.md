# 03 · Vectorization vs Loops

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Every code sample below was hand-traced against MATLAB's documented
    array-operator and broadcasting ("implicit expansion") semantics, and
    cross-checked against equivalent NumPy behavior where applicable. It
    was not executed in MATLAB itself.

MATLAB's entire design center is the matrix. That means the fastest, most
idiomatic MATLAB code almost never contains an explicit loop over array
elements — it expresses the whole computation as one array operation.
This module is about recognizing loop patterns that should become
vectorized expressions, and knowing when a loop is still the right tool.

## Why vectorization matters

A `for` loop over `1:N` in MATLAB pays per-iteration overhead: bounds
checks, interpreter dispatch, and (historically) JIT-compilation limits.
A vectorized expression instead calls into MATLAB's underlying compiled
array library once for the whole array. The difference is not cosmetic —
on large arrays it is routinely a 10-100x speedup, and it also tends to
produce shorter, more readable code once you're fluent in the idiom.

```matlab
n = 1e6;
x = 1:n;

% Loop version
tic;
y_loop = zeros(1, n);
for i = 1:n
    y_loop(i) = x(i)^2 + 3*x(i) - 1;
end
t_loop = toc;

% Vectorized version
tic;
y_vec = x.^2 + 3*x - 1;
t_vec = toc;
```

Both produce identical results element-for-element, but `t_vec` is
typically one to two orders of magnitude smaller than `t_loop`. Modern
MATLAB's JIT accelerator narrows this gap for simple loops, but the gap
never fully closes, and vectorized code stays easier to read and to
compose with the rest of the array-oriented toolbox (plotting, reductions,
logical indexing).

## The core idiom: element-wise operators

Recall from Level 1 that `.^`, `.*`, `./` operate element-by-element,
while `^`, `*`, `/` are matrix operations. Vectorization leans on the
element-wise family constantly:

```matlab
x = linspace(0, 2*pi, 5);
y = sin(x) .* exp(-x/4);   % element-wise multiply, not matrix multiply
```

A loop written as:

```matlab
for i = 1:length(x)
    y(i) = sin(x(i)) * exp(-x(i)/4);
end
```

is mechanically translated to the vectorized form by replacing `x(i)` with
`x` and every `*`/`/`/`^` that acts elementwise with its dotted form.
This translation works whenever loop iteration `i` only reads/writes
index `i` — no dependency on `i-1` or accumulation across iterations.

## Logical indexing replaces `if` inside loops

A very common loop pattern is "collect elements meeting a condition":

```matlab
data = [-3, 7, -1, 9, 0, -5, 4];
positives_loop = [];
for i = 1:length(data)
    if data(i) > 0
        positives_loop(end+1) = data(i);
    end
end
```

The vectorized equivalent uses a **logical mask**:

```matlab
mask = data > 0;          % logical array: [0 1 0 1 0 0 1]
positives = data(mask);   % [7 9 4]
```

`data > 0` produces a logical array the same size as `data`. Indexing
`data` with a logical array of matching size selects exactly the `true`
positions, in order — no manual bookkeeping of an output index.

Assignment works the same way, and is where logical indexing really
shines over loops:

```matlab
data(data < 0) = 0;   % clamp all negative values to zero, in place
```

The loop equivalent is four lines with an `if`; the vectorized form is
one line and reads as "wherever `data` is negative, set it to zero."

### `find` when you need indices, not values

Sometimes you need *positions*, not values — `find` returns the linear
indices where a condition holds:

```matlab
idx = find(data > 0);     % [2 4 7]
first_idx = find(data > 0, 1);          % 2 (first match only)
last_idx  = find(data > 0, 1, 'last');  % 7
```

`find(cond, 1)` short-circuits after the first match and is the
vectorized replacement for a loop with a `break` once a condition is hit.

## Accumulation patterns: `cumsum`, `cumprod`, `diff`

Loops that carry a running total across iterations map onto built-in
cumulative functions:

```matlab
values = [10 20 5 15 30];

running_total_loop = zeros(size(values));
total = 0;
for i = 1:length(values)
    total = total + values(i);
    running_total_loop(i) = total;
end
% running_total_loop = [10 30 35 50 80]

running_total = cumsum(values);   % identical result, one call
```

`diff(values)` computes successive differences (`values(2:end) -
values(1:end-1)`), the vectorized inverse of a loop that computes
period-over-period change:

```matlab
diff(values)   % [10 -15 10 15]
```

## Vectorizing 2D operations: `meshgrid` / broadcasting

A common beginner loop evaluates a function of two variables over a grid:

```matlab
xv = -2:0.5:2;
yv = -2:0.5:2;
Z_loop = zeros(length(yv), length(xv));
for i = 1:length(yv)
    for j = 1:length(xv)
        Z_loop(i, j) = xv(j)^2 + yv(i)^2;
    end
end
```

`meshgrid` builds coordinate matrices once, then a single vectorized
expression evaluates the whole grid:

```matlab
[X, Y] = meshgrid(xv, yv);
Z = X.^2 + Y.^2;    % same values as Z_loop, no nested loop
```

Modern MATLAB (R2016b+) also supports **implicit expansion**
(broadcasting): a row vector and a column vector combine directly without
`meshgrid` at all —

```matlab
x = -2:0.5:2;        % 1x9 row
y = (-2:0.5:2)';      % 9x1 column
Z2 = x.^2 + y.^2;     % 9x9, broadcast automatically
```

`Z2` is numerically identical to `Z` above; `x.^2` broadcasts across rows
and `y.^2` broadcasts across columns, and `+` combines them into a full
9x9 matrix. This is the single biggest vectorization upgrade over older
MATLAB code you'll see in legacy scripts still using `meshgrid` /
`repmat` out of habit.

## `arrayfun` and `cellfun`: vectorizing calls to non-vectorizable functions

Sometimes the per-element operation is a function that doesn't support
array inputs natively — often a function with internal branching. `arrayfun`
applies a function to every element of a numeric array and collects the
results; `cellfun` does the same for cell arrays:

```matlab
classify = @(v) merge(v > 0, 'pos', merge(v < 0, 'neg', 'zero'));
% (illustrative only: MATLAB has no `merge`; a real classify would use if/else)

function s = classify_sign(v)
    if v > 0
        s = "pos";
    elseif v < 0
        s = "neg";
    else
        s = "zero";
    end
end

labels = arrayfun(@classify_sign, data);   % applies elementwise, output as an array
```

Because `classify_sign` returns a string scalar (non-uniform across
calls in general), you'd pass `'UniformOutput', false` if outputs vary
in type or size:

```matlab
words = arrayfun(@(v) sprintf('%d units', v), data, 'UniformOutput', false);
% words is a cell array of char row vectors, since sprintf output length varies
```

`arrayfun`/`cellfun` are not automatically faster than an equivalent
`for` loop — they still call the function once per element under the
hood — but they read cleanly as a vectorized statement of intent, and
they compose well with `cellfun(@isempty, c)`-style existing-function
checks over cell arrays.

## When to keep the loop

Vectorization is not a universal replacement. Keep an explicit loop when:

- **The iteration is inherently sequential** — each output depends on the
  *previous* output, such as a discrete-time simulation
  `x(k+1) = f(x(k))` or a recursive filter. There's no way to compute
  `x(k+1)` before `x(k)` exists, so no array expression replaces it.
- **The vectorized form would need to materialize a huge intermediate
  array** — e.g. an `NxN` distance matrix when `N` is in the millions —
  and a loop (possibly chunked) keeps memory bounded.
- **Readability actually suffers.** A three-level broadcasting expression
  that takes a paragraph to explain is worse code than a four-line loop
  with a comment, even if it benchmarks faster.
- **The body has genuine side effects per iteration** — writing files,
  updating a GUI, calling external hardware — where "at position `i`,
  do X" is the actual semantics, not a data transformation.

A recursive/sequential example that must stay a loop:

```matlab
n = 20;
fib = zeros(1, n);
fib(1) = 1; fib(2) = 1;
for i = 3:n
    fib(i) = fib(i-1) + fib(i-2);   % depends on prior loop outputs
end
```

There is no vectorized Fibonacci here (short of a closed-form
Binet's-formula approximation, which trades exactness for speed and isn't
the same computation).

## A worked before/after

Loop version — compute a weighted moving average of window 3:

```matlab
x = [4 8 6 5 9 3 7];
n = length(x);
w = [0.2 0.6 0.2];
y_loop = zeros(1, n-2);
for i = 1:n-2
    window = x(i:i+2);
    y_loop(i) = sum(window .* w);
end
```

Vectorized version using `conv`:

```matlab
y = conv(x, fliplr(w), 'valid');
```

`conv` flips one input internally (convolution vs. correlation), so
`fliplr(w)` cancels that flip and reproduces the same weighted-sum
semantics as the loop. `'valid'` restricts output to positions where the
window fully overlaps `x`, matching the loop's `n-2` output length.
Both give `y_loop = y = [0.2*4+0.6*8+0.2*6, ...] = [6.8, 6.6, 6.6, 6.8, 5.6]`.

## Summary

| Loop pattern | Vectorized replacement |
|---|---|
| `for i, y(i) = f(x(i))` elementwise math | `.^`, `.*`, `./` on whole arrays |
| `if` inside loop collecting matches | logical indexing: `x(x > 0)` |
| loop tracking a running total | `cumsum`, `cumprod` |
| loop computing successive differences | `diff` |
| nested loop over a 2D grid | `meshgrid` + `.^`/`.*`, or implicit expansion |
| loop calling a non-vectorized function per element | `arrayfun` / `cellfun` |
| loop where output(k) depends on output(k-1) | keep the loop |

The practical workflow: write the loop first if that's the clearest way
to think through the logic, get it correct on a small test case, then ask
"does iteration `i` only touch index `i`?" — if yes, vectorize; if the
iteration threads state forward, leave it as a loop.
