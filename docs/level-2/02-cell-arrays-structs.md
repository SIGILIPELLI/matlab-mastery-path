# 02 · Cell Arrays & Structs

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    The behavior described here (indexing rules for `{}` vs `()`, struct
    field access, `struct` array creation) is MATLAB's documented,
    deterministic language semantics — not something that varies by run —
    and was hand-traced against the MATLAB Language Fundamentals
    documentation rather than executed in MATLAB itself.

Level 1 used plain numeric arrays and a little bit of cell arrays for
strings. This module covers the two container types that let you organize
genuinely mixed, heterogeneous, or named data: **cell arrays** and
**structs**.

## Cell arrays: mixed-type containers

A cell array holds *anything* in each slot — different types, different
sizes — unlike a numeric array where every element must be the same type
and matrices must be rectangular.

```matlab
c = {42, 'hello', [1 2 3], {1, 2}};
class(c)
```

```matlab
ans =

    'cell'
```

### `()` vs `{}` — the rule that trips everyone up

- `c(1)` returns a **1x1 cell array** containing the value (a "slice").
- `c{1}` returns the **value itself**, unwrapped.

```matlab
a = c(1)
b = c{1}
```

```matlab
a =

  1x1 cell array

    {[42]}

b =

    42
```

`class(a)` is `'cell'`; `class(b)` is `'double'`. This is the single most
common source of "why is my code getting a `1x1 cell` instead of a number"
bugs — anytime a cell array element behaves unexpectedly downstream, check
whether `()` was used where `{}` was needed.

### Looping over a cell array

```matlab
names = {'Ana', 'Ben', 'Cara'};
for i = 1:length(names)
    fprintf('Hello, %s!\n', names{i});   % {} to get the char array out
end
```

```
Hello, Ana!
Hello, Ben!
Hello, Cara!
```

Using `names(i)` instead of `names{i}` here would pass a `1x1 cell` to
`fprintf`'s `%s`, which errors — `%s` expects a char array, not a cell.

## Structs: named fields

A struct groups related data under named fields, like a lightweight record:

```matlab
student.name = 'Ana';
student.score = 92;
student.passed = true;
```

```matlab
student =

  struct with fields:

       name: 'Ana'
      score: 92
     passed: 1
```

Access fields with dot notation: `student.name` returns `'Ana'`. This reads
far more clearly than tracking which numeric index means what in a plain
array.

## Struct arrays — many records, same shape

```matlab
students(1).name = 'Ana';
students(1).score = 92;
students(2).name = 'Ben';
students(2).score = 78;
students(3).name = 'Cara';
students(3).score = 85;
```

```matlab
students(2)
```

```matlab
ans =

  struct with fields:

       name: 'Ben'
      score: 78
```

`students` is now a `1x3 struct array` — each element has the same fields.
Pull all scores into a plain numeric array with `[students.score]`:

```matlab
all_scores = [students.score]
avg = mean(all_scores)
```

```matlab
all_scores =

    92    78    85

avg =

   85
```

`[students.score]` is a "comma-separated list" expansion — MATLAB expands
`students.score` into `92, 78, 85` as three separate values, and wrapping
them in `[]` concatenates them into one array. This pattern (`[s.field]` or
`{s.field}`) is the standard way to pull one column out of a struct array
without writing an explicit loop.

## Nesting cells and structs

```matlab
record.name = 'Ana';
record.grades = {'A', 'B+', 'A-'};   % cell array inside a struct field
record.grades{2}
```

```matlab
ans =

    'B+'
```

Structs and cells combine freely — a struct field can be a cell array, a
cell array can hold structs, and this nesting is how MATLAB represents
genuinely hierarchical data (JSON parsed with `jsondecode`, for instance,
comes back as nested structs and cells).

## `fieldnames`, `isfield`, and `rmfield`

```matlab
fieldnames(student)
isfield(student, 'score')
student2 = rmfield(student, 'passed');
```

```matlab
ans =

  3x1 cell array

    {'name'  }
    {'score' }
    {'passed'}

ans =

  logical

   1
```

`fieldnames` returns a cell array of field name strings — useful for
generic code that processes structs without knowing their fields in
advance. `isfield` checks existence before accessing a field that might not
be there (safer than a direct access, which errors if the field is
missing).

## Cheat sheet

| Task | Syntax |
|------|--------|
| Create a cell array | `{val1, val2, ...}` |
| Get a value out of a cell | `c{i}` |
| Get a 1x1 cell "slice" | `c(i)` |
| Create/access a struct field | `s.field = val` / `s.field` |
| Build a struct array | `s(1).field = ...`, `s(2).field = ...` |
| Pull one field from all elements | `[s.field]` or `{s.field}` |
| List field names | `fieldnames(s)` |
| Check a field exists | `isfield(s, 'name')` |

## Exercise

Build a `1x4` struct array `inventory` where each element has fields
`item` (char), `qty` (double), and `price` (double), for four made-up
products. Then, without a loop, compute the total inventory value using
`sum([inventory.qty] .* [inventory.price])`, and use a `for` loop with
`{}`-indexing to print each item's name and its `qty * price` subtotal in a
formatted line.
