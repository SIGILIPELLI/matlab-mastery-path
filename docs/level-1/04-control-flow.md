# 04 · Control Flow

## `if` / `elseif` / `else`

```matlab
score = 78;

if score >= 90
    disp('A')
elseif score >= 80
    disp('B')
elseif score >= 70
    disp('C')
else
    disp('F')
end
% C
```

Every block in MATLAB — `if`, `for`, `while`, `function` — is closed with
the keyword `end` (not braces, not indentation). MATLAB is not whitespace
sensitive; the indentation above is a style convention, not a syntax
requirement, but the Editor auto-indents to match it.

`elseif` is **one word** — `else if` (two words) is technically also legal
in MATLAB, but it opens a *second, nested* `if` block that then needs its
*own* closing `end`, which is almost never what you want. Stick to the
one-word `elseif` for a chain of conditions at the same level.

## Comparison and logical operators

```matlab
5 > 3          % 1 (true)
5 == 5         % 1
5 ~= 3         % 1 -- MATLAB uses ~= for "not equal", not !=
```

For combining conditions, MATLAB has two pairs of operators, and choosing
the right one matters:

```matlab
a = true; b = false;
a && b         % scalar short-circuit AND -- requires SCALAR operands
a || b         % scalar short-circuit OR
a & b          % element-wise AND -- works on arrays, does NOT short-circuit
a | b          % element-wise OR
```

`&&` and `||` only accept scalar (single-value) operands and stop
evaluating as soon as the result is determined (short-circuiting) — this
matters when the second condition would error if evaluated (e.g. checking
`~isempty(x) && x(1) > 0`, where `x(1)` would error on an empty `x` if it
were evaluated regardless). `&` and `|` evaluate **both** sides always and
work element-wise across whole arrays — essential for filtering vectors,
but the wrong choice inside an `if` condition guarding against an error.

```matlab
x = [];
if ~isempty(x) && x(1) > 0    % safe: short-circuits, x(1) never evaluated
    disp('positive')
end
% (nothing printed, no error)
```

## `if` requires a scalar logical condition

```matlab
scores = [95, 60, 74];
if scores > 70
    disp('all passing')
end
```

```
Error using if
Operands to the || and && operators must be convertible to logical
scalar values.
```

An `if` in MATLAB needs one true/false value, not an array of them. To
branch on every element of a vector at once, reach for logical indexing or
`all()`/`any()` instead:

```matlab
if all(scores > 70)
    disp('all passing')
end
% all passing

passing = scores(scores > 70)   % logical indexing: keep only qualifying elements
% passing = 95    74
```

## `for` loops

```matlab
for i = 1:5
    disp(i)
end
% 1
% 2
% 3
% 4
% 5
```

A MATLAB `for` iterates over the **columns** of whatever follows `=` — for
a row vector like `1:5`, each "column" is a single scalar, which is the
common case. Iterating over a matrix hands each column (as a full column
vector) to the loop variable, one at a time:

```matlab
for col = [1 2; 3 4; 5 6]
    disp(col)
end
% 1
% 3
% 5
%
% 2
% 4
% 6
```

```matlab
fruits = {'apple', 'banana', 'cherry'};   % cell array (covered fully in Level 2)
for i = 1:length(fruits)
    fprintf('I like %s\n', fruits{i});
end
% I like apple
% I like banana
% I like cherry
```

!!! warning "Loops vs. vectorization"
    MATLAB is optimized for whole-array operations, not explicit loops.
    `scores * 2` or `scores(scores > 70)` operate on an entire array at
    once, internally in optimized code — a `for` loop doing the same thing
    element-by-element is typically slower and less idiomatic MATLAB.
    Loops are worth learning first because they make the underlying logic
    explicit; you'll see vectorized alternatives in detail in
    [Level 2](../level-2/03-vectorization-vs-loops.md).

## `while` loops

```matlab
count = 1;
while count <= 3
    disp(count)
    count = count + 1;
end
% 1
% 2
% 3
```

## `break` and `continue`

`break` exits a loop immediately; `continue` skips to the next iteration:

```matlab
for i = 1:10
    if i == 5
        break          % stop the loop entirely once i reaches 5
    end
    disp(i)
end
% 1 2 3 4

for i = 1:6
    if mod(i, 2) == 0
        continue       % skip even numbers
    end
    disp(i)
end
% 1 3 5
```

Note `mod(i, 2)`, not `i % 2` — MATLAB has no `%` modulo operator (`%`
means comment); `mod()` is the function for remainder.

## `switch` / `case`

`switch` matches a value against a set of cases — useful when an
`if`/`elseif` chain would otherwise test the same variable repeatedly:

```matlab
function describe_day(day)
    switch day
        case 'Mon'
            disp('Start of the work week')
        case 'Fri'
            disp('Almost the weekend')
        case {'Sat', 'Sun'}     % a cell array of values matches ANY of them
            disp('Weekend!')
        otherwise
            disp('Just a regular day')
    end
end

describe_day('Fri')
% Almost the weekend
describe_day('Sat')
% Weekend!
describe_day('Tue')
% Just a regular day
```

Unlike C or Java's `switch`, MATLAB's does **not** fall through between
cases — each `case` is independent and only its own block runs, so no
`break` statement is needed (or allowed) at the end of a case.

## Control flow cheat sheet

| Construct | Syntax |
|-----------|--------|
| If/elseif/else | `if cond ... elseif cond2 ... else ... end` |
| Scalar AND/OR (short-circuit) | `&&`, `\|\|` |
| Element-wise AND/OR | `&`, `\|` |
| Not equal | `~=` |
| For loop | `for i = 1:n ... end` |
| While loop | `while cond ... end` |
| Exit loop early | `break` |
| Skip to next iteration | `continue` |
| Multi-way branch | `switch x case v1 ... case {v2,v3} ... otherwise ... end` |
| Vector-wide condition | `all(cond)`, `any(cond)` |

## Exercise

Write a script that loops over the vector `nums = [4, 15, 8, 23, 42, 7]`
with a `for` loop, printing `'even'` or `'odd'` for each number using
`mod(nums(i), 2)`. Then rewrite the same result using logical indexing —
`nums(mod(nums, 2) == 0)` for the evens and the complementary expression for
the odds — and confirm both approaches agree. Finally, write a `switch`
statement that takes a grade letter (`'A'`, `'B'`, `'C'`, or anything else)
and displays a matching comment, grouping `'D'` and `'F'` into a single
`case` using cell-array syntax.
