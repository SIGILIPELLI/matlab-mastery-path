# 01 · What Is MATLAB?

MATLAB ("MATrix LABoratory") is a numerical computing environment and
programming language built around one central idea: **every value is a
matrix**. A single number is a 1×1 matrix, a list of numbers is a 1×N (or
N×1) matrix, and a table of numbers is an M×N matrix — there's no separate
"scalar" type living apart from that model. This matrix-first design is what
makes MATLAB unusually good at linear algebra, signal processing, control
systems, and any workflow that's naturally expressed as operations on arrays
of numbers.

You interact with MATLAB through its desktop application, which bundles
several panels into one window:

| Panel | Purpose |
|-------|---------|
| **Command Window** | Type expressions and see results immediately — the interactive REPL |
| **Workspace** | Lists every variable currently defined, with its size, type, and value preview |
| **Current Folder** | A file browser scoped to your working directory |
| **Editor** | Where you write and edit `.m` script and function files |
| **Command History** | A log of every command you've run in the Command Window, across sessions |

## The Command Window — MATLAB as a calculator

Typing an expression without a trailing semicolon echoes the result back,
labeled `ans` if you didn't assign it to a variable:

```matlab
>> 3 + 4
ans =

     7

>> 2^10
ans =

        1024

>> sqrt(16)
ans =

     4
```

Ending a line with a semicolon (`;`) suppresses that echoed output — useful
once you're writing longer scripts where you don't want every intermediate
result printed:

```matlab
>> x = 5;
>> y = x * 2
y =

    10
```

Notice `x = 5;` printed nothing, but `y = x * 2` (no semicolon) printed `y`'s
value. This is one of the most common beginner adjustments coming from other
languages: **the semicolon in MATLAB controls display, not statement
termination** — a line without one is still perfectly valid, it just also
echoes its result.

## Scripts vs. functions vs. live scripts

MATLAB code lives in `.m` files, and there are two fundamentally different
kinds:

**Scripts** (`myscript.m`) are just a sequence of commands, run top to
bottom, sharing MATLAB's base workspace — any variable a script creates
stays around in the workspace after the script finishes, exactly as if
you'd typed each line into the Command Window yourself.

```matlab
% analyze.m — a script
data = [12, 45, 7, 89, 23];
average = mean(data);
disp(average)
```

Running `analyze` (typing the filename, no extension, no parentheses) at the
Command Window executes every line in order.

**Functions** (`myfunction.m`) declare inputs and outputs explicitly and run
in their own isolated workspace — variables created inside a function
disappear once it returns, unlike a script's variables:

```matlab
% square_it.m — a function
function result = square_it(x)
    result = x^2;
end
```

```matlab
>> square_it(5)
ans =

    25
>> x
Undefined function or variable 'x'.
```

That last error is expected and important: `x` was a parameter *local* to
`square_it`, not a variable in your base workspace — functions don't leak
their internals the way scripts do. Module 05 covers writing functions in
depth.

**Live scripts** (`.mlx`, created via the Editor's "New Live Script") are a
newer, notebook-style format: code, its output (including inline plots and
formatted tables), and rich narrative text (headings, formatted equations,
images) all live together in one scrollable document, similar in spirit to
a Jupyter notebook. They're excellent for exploratory analysis, tutorials,
and reports where you want the explanation and the result side by side, but
under the hood they're still MATLAB code — anything you can do in a plain
script you can do in a live script, plus richer formatting.

## The `.m` file and the current folder

A script or function only runs by name (e.g. `analyze`, not `analyze.m`) if
it's either in MATLAB's **current folder** (shown in the Current Folder
panel) or somewhere on MATLAB's search **path**. A very common first-day
error is writing a file, saving it somewhere unrelated, then wondering why
MATLAB reports it as undefined:

```matlab
>> analyze
Undefined function or variable 'analyze'.
```

The fix is almost always to check the Current Folder panel and `cd` to
(or open) the folder containing the file, or use `addpath` to add that
folder to MATLAB's search path permanently for the session.

## Getting help without leaving MATLAB

Two commands cover most "how do I use this function" questions:

```matlab
>> help sqrt
 sqrt - Square root
    ...

>> doc sqrt   % opens the same information in a browsable, searchable window
```

`help` prints a plain-text summary right in the Command Window; `doc` opens
the full formatted documentation page, usually with runnable examples —
reach for `doc` when `help`'s summary isn't enough.

## Semicolon, comments, and basic syntax cheat sheet

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

## Exercise

Open MATLAB (or note down the commands if you're reading without it
installed yet). In the Command Window: compute `12 * 8` without a semicolon
and observe the `ans` output; then assign the result of `100 / 7` to a
variable called `result` with a semicolon so nothing prints; then type
`result` alone on a line to display it. Finally, create a script file called
`hello.m` containing `disp('Hello from a script')`, save it in your current
folder, and run it by typing `hello` in the Command Window.
