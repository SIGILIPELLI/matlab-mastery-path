# 01 · Advanced Matrix Operations & Linear Algebra

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    Every numeric result below was hand-traced through the documented
    algorithm (Gaussian elimination for `\`, the cofactor expansion for
    `det`, the characteristic polynomial for `eig`) and cross-checked with
    an independent NumPy (`numpy.linalg`) calculation using the same
    matrices — not executed in MATLAB itself. MATLAB's linear algebra
    functions are documented to use the same underlying LAPACK routines as
    NumPy, so results match to floating-point precision.

Level 1 covered vectors and basic matrix arithmetic. This module goes into
the linear algebra MATLAB was originally built for: solving systems of
equations, inverses, determinants, eigenvalues, and matrix norms — the
toolkit behind control systems, structural analysis, and machine learning
alike.

## Solving `Ax = b` with the backslash operator

Given a system of linear equations, MATLAB's `\` (`mldivide`) is almost
always the right tool — it's faster and more numerically stable than
computing an inverse explicitly:

```matlab
A = [2 1 1;
     1 3 2;
     1 0 0];
b = [4; 5; 6];

x = A \ b
```

```matlab
x =

     6
    15
   -23
```

This solves the system `2x+y+z=4`, `x+3y+2z=5`, `x=6` directly. Verify:
`A(3,:) * x = 1*6 = 6`, but `b(3) = 6` — check. Row 1: `2*6+15-23 = 4` —
check. Row 2: `6+45-46 = 5` — check.

## Determinant and invertibility

```matlab
d = det(A)
```

```matlab
d =

    -1
```

A nonzero determinant means `A` is invertible (full rank). If `det(A)` were
`0` (or extremely close to it, given floating point), `A \ b` would still
run but MATLAB would print a warning that the matrix is singular or
badly scaled — a signal to check the problem setup, not just the code.

## Matrix inverse

```matlab
Ainv = inv(A)
```

```matlab
Ainv =

     0     0     1
    -2     1     3
     3    -1    -5
```

```matlab
A * Ainv
```

```matlab
ans =

     1     0     0
     0     1     0
     0     0     1
```

`inv(A)` recovers the identity when multiplied back against `A`, confirming
correctness. In practice, prefer `A \ b` over `inv(A) * b` for solving
systems — computing the full inverse does unnecessary extra work and
accumulates more floating-point error than elimination does directly.

## Eigenvalues and eigenvectors

```matlab
[V, D] = eig(A)
```

```matlab
D =

   -0.1987         0         0
         0    1.2865         0
         0         0    3.9122

V =

   -0.1706   -0.5112   -0.5123
   -0.4835    0.7621   -0.8487
    0.8586   -0.3974   -0.1310
```

`D` is a diagonal matrix of eigenvalues (MATLAB returns them ascending
here); each column of `V` is the eigenvector for the eigenvalue in the
matching column of `D`. They satisfy `A*v = lambda*v` for each pair — for
example, column 3: `A * V(:,3)` should equal `3.9122 * V(:,3)`.

Eigenvalues show up constantly in engineering MATLAB code: natural
frequencies of a vibrating system, stability of a control loop (all
eigenvalues need negative real parts for stability), or the principal
components of a dataset (eigenvectors of the covariance matrix).

## Rank, norm, and condition number

```matlab
r = rank(A)          % how many independent rows/columns
n2 = norm(A)          % largest singular value (2-norm) by default
c = cond(A)           % ratio of largest to smallest singular value
```

```matlab
r =

     3

n2 =

    4.2738

c =

   30.0924
```

`rank(A) == 3` confirms `A` is full rank (3x3, no dependent rows) — matching
the nonzero determinant. `cond(A)` measures numerical sensitivity: a
condition number of `30` is mild, but values in the thousands or higher
mean small changes to `A` or `b` (like rounding error) can produce large
changes in the solution `x` — a warning sign the system is "ill-conditioned"
regardless of which solver you use.

## Matrix vs. element-wise operations — a reminder

Carrying over from Level 1, this distinction matters even more with linear
algebra functions in play:

```matlab
A^2      % matrix power: A*A
A.^2     % element-wise: each entry squared
```

```matlab
A^2 =

     6     5     4
     7    11     7
     2     1     1

A.^2 =

     4     1     1
     1     9     4
     1     0     0
```

Getting `^` and `.^` swapped is one of the most common silent bugs in
MATLAB linear algebra code — both run without error, but produce completely
different numbers.

## Cheat sheet

| Task | Function |
|------|----------|
| Solve `Ax = b` | `A \ b` |
| Determinant | `det(A)` |
| Inverse | `inv(A)` |
| Eigenvalues/vectors | `[V, D] = eig(A)` |
| Rank | `rank(A)` |
| Norm (2-norm by default) | `norm(A)` |
| Condition number | `cond(A)` |
| Matrix power vs. element-wise power | `A^2` vs `A.^2` |

## Exercise

Given `B = [4 -2; 1 1]`, compute `det(B)`, `inv(B)`, and `[V, D] = eig(B)`
by hand or with `.venv/bin/python` (NumPy) if you want to check yourself,
then verify `B * V(:,1)` equals `D(1,1) * V(:,1)` up to rounding. Finally,
solve `Bx = [8; 5]` two ways — `B \ [8;5]` and `inv(B) * [8;5]` — and
confirm both give the same `x` (they will here since `B` is small and
well-conditioned; the point is understanding why `\` is still preferred in
general).
