# 08 · String & Text Processing

Module 02 introduced the two text types, `char` and `string`. This module
covers the functions you'll actually use to build, inspect, and transform
text — most work on both types, with a few differences worth knowing.

## Building text: concatenation and formatting

```matlab
>> first = "Alice"; last = "Smith";
>> full = first + " " + last          % string concatenation with +
full =

    "Alice Smith"

>> full2 = strcat(first, " ", last)   % works for char AND string
full2 =

    "Alice Smith"
```

For formatted output with embedded values, `sprintf` (build a string) and
`fprintf` (print directly) use C-style format specifiers:

```matlab
>> name = "Bob"; age = 30;
>> msg = sprintf('%s is %d years old', name, age)
msg =

    'Bob is 30 years old'

>> fprintf('%s is %d years old\n', name, age)
Bob is 30 years old
```

| Specifier | Meaning |
|-----------|---------|
| `%s` | text (`char` or `string`) |
| `%d` | integer |
| `%f` | floating point, e.g. `%.2f` for 2 decimal places |
| `\n` | newline |

`sprintf` returns a `char` array (not a `string`), regardless of what type
its arguments were — worth knowing if you chain `sprintf`'s result into
something that specifically expects a `string`.

## Inspecting text

```matlab
>> s = "Hello, World!";
>> length(s)          % total character count
ans =

    13

>> upper(s)
ans =

    "HELLO, WORLD!"

>> lower(s)
ans =

    "hello, world!"

>> strtrim("   padded   ")   % remove leading/trailing whitespace
ans =

    "padded"
```

## Searching within text

```matlab
>> s = "The quick brown fox";
>> contains(s, "quick")
ans =

  logical

   1

>> startsWith(s, "The")
ans =

  logical

   1

>> strfind(s, "o")          % ALL starting positions of a match (numeric array)
ans =

    13    17
```

`contains`, `startsWith`, and `endsWith` return a clean logical `true`/
`false` and are the modern, readable choice for a yes/no check. `strfind`
instead returns every index where a match starts — useful when you need
*where* a substring occurs, not just whether it does.

## Splitting and joining

```matlab
>> csv_line = "Alice,30,Boston";
>> parts = split(csv_line, ",")
parts =

  3x1 string array

    "Alice"
    "30"
    "Boston"

>> strjoin(["Alice", "30", "Boston"], " | ")
ans =

    'Alice | 30 | Boston'
```

`split` on a `string` returns a **column** of pieces by default, not a row
— a common shape surprise if you then try to concatenate the result with
something else expecting a row vector. `strjoin` requires a **cell array**
of `char`, or a `string` array, and always returns `char`.

## Replacing text

```matlab
>> s = "The cat sat on the mat";
>> replace(s, "at", "og")
ans =

    "The cog sog on the mog"

>> regexprep(s, '[aeiou]', '*')   % regex-based replace
ans =

    "Th* c*t s*t *n th* m*t"
```

`replace` does simple literal substring replacement; `regexprep` accepts a
full regular expression pattern for anything more sophisticated (multiple
patterns, character classes, capture groups).

## Converting between text and numbers

```matlab
>> str2double("42.5")
ans =

   42.5000

>> num2str(42.5)
ans =

    '42.5'

>> num2str(pi, 4)          % second argument controls significant digits
ans =

    '3.142'
```

`str2double` on text that isn't a valid number returns `NaN` rather than
erroring — always worth checking for when parsing user-provided or
file-sourced text:

```matlab
>> str2double("not a number")
ans =

   NaN

>> isnan(str2double("not a number"))
ans =

  logical

   1
```

## Comparing text

```matlab
>> strcmp("hello", "hello")     % exact comparison, case-SENSITIVE
ans =

  logical

   1

>> strcmp("Hello", "hello")
ans =

  logical

   0

>> strcmpi("Hello", "hello")    % the 'i' suffix means case-INsensitive
ans =

  logical

   1
```

Plain `==` also works for comparing two `string` values directly (unlike
`char`, where `==` compares character-by-character and requires equal
lengths) — `"hello" == "hello"` gives `true`, but `'hello' == 'hi'` errors
because the `char` arrays have different lengths. `strcmp`/`strcmpi` work
consistently for both types and are the safer default habit.

## Regular expressions with `regexp`

```matlab
>> [tokens] = regexp("Order #4521 shipped", '\d+', 'match')
tokens =

  1x1 cell array

    {'4521'}
```

`regexp`'s behavior depends heavily on its trailing option string
(`'match'`, `'tokens'`, `'start'`, `'names'`, etc.) — `'match'` returns the
actual matched text (in a cell array), which is the most common case for
simple extraction tasks.

## Text processing cheat sheet

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

## Exercise

Given `record = "Alice,29,Engineer"`, use `split` to break it into three
pieces on the comma, convert the age piece to a number with `str2double`,
and use `sprintf` to build the sentence `"Alice is 29 years old and works
as an Engineer."`. Then take the sentence `"The rain in Spain stays mainly
in the plain"` and use `regexp` with pattern `'\w*ain\w*'` and the
`'match'` option to extract every word ending in "ain".
