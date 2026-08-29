# 05 · Functions in MATLAB

## Basic function syntax

A function file's name must match its function name exactly (`square_it.m`
must define `function ... = square_it(...)`):

```matlab
% square_it.m
function result = square_it(x)
    result = x^2;
end
```

```matlab
>> square_it(5)
ans =

    25
```

Since R2016b, you can also define **local functions** inside a script file
(after the main script code) or write a whole file as a **function file**
where the function itself is the entire file — both are common; script-local
functions are convenient for helper logic you don't want to publish as its
own file.

## Multiple input and output arguments

```matlab
function [total, average] = summarize(values)
    total = sum(values);
    average = mean(values);
end
```

```matlab
>> [t, a] = summarize([10, 20, 30])
t =

    60

a =

    20
```

Calling `summarize([10, 20, 30])` with only one output captures just
`total` — MATLAB doesn't error, it silently gives you the first output only:

```matlab
>> t = summarize([10, 20, 30])
t =

    60
```

If you want the second output but not the first, use `~` as a placeholder:

```matlab
>> [~, a] = summarize([10, 20, 30])
a =

    20
```

## Default-like behavior with `nargin`

MATLAB has no default-parameter syntax in the function signature itself.
Instead, `nargin` (a built-in that reports how many inputs were actually
passed) lets a function detect a missing argument and supply its own
default:

```matlab
function y = scale_value(x, factor)
    if nargin < 2
        factor = 2;    % default when the caller omits the second argument
    end
    y = x * factor;
end
```

```matlab
>> scale_value(5)
ans =

    10

>> scale_value(5, 3)
ans =

    15
```

`nargin` (with no arguments, called from inside a function) always reports
how many arguments *this specific call* actually supplied — not how many
the function is declared to accept.

## `nargout` — how many outputs the caller wants

Symmetrically, `nargout` reports how many output values the caller
requested, which matters if computing a second output is expensive and
should be skipped when not needed:

```matlab
function [result, details] = analyze(x)
    result = mean(x);
    if nargout > 1
        details = struct('min', min(x), 'max', max(x));  % only computed if asked for
    end
end
```

```matlab
>> m = analyze([1 5 9])
m =

     5

>> [m, d] = analyze([1 5 9])
m =

     5

d =

  struct with fields:

    min: 1
    max: 9
```

## Variable numbers of arguments: `varargin` and `varargout`

```matlab
function total = add_all(varargin)
    total = 0;
    for i = 1:length(varargin)
        total = total + varargin{i};
    end
end
```

```matlab
>> add_all(1, 2, 3)
ans =

     6

>> add_all(1, 2, 3, 4, 5)
ans =

    15
```

`varargin` collects any number of trailing arguments into a **cell array**
(hence indexing with curly braces `{i}`, not parentheses — cell arrays are
covered fully in Level 2). `varargin` must be the *last* parameter in the
function's signature; any named parameters before it are matched normally.

## Anonymous functions

For small, throwaway expressions, an anonymous function avoids creating a
separate `.m` file:

```matlab
>> square = @(x) x^2;
>> square(5)
ans =

    25

>> add = @(x, y) x + y;
>> add(3, 4)
ans =

     7
```

Anonymous functions **capture** the value of any variable from the
enclosing workspace at the moment they're defined — changing that variable
afterward does not affect the already-created function:

```matlab
>> factor = 10;
>> multiply = @(x) x * factor;
>> factor = 100;       % changing factor AFTER defining multiply...
>> multiply(5)          % ...has no effect: still uses factor=10, captured earlier
ans =

    50
```

This is a common surprise: `multiply` "remembers" `factor` as it was at
definition time, not as a live reference to the variable.

## Local scope — functions don't see the caller's workspace

```matlab
function show_x()
    disp(exist('x', 'var'))    % 0 if 'x' does not exist in THIS function's scope
end
```

```matlab
>> x = 42;
>> show_x()
     0
```

Even though `x` exists in the base workspace when `show_x()` is called, the
function has its own separate workspace and cannot see `x` unless it's
passed in as an argument — this isolation is exactly what makes functions
safer and more predictable to reason about than scripts, which do share the
base workspace.

## Recursion

Functions can call themselves, as in any language:

```matlab
function result = factorial_r(n)
    if n <= 1
        result = 1;
    else
        result = n * factorial_r(n - 1);
    end
end
```

```matlab
>> factorial_r(5)
ans =

   120
```

(MATLAB also has a built-in `factorial()` — this is purely for illustrating
recursion.)

## Function cheat sheet

| Task | Syntax |
|------|--------|
| Basic function | `function out = name(in) ... end` |
| Multiple outputs | `function [a, b] = name(x) ... end` |
| Call, ignore an output | `[~, b] = name(x)` |
| Count of inputs actually passed | `nargin` |
| Count of outputs requested | `nargout` |
| Default value for omitted arg | `if nargin < N, arg = default; end` |
| Variable-length input | `function out = f(varargin) ... end` |
| Anonymous function | `f = @(x) x^2;` |
| Check if a variable exists in scope | `exist('name', 'var')` |

## Exercise

Write a function `stats_report(values, verbose)` where `verbose` is
optional (default `false`, via `nargin`). It should always return the mean
as its first output; when `verbose` is `true`, it should also return a
second output — a struct with `min`, `max`, and `std` fields — but only
compute that struct when `nargout > 1` (i.e., only when the caller actually
asks for it). Test it three ways: `m = stats_report([2 4 6 8])`,
`[m, d] = stats_report([2 4 6 8], true)`, and confirm calling with one
output never triggers the struct computation (add a `disp('computing
details')` inside the `nargout > 1` branch to observe this).
