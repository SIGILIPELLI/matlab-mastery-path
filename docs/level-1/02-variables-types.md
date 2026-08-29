# 02 · Variables & Basic Data Types

## Assignment

MATLAB assignment uses `=`, and variable names are created the moment you
first assign to them — no declaration keyword needed:

```matlab
>> age = 30
age =

    30

>> name = "Alice"
name =

    "Alice"
```

Variable names must start with a letter, can contain letters, digits, and
underscores, and are **case-sensitive** — `Age` and `age` are two different
variables. By convention MATLAB code favors `camelCase` or `lower_snake`
names; reserved words like `if`, `for`, and `end` cannot be used as variable
names.

## Numeric types

By default, every number in MATLAB is a **double** (64-bit floating point) —
unlike C or Java, you don't choose `int` vs `float` unless you specifically
need to:

```matlab
>> x = 5;
>> class(x)
ans =

    'double'

>> y = 5.5;
>> class(y)
ans =

    'double'
```

Both `5` and `5.5` are the same underlying type. Integer types exist
(`int8`, `int16`, `int32`, `int64`, and their `uint*` unsigned counterparts)
but must be created explicitly, and are mainly used when interfacing with
hardware, embedded targets, or memory-constrained data:

```matlab
>> z = int32(5);
>> class(z)
ans =

    'int32'

>> z + 2.9      % integer types round arithmetic results, they don't truncate
ans =

  int32

   8
```

`int32(5) + 2.9` rounds to `8` (5 + 2.9 = 7.9, rounded to the nearest
integer), not truncated to `7` — a subtlety worth remembering if you ever
mix integer and double arithmetic.

## `char` vs `string` — MATLAB has two text types

This trips up almost everyone coming from another language. MATLAB has
**both** a legacy `char` array type (single quotes) and a newer `string`
type (double quotes, added in R2016b) that behave differently:

```matlab
>> a = 'hello';       % char array
>> class(a)
ans =

    'char'

>> b = "hello";       % string
>> class(b)
ans =

    'string'
```

A `char` array is really a row vector of character codes — indexing it
gives you individual characters, and concatenating two `char` values with
different treatment than strings requires `strcat` or square brackets:

```matlab
>> a(1)
ans =

    'h'

>> ['hello', ' ', 'world']
ans =

    'hello world'
```

A `string` behaves more like text in other modern languages — you can
concatenate with `+`, and a `string` array can hold multiple independent
text elements, each of arbitrary length, like a cell array of strings but
with cleaner syntax:

```matlab
>> "hello" + " " + "world"
ans =

    "hello world"

>> names = ["Alice", "Bob", "Carol"];
>> names(2)
ans =

    "Bob"
```

!!! warning "Which one should you use?"
    New code should generally prefer `string` (double quotes) — it has
    cleaner semantics and better error messages. But a huge amount of
    existing MATLAB code, and many built-in functions (like `fieldnames`,
    error identifiers, and older file I/O functions), still expect or
    return `char`. Expect to see both, and know that `string(x)` and
    `char(x)` convert between them.

## Logical (boolean) type

```matlab
>> t = true;
>> class(t)
ans =

    'logical'

>> 5 > 3
ans =

  logical

   1

>> class(5 > 3)
ans =

    'logical'
```

A `logical` displays as `1`/`0`, not `true`/`false`, but `class()` confirms
it's a distinct type from `double` — this matters because logical arrays
are used directly for indexing (covered in Module 03).

## Checking and converting types

```matlab
>> x = 5;
>> isa(x, 'double')
ans =

  logical

   1

>> isnumeric(x)
ans =

  logical

   1

>> ischar('hello')
ans =

  logical

   1

>> isstring("hello")
ans =

  logical

   1
```

Conversion functions follow a consistent `type(value)` naming pattern:

```matlab
>> num2str(42)          % number -> char, e.g. for building messages
ans =

    '42'

>> str2double('3.14')   % text -> double
ans =

    3.1400

>> str2num('42')        % text -> double, but evaluates as MATLAB code (avoid on untrusted input)
ans =

    42
```

`str2double` is generally safer than `str2num` for parsing plain numeric
text, since `str2num` runs its input through MATLAB's interpreter — fine for
`'42'`, but a risk if the text ever comes from an untrusted source.

## The `whos` command — inspecting what's in memory

```matlab
>> x = 5; name = "Alice"; flag = true;
>> whos
  Name        Size            Bytes  Class      Attributes

  flag        1x1                 1  logical
  name        1x1               142  string
  x           1x1                 8  double
```

`whos` is the fastest way to answer "wait, what type is this variable
again, and how big is it" without printing the whole value.

## Type cheat sheet

| Type | Created with | `class()` result |
|------|---------------|-------------------|
| Double (default numeric) | `5`, `5.5` | `double` |
| Integer (explicit) | `int32(5)`, `uint8(5)` | `int32`, `uint8`, etc. |
| Char array | `'hello'` | `char` |
| String | `"hello"` | `string` |
| Logical | `true`, `5 > 3` | `logical` |

## Exercise

Create a variable `temperature` holding `98.6` and confirm its class is
`double`. Create a `char` variable `city = 'Boston'` and a `string` variable
`country = "USA"`; concatenate them into one message reading `"Boston, USA"`
using square-bracket `char` concatenation for one version and `+` string
concatenation for another. Finally, use `str2double` to convert the text
`'451'` into a number and add `10` to confirm it behaves as a true numeric
value (not text).
