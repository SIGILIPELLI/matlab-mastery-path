# 04 · Writing Robust Functions

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Function-argument validation syntax (`arguments` blocks), error
    identifiers, and `nargin`/`nargout` semantics are documented,
    version-stable MATLAB language features, hand-traced here against the
    MATLAB Language Fundamentals documentation rather than run in MATLAB.

Level 1 covered writing basic functions with fixed input/output lists.
Real functions — the ones you reuse across projects and hand to
teammates — need to defend themselves against bad input, support
optional arguments, and fail with messages that explain *what went
wrong*, not just crash on the wrong line. This module covers the tools
MATLAB gives you for that.

## Anatomy of a robust function file

```matlab
function result = safe_divide(a, b)
%SAFE_DIVIDE Divide a by b, guarding against division by zero.
%   RESULT = SAFE_DIVIDE(A, B) returns A/B elementwise. Throws an error
%   if any element of B is zero.
%
%   Example:
%       safe_divide([10 20], [2 5])   % returns [5 4]

    if any(b(:) == 0)
        error('safe_divide:divByZero', 'b contains a zero element; division undefined.');
    end
    result = a ./ b;
end
```

Three habits distinguish this from a bare `function result = f(a,b) ...
end`:

1. A **help comment block** immediately after the `function` line (no
   blank line between them) — this is what `help safe_divide` and `doc
   safe_divide` display, and the first line becomes the one-line summary
   shown in `lookfor`/autocomplete tooltips.
2. **Input validation** before doing any work, so failures happen at the
   boundary with a clear message rather than deep inside a computation
   with a cryptic `NaN` or index error.
3. An **error identifier** (`'safe_divide:divByZero'`) — a colon-separated
   component:mnemonic string — rather than just a message. Identifiers let
   calling code catch *specific* failure modes with `try/catch` and
   `err.identifier`, instead of pattern-matching on message text.

## `arguments` blocks (R2019b+)

Modern MATLAB supports a declarative `arguments` block that replaces
manual `if nargin < 2` checks with validation MATLAB enforces
automatically at the call boundary:

```matlab
function y = moving_average(x, windowSize, options)
    arguments
        x (1,:) double
        windowSize (1,1) double {mustBePositive, mustBeInteger} = 3
        options.Mode (1,1) string {mustBeMember(options.Mode, ["same","valid"])} = "same"
    end

    kernel = ones(1, windowSize) / windowSize;
    y = conv(x, kernel, options.Mode);
end
```

Reading this declaration:

- `x (1,:) double` — `x` must be a row vector (`1,:` size spec) of type
  `double`. Passing a column vector or a `single` errors before the
  function body ever runs.
- `windowSize (1,1) double {mustBePositive, mustBeInteger} = 3` — a
  scalar double, validated by two built-in validation functions, with a
  **default value** of `3` if the caller omits it.
- `options.Mode ...` — a field on a `struct`-like trailing `options`
  argument, letting callers pass `Mode="valid"` as a **name-value pair**
  without positional ordering: `moving_average(x, 5, Mode="valid")`.

Calling `moving_average(data)` uses `windowSize=3` and `Mode="same"`
automatically; calling `moving_average(data, 0)` fails immediately with a
`mustBePositive` validation error rather than silently producing garbage
from a zero-length kernel.

### Pre-R2019b style: manual validation

Not all codebases are on a modern release, and understanding the manual
pattern is useful for maintaining legacy code:

```matlab
function y = moving_average_legacy(x, windowSize)
    if nargin < 2
        windowSize = 3;
    end
    validateattributes(x, {'numeric'}, {'row'});
    validateattributes(windowSize, {'numeric'}, {'scalar', 'positive', 'integer'});

    kernel = ones(1, windowSize) / windowSize;
    y = conv(x, kernel, 'same');
end
```

`nargin` (number of input arguments actually supplied at the call site)
is how pre-`arguments` code implements optional parameters and defaults.
`validateattributes` is the imperative counterpart to the declarative
`{mustBePositive, ...}` constraints — same checks, more verbose call
syntax.

## `nargin` and `nargout` for variable arity

Both `nargin` and `nargout` (used without parentheses, inside a function
body) report how many inputs/outputs the *caller* actually supplied or
requested — letting one function adapt its behavior:

```matlab
function varargout = describe_stats(x)
    m = mean(x);
    s = std(x);
    fprintf('mean = %.4f\n', m);
    if nargout >= 1
        varargout{1} = m;
    end
    if nargout >= 2
        varargout{2} = s;
    end
end
```

- `describe_stats(data)` — prints the mean, returns nothing usable (no
  output captured).
- `m = describe_stats(data)` — `nargout` is 1; `varargout{1}` is set to
  the mean and returned as `m`.
- `[m, s] = describe_stats(data)` — `nargout` is 2; both values returned.

`varargin`/`varargout` are cell arrays that soak up any number of
trailing positional inputs/outputs — useful for wrapper functions that
forward arguments to another function:

```matlab
function result = timed_call(fn, varargin)
    tic;
    result = fn(varargin{:});   % {:} expands the cell array into a comma-separated list
    fprintf('elapsed: %.4fs\n', toc);
end

timed_call(@plus, 3, 4);        % forwards 3, 4 to plus(3,4)
```

## Defensive checks that matter in practice

**Type and shape mismatches** are the most common runtime failure in
numeric code:

```matlab
function C = matmul_checked(A, B)
    if size(A, 2) ~= size(B, 1)
        error('matmul_checked:dimMismatch', ...
            'Inner dimensions must agree: A is %dx%d, B is %dx%d.', ...
            size(A,1), size(A,2), size(B,1), size(B,2));
    end
    C = A * B;
end
```

Compare this to letting MATLAB's own error surface: `A * B` alone
produces `Error using * ... Inner matrix dimensions must agree.` — correct
but generic. The wrapped version reports the *actual offending sizes*,
which matters a great deal when `A` and `B` are computed several function
calls upstream from where the mismatch is discovered.

**NaN/Inf propagation** silently corrupts downstream results without
ever throwing an error, since NaN arithmetic is "valid" IEEE-754
arithmetic. Guard explicitly where NaNs shouldn't occur:

```matlab
function y = safe_log(x)
    if any(x(:) <= 0)
        error('safe_log:nonPositive', 'safe_log requires all elements > 0.');
    end
    y = log(x);
end
```

## `try`/`catch` and structured error handling

```matlab
function result = load_and_process(filename)
    try
        data = readmatrix(filename);
    catch ME
        switch ME.identifier
            case 'MATLAB:readtable:FileNotFound'
                error('load_and_process:missingFile', 'File not found: %s', filename);
            otherwise
                rethrow(ME);   % unknown failure: don't hide it, propagate with full stack
        end
    end
    result = mean(data, 1);
end
```

`ME` (a `MException` object) carries `.identifier`, `.message`, and
`.stack`. Branching on `.identifier` handles *known* failure categories
specifically; `rethrow(ME)` preserves the original error's identity and
stack trace for anything unanticipated, rather than swallowing it or
re-wrapping it into a less-informative generic error. Never leave a
`catch` block empty — a silently swallowed error is far harder to debug
than a program that stops with a clear message.

## Assertions for internal invariants

`assert` is for conditions that should be **impossible** if the rest of
the code is correct — different from `error`, which handles conditions
that are *expected* to occur sometimes (bad user input, a missing file):

```matlab
function y = normalize(x)
    y = (x - mean(x)) / std(x);
    assert(abs(mean(y)) < 1e-10, 'normalize:postcondition', ...
        'Normalized mean should be ~0, got %.2e', mean(y));
end
```

If this assertion ever fires, it indicates a bug in `normalize` itself
(or a pathological input like all-identical values causing `std(x)==0`),
not a user mistake — which is exactly the distinction that should guide
`assert` vs `error` in your own code.

## Summary checklist for a "robust enough" function

- [ ] Help comment block (H1 line + description) immediately after
      `function`
- [ ] Input validation via `arguments` block (or `validateattributes` on
      older releases) — fail at the boundary, not three calls deep
- [ ] Meaningful, namespaced error identifiers (`'myFunc:reason'`) instead
      of bare `error('message')`
- [ ] Explicit handling of edge cases relevant to the domain (division by
      zero, empty input, size mismatches, NaN/Inf)
- [ ] `try/catch` only where you can add value (translate, retry, or
      convert to a clearer error) — otherwise let errors propagate
- [ ] `assert` reserved for internal invariants, `error` for
      caller-facing problems

These habits cost a few extra lines per function and pay for themselves
the first time someone (including future you) calls the function with
input it wasn't quite designed for.
