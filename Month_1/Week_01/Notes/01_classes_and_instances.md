# Python OOP — Topic 1: Classes, Instances & `__init__`

> Part of my Python OOP learning journey. 
> Goal of these notes: not just syntax — *why* each feature exists.

---

## Topic 1: Why Classes Exist

**The problem without classes:**

Imagine storing 50 employees using plain variables:

```python
emp1_first = "bharat"
emp1_last = "bhushan"
emp1_pay = 50000

emp2_first = "Test"
emp2_last = "User"
emp2_pay = 60000
```

This scales badly — 50 employees = 150+ disconnected variables, nothing ties
`emp1_first` and `emp1_pay` together as "one employee," and every function
needs 3+ separate parameters passed in the right order.

**The fix:** a class bundles related **data** (attributes) and **behavior**
(methods) into one reusable unit.

```python
class Employee:
    pass
```

- `class Employee:` defines a **blueprint** — no actual employee exists yet.
- `pass` is just a placeholder so Python doesn't throw a syntax error on an
  empty block. It does nothing at runtime.

**Class vs. Instance**

```python
emp1 = Employee()
emp2 = Employee()
```

- `Employee` = the cookie cutter (blueprint).
- `emp1`, `emp2` = actual cookies (instances) — separate objects in memory.

```python
print(emp1)  # <__main__.Employee object at 0x000001A2B3C4D5E6>
print(emp2)  # <__main__.Employee object at 0x000001A2B3C4D7F8>
```

Different memory addresses confirm they're independent objects, even though
both came from the same class.

---

## Topic 2: Manual Attribute Assignment (and why it breaks)

You *can* attach data directly to an instance:

```python
emp1 = Employee()
emp1.first = 'bharat'
emp1.last = 'bhushan'
emp1.email = 'bharat.bhushan@company.com'
emp1.pay = 50000
```

**Why this is a bad pattern at scale:**

1. 4 lines of manual typing *per employee* — error-prone busywork.
2. A typo like `emp1.frist = 'bharat'` creates a brand-new attribute silently.
   No error at the time of the mistake.
3. Forgetting to set `.pay` causes **no error immediately** — it only blows
   up later, far from the actual bug, when something tries to read it.
4. `email` is *derived* from `first`/`last`, but typed out by hand every
   time — easy for it to become inconsistent with the actual name.

> **Core lesson:** manual, repeated initialization scales linearly with
> mistakes. Nothing in the code *enforces* a consistent contract for what
> every Employee must have.

---

## Topic 3: `__init__` — The Constructor

**The mental bridge:** you already know that regular functions enforce
required arguments:

```python
def add(a, b):
    return a + b

add(5)
# TypeError: add() missing 1 required positional argument: 'b'
```

`__init__` is just a function that Python **automatically calls** the
moment you create an instance — so it gets the same enforcement for free.

```python
class Employee:
    def __init__(self, first, last, pay):
        self.first = first
        self.last = last
        self.pay = pay
        self.email = first + '.' + last + '@company.com'
```

```python
emp1 = Employee('bharat', 'bhushan', 50000)
emp2 = Employee('Test', 'User', 60000)
```

**What `self` actually is:**

Python always passes the newly created object as the **first** positional
argument to `__init__` — regardless of what you name that parameter. We
*call* it `self` by convention, not by requirement.

```python
emp1 = Employee('bharat', 'bhushan', 50000)
# Internally maps to:
#   self  -> the new object being built
#   first -> 'bharat'
#   last  -> 'bhushan'
#   pay   -> 50000
```

If you omit `self` from the parameter list, every value shifts over by one
slot, and the call no longer matches the function's signature:

```python
def __init__(first, last, pay):   # only 3 slots
    ...

# Employee('bharat', 'bhushan', 50000) actually sends 4 things:
# (new object, 'bharat', 'bhushan', 50000) -> 4 items, 3 slots
# TypeError: __init__() takes 3 positional arguments but 4 were given
```

**Design principle — derive vs. store:**

| Situation | What to do | Why |
|---|---|---|
| Value is *computable* from data you already have (e.g. `email` from `first` + `last`) | Derive it | Can never disagree with the source data |
| Value is *independent* real-world info (e.g. `pay`) | Take it as a parameter, store it | No formula can produce it |

If you store a derived value instead of computing it, it can go **stale**:

```python
self.email = first + '.' + last + '@company.com'   # derived, always correct

# vs. manually passed in:
self.email = email   # could be typed wrong, can disagree with first/last
```

**Fail-fast principle:** `__init__` requiring all fields means mistakes
(missing data) are caught immediately as a `TypeError` at creation time —
not later as a mysterious `AttributeError` somewhere deep in the program.

---

## Topic 4: Adding Methods — `full_name()`

**The problem:** without a method, getting a full name means repeating the
same logic everywhere it's needed:

```python
print(emp1.first + ' ' + emp1.last)   # repeated in 10 different places
```

If the display format ever needs to change, you must find and fix every
copy by hand.

**The fix — wrap the logic once, inside the class:**

```python
class Employee:
    def __init__(self, first, last, pay):
        self.first = first
        self.last = last
        self.pay = pay
        self.email = first + '.' + last + '@company.com'

    def full_name(self):
        return '{} {}'.format(self.first, self.last)
        # equally valid: return self.first + ' ' + self.last
```

```python
print(emp1.full_name())   # bharat bhushan
```

Change the formatting once inside `full_name`, and every call site in the
program reflects the update automatically.

**`.format()` vs `+` concatenation:** both work fine for two strings. `+`
needs manual `str()` conversion when mixing in numbers (e.g. `self.pay`);
`.format()` (or f-strings) handle that automatically and scale better as
strings get more complex. Neither is "wrong" for simple cases.

**Why `full_name` is a method, not a stored attribute:**

```python
# BAD — stores a derived value, which can go stale:
self.full_name = first + ' ' + last   # set once in __init__

emp1.first = 'John'
print(emp1.full_name)   # still 'bharat bhushan' — WRONG, stale data!
```

```python
# GOOD — recalculated fresh every call, can never go stale:
def full_name(self):
    return self.first + ' ' + self.last

emp1.first = 'John'
print(emp1.full_name())   # 'John bhushan' — correct
```

> Same underlying rule as `email` in Topic 3: never store a value that is
> 100% derivable from other attributes — compute it on demand instead.

---

## Topic 5: Common Mistakes — Missing `self`, Missing `()`

**Mistake 1 — forgetting `self` in a method definition:**

```python
def full_name():                     # no self
    return '{} {}'.format(self.first, self.last)

emp1.full_name()
# TypeError: full_name() takes 0 positional arguments but 1 was given
```
Python still auto-passes `emp1` as an argument (since it's called via an
instance) — but the function has zero slots to receive it.

**Mistake 2 — forgetting `()` when calling a method:**

```python
print(emp1.full_name)     # no parentheses — does NOT call the method
# <bound method Employee.full_name of <__main__.Employee object at 0x...>>

print(emp1.full_name())   # correct — actually executes it
# bharat bhushan
```

`emp1.full_name` (no parens) only **refers** to the method, bound to
`emp1`, ready to run — it does not execute the function body. No crash
happens, which makes this a **silent bug**: the program runs fine, it just
prints the wrong thing.

> General Python rule (applies outside classes too): writing a function
> name without `()` never runs it — it just gives you the function object.

**Attributes vs. methods — why one needs `()` and the other doesn't:**

| | What it is | Access |
|---|---|---|
| `self.first` | a stored value | `emp1.first` → value directly, no `()` |
| `def full_name(self)` | a set of instructions to run | `emp1.full_name()` → must call it |

`print(emp1)` (no method at all) prints the default
`<__main__.Employee object at 0x...>` — Python doesn't know how you want an
object displayed. Fixed properly later via `__repr__`/`__str__`.

---

## Topic 6: Calling Methods — Instance vs. Class Name

Two ways to call the same method, same result:

```python
emp1.full_name()              # idiomatic — Python auto-passes emp1 as self
Employee.full_name(emp1)      # explicit — you manually pass the instance
```

- `emp1.full_name()` — the dot-on-instance syntax triggers Python's
  auto-fill: `emp1` is silently slotted into `self`.
- `Employee.full_name(emp1)` — calling through the **class** (the
  blueprint, not an object) removes that auto-fill trigger. There's no
  instance "before the dot" for Python to grab, so `self`'s slot must be
  filled manually — same as any normal function call.

Both produce `'bharat bhushan'`. The mechanism differs; the result doesn't.
`emp1.full_name()` is what you'll use 99% of the time — the class-name form
mainly proves that `self` is just a regular parameter, not magic.

---

## Key Takeaways from Lecture 1

- A class is a **blueprint**; an instance is a real, independent object
  built from it.
- `__init__` automates attribute setup and **enforces required data** at
  creation time (fail fast).
- `self` is just the first parameter — Python auto-fills it with "whichever
  object the method was called on."
- **Derive, don't store**, any value computable from existing attributes
  (`email`, `full_name`) — storing it risks stale, contradictory data.
- Methods need `()` to execute; without it, you only get a reference.
- Methods can be called via an instance (auto `self`) or via the class
  name (manual `self`) — same outcome, different syntax.
