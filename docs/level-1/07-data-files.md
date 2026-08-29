# 07 · Working with Data Files

## Reading a CSV with `readtable`

`readtable` is the modern, recommended way to load tabular data — it
returns a **table**, MATLAB's spreadsheet-like container with named,
typed columns:

```matlab
% sales.csv:
% name,quarter,revenue
% Alice,Q1,120
% Bob,Q1,95
% Alice,Q2,140

data = readtable('sales.csv');
```

```matlab
>> data
data =

  3x3 table

     name      quarter    revenue
    ________    _______    _______

    {'Alice'}    {'Q1' }     120
    {'Bob'  }    {'Q1' }      95
    {'Alice'}    {'Q2' }     140
```

`readtable` auto-detects column names from the header row and infers each
column's type — text columns become `cell` arrays of `char` by default (the
curly braces `{...}` in the display), while `revenue` was correctly
detected as numeric.

## Accessing table data

```matlab
>> data.revenue                  % dot notation: get a whole column
ans =

   120
    95
   140

>> data.revenue(1)               % first value in that column
ans =

   120

>> data(1, :)                    % first ROW, all columns (still a table)
ans =

  1x3 table

     name      quarter    revenue
    ________    _______    _______

    {'Alice'}    {'Q1' }     120

>> data(data.revenue > 100, :)   % logical filtering, same idea as vector indexing
ans =

  2x3 table

     name      quarter    revenue
    ________    _______    _______

    {'Alice'}    {'Q1' }     120
    {'Alice'}    {'Q2' }     140
```

Table filtering works exactly like the logical indexing from Module 03 —
`data.revenue > 100` produces a logical column, and `data(logical_col, :)`
keeps only the matching rows.

## Writing a table back out

```matlab
writetable(data, 'sales_filtered.csv')
```

`writetable` writes headers and values in the same CSV shape `readtable`
expects, so round-tripping (`readtable` → modify → `writetable`) preserves
structure cleanly.

## Reading plain numeric data: `readmatrix` and `csvread`

When a file is pure numbers with no header row or text columns,
`readmatrix` is simpler than `readtable` since it skips the table
machinery entirely and returns a plain numeric matrix:

```matlab
% numbers.csv:
% 1,2,3
% 4,5,6
% 7,8,9

M = readmatrix('numbers.csv');
```

```matlab
>> M
M =

     1     2     3
     4     5     6
     7     8     9
```

`csvread()` is an older function that does something similar but is
formally discouraged in current MATLAB documentation in favor of
`readmatrix`; you'll still see it in older code and tutorials.

## MATLAB's own binary format: `.mat` files

`.mat` files are MATLAB's native way to save and reload entire workspace
variables — any type, any size, with full fidelity (a struct stays a
struct, a table stays a table), which a text format like CSV can't
guarantee:

```matlab
x = [1 2 3];
name = "Alice";
data_table = readtable('sales.csv');

save('my_data.mat', 'x', 'name', 'data_table')   % save specific variables
save('everything.mat')                            % save the ENTIRE workspace
```

```matlab
>> clear                    % wipe the workspace to prove the reload works
>> load('my_data.mat')
>> x
x =

     1     2     3
```

`load('my_data.mat')` restores every variable that was saved into it,
under their original names, directly into the current workspace — a common
pattern for caching an expensive computation's result so you don't have to
redo it every session.

```matlab
S = load('my_data.mat');   % load into a STRUCT instead of the workspace
S.x                         % access via the struct, avoids overwriting existing variables
```

Loading into a struct (by capturing `load`'s return value) is the safer
choice inside a function or script where blindly injecting variables into
the current scope could silently overwrite something already in use.

## Checking whether a file exists first

```matlab
if isfile('sales.csv')
    data = readtable('sales.csv');
else
    error('sales.csv not found in the current folder')
end
```

`isfile()` is a clean, readable existence check (there's also `isfolder()`
for directories); it avoids a harder-to-read `exist('sales.csv', 'file') ==
2` check from older MATLAB code.

## Reading plain text line by line

For formats `readtable`/`readmatrix` don't handle directly, low-level file
I/O is available:

```matlab
fid = fopen('notes.txt', 'r');
line1 = fgetl(fid);      % reads one line, without the newline character
fclose(fid);              % ALWAYS close what you open
```

!!! warning "Always pair `fopen` with `fclose`"
    An open file handle (`fid`) that's never closed can lock the file or
    leak resources across a long session. Wrap risky file operations in a
    `try`/`catch` (Level 2 covers this) that still calls `fclose(fid)` on
    the way out, or prefer the higher-level `readtable`/`readmatrix`
    functions, which manage the file handle for you automatically.

## Data file cheat sheet

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

## Exercise

Create a CSV file `students.csv` with columns `name,score` and four rows of
sample data. Load it with `readtable`, use logical filtering to keep only
rows where `score >= 70`, and write the result to `passing_students.csv`
with `writetable`. Then save the filtered table into a `.mat` file, `clear`
your workspace, reload the `.mat` file into a struct with `S = load(...)`,
and confirm `S.filtered` (or whatever name you used) matches what you
saved.
