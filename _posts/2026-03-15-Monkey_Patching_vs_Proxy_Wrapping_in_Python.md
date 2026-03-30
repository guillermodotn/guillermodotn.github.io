---
title: "Monkey Patching vs Proxy Wrapping in Python: Two Ways to Intercept Behavior at Runtime"
date: 2026-03-15 10:00:00 +0100
description: "A comparison of monkey patching and proxy wrapper patterns in Python, with code examples and a real-world case study."
categories: [Python, Software Design]
tags: [python, design-patterns, monkey-patching, proxy-pattern, testing, debugging]     # TAG names should always be lowercase

---

Python lets you modify program behavior at runtime in ways most languages won't allow. Two of the most common techniques for doing this are **monkey patching** and the **proxy wrapper pattern**. Both let you intercept and modify how objects behave without changing source code, but they work in fundamentally different ways, carry different risks, and shine in different situations.

I've been thinking about these patterns a lot lately while working on a side project[^flowview] that intercepts Polars DataFrame operations at runtime. Figuring out the right approach sent me down a rabbit hole, and I wanted to write down what I found.

## What is Monkey Patching

Monkey patching is the act of modifying or replacing attributes of a class, module, or object **at runtime** -- after it's already been defined and loaded into memory.

The simplest example:

```python
class Calculator:
    def add(self, a, b):
        return a + b

calc = Calculator()
calc.add(2, 3)  # 5

# Monkey patch the class method
Calculator.add = lambda self, a, b: a * b

calc.add(2, 3)  # 6 — every instance is affected
```

You didn't subclass `Calculator`. You didn't modify its source file. You reached into the class at runtime and swapped out one of its methods. Every existing and future instance of `Calculator` now uses the new `add` method.

### How it works under the hood

When you call `obj.method()`, Python looks up `method` in the object's class namespace (and then the parent classes, following the MRO). When you assign `Calculator.add = something`, you're replacing that entry. The next call to `.add()` finds the new function.

```python
# Before patching
Calculator.__dict__['add']  # <function Calculator.add at 0x...>

# After patching
Calculator.add = lambda self, a, b: a * b
Calculator.__dict__['add']  # <function <lambda> at 0x...>
```
{: .nolineno }

This is a **global mutation**. It changes the class for every instance in the entire process.

### A practical example -- patching for tests

The most common and widely accepted use of monkey patching is in testing. Python's `unittest.mock.patch`[^unittest-mock] uses structured monkey patching extensively. pytest's `monkeypatch` fixture[^pytest] takes a similar approach:

```python
import requests
from unittest.mock import patch, Mock

def get_user_name(user_id):
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()["name"]

# In a test:
def test_get_user_name():
    mock_response = Mock()
    mock_response.json.return_value = {"name": "Alice"}

    with patch("requests.get", return_value=mock_response):
        # Inside this block, requests.get is replaced
        assert get_user_name(42) == "Alice"

    # Outside the block, requests.get is restored
```

The `patch` context manager replaces `requests.get` with a mock, runs your test, then restores the original. This is monkey patching -- but it's scoped, temporary, and restores the original state.

### Intercepting a specific method

Imagine you're building a debugging tool that traces data pipeline steps. You could temporarily replace `pl.DataFrame.pipe` to capture step data:

```python
import polars as pl

# Save the original
_original_pipe = pl.DataFrame.pipe

def _traced_pipe(self, function, *args, **kwargs):
    """Replacement pipe that captures step data."""
    print(f"Before {function.__name__}: {self.shape}")
    result = _original_pipe(self, function, *args, **kwargs)
    print(f"After {function.__name__}: {result.shape}")
    return result

# Install the patch
pl.DataFrame.pipe = _traced_pipe

# Run a pipeline — every .pipe() call is now traced
result = (
    df.pipe(clean_names)
      .pipe(filter_active)
      .pipe(add_revenue)
)

# Restore
pl.DataFrame.pipe = _original_pipe
```

During the window where the patch is active, every Polars DataFrame in the entire Python process uses `_traced_pipe` instead of the original `.pipe()`. This works because the Python-level methods on Polars DataFrames are patchable, even though the core engine is implemented in Rust.

## What is the Proxy Wrapper Pattern

The proxy pattern takes a different approach: instead of modifying a class, you wrap an object with another object that intercepts attribute access and method calls, delegating most operations through while adding behavior to specific ones.

Here is an implementation using `__getattr__`:

```python
class LoggingProxy:
    def __init__(self, obj):
        # Store the real object
        object.__setattr__(self, '_obj', obj)

    def __getattr__(self, name):
        attr = getattr(self._obj, name)
        if callable(attr):
            def wrapper(*args, **kwargs):
                print(f"Calling {name}({args}, {kwargs})")
                result = attr(*args, **kwargs)
                print(f"  -> returned {type(result).__name__}")
                return result
            return wrapper
        return attr

# Usage
real_list = [3, 1, 4, 1, 5]
proxy = LoggingProxy(real_list)
proxy.sort()
# Calling sort((), {})
#   -> returned NoneType
proxy.append(9)
# Calling append((9,), {})
#   -> returned NoneType
```

The proxy doesn't modify `list`. It doesn't affect any other list in your program. It only intercepts operations on `proxy` -- the original `real_list` is untouched (though its contents change since `sort()` and `append()` mutate in place).

### How `__getattr__` works

`__getattr__` is only called when normal attribute lookup fails:

1. Python first checks the instance's `__dict__`
2. Then the class's `__dict__` (and parent classes via MRO)
3. Only if nothing is found does it call `__getattr__`

This is important because it means your proxy can define real methods (which take priority) while delegating everything else to the wrapped object.

There's also `__getattribute__`, which is called for *every* attribute access -- but it's harder to use correctly because you need to be careful not to create infinite recursion when accessing your own attributes.

> `__getattr__` only intercepts regular attribute access. Python looks up dunder methods like `__len__`, `__iter__`, `__repr__`, and `__eq__` on the *type*, not the instance. This means `len(proxy)` or `for x in proxy:` will fail unless your proxy class explicitly defines those methods. This is the proxy pattern's most important limitation.
{: .prompt-warning }

### A practical proxy -- DataFrame tracing

Here's how a proxy wrapper can trace Polars DataFrame operations without modifying the DataFrame class:

```python
import time
import polars as pl

class TracedDataFrame:
    def __init__(self, df, steps=None):
        object.__setattr__(self, '_df', df)
        object.__setattr__(self, '_steps', steps or [])

    @property
    def __class__(self):
        # Makes isinstance(proxy, pl.DataFrame) return True
        return pl.DataFrame

    def __getattr__(self, name):
        attr = getattr(self._df, name)

        if not callable(attr):
            return attr

        proxy = self

        def wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = attr(*args, **kwargs)
            elapsed = (time.perf_counter() - start) * 1000

            if isinstance(result, pl.DataFrame):
                before = proxy._df.shape
                after = result.shape
                proxy._steps.append({
                    'method': name,
                    'before': before,
                    'after': after,
                    'time_ms': elapsed,
                })
                # Re-wrap so the next call is also traced
                return TracedDataFrame(result, proxy._steps)

            return result

        return wrapper
```

```python
# Usage
df = pl.DataFrame({
    "status": ["active", "inactive", "active"],
    "price": [10.0, 20.0, 30.0],
    "quantity": [2, 1, 3],
})

traced = TracedDataFrame(df)
result = (
    traced.filter(pl.col("status") == "active")
          .with_columns(
              (pl.col("price") * pl.col("quantity")).alias("revenue")
          )
          .select("status", "revenue")
)

# result._steps contains the full trace:
# [
#   {'method': 'filter', 'before': (3, 3), 'after': (2, 3), ...},
#   {'method': 'with_columns', 'before': (2, 3), 'after': (2, 4), ...},
#   {'method': 'select', 'before': (2, 4), 'after': (2, 2), ...},
# ]
```
{: .nolineno }

Notice what's different from monkey patching:

- `pl.DataFrame` is never modified
- Only the `traced` instance is affected
- Other DataFrames in your program behave normally
- The trace state (`_steps`) is carried forward through the chain by re-wrapping each result

For debugging scenarios, the performance overhead of the proxy is not a concern -- you're adding one extra function call per method invocation, which is nothing compared to the DataFrame operations themselves.

### The `__class__` trick

One challenge with proxies is that they break `isinstance()` checks:

```python
traced = TracedDataFrame(df)
isinstance(traced, pl.DataFrame)  # False — it's a TracedDataFrame
```
{: .nolineno }

This can cause problems if downstream code checks the type. The fix is to override the `__class__` property:

```python
class TracedDataFrame:
    @property
    def __class__(self):
        return pl.DataFrame
```
{: .nolineno }

Now `isinstance(traced, pl.DataFrame)` returns `True`. It works because `isinstance()` checks `__class__`, not `type()`. Most Python code uses `isinstance()`, so the override covers the vast majority of cases. Code that uses `type()` directly is rare, but it exists -- particularly in some serialization libraries and C extensions.

## Head-to-Head Comparison

### Scope

Monkey patching changes things **globally**. If you patch `pl.DataFrame.pipe`, every DataFrame in every thread, in every library, in your entire Python process uses your patched version. The proxy only affects the single object you wrap.

### Thread Safety

**Monkey patching is inherently unsafe in multithreaded code.** If Thread A restores the original method while Thread B is still running, Thread B's remaining calls silently skip tracing. You can mitigate this with reference counters and locks, but that adds complexity and still has edge cases.

> The proxy wrapper is inherently thread-safe. Each thread wraps its own DataFrame with its own proxy. There's no shared mutable state, no race conditions, and no locks needed.
{: .prompt-tip }

### Maintenance Burden

**Monkey patching a single method is low-maintenance.** You match one function signature, save the original, and restore it when done.

**Monkey patching many methods is a maintenance nightmare.** If you need to patch `filter`, `with_columns`, `select`, `drop`, `rename`, `sort`, and 20 more methods, each one needs its own wrapper, each needs to be saved and restored, and every upstream release could change any of these signatures.

The proxy's `__getattr__` delegates to the real object. When the upstream library adds a new method in a future release, the proxy automatically supports it -- no code changes needed. When a method signature changes, the proxy still works because it just passes `*args, **kwargs` through.

> The proxy is forward-compatible by default. This is its key architectural advantage over monkey patching when you need to intercept many methods.
{: .prompt-tip }

### Type Safety and IDE Support

**Monkey patching preserves types perfectly.** After patching `pl.DataFrame.pipe`, every object is still a real `pl.DataFrame`. IDE autocomplete works. Type checkers are happy.

**Proxies break types (partially).** Your IDE sees `TracedDataFrame`, not `pl.DataFrame`, so you lose autocomplete and type checking inside the decorated function. The `__class__` override fixes `isinstance()` at runtime, but static analysis doesn't know about it. The code runs correctly, but the development experience is slightly degraded.

This is the proxy's main practical drawback.

### Transparency

**Monkey patching is invisible to the user's code** -- the function receives a real `pl.DataFrame` and has no idea `.pipe()` is patched.

**The proxy is almost invisible.** The function works the same, `isinstance()` returns `True`, all methods delegate correctly. But if the user inspects the object closely (`type(df)`), they'll see it's not a real DataFrame.

In practice, this rarely matters. Pipeline functions transform data -- they don't introspect the DataFrame type.

## When to Use Each

### Use monkey patching when

1. **You need to intercept one specific, stable method** -- Patching a single well-known method (like `.pipe()`) is simple and has minimal risk.

2. **Testing** -- `unittest.mock.patch` is monkey patching, and it's the standard approach. It's scoped, temporary, and well-understood.

3. **You can't control the call site** -- If you need to intercept behavior in third-party code that creates its own objects, monkey patching the class is sometimes the only option.

4. **Total transparency is required** -- If the intercepted code does strict type checking with `type()`, or if IDE support is critical, monkey patching preserves types perfectly.

### Use the proxy wrapper when

1. **You need to intercept many methods** -- The proxy's `__getattr__` handles all methods generically. No per-method code needed, and it's forward-compatible with future API changes.

2. **Thread safety matters** -- No global state mutation, no race conditions.

3. **You want scoped interception** -- The proxy only affects the wrapped instance. Other code continues to use real objects.

4. **The upstream library changes frequently** -- The proxy delegates transparently, so upstream API changes don't break your code.

5. **You're building a library** -- Monkey patching in a library is risky because you're modifying global state that other code might rely on. A proxy is self-contained.

> Consider avoiding both if **subclassing** or **composition** solve the problem -- they're more explicit and better supported by type checkers. Runtime interception is best for temporary or conditional changes.
{: .prompt-info }

## Putting It Together -- A Real Decision

The choice between these patterns becomes clearer when you walk through a real decision. Say you're building a tool that traces Polars DataFrame transformations -- showing row counts, schema changes, sample data, and timing at each step of a pipeline. This is roughly the problem I was trying to solve.

### Starting with monkey patching

The first version only needed to intercept `.pipe()` calls -- a single, stable method on `pl.DataFrame`. Users structure their pipelines like this:

```python
@trace
def process(df):
    return (
        df.pipe(clean_names)
          .pipe(filter_active)
          .pipe(add_revenue)
    )
```
{: .nolineno }

For this, monkey patching was the obvious choice. Save the original `.pipe()`, replace it with a version that captures snapshots before and after each step, and restore it when the function returns:

```python
_original_pipe = pl.DataFrame.pipe
pl.DataFrame.pipe = _traced_pipe
try:
    result = func(*args, **kwargs)
finally:
    pl.DataFrame.pipe = _original_pipe
```
{: .nolineno }

Simple. One method patched, one method restored. Type safety is perfect -- every object is a real `pl.DataFrame`. IDE autocomplete works. Tests pass. For a focused MVP, this was genuinely the right choice.

### The moment it breaks down

Then you discover that many Polars users don't write `.pipe()` chains. Idiomatic Polars looks like this:

```python
def process(df):
    return (
        df.filter(pl.col("status") == "active")
          .with_columns(
              (pl.col("price") * pl.col("quantity")).alias("revenue")
          )
          .group_by("category")
          .agg(pl.col("revenue").sum())
    )
```
{: .nolineno }

No `.pipe()`. Direct method chains. The tool doesn't see any of these operations.

The instinctive reaction is to patch more methods:

```python
pl.DataFrame.filter = traced_filter
pl.DataFrame.with_columns = traced_with_columns
pl.DataFrame.select = traced_select
pl.DataFrame.drop = traced_drop
pl.DataFrame.rename = traced_rename
pl.DataFrame.sort = traced_sort
pl.DataFrame.join = traced_join
# ... and 15 more
```
{: .nolineno }

Now you're patching 20+ methods. Each one needs its own wrapper. Each one needs to be saved and restored in a `finally` block. Every upstream release could change any of these signatures. And all of these patches are global -- affecting every DataFrame in every thread.

> This is likely what happened to pandas-log[^pandas-log]. It patched dozens of pandas methods and couldn't keep up with API changes across releases.
{: .prompt-warning }

### Why proxy wrapper wins here

The proxy wrapper solves all of these problems with the `TracedDataFrame` pattern shown earlier. One `__getattr__` implementation handles every method -- `filter`, `with_columns`, `select`, `drop`, `sort`, `join`, and any method Polars adds in future releases. There's no per-method maintenance, and thread safety comes for free since each call creates its own proxy with its own state.

### The tradeoff accepted

The proxy isn't perfect. There are two practical compromises:

**IDE autocomplete degrades inside the decorated function.** The IDE sees `TracedDataFrame`, not `pl.DataFrame`, so method suggestions may not appear. The code runs correctly -- it's a development experience issue, not a correctness issue.

**`type(df)` returns `TracedDataFrame` instead of `pl.DataFrame`.** The `__class__` override makes `isinstance()` work, but direct `type()` checks will see the proxy. In practice, pipeline functions transform data -- they don't introspect the DataFrame type. This edge case rarely matters.

Both compromises are acceptable because the decorator is temporary -- you add `@trace` to debug, remove it when done.

### The lesson

In this case, monkey patching was a reasonable starting point when the goal was just intercepting `.pipe()`. The proxy made more sense once the scope grew to "trace any DataFrame operation." That kind of progression is pretty natural -- start simple, adapt when the problem outgrows the tool.

> Start with the simplest interception mechanism that solves your current problem, and let the requirements guide you to a more sophisticated pattern when the scope demands it.
{: .prompt-tip }

## Decision Framework

When choosing between the two, these questions help:

| Question | Monkey Patching | Proxy Wrapper |
|---|---|---|
| How many methods to intercept? | One or two | Many or all |
| Multithreaded code? | No | Yes |
| How often does the upstream API change? | Rarely | Frequently |
| Need perfect type safety? | Yes | Can tolerate minor gaps |
| Building a library or an application? | Application | Library |
| Is the interception temporary? | Yes | Either |
| Can you control what objects the code receives? | No | Yes |

If most of your answers fall in the left column, monkey patching is simpler and sufficient. If they fall in the right column, the proxy wrapper is the safer, more maintainable choice.

## Summary

**Monkey patching** is a powerful, simple technique for runtime behavior modification. It's ideal for testing, for patching one or two well-known methods, and for situations where you can't control the call site. Its main weakness is that it mutates global state, making it thread-unsafe and risky in production code.

**The proxy wrapper** is a more disciplined approach that provides the same interception capability without global side effects. It's ideal for intercepting many methods, for thread-safe code, and for libraries that need to be robust against upstream changes. Its main weakness is that it introduces a new type that can confuse static analysis and IDE tooling, and dunder methods need explicit handling.

Neither is universally better. They solve the same problem -- intercepting behavior at runtime -- but they make different tradeoffs around scope, safety, and complexity.

## References

[^flowview]: <https://github.com/guillermodotn/flowview>
[^unittest-mock]: <https://docs.python.org/3/library/unittest.mock.html#unittest.mock.patch>
[^pandas-log]: <https://github.com/eyaltrabelsi/pandas-log>
[^pytest]: <https://docs.pytest.org/en/stable/how-to/monkeypatch.html>
