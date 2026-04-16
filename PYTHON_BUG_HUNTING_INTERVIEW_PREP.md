# Python Bug Hunting – Technical Interview Prep Guide

> Dense, practical, no fluff. Each section is a category of bug you will see in interviews. Study the buggy version first, diagnose it yourself, then read the analysis. Good luck today.

---

## Table of Contents

1. [Mutable Default Arguments](#1-mutable-default-arguments)
2. [Off-by-One Errors and Incorrect Loop Bounds](#2-off-by-one-errors-and-incorrect-loop-bounds)
3. [Incorrect Use of `is` vs `==`](#3-incorrect-use-of-is-vs-)
4. [Variable Scoping Issues (Closures, Global/Local)](#4-variable-scoping-issues-closures-globallocal)
5. [Exception Handling Anti-Patterns](#5-exception-handling-anti-patterns)
6. [Shallow vs Deep Copy Bugs](#6-shallow-vs-deep-copy-bugs)
7. [Integer vs Float Division Pitfalls](#7-integer-vs-float-division-pitfalls)
8. [Modifying a List/Dict While Iterating](#8-modifying-a-listdict-while-iterating)
9. [Missing or Unintentional None Returns](#9-missing-or-unintentional-none-returns)
10. [Incorrect String/List Indexing or Slicing](#10-incorrect-stringlist-indexing-or-slicing)
11. [Race Conditions and Shared State in Threads](#11-race-conditions-and-shared-state-in-threads)
12. [Class vs Instance Variable Confusion](#12-class-vs-instance-variable-confusion)
13. [Interview Self-Review Checklist](#13-interview-self-review-checklist)

---

## 1. Mutable Default Arguments

### Why This Is a Pitfall

Python evaluates default argument values **once at function definition time**, not on each call. If a mutable object (list, dict, set) is used as a default, every call that relies on that default shares the exact same object. Mutations in one call persist into future calls.

This catches even experienced developers off guard because it violates the intuitive expectation that defaults are "reset" each invocation.

---

### Buggy Code

```python
def add_item(item, cart=[]):
    cart.append(item)
    return cart

order1 = add_item("apple")
order2 = add_item("banana")
order3 = add_item("cherry", [])

print(order1)  # Expected: ['apple']
print(order2)  # Expected: ['banana']
print(order3)  # Expected: ['cherry']
```

### Issues

- `cart=[]` is evaluated once. `order1` and `order2` both point to the **same** list object.
- After both calls, `order1` and `order2` are `['apple', 'banana']` — not what either caller intended.
- `order3` is fine only because an explicit list was passed in.
- This is a **silent data corruption bug**: no exception is raised, the wrong data is returned.

### Corrected Code

```python
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart

order1 = add_item("apple")
order2 = add_item("banana")
order3 = add_item("cherry", [])

print(order1)  # ['apple']
print(order2)  # ['banana']
print(order3)  # ['cherry']
```

### What to Say Out Loud

> "My first flag here is the mutable default argument — `cart=[]`. In Python, defaults are bound at definition time, so all callers that omit `cart` are sharing one list across the entire program's lifetime. The fix is the sentinel pattern: default to `None`, then assign a fresh list inside the body. I'd also ask whether `cart` should be returned at all, or if this should be a method on a Cart class."

---

## 2. Off-by-One Errors and Incorrect Loop Bounds

### Why This Is a Pitfall

Python's `range(n)` produces `0` through `n-1`. Forgetting this — or confusing "length" with "last valid index" — is the most mechanically common bug in interview code. It shows up in loops, slices, binary search implementations, and pagination logic.

---

### Buggy Code

```python
def find_max(numbers):
    max_val = numbers[0]
    for i in range(1, len(numbers) + 1):  # bug here
        if numbers[i] > max_val:
            max_val = numbers[i]
    return max_val

scores = [3, 7, 2, 9, 4]
print(find_max(scores))
```

### Issues

- `range(1, len(numbers) + 1)` iterates `i` up to `len(numbers)`, which is **one past the last valid index**.
- On the final iteration, `numbers[i]` raises `IndexError`.
- Even if the list had one element, the loop body would immediately go out of bounds.

### Corrected Code

```python
def find_max(numbers):
    if not numbers:
        raise ValueError("Cannot find max of empty list")
    max_val = numbers[0]
    for i in range(1, len(numbers)):  # correct upper bound
        if numbers[i] > max_val:
            max_val = numbers[i]
    return max_val

scores = [3, 7, 2, 9, 4]
print(find_max(scores))  # 9
```

> Note: In production Python you'd use `max(numbers)`, but interviews often ask you to implement it manually to test exactly this.

### What to Say Out Loud

> "I see `range(1, len(numbers) + 1)`. My mental model for `range` is always: the stop value is **exclusive**, so the last element reached is `stop - 1`. That means this loop reaches index `len(numbers)`, which is out of bounds. I'd change it to `range(1, len(numbers))`. I'd also add an empty-list guard before `numbers[0]` because that would panic too."

---

## 3. Incorrect Use of `is` vs `==`

### Why This Is a Pitfall

`is` checks **object identity** (same location in memory). `==` checks **value equality** (defined by `__eq__`). They coincide for small integers and interned strings due to CPython implementation details, which creates bugs that work on some inputs and silently fail on others — making them extremely hard to catch in testing.

---

### Buggy Code

```python
def classify_response(code):
    if code is 200:
        return "OK"
    elif code is 404:
        return "Not Found"
    elif code is 500:
        return "Server Error"
    else:
        return "Unknown"

print(classify_response(200))    # May work
print(classify_response(1000))   # Will not match even if == 1000
```

### Issues

- `is` compares object identity, not value. CPython caches small integers (typically -5 to 256), so `code is 200` accidentally works for 200. But `code is 1000` will **never** be `True` because large integers are freshly allocated objects.
- This code produces a `SyntaxWarning` in Python 3.8+ for exactly this reason.
- A related bug: `if user_input is "yes"` fails on user-supplied strings that aren't interned.

### Corrected Code

```python
def classify_response(code):
    if code == 200:
        return "OK"
    elif code == 404:
        return "Not Found"
    elif code == 500:
        return "Server Error"
    else:
        return "Unknown"

print(classify_response(200))   # "OK"
print(classify_response(1000))  # "Unknown"
```

> Legitimate uses of `is`: checking against `None` (`if x is None`), checking against singletons like `True`/`False` when you need strict type checking.

### What to Say Out Loud

> "Every `is` comparison here is a red flag. `is` tests identity — same object in memory — not equality. CPython interns small integers, so `is 200` happens to work, but `is 1000` never will. The fix is `==` throughout. The only correct uses of `is` are comparisons to `None`, `True`, or `False` as explicit singletons, and even then it's a style choice, not a correctness requirement for `True`/`False`."

---

## 4. Variable Scoping Issues (Closures, Global/Local)

### Why This Is a Pitfall

Python resolves variable names at **runtime** using the LEGB rule (Local → Enclosing → Global → Built-in). Closures capture the **variable** (a reference to the cell), not the **value** at the time of closure creation. This means all closures in a loop see the loop variable's **final** value.

---

### Buggy Code

```python
def make_multipliers():
    multipliers = []
    for i in range(5):
        multipliers.append(lambda x: x * i)
    return multipliers

funcs = make_multipliers()
print(funcs[0](10))  # Expected: 0,  Actual: 40
print(funcs[2](10))  # Expected: 20, Actual: 40
print(funcs[4](10))  # Expected: 40, Actual: 40
```

### Issues

- All five lambdas close over the **same** variable `i` in the enclosing scope.
- After the loop completes, `i` is `4`.
- Every lambda evaluates `x * i` lazily at call time, and at that point `i == 4` for all of them.
- This is the classic "late binding closure" bug.

### Deep Dive: What Is Actually Happening

**The memory model**

When Python executes `def make_multipliers()`, the body creates a **local scope**. Inside that scope, `i` is a single variable — one slot in a lookup table. It is not five separate variables; it is one variable that gets rebound to a new integer on each iteration.

When you write `lambda x: x * i`, Python does **not** evaluate `i` at that moment. It creates a function object that contains a pointer to the enclosing scope's lookup table — specifically, to the cell holding `i`. This pointer is called a **closure cell**.

Think of it this way: the lambda doesn't write down the *value* of `i` on a sticky note. It writes down the *address of the drawer* where `i` lives.

**Tracing the loop**

```
Iteration 0: i rebound to 0 → lambda_0 created → holds a reference to the "i" cell
Iteration 1: i rebound to 1 → lambda_1 created → holds a reference to the SAME "i" cell
Iteration 2: i rebound to 2 → lambda_2 created → same cell
Iteration 3: i rebound to 3 → lambda_3 created → same cell
Iteration 4: i rebound to 4 → lambda_4 created → same cell

Loop ends. i = 4. The "i" cell now permanently holds 4.
make_multipliers() returns the list of 5 lambdas.
```

At this point all five lambdas are pointing at the same cell, and that cell contains `4`.

**What each call actually evaluates**

```
funcs[0](10)  →  lambda x: x * i  →  10 * i  →  10 * 4  =  40
funcs[1](10)  →  lambda x: x * i  →  10 * i  →  10 * 4  =  40
funcs[2](10)  →  lambda x: x * i  →  10 * i  →  10 * 4  =  40
funcs[3](10)  →  lambda x: x * i  →  10 * i  →  10 * 4  =  40
funcs[4](10)  →  lambda x: x * i  →  10 * i  →  10 * 4  =  40
```

Every single one returns 40. Not because they are the same lambda object — they are five distinct function objects — but because they all look up `i` in the same place, and that place says `4`.

**You can verify the closure cell directly**

```python
funcs = make_multipliers()
print(funcs[0].__code__.co_freevars)                          # ('i',)
print(funcs[0].__closure__[0].cell_contents)                  # 4
print(funcs[2].__closure__[0].cell_contents)                  # 4
print(funcs[0].__closure__[0] is funcs[2].__closure__[0])     # True
```

That last line is the smoking gun: `funcs[0]` and `funcs[2]` don't just have the same *value* in their closure — they reference the exact same *cell object*.

**Why each fix works**

*Default argument fix (`lambda x, i=i`):* Default arguments are evaluated eagerly at *definition time* — the moment the `lambda` expression is executed inside the loop. On iteration 0, `i=i` captures the value `0` into the default, not a reference to the cell. Each lambda gets its own frozen copy.

```
Iteration 0: i=0 → lambda_0's default i is baked in as 0
Iteration 1: i=1 → lambda_1's default i is baked in as 1
Iteration 2: i=2 → lambda_2's default i is baked in as 2
...
```

*Factory function fix:* Each call to `make_multiplier(i)` creates a brand new scope with its own `n` variable. `lambda x: x * n` closes over `n` in *that* scope, not over the loop's `i`. Since each call to the factory gets a fresh scope, the lambdas never share a cell.

```
make_multiplier(0) → new scope, n=0 → lambda closes over this scope's n
make_multiplier(1) → new scope, n=1 → lambda closes over a DIFFERENT scope's n
make_multiplier(2) → new scope, n=2 → lambda closes over yet another scope's n
```

**The one-line mental model to carry into any interview:**

> Closures close over **variables**, not **values**. Every time you see a lambda or inner function defined inside a loop, ask: "when this is called later, what value will the loop variable hold?"

### Corrected Code — Option A: Default Argument Capture

```python
def make_multipliers():
    multipliers = []
    for i in range(5):
        multipliers.append(lambda x, i=i: x * i)  # captures current value
    return multipliers

funcs = make_multipliers()
print(funcs[0](10))  # 0
print(funcs[2](10))  # 20
print(funcs[4](10))  # 40
```

### Corrected Code — Option B: Factory Function

```python
def make_multiplier(n):
    return lambda x: x * n

def make_multipliers():
    return [make_multiplier(i) for i in range(5)]

funcs = make_multipliers()
print(funcs[0](10))  # 0
print(funcs[2](10))  # 20
print(funcs[4](10))  # 40
```

### What to Say Out Loud

> "This is the late-binding closure trap. Each lambda doesn't snapshot the value of `i` — it holds a reference to the variable `i` itself. Once the loop ends, `i` is 4, so every lambda returns `x * 4`. Two clean fixes: use a default argument `i=i` to bind the current value eagerly, or extract a factory function that creates a new scope per iteration. I prefer the factory function in production because it's less surprising to readers."

---

## 5. Exception Handling Anti-Patterns

### Why This Is a Pitfall

Exception handling is where programs degrade from "broken" to "silently wrong." Bare `except` clauses catch everything including `SystemExit`, `KeyboardInterrupt`, and `MemoryError`. Swallowing exceptions hides bugs. Catching overly broad exception types masks root causes.

---

### Buggy Code

```python
import json

def load_config(path):
    try:
        with open(path) as f:
            data = json.load(f)
        return data["settings"]
    except:
        return {}

config = load_config("config.json")
print(config.get("timeout", 30))
```

### Issues

- **Bare `except`** catches `BaseException`, including `KeyboardInterrupt` and `SystemExit`. A user pressing Ctrl+C during a file read will be silently absorbed.
- If `config.json` doesn't exist (`FileNotFoundError`), the caller gets `{}` with no indication of the failure.
- If `config.json` is malformed JSON (`json.JSONDecodeError`), same silent failure.
- If the key `"settings"` is absent (`KeyError`), same silent failure.
- Returning `{}` for all these distinct failure modes makes debugging production issues nearly impossible.

### Corrected Code

```python
import json
import logging

logger = logging.getLogger(__name__)

def load_config(path):
    try:
        with open(path) as f:
            data = json.load(f)
    except FileNotFoundError:
        logger.warning("Config file not found: %s", path)
        return {}
    except json.JSONDecodeError as e:
        logger.error("Malformed config JSON in %s: %s", path, e)
        return {}

    try:
        return data["settings"]
    except KeyError:
        logger.error("Missing 'settings' key in config: %s", path)
        return {}
```

### What to Say Out Loud

> "The bare `except` is an immediate red flag — it catches `BaseException`, including signals that Python uses to exit the process. I'd always use `except Exception` at minimum, or ideally name the specific exceptions I expect. The bigger issue here is that all failure modes return `{}` silently. The caller has no way to distinguish 'file not found' from 'malformed JSON' from 'wrong schema'. I'd separate the exception types and log with enough context to debug later. Whether to re-raise or return a sentinel depends on whether the caller can meaningfully recover."

---

## 6. Shallow vs Deep Copy Bugs

### Why This Is a Pitfall

Assignment in Python never copies an object — it binds a new name to the same object. `list.copy()` and `dict.copy()` create **shallow** copies: the top-level container is new, but nested objects are still shared references. Mutating a nested object in the "copy" mutates the original.

---

### Buggy Code

```python
import copy

original = {"user": "alice", "scores": [95, 87, 92]}

snapshot = original.copy()   # shallow copy
snapshot["user"] = "bob"
snapshot["scores"].append(100)

print(original["user"])    # "alice" — fine
print(original["scores"])  # [95, 87, 92, 100] — NOT fine
```

### Issues

- `original.copy()` creates a new dict, but the value at `"scores"` is still the **same list object** in both dicts.
- Appending to `snapshot["scores"]` modifies the shared list, mutating `original` as a side effect.
- Changing `snapshot["user"]` is safe only because strings are immutable and reassignment replaces the reference.

### Corrected Code

```python
import copy

original = {"user": "alice", "scores": [95, 87, 92]}

snapshot = copy.deepcopy(original)
snapshot["user"] = "bob"
snapshot["scores"].append(100)

print(original["user"])    # "alice"
print(original["scores"])  # [95, 87, 92]  — unaffected
```

> When `deepcopy` is too expensive (large object graphs), consider structuring data as immutable (tuples, frozensets, `dataclasses` with `frozen=True`) or using explicit per-level copies.

### What to Say Out Loud

> "The `dict.copy()` here is a shallow copy — it copies the container but not the contents. The `scores` list is the same object in both dicts. Any mutation to a nested mutable value affects both. The fix is `copy.deepcopy()` for a full recursive copy. I'd also ask why we need a mutable snapshot at all — if the goal is immutability, using frozen data structures or building a new dict from scratch with reconstructed values is often safer and more explicit."

---

## 7. Integer vs Float Division Pitfalls

### Why This Is a Pitfall

Python 3 changed `/` to always return a float, but `//` (floor division) truncates toward negative infinity, not toward zero. Mixing these — or porting Python 2 code where `/` was integer division — produces subtle numeric bugs that rarely raise exceptions.

---

### Buggy Code

```python
def average(numbers):
    total = 0
    for n in numbers:
        total += n
    return total / len(numbers)

def paginate(total_items, page_size):
    num_pages = total_items / page_size   # should be integer
    return list(range(1, num_pages + 1))  # TypeError if float

print(average([1, 2, 3, 4]))  # 2.5 — correct here, but...
print(paginate(10, 3))        # TypeError: 'float' object cannot be interpreted as integer
```

### Issues

- `total_items / page_size` returns a `float` (e.g., `3.3333...`). Passing it to `range()` raises `TypeError`.
- The intent is integer (ceiling) division: 10 items at 3 per page → 4 pages.
- Using `//` here gives floor division (3 pages), which would drop the last partial page — also wrong.
- `average` works but returns a float even when the average is a whole number; this may matter in downstream comparisons.

### Corrected Code

```python
import math

def average(numbers):
    return sum(numbers) / len(numbers)  # float division is correct for average

def paginate(total_items, page_size):
    num_pages = math.ceil(total_items / page_size)  # ceiling division
    return list(range(1, num_pages + 1))

print(average([1, 2, 3, 4]))  # 2.5
print(paginate(10, 3))        # [1, 2, 3, 4]
```

> Alternative for ceiling division without `math`: `(total_items + page_size - 1) // page_size` — useful in environments where importing math is restricted.

### What to Say Out Loud

> "The division in `paginate` returns a float in Python 3. `range()` only accepts integers, so this crashes. The conceptual bug is that page count requires ceiling division — 10 items at 3 per page needs 4 pages, not 3. I'd use `math.ceil`. I always double-check divisions and ask: should this be exact float division, floor integer division, or ceiling integer division? The answer changes the entire output."

---

## 8. Modifying a List/Dict While Iterating

### Why This Is a Pitfall

Python's list and dict iterators are backed by the live data structure. Inserting or deleting elements during iteration shifts indices or invalidates the iterator, causing elements to be skipped, processed twice, or a `RuntimeError` to be raised (for dicts in Python 3).

---

### Buggy Code

```python
def remove_negatives(numbers):
    for i, n in enumerate(numbers):
        if n < 0:
            numbers.remove(n)  # modifies list during iteration
    return numbers

data = [1, -2, -3, 4, -5]
print(remove_negatives(data))  # Expected: [1, 4], Actual: [1, -3, 4]
```

### Issues

- `numbers.remove(n)` removes the **first occurrence** of `n`, shifting all subsequent elements left by one.
- The iterator's internal index advances by 1 after each step, so the element that shifted into the removed position is **skipped**.
- `-2` is removed (index 1), `-3` shifts to index 1, iterator moves to index 2 — `-3` is never visited.
- For dicts, Python 3 raises `RuntimeError: dictionary changed size during iteration`.

### Corrected Code — Option A: List Comprehension (preferred)

```python
def remove_negatives(numbers):
    return [n for n in numbers if n >= 0]

data = [1, -2, -3, 4, -5]
print(remove_negatives(data))  # [1, 4]
```

### Corrected Code — Option B: Iterate a Copy

```python
def remove_negatives(numbers):
    for n in numbers[:]:      # iterate over a shallow copy
        if n < 0:
            numbers.remove(n)
    return numbers
```

### What to Say Out Loud

> "Modifying a list while iterating over it — this is a classic. When you remove an element, the underlying array shifts left, but the iterator's index keeps advancing, so it skips the next element. The safest fix is a list comprehension, which builds a new list and doesn't touch the original during the loop. If in-place mutation is required, iterate over a copy with `numbers[:]`. For dicts, Python 3 is strict about this and raises a `RuntimeError` immediately, but lists fail silently, which is more dangerous."

---

## 9. Missing or Unintentional None Returns

### Why This Is a Pitfall

Every Python function that lacks an explicit `return` statement returns `None`. This is silent — no error at the function definition site. The bug surfaces later when the caller tries to use the return value, sometimes far away from the function itself.

---

### Buggy Code

```python
def double_list(numbers):
    numbers = [n * 2 for n in numbers]  # creates a new list, doesn't return it

def process(data):
    result = double_list(data)
    total = sum(result)   # TypeError: 'NoneType' object is not iterable
    return total

print(process([1, 2, 3]))
```

### Issues

- `double_list` builds a new list and assigns it to the local name `numbers`, but **never returns it**.
- The original list passed in is unchanged (lists are mutable but `numbers = [...]` rebinds the local name, not the caller's reference).
- `result` is `None`, and `sum(None)` raises `TypeError`.
- A related variant: a function returns a value in some branches but falls through to `None` in others.

### Corrected Code

```python
def double_list(numbers):
    return [n * 2 for n in numbers]

def process(data):
    result = double_list(data)
    total = sum(result)
    return total

print(process([1, 2, 3]))  # 12
```

### What to Say Out Loud

> "The function builds a list comprehension and assigns it locally, but there's no `return` statement, so it implicitly returns `None`. The caller assigns that `None` to `result` and then crashes when `sum` tries to iterate it. I'd add `return` and also note the subtle rebinding issue: `numbers = [...]` inside the function doesn't mutate the caller's list — it just rebinds the local name. If in-place mutation were the goal, you'd need `numbers[:] = [...]` or to modify the list elements directly."

---

## 10. Incorrect String/List Indexing or Slicing

### Why This Is a Pitfall

Python slices never raise `IndexError` — an out-of-range slice silently returns an empty sequence. But direct index access does raise `IndexError`. Negative indices and slice semantics (`start:stop:step`) are frequently misapplied, especially in reverse-iteration or windowing scenarios.

---

### Buggy Code

```python
def get_last_three(items):
    return items[len(items) - 3 : len(items) + 1]  # bug in stop index

def reverse_string(s):
    result = ""
    for i in range(len(s), 0, -1):   # bug: misses index 0
        result += s[i]               # bug: index len(s) is out of bounds
    return result

print(get_last_three([10, 20, 30, 40, 50]))
print(reverse_string("hello"))
```

### Issues

**`get_last_three`:**
- `len(items) + 1` as a stop index is harmless (slices clamp silently) but misleading — it implies the author doesn't know Python slice semantics.
- The real intent is cleaner with negative indexing: `items[-3:]`.

**`reverse_string`:**
- `range(len(s), 0, -1)` starts at `len(s)` (e.g., 5 for "hello"). `s[5]` is out of bounds → `IndexError`.
- Even if fixed to `range(len(s)-1, 0, -1)`, it would miss index 0, dropping the first character.
- The correct range is `range(len(s)-1, -1, -1)` (stop at -1 exclusive = 0 inclusive).

### Corrected Code

```python
def get_last_three(items):
    return items[-3:]  # clean, idiomatic, handles lists shorter than 3

def reverse_string(s):
    return s[::-1]  # idiomatic Python reverse

# Manual implementation for interview:
def reverse_string_manual(s):
    result = ""
    for i in range(len(s) - 1, -1, -1):  # from last index down to 0 inclusive
        result += s[i]
    return result

print(get_last_three([10, 20, 30, 40, 50]))  # [30, 40, 50]
print(reverse_string("hello"))               # "olleh"
```

### What to Say Out Loud

> "Two separate bugs here. In `get_last_three`, the stop index `len(items) + 1` works accidentally because slice stops clamp to the array length — but it's wrong conceptually. The idiomatic form is `items[-3:]`. In `reverse_string`, the range starts at `len(s)` which is immediately out of bounds. Even if that's fixed, stopping at `0` exclusive means index 0 is never visited and the first character is dropped. The correct stop for a range going downward that includes index 0 is `-1`. Whenever I see manual index arithmetic, I mentally trace through the first and last iteration."

---

## 11. Race Conditions and Shared State in Threads

### Why This Is a Pitfall

Python threads share heap memory. The GIL (Global Interpreter Lock) prevents true parallel bytecode execution in CPython, but it does not make high-level operations atomic. A read-modify-write sequence across multiple bytecodes can be interrupted between steps, corrupting shared state.

---

### Buggy Code

```python
import threading

counter = 0

def increment(n):
    global counter
    for _ in range(n):
        counter += 1   # not atomic: read → increment → write

threads = [threading.Thread(target=increment, args=(100000,)) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # Expected: 1000000, Actual: varies (e.g., 743291)
```

### Issues

- `counter += 1` compiles to multiple bytecodes: `LOAD_GLOBAL counter`, `LOAD_CONST 1`, `BINARY_ADD`, `STORE_GLOBAL counter`. The GIL can be released between any of these.
- Two threads can both read the same value of `counter`, both increment it, and both write back — effectively losing one increment.
- The final value is non-deterministic and always ≤ 1,000,000.
- Using `global counter` is a code smell on its own, but the race condition is the correctness bug.

### Corrected Code

```python
import threading

counter = 0
lock = threading.Lock()

def increment(n):
    global counter
    for _ in range(n):
        with lock:
            counter += 1

threads = [threading.Thread(target=increment, args=(100000,)) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # 1000000, deterministic
```

> For high-contention counters, prefer `threading.local()` for per-thread accumulators, or use `queue.Queue` to funnel updates to a single writer thread. `collections.Counter` is also not thread-safe.

### What to Say Out Loud

> "`counter += 1` looks atomic but it's not — it's a read-modify-write across at least three bytecodes. The GIL doesn't protect logical operations, only individual bytecode instructions. Two threads can race through the read and both write back the same incremented value, effectively losing an update. The fix is a `threading.Lock` wrapping the read-modify-write. I'd also mention that for pure counting, `threading.Semaphore` or an atomic approach using a thread-local accumulator and a final reduce is more scalable at high concurrency."

---

## 12. Class vs Instance Variable Confusion

### Why This Is a Pitfall

Class-level attributes are shared across all instances of a class. Instance attributes (set via `self.attr = ...`) are per-object. The confusion arises because you can **read** a class attribute through `self`, but the moment you **write** to `self.attr`, you create a new instance attribute that shadows the class one — for mutable objects this distinction is critical.

---

### Buggy Code

```python
class Student:
    grades = []          # class-level attribute — shared by all instances

    def __init__(self, name):
        self.name = name

    def add_grade(self, grade):
        self.grades.append(grade)  # mutates the SHARED class list

alice = Student("Alice")
bob = Student("Bob")

alice.add_grade(90)
alice.add_grade(85)
bob.add_grade(70)

print(alice.grades)  # Expected: [90, 85], Actual: [90, 85, 70]
print(bob.grades)    # Expected: [70],     Actual: [90, 85, 70]
```

### Issues

- `grades = []` is a class attribute. All `Student` instances share the exact same list object.
- `self.grades.append(grade)` accesses the class attribute through `self` (no instance attribute named `grades` exists yet), then mutates the shared list.
- Unlike `self.grades = something_new` (which would create an instance attribute), `.append()` modifies in-place and never creates an instance attribute.
- Alice and Bob are sharing the same grades list.

### Corrected Code

```python
class Student:
    def __init__(self, name):
        self.name = name
        self.grades = []   # instance attribute — fresh list per object

    def add_grade(self, grade):
        self.grades.append(grade)

alice = Student("Alice")
bob = Student("Bob")

alice.add_grade(90)
alice.add_grade(85)
bob.add_grade(70)

print(alice.grades)  # [90, 85]
print(bob.grades)    # [70]
```

> **Legitimate use of class attributes:** constants shared across instances (`MAX_GRADE = 100`), class-wide counters updated through `ClassName.counter += 1` (not `self.counter += 1`), and flyweight-pattern shared immutable data.

### What to Say Out Loud

> "The `grades = []` at class level is the bug — it's a single list shared by every `Student` instance. When `self.grades.append(grade)` runs, Python looks up `grades` on the instance, doesn't find it, then finds it on the class — and mutates that shared class-level list. This is different from `self.grades = []`, which would create a new instance attribute and shadow the class one. The fix is to initialize `self.grades = []` in `__init__`, giving each instance its own list. I'd call this out as a variant of the mutable default argument bug — same root cause, different syntax."

---

## 13. Interview Self-Review Checklist

When you receive an unfamiliar Python code snippet in an interview, walk through this checklist before writing a single word of analysis. Speak these as questions out loud — interviewers want to hear the diagnostic process.

---

### Inputs and Boundaries

- [ ] What are the valid inputs? What happens with `None`, empty list, empty string, or `0`?
- [ ] What if the input is larger than expected? What if it's negative?
- [ ] Does the function modify its inputs, or create new objects?

### Names and References

- [ ] Are any default arguments mutable (list, dict, set)? → Mutable default argument trap.
- [ ] Does the code use `is` to compare values? → Should it be `==`?
- [ ] Are there any variables that shadow built-ins (`list`, `id`, `type`, `input`, `max`)?

### Loops and Iteration

- [ ] What are the exact bounds of every `range()`? Trace the first and last value.
- [ ] Is any collection being modified inside a loop that iterates over it?
- [ ] Are there nested loops with shared index variable names (e.g., both loops using `i`)?

### Returns and None

- [ ] Does every code path return a value, or can any path fall through to implicit `None`?
- [ ] Is the return value of a function being used by the caller? Is it discarded?
- [ ] Are any results assigned and then never used?

### Copies and Mutations

- [ ] When an object is "copied", is it a shallow or deep copy?
- [ ] Does the code assume a passed-in list/dict won't be mutated? Is that assumption safe?

### Numeric Arithmetic

- [ ] Is any division expected to produce an integer? Is `//` or `math.ceil` needed?
- [ ] Are there float comparisons with `==`? (Floating point equality is unreliable.)
- [ ] Can any denominator be zero?

### Exception Handling

- [ ] Is there a bare `except`? Should it be `except Exception` or a specific type?
- [ ] Are exceptions being swallowed (caught and ignored)? Will the caller know something went wrong?
- [ ] Is cleanup code (file close, lock release) in a `finally` block or using a context manager?

### Classes and Objects

- [ ] Are any attributes defined at class level that should be instance attributes?
- [ ] Does `self.attr = ...` inside `__init__` shadow a class attribute intentionally?
- [ ] Are class methods mutating `self` when they should return new objects?

### Concurrency

- [ ] Is any shared mutable state accessed from multiple threads without a lock?
- [ ] Does "atomic-looking" code (`+=`, `append`, dict update) actually need protection?
- [ ] Are there `global` variable accesses in threaded code?

### String and Sequence Indexing

- [ ] Does any index access come from length arithmetic? Trace the first and last iteration.
- [ ] Are slices used correctly — especially with negative step values?
- [ ] For strings: is the code assuming ASCII when it should handle Unicode?

---

### The Three Questions to Always Ask the Interviewer

1. **"What should happen on invalid input?"** — clarifies whether you need guards or can assume clean data.
2. **"Is this expected to run in a concurrent environment?"** — determines whether thread safety matters.
3. **"Is this performance-sensitive?"** — clarifies whether `O(n²)` is acceptable or whether algorithmic correctness is the entire point.

---

*Good luck today. Read the code once for understanding, then again looking for exactly the categories above. State each issue you find, its consequence, and your fix — in that order.*
