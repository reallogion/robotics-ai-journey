# Python OOP — Lecture 2: Class Variables vs. Instance Variables
> Builds directly on Lecture 1 (`self`, `__init__`, attribute lookup basics).

---

## Topic 7: Why Class Variables Exist

**Scenario:** the company gives every employee the same annual raise — say
4% — and you want a method to apply it.

**The naive attempt — hardcode the rate directly in the method:**

```python
class Employee:
    def __init__(self, first, last, pay):
        self.first = first
        self.last = last
        self.pay = pay

    def apply_raise(self):
        self.pay = int(self.pay * 1.04)
```

```python
emp1 = Employee('bharat', 'bhushan', 50000)
emp1.apply_raise()
print(emp1.pay)   # 52000
```

This works — but it hides a scaling problem:

1. The rate `1.04` is invisible from outside `apply_raise()`. You can't do
   `emp1.raise_amount` to display it elsewhere — it doesn't exist as an
   attribute at all:
   ```python
   print(emp1.raise_amount)
   # AttributeError: 'Employee' object has no attribute 'raise_amount'
   ```
2. If the rate is hardcoded in multiple places across the codebase (the
   raise method, a report generator, a test file...), changing it next
   year means hunting down **every** occurrence by hand. Miss one, and
   part of the program silently uses the wrong rate — no crash, no
   warning, just quietly wrong numbers.

> Same disease as Topic 2 (duplicated attribute assignment) and Topic 4
> (duplicated `full_name` logic): **a value used in multiple places with no
> single source of truth.**

**The need:** one shared value, in exactly one place, readable from outside
any method, used by every employee.

---

## Topic 8: Creating and Accessing a Class Variable

```python
class Employee:
    raise_amount = 1.04   # class variable — lives OUTSIDE any method

    def __init__(self, first, last, pay):
        self.first = first
        self.last = last
        self.pay = pay
```

`raise_amount = 1.04` sits directly in the class body, not inside
`__init__` or any method — so it belongs to the **class**, not to any one
employee.

**Accessing it inside a method — bare name does NOT work:**

```python
def apply_raise(self):
    self.pay = int(self.pay * raise_amount)   # bare name
```
```
NameError: name 'raise_amount' is not defined
```

Python only auto-searches **local variables and parameters** for a bare
name — `raise_amount` was never one of those, so the lookup never even
reaches the class. (Different from `AttributeError`: that's when an
*object* lacks an attribute; `NameError` is when a *bare name* isn't
findable anywhere reachable.)

**The fix — go through `self.` or the class name:**

```python
def apply_raise(self):
    self.pay = int(self.pay * self.raise_amount)
```

### Why `self.raise_amount` works — attribute lookup order

`self.x` is not a name search — it's a lookup **on an object**, following
this order:
1. Check if the instance (`emp1`) has `x` stored directly on itself.
2. If not found, fall back to checking the **class**.

`raise_amount` was never set via `self.raise_amount = ...` in `__init__`,
so step 1 fails — but step 2 finds it on `Employee`. This fallback chain is
exactly why `self.raise_amount` succeeds even though nothing was ever
stored on the instance itself.

```python
print(Employee.raise_amount)   # 1.04
print(emp1.raise_amount)       # 1.04 — falls back to class
print(emp2.raise_amount)       # 1.04 — falls back to class
```

### Why not just define it locally inside the method?

```python
def apply_raise(self):
    raise_amount = 1.04   # local variable
    self.pay = int(self.pay * raise_amount)
```

This avoids the `NameError`, but solves nothing real:
- A local variable disappears the moment the method finishes — it's not
  attached to the instance or the class, so `emp1.raise_amount` still
  fails with `AttributeError`.
- Any *other* method needing the rate (e.g. a `projected_pay()` method)
  would need its own separate `1.04` line — duplication is back.

**Requirement check across all three options:**

| Placement | Readable without calling a method | Shared by all instances | Single source of truth |
|---|---|---|---|
| Local variable inside a method | ❌ | ❌ | ❌ |
| `self.raise_amount = ...` in `__init__` | ✅ | ❌ (copied into every instance separately) | ❌ |
| Class-level (`raise_amount = 1.04` in class body) | ✅ | ✅ | ✅ |

Only the class-level placement satisfies all three — which is exactly why
it's the correct design.

---

## Topic 9: Class Name vs. `self` — Reading vs. Assigning

**Reading** and **assigning** a class variable through `self` are *not*
symmetric — this is the most important distinction in this lecture.

### Changing it via the class name — affects everyone

```python
Employee.raise_amount = 1.05
```
```python
print(Employee.raise_amount)   # 1.05
print(emp1.raise_amount)       # 1.05 — no instance override, falls back
print(emp2.raise_amount)       # 1.05 — same
```

### Changing it via an instance — creates a personal override

```python
emp1.raise_amount = 1.05
```

This does **not** modify `Employee.raise_amount`. Assignment with a dot
always writes **directly onto the object on the left of the dot** — it
never reaches past the instance to touch the class. Python allows this
freely; no special permission needed (same rule as freely attaching new
attributes in Topic 2).

```python
print(emp1.raise_amount)       # 1.05 — its own private copy now
print(emp2.raise_amount)       # 1.04 — unaffected, still falls back to class
print(Employee.raise_amount)   # 1.04 — unaffected, never touched
```

`emp1` now **shadows** the class variable with its own copy. This is
permanent for `emp1` regardless of future changes to `Employee.raise_amount`.

**The rule:**
| Action | Effect |
|---|---|
| `Employee.raise_amount = 1.05` | Changes the shared value → affects everyone with no personal override |
| `emp1.raise_amount = 1.05` | Creates a personal override on `emp1` only → everyone else unaffected |

### Real use case for instance-level override

A senior/top-performing employee gets a different raise rate than the
company default:

```python
senior_emp.raise_amount = 1.10
```

No new method, no special-case branching — `apply_raise()` written with
`self.raise_amount` automatically picks up `1.10` for this one employee
and `1.04` for everyone else, purely because of the lookup order. If
`apply_raise()` had used `Employee.raise_amount` instead, this override
would be silently ignored — a real and easy-to-make bug.

> **Design takeaway:** use `self.x` inside methods when per-instance
> overrides are a *desired* feature (like `raise_amount`). Use the class
> name directly when the value must never vary per instance (see
> `num_of_employees` below).

---

## Topic 10: Inspecting Namespaces with `__dict__`

`__dict__` shows what's actually stored, proving the lookup behavior
directly instead of just trusting the explanation.

```python
print(emp1.__dict__)
# {'first': 'bharat', 'last': 'bhushan', 'pay': 50000}
```
No `raise_amount` listed — confirms it is **not** an instance attribute;
it isn't stored on `emp1` at all.

```python
print(Employee.__dict__)
```
`raise_amount` appears here instead — confirming it lives on the class.

If `emp1.raise_amount = 1.10` had been set, `emp1.__dict__` would then
show `raise_amount` listed for that instance — visible proof of shadowing.

---

## Topic 11: A Second Class Variable — `num_of_employees`

A different use case from `raise_amount`: a value that must **never** be
overridden per instance — a genuine running total.

```python
class Employee:
    num_of_employees = 0
    raise_amount = 1.04

    def __init__(self, first, last, pay):
        self.first = first
        self.last = last
        self.pay = pay
        Employee.num_of_employees += 1   # note: Employee., NOT self.
```

### Why `self.num_of_employees += 1` would break this

`self.x += 1` secretly expands into **two separate operations**:
```python
self.num_of_employees = self.num_of_employees + 1
```

- **Right side** (`self.num_of_employees`) is a *read* → lookup falls back
  to the class → finds `0`.
- **Left side** (`self.num_of_employees = ...`) is an *assignment* → always
  writes directly onto the instance → creates a **new personal copy**.

Result: every employee ends up with their own private `num_of_employees = 1`,
and `Employee.num_of_employees` never moves past `0`. There is no real
shared count anywhere — just isolated `1`s, each instance "thinking" it's
employee number one.

### The correct version

```python
Employee.num_of_employees += 1
```

Expands to `Employee.num_of_employees = Employee.num_of_employees + 1` —
both the read and the write happen directly on the class, no instance
involved, no shadowing risk. The counter genuinely accumulates:
`0 → 1 → 2 → 3 → ...` correctly across every employee created.

```python
print(Employee.num_of_employees)   # 0, before any employees exist

emp1 = Employee('bharat', 'bhushan', 50000)
emp2 = Employee('Test', 'User', 60000)

print(Employee.num_of_employees)   # 2
```

---

## Key Takeaways from Part 2.

- **Instance variables** are unique per object (`self.first`, `self.pay`).
  **Class variables** are shared across all instances of a class.
- A bare name inside a method only searches local scope — it never
  auto-reaches into the class. Use `self.x` or `ClassName.x` explicitly.
- `self.x` (read) follows lookup order: instance first, then class.
  `self.x = ...` (assign) always writes to the instance directly, even
  if a class variable with the same name exists — this creates shadowing,
  not a modification of the shared value.
- Use `self.x` inside methods when **per-instance override is desired**
  (e.g. a custom raise rate for one employee).
- Use `ClassName.x` (never `self.x`) when a value must be a **true shared
  total or constant** with no per-instance variation (e.g. employee count).
- `instance.__dict__` and `ClassName.__dict__` let you directly inspect
  what's actually stored where — useful for confirming lookup behavior.
