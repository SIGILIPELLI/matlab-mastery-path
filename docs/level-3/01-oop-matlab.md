---
description: "Object-Oriented Programming in MATLAB — Everything through Level 2 used functions and structs to organize data and behavior separately. classdef lets you…"
---

# 01 · Object-Oriented Programming in MATLAB

!!! note "Verification note"
    MATLAB was not available in the environment used to write this page.
    `classdef` syntax and semantics (properties, methods, handle vs.
    value classes, inheritance) are documented, version-stable MATLAB
    language features, hand-traced against the MATLAB OOP documentation
    rather than executed in MATLAB itself.

Everything through Level 2 used functions and structs to organize data
and behavior separately. `classdef` lets you bundle data (**properties**)
and the operations on that data (**methods**) into one reusable
definition — a **class** — and create as many independent instances
(**objects**) of it as needed.

## A minimal class

```matlab
classdef BankAccount
    properties
        Balance = 0
        Owner = ""
    end

    methods
        function obj = BankAccount(owner, initialBalance)
            if nargin > 0
                obj.Owner = owner;
                obj.Balance = initialBalance;
            end
        end

        function obj = deposit(obj, amount)
            if amount <= 0
                error('BankAccount:invalidAmount', 'Deposit amount must be positive.');
            end
            obj.Balance = obj.Balance + amount;
        end

        function obj = withdraw(obj, amount)
            if amount > obj.Balance
                error('BankAccount:insufficientFunds', 'Cannot withdraw %.2f from balance %.2f.', amount, obj.Balance);
            end
            obj.Balance = obj.Balance - amount;
        end
    end
end
```

A `classdef` file (`BankAccount.m`, filename must match the class name)
declares `properties` blocks for data fields and `methods` blocks for
functions that operate on instances. The method with the same name as
the class (`BankAccount(...)`) is the **constructor** — called
automatically by `BankAccount(...)` to create a new object.

```matlab
acc = BankAccount("Priya", 100);
acc = acc.deposit(50);      % acc.Balance is now 150
acc = acc.withdraw(30);     % acc.Balance is now 120
disp(acc.Balance);          % 120
```

## Value classes vs. handle classes — the critical distinction

Notice `deposit` and `withdraw` both **return** `obj` and the caller
reassigns `acc = acc.deposit(50)`. This is because a plain `classdef ...
end` (no superclass) defines a **value class** — like a `double` or a
`struct`, every object is copied on assignment and passed to functions by
value. A method that mutates `obj.Balance` internally only changes its
*local copy*; without `obj = acc.deposit(50)` reassigning the result, the
caller's `acc` would be unchanged.

```matlab
acc.deposit(50);     % WRONG for a value class: return value discarded, acc unchanged
acc = acc.deposit(50);   % correct: acc now points to the updated copy
```

**Handle classes** (`classdef BankAccount < handle`) behave like
references/pointers instead — every variable referring to the same
object shares the same underlying data, and methods mutate it in place
without needing to return and reassign:

```matlab
classdef BankAccountH < handle
    properties
        Balance = 0
    end
    methods
        function deposit(obj, amount)
            obj.Balance = obj.Balance + amount;   % mutates the shared object directly
        end
    end
end
```

```matlab
acc = BankAccountH;
acc.deposit(50);          % no reassignment needed — acc.Balance is 50
acc2 = acc;                % acc2 is the SAME object, not a copy
acc2.deposit(25);
disp(acc.Balance);         % 75 — acc "sees" acc2's change, since they're the same object
```

Choosing wrong is a common source of confusing bugs: a value class where
you expected shared mutable state (methods silently "not working" because
you forgot to reassign), or a handle class where you expected an
independent copy (`acc2 = acc` unexpectedly aliasing, so mutating one
mutates both). Rule of thumb: model **things with independent identity
that change over time and get passed around** (a GUI control, a live
device connection, a growing log) as handle classes; model **pure data
values** (a coordinate, a measurement, an immutable configuration) as
value classes.

## Property validation, same idiom as function arguments

```matlab
classdef Rectangle
    properties
        Width  (1,1) double {mustBePositive} = 1
        Height (1,1) double {mustBePositive} = 1
    end
    methods
        function a = area(obj)
            a = obj.Width * obj.Height;
        end
        function p = perimeter(obj)
            p = 2*(obj.Width + obj.Height);
        end
    end
end
```

`Rectangle` here reuses the same `(size) type {validators}` syntax from
Level 2's `arguments` blocks — `properties` blocks support it directly,
so `r.Width = -5` throws a validation error immediately rather than
letting a negative width silently corrupt later `area()` calls.

## Inheritance

```matlab
classdef Square < Rectangle
    methods
        function obj = Square(side)
            obj@Rectangle();     % call the superclass constructor
            obj.Width = side;
            obj.Height = side;
        end
    end
end
```

`Square` inherits `area()` and `perimeter()` from `Rectangle` without
redefining them — the whole point of inheritance is that behavior common
to both stays defined in one place. `obj@Rectangle()` explicitly invokes
the superclass constructor, required whenever the superclass constructor
takes arguments or needs to run before the subclass's own initialization.

```matlab
sq = Square(4);
sq.area()         % 16, using Rectangle's area() unmodified
```

### Overriding a method

```matlab
classdef Circle
    properties
        Radius (1,1) double {mustBePositive} = 1
    end
    methods
        function a = area(obj)
            a = pi * obj.Radius^2;
        end
    end
end
```

`Circle` isn't a subclass of `Rectangle` (a circle isn't a kind of
rectangle) but defines its own `area()` with the same *name* —
**polymorphism**: code that calls `.area()` on any shape object gets the
right computation without needing to know or check which concrete class
it's holding:

```matlab
shapes = {Rectangle(3,4), Square(5), Circle(2)};
for i = 1:numel(shapes)
    fprintf('Area: %.2f\n', shapes{i}.area());
end
% Area: 12.00
% Area: 25.00
% Area: 12.57
```

## Static-like behavior: `Constant` properties and methods without `obj`

```matlab
classdef MathConstants
    properties (Constant)
        Pi = 3.14159265358979
        E  = 2.71828182845905
    end
    methods (Static)
        function r = degToRad(deg)
            r = deg * MathConstants.Pi / 180;
        end
    end
end
```

`Constant` properties belong to the class itself, not any instance —
`MathConstants.Pi` works without ever constructing a `MathConstants`
object. `Static` methods likewise don't take `obj` as their first
argument and are called as `MathConstants.degToRad(90)` — the MATLAB
analog of a "utility function namespaced under a class" pattern common in
other object-oriented languages.

## `properties (Access = private)` — encapsulation

```matlab
classdef Counter < handle
    properties (Access = private)
        Count = 0
    end
    methods
        function increment(obj)
            obj.Count = obj.Count + 1;
        end
        function n = getCount(obj)
            n = obj.Count;
        end
    end
end
```

`Access = private` restricts reading/writing `Count` to code inside the
class's own methods — external code must go through `getCount()`, so the
class controls exactly how its internal state can be modified.
`c.Count` from outside errors with an access-restriction message rather
than silently succeeding, which is the point: it forces callers through
the class's intended interface instead of reaching into internals that
might change in a future version of the class.

## When to reach for `classdef` vs. a struct + functions

| Use `classdef` when... | Use a struct + plain functions when... |
|---|---|
| The data has behavior tightly coupled to it (validation, invariants to maintain) | Data is a simple, mostly-passive bag of fields |
| You need inheritance/polymorphism across several related types | There's only one "kind" of thing, no natural hierarchy |
| Encapsulation (hiding internal representation) actually matters | Any code can safely see/modify every field |
| Object identity and shared mutable state matter (→ handle class) | Everything is naturally copy-by-value data |

Overusing `classdef` for what's really just a data record adds ceremony
(constructor boilerplate, a whole file per type) without benefit — many
well-written MATLAB programs use structs and functions throughout and
only reach for classes when genuine object-oriented structure (shared
behavior across variants, encapsulated mutable state) is actually needed.

## How It Actually Works

MATLAB has **two fundamentally different object models**, and which one a
class uses changes its runtime semantics completely. A `classdef` that
inherits from `handle` produces **reference-semantics** objects — exactly
like the graphics handles from Module 05: a variable holding a handle
object is a pointer to one shared instance, so `b = a` makes `b` and `a`
refer to the *same* object, and `b.prop = 5` is visible through `a` too.
A `classdef` that does **not** inherit from `handle` produces
**value-semantics** objects, behaving like MATLAB's built-in numeric and
struct types: `b = a` triggers the same copy-on-write mechanism covered in
Module 02 — no copy happens at assignment time, but the first mutation to
either `a` or `b` forces a private copy of that object's data before the
change is applied, so afterward they are fully independent.

Method dispatch for `classdef` objects goes through MATLAB's class
metadata system (`meta.class`) at call time: `obj.method(args)` resolves
`method` by walking the object's class hierarchy (checking the object's
own class, then its superclasses in declaration order) to find the first
matching method definition — this lookup happens once per call, not once
per class definition, though MATLAB does cache resolved method handles
internally to avoid repeating the full hierarchy walk on every single
call in a hot loop.

Property validation blocks (`properties (Access = private)
value (1,1) double {mustBePositive} end`, R2021a+) are enforced by
generated setter code the class system inserts automatically — assigning
to that property runs the validation function *before* the new value is
stored, using the same validation-function mechanism as function
`arguments` blocks (Module 04).

*Note: based on MathWorks' documented `classdef` value/handle semantics
and method-resolution order; not executed in MATLAB itself.*

## Summary

- `classdef` bundles `properties` (data) and `methods` (behavior) into a
  reusable type; the constructor method shares the class's name.
- **Value classes** copy on assignment and require methods to return and
  be reassigned (`obj = obj.method(...)`); **handle classes**
  (`< handle`) behave as shared references and mutate in place. Picking
  the wrong one is the most common `classdef` bug.
- Property validators reuse the same `{mustBePositive, ...}` syntax as
  function `arguments` blocks.
- Inheritance (`<`), method overriding, `Constant` properties, and
  `Static` methods give MATLAB the standard OOP toolkit, used when a
  problem genuinely has shared behavior across related types rather than
  as a default way to organize any data.
