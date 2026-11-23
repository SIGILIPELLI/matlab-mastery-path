---
title: "MATLAB Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free MATLAB cheat sheet: 11 quick-reference tables from every level of the MATLAB Mastery Path, each linked to its full lesson."
---

# MATLAB Cheat Sheet

Quick-reference tables from **11 lessons** of the free MATLAB Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Is MATLAB?](level-1/01-what-is-matlab.md)

| Syntax | Meaning |
|--------|---------|
| `;` at end of line | Suppress echoing the result |
| `%` | Single-line comment |
| `%{ ... %}` | Block comment (each delimiter on its own line) |
| `...` at end of line | Continue a long statement onto the next line |
| `ans` | Automatic variable holding the last unassigned result |
| `clc` | Clear the Command Window display (not variables) |
| `clear` | Remove all variables from the workspace |
| `who` / `whos` | List workspace variables (`whos` adds size/type/bytes) |

### [02 · Variables &amp; Basic Data Types](level-1/02-variables-types.md)

| Type | Created with | `class()` result |
|------|---------------|-------------------|
| Double (default numeric) | `5`, `5.5` | `double` |
| Integer (explicit) | `int32(5)`, `uint8(5)` | `int32`, `uint8`, etc. |
| Char array | `'hello'` | `char` |
| String | `"hello"` | `string` |
| Logical | `true`, `5 > 3` | `logical` |

### [03 · Vectors &amp; Matrix Operations](level-1/03-vectors-matrices.md)

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

### [04 · Control Flow](level-1/04-control-flow.md)

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

### [05 · Functions in MATLAB](level-1/05-functions.md)

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

### [06 · Plotting &amp; Visualization Basics](level-1/06-plotting-basics.md)

| Task | Function |
|------|----------|
| Line plot | `plot(x, y)` |
| Scatter (unconnected points) | `scatter(x, y)` |
| Bar chart (categories) | `bar(values)` |
| Histogram (distribution of raw data) | `histogram(data)` |
| Axis labels / title | `xlabel()`, `ylabel()`, `title()` |
| Multiple series, one legend | `legend('name1', 'name2')` |
| Keep adding to same axes | `hold on` / `hold off` |
| Gridlines | `grid on` |
| New figure window | `figure` |
| Multiple plots, one figure | `subplot(rows, cols, index)` |
| Save to file | `saveas(gcf, 'name.png')` |

### [07 · Working with Data Files](level-1/07-data-files.md)

| Task | Function |
|------|----------|
| Read CSV into a table | `readtable('file.csv')` |
| Read pure numeric CSV into a matrix | `readmatrix('file.csv')` |
| Write a table to CSV | `writetable(T, 'file.csv')` |
| Save workspace variables | `save('file.mat', 'var1', 'var2')` |
| Save entire workspace | `save('file.mat')` |
| Load `.mat` into workspace | `load('file.mat')` |
| Load `.mat` into a struct (safer) | `S = load('file.mat')` |
| Check a file exists | `isfile('file.csv')` |
| Low-level line-by-line read | `fopen`, `fgetl`, `fclose` |

### [08 · String &amp; Text Processing](level-1/08-string-text-processing.md)

| Task | Function |
|------|----------|
| Concatenate | `a + b` (string), `strcat(a, b)`, `[a, b]` (char) |
| Format with values | `sprintf('%s: %d', name, val)` |
| Length | `length(s)` |
| Upper/lower case | `upper(s)`, `lower(s)` |
| Trim whitespace | `strtrim(s)` |
| Contains / starts / ends | `contains`, `startsWith`, `endsWith` |
| Find position(s) | `strfind(s, pattern)` |
| Split | `split(s, delimiter)` |
| Join | `strjoin(cellOrStringArray, delimiter)` |
| Replace (literal) | `replace(s, old, new)` |
| Replace (regex) | `regexprep(s, pattern, replacement)` |
| Text -> number | `str2double(s)` (returns `NaN` on failure) |
| Number -> text | `num2str(n)` |
| Compare (exact / case-insensitive) | `strcmp(a, b)`, `strcmpi(a, b)` |
| Regex extraction | `regexp(s, pattern, 'match')` |

### [09 · Basic Numerical Methods](level-1/09-numerical-methods.md)

| Task | Function | Needs derivative? |
|------|----------|---------------------|
| Find a root near a guess | `fzero(f, x0)` | No |
| Find a root in an interval (by hand) | Bisection | No |
| Find a root fast (by hand) | Newton's method | Yes |
| Integrate sampled data | `trapz(x, y)` | — |
| Integrate a known function accurately | `integral(f, a, b)` | — |
| Minimize a function | `fminbnd(f, a, b)` | No |

## Level 2 · Intermediate

### [01 · Advanced Matrix Operations &amp; Linear Algebra](level-2/01-advanced-matrix-linear-algebra.md)

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

### [02 · Cell Arrays &amp; Structs](level-2/02-cell-arrays-structs.md)

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

---

Go deeper with the full [MATLAB Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
