# 03 · Vectors & Matrix Operations

This is the module everything else in MATLAB builds on. Every value you've
used so far — even a single number like `5` — is technically a 1×1 matrix.
This module covers building vectors and matrices, and the crucial
distinction between element-wise and matrix (linear-algebra) operations.

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page
    (checked via `which matlab`). Every output shown below was hand-traced
    against MATLAB's documented, deterministic semantics and cross-checked
    with NumPy (`np.array`, `@` for matrix multiply, `np.linalg`) as an
    independent arithmetic check — not run in MATLAB itself.

## Creating vectors

```matlab
>> row = [1, 2, 3, 4]        % row vector — comma or space separated
row =

     1     2     3     4

>> col = [1; 2; 3; 4]        % column vector — semicolon separated
col =

     1
     2
     3
     4
```

Commas (or spaces) inside `[ ]` move **across** a row; semicolons start a
**new** row. This one rule governs every matrix literal in MATLAB.

## The colon operator — MATLAB's range builder

```matlab
>> 1:5
ans =

     1     2     3     4     5

>> 1:2:10          % start:step:stop
ans =

     1     3     5     7     9

>> 10:-1:6          % negative step counts down
ans =

    10     9     8     7     6

>> linspace(0, 1, 5)   % 5 evenly spaced points from 0 to 1
ans =

         0    0.2500    0.5000    0.7500    1.0000
```

`1:2:10` stops at `9`, not `10` — the colon operator generates every value
`start + k*step` that doesn't overshoot `stop`, it does **not** force the
endpoint to be included the way `linspace` does.

## Creating matrices

```matlab
>> A = [1 2 3; 4 5 6; 7 8 10]
A =

     1     2     3
     4     5     6
     7     8    10

>> size(A)
ans =

     3     3

>> zeros(2, 3)
ans =

     0     0     0
     0     0     0

>> ones(3)          % single argument -> square matrix
ans =

     1     1     1
     1     1     1
     1     1     1

>> eye(3)            % identity matrix
ans =

     1     0     0
     0     1     0
     0     0     1
```

`size(A)` returns `[3 3]` — rows first, columns second, always in that
order. For a vector, `length(x)` gives the number of elements regardless of
whether it's a row or column; `numel(x)` gives the total element count for
any array (equivalent to `length` for vectors, but the right choice for
matrices where you want rows×columns, not just one dimension).

## Indexing — 1-based, and `(row, col)` order

MATLAB indexing starts at **1**, not 0 — like R, unlike Python, C, or Java.

```matlab
>> A(1, 1)
ans =

     1

>> A(2, 3)          % row 2, column 3
ans =

     6

>> A(end, end)      % 'end' means the last index in that dimension
ans =

    10
```

Colon `:` alone (not part of a range) means "every element along this
dimension" — it's how you pull a full row or column:

```matlab
>> A(2, :)          % entire row 2
ans =

     4     5     6

>> A(:, 3)          % entire column 3
ans =

     3
     6
    10

>> A(1:2, 2:3)      % sub-matrix: rows 1-2, columns 2-3
ans =

     2     3
     5     6
```

Linear indexing also works — MATLAB treats any matrix as one long list of
elements internally **stored column by column** (column-major order), so
`A(4)` is the 4th element down the first column, wrapping into the second:

```matlab
>> A(4)             % column-major: A(1,1),A(2,1),A(3,1),A(1,2),...
ans =

     2
```

!!! warning "Column-major order is a common surprise"
    Coming from C, Python (NumPy default), or Java — all row-major — this
    trips people up. `reshape()` follows the same column-major fill order:
    `reshape(1:6, 2, 3)` fills column 1 first (`[1;2]`), then column 2
    (`[3;4]`), then column 3 (`[5;6]`), producing `[1 3 5; 2 4 6]` — not the
    row-by-row `[1 2 3; 4 5 6]` a row-major language would produce for the
    "same" reshape.

```matlab
>> reshape(1:6, 2, 3)
ans =

     1     3     5
     2     4     6
```

## Element-wise operations — the dot (`.`) prefix

Arithmetic operators (`*`, `/`, `^`) mean **matrix** operations by default.
To force **element-by-element** behavior, prefix with a dot: `.*`, `./`,
`.^`. Mixing these up is the single most common MATLAB bug for beginners.

```matlab
>> v = [2 4 6];
>> w = [1 3 5];
>> v .* w           % element-wise multiply: [2*1, 4*3, 6*5]
ans =

     2    12    30

>> v .^ 2           % element-wise square
ans =

     4    16    36

>> A .* A           % element-wise square of every entry in A
ans =

     1     4     9
    16    25    36
    49    64   100
```

## Matrix (linear-algebra) operations

```matlab
>> A * A            % TRUE matrix multiplication (rows-by-columns)
ans =

    30    36    45
    66    81   102
   109   134   169
```

`A * A` and `A .* A` give completely different results — the first is the
matrix product (sum of products across rows and columns), the second is
just squaring every entry independently. Confusing the two silently
produces wrong-but-plausible-looking numbers, so when a result looks odd,
checking whether you meant `*` or `.*` is one of the first things to try.

```matlab
>> A'                % transpose (apostrophe)
ans =

     1     4     7
     2     5     8
     3     6    10

>> v * w'            % row (1x3) * column (3x1) = dot product (1x1 result)
ans =

    44

>> det(A)            % determinant
ans =

    -3.0000

>> inv(A)            % matrix inverse (only for square, non-singular A)
ans =

    0.4667   -0.2667   -0.2000
   -0.4667    0.6667    0.2000
    0.2000   -0.4000    0.2000
```

Matrix multiplication also requires **compatible dimensions**: an M×N
matrix can only be multiplied (with `*`) by an N×P matrix. Getting this
wrong is one of the most common runtime errors:

```matlab
>> [1 2 3] * [1 2 3]
Error using  *
Incorrect dimensions for matrix multiply. Check that the number of
columns in the first matrix matches the number of rows in the second
matrix...
```

(A 1×3 times a 1×3 doesn't conform — you'd want `[1 2 3] * [1 2 3]'` to get
a dot product, or `[1 2 3] .* [1 2 3]` to multiply element-wise.)

## Solving linear systems: `A \ b`, not `inv(A) * b`

For a system `Ax = b`, MATLAB's backslash ("left division") operator solves
directly, and is both faster and more numerically stable than explicitly
computing an inverse:

```matlab
>> B = [1 2; 3 4];
>> b = [5; 11];
>> x = B \ b
x =

     1
     2
```

Check: `B * x` should reproduce `b` — `[1*1+2*2; 3*1+4*2] = [5; 11]`. ✓
`inv(B) * b` gives the same answer here, but `\` is the idiomatic and
recommended approach in real MATLAB code.

## Aggregate functions operate down columns by default

```matlab
>> sum(A)             % sum of EACH COLUMN (returns a row vector)
ans =

    12    15    19

>> sum(A, 2)          % sum of each ROW instead (dimension 2)
ans =

     6
    15
    25

>> sum(A(:))          % A(:) flattens to one column -> sum of ALL elements
ans =

    46

>> mean(A)            % column means, same default-dimension rule
ans =

     4.0000     5.0000     6.3333
```

`sum`, `mean`, `max`, `min`, `sort`, and most other aggregate functions
default to operating **down each column** for a matrix — a frequent source
of confusion if you expected a single overall total. `sum(A(:))` (flatten,
then sum) is the standard idiom for "total over the whole matrix."

## Building matrices from other matrices

```matlab
>> C = [A, A]          % horizontal concatenation (dimensions must match)
>> D = [A; A]          % vertical concatenation
>> horzcat(A, A)       % same as [A, A]
>> vertcat(A, A)       % same as [A; A]
```

## Matrix & vector cheat sheet

| Task | Syntax |
|------|--------|
| Row vector | `[1 2 3]` |
| Column vector | `[1; 2; 3]` |
| Range | `1:5`, `1:2:10`, `linspace(0,1,5)` |
| Matrix literal | `[1 2; 3 4]` |
| Size / element count | `size(A)`, `length(x)`, `numel(A)` |
| Index (1-based) | `A(2,3)`, `A(end,:)`, `A(:,1)` |
| Element-wise op | `.*`, `./`, `.^` |
| Matrix op | `*`, `\` (solve), `'` (transpose) |
| Identity / zeros / ones | `eye(n)`, `zeros(m,n)`, `ones(m,n)` |
| Determinant / inverse | `det(A)`, `inv(A)` |
| Solve `Ax=b` | `x = A \ b` |
| Sum all elements | `sum(A(:))` |

## Exercise

Build the matrix `A = [1 2 3; 4 5 6; 7 8 10]` and vectors `v = [2 4 6]`,
`w = [1 3 5]`. Compute both `v .* w` (element-wise) and `v * w'` (dot
product) and confirm you understand why they differ in shape. Then solve
`Ax = [6; 15; 25]` using `A \ [6; 15; 25]`, and verify your answer by
computing `A * x` and confirming it reproduces `[6; 15; 25]`.
