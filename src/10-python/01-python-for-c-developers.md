# Python for C# Developers

> **🔄 Current (as of October 2026):** Python 3.14 (released October 2025) is the current
> stable line, and Python 3.15 is scheduled for release in October 2026. Python releases a new
> minor version every October, each supported for five years. Examples here work on 3.12+.

Why does a C#/.NET book have a Python section? Because Python is the **lingua franca of AI
and data work**. Most AI libraries, model provider SDKs, evaluation tools, data processing
libraries and examples appear in Python first, sometimes only in Python. You don't need to
become a Python expert, but you need to read it fluently, write solid scripts and small
services, and know where it differs from C# in ways that cause bugs.

This chapter teaches Python through a C# lens: what maps directly, what's different, and
what will surprise you.

---

## 1. The problem: a second language, quickly

Learning a second language as an experienced developer is mostly about **mapping concepts**
and **unlearning assumptions**. You already know variables, types, functions, classes,
collections, exceptions, async and modules. The work is learning Python's versions of them,
its idioms ("Pythonic" code), and its different trade-offs.

---

## 2. The mental model: dynamic, interpreted, batteries included

| | C# | Python |
|---|---|---|
| Typing | Static, checked by the compiler | **Dynamic**: types belong to values; optional type hints checked by separate tools |
| Execution | Compiled to IL, JIT-compiled | **Interpreted** by CPython (bytecode + a virtual machine; an experimental JIT exists in recent versions) |
| Performance | Fast | Slower for pure-Python loops; fast when calling C/Rust/CUDA libraries (NumPy, PyTorch) |
| Concurrency | Threads + async, truly parallel | `asyncio`; threads limited by the **GIL** for CPU work (changing: section 9); processes for parallelism |
| Blocks | `{ }` braces | **Indentation** is syntax |
| Packages | NuGet | PyPI, installed with pip/uv (Chapter 2) |
| Standard library | Large BCL | "Batteries included": large standard library |

Python's philosophy (`import this` prints "The Zen of Python") favors readability, one obvious
way to do things, and explicitness. Much Python code is short scripts and glue; much is also
large production systems (Instagram, much of YouTube's tooling, scientific computing, nearly
all ML infrastructure).

---

## 3. Syntax side by side

```csharp
// C#
public static string Describe(Ticket ticket)
{
    if (ticket.Assignee is null)
        return $"{ticket.Id}: {ticket.Title} (unassigned)";
    return $"{ticket.Id}: {ticket.Title} → {ticket.Assignee}";
}
```

```python
# Python
def describe(ticket: Ticket) -> str:
    if ticket.assignee is None:
        return f"{ticket.id}: {ticket.title} (unassigned)"
    return f"{ticket.id}: {ticket.title} → {ticket.assignee}"
```

Differences to absorb immediately:

- **Indentation defines blocks** (4 spaces by convention). No braces, no semicolons.
- **`None`** instead of `null`; check with `is None`, not `== None`.
- **`snake_case`** for functions, variables and modules; `PascalCase` for classes;
  `UPPER_CASE` for constants (PEP 8, Python's style guide).
- **f-strings** (`f"{x}"`) are interpolation.
- **`and`, `or`, `not`** instead of `&&`, `||`, `!`.
- **Truthiness**: empty collections, `0`, `""` and `None` are falsy (like JavaScript, Book V,
  Chapter 1). `if items:` checks for a non-empty list.
- **Comments** with `#`; docstrings (`"""..."""`) document functions and classes.

### Core types

| C# | Python | Notes |
|---|---|---|
| `int`, `long` | `int` | **Arbitrary precision**: no overflow |
| `double` | `float` | 64-bit float, same rounding issues |
| `decimal` | `decimal.Decimal` | Use for money |
| `bool` | `bool` (`True`/`False`) | |
| `string` | `str` | Immutable, Unicode |
| `byte[]` | `bytes` / `bytearray` | |
| `List<T>` | `list` | `[1, 2, 3]`, mutable |
| `T[]` (fixed) / tuple | `tuple` | `(1, 2)`, immutable |
| `Dictionary<K,V>` | `dict` | `{"a": 1}`; preserves insertion order |
| `HashSet<T>` | `set` / `frozenset` | `{1, 2}` |
| `DateTimeOffset` | `datetime` (timezone-aware) | Always use **aware** datetimes: `datetime.now(timezone.utc)` |

---

## 4. Functions

```python
def create_ticket(title: str, priority: str = "Normal", *, tags: list[str] | None = None) -> dict:
    """Create a ticket payload. Arguments after * are keyword-only."""
    return {"title": title, "priority": priority, "tags": tags or []}

create_ticket("VPN drops", "High", tags=["network"])
create_ticket(title="Printer offline")
```

- **Default arguments**, **keyword arguments** (like C# named arguments), and `*` to force
  keyword-only parameters (great for readability).
- `*args` and `**kwargs` collect extra positional and keyword arguments (like `params`).
- Functions are first-class; `lambda x: x * 2` is a small anonymous function (single
  expression only).

> **⚠️ What can go wrong:** **Mutable default arguments** are evaluated **once**, when the
> function is defined, not per call:
> ```python
> def add_tag(tag, tags=[]):      # ✗ the same list is shared across calls
>     tags.append(tag)
>     return tags
> add_tag("a"); add_tag("b")      # ['a', 'b'] !
> ```
> Use `None` as the default and create the list inside: `tags = tags or []` (or
> `if tags is None: tags = []`).

---

## 5. Collections and comprehensions

Python's **comprehensions** are its LINQ (Book I, Chapter 7) for most everyday cases:

```python
active = [t for t in tickets if t.status in ("Open", "InProgress")]            # Where
titles = [t.title for t in tickets]                                            # Select
by_id = {t.id: t for t in tickets}                                             # ToDictionary
assignees = {t.assignee for t in tickets if t.assignee}                        # ToHashSet + Where
total_comments = sum(len(t.comments) for t in tickets)                         # generator expression
```

| LINQ | Python |
|---|---|
| `Where` / `Select` | comprehension or `filter` / `map` |
| `Any` / `All` | `any(...)` / `all(...)` |
| `OrderBy(x => x.Key)` | `sorted(items, key=lambda x: x.key)`; `list.sort()` in place |
| `GroupBy` | `itertools.groupby` (needs sorted input) or `collections.defaultdict(list)` |
| `First` / `FirstOrDefault` | `next(iter)` / `next(iter, None)` |
| `Count()` | `len(...)` (collections) |
| `Zip`, `Take`, `Skip` | `zip`, `itertools.islice` |
| `Distinct` | `set(...)` or `dict.fromkeys(...)` (keeps order) |

**Generators** (`yield`) are lazy sequences, exactly like C# iterators (`yield return`), and
**generator expressions** `(x for x in ...)` are lazy comprehensions. Like LINQ, they
execute when iterated (Book I, Chapter 7's deferred execution).

Useful standard modules: `collections` (`Counter`, `defaultdict`, `deque`), `itertools`,
`functools` (`lru_cache`, `partial`), `dataclasses`, `pathlib`, `json`, `datetime`, `re`,
`logging`, `asyncio`, `typing`.

### Slicing and unpacking

```python
items[0], items[-1]          # first, last
items[1:4], items[:3], items[::2]   # slices (new lists)
first, *rest = items         # unpacking
for i, t in enumerate(tickets): ...
for key, value in by_id.items(): ...
```

---

## 6. Classes, dataclasses and protocols

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum

class Priority(str, Enum):
    LOW = "Low"
    NORMAL = "Normal"
    HIGH = "High"
    URGENT = "Urgent"

@dataclass(frozen=True, slots=True)          # like a C# record: generated __init__, __eq__, __repr__; immutable
class Comment:
    author: str
    body: str
    created_at: datetime

@dataclass
class Ticket:
    id: str
    title: str
    priority: Priority = Priority.NORMAL
    status: str = "Open"
    assignee: str | None = None
    comments: list[Comment] = field(default_factory=list)   # safe mutable default

    def assign_to(self, assignee: str) -> None:
        if self.status == "Closed":
            raise ValueError(f"Ticket {self.id} is closed")
        self.assignee = assignee
        if self.status == "Open":
            self.status = "InProgress"
```

- **`self`** is explicit: every instance method takes it as the first parameter.
- **No access modifiers**: a leading underscore (`_internal`) is a convention for "private";
  double underscore triggers name mangling. Python trusts developers ("we're all consenting
  adults").
- **`@dataclass`** removes boilerplate; `frozen=True` makes instances immutable (like records,
  Book I, Chapter 2).
- **Dunder methods** (`__init__`, `__repr__`, `__eq__`, `__lt__`, `__len__`, `__iter__`) are
  how classes plug into language features, like operator overloading and interfaces combined.
- **`@property`** for computed attributes; `@staticmethod` and `@classmethod`.

### Duck typing and Protocols

Python is **duck-typed**: if an object has the methods you call, it works. For static type
checking, **`Protocol`** describes a shape structurally, much like TypeScript interfaces
(Book V, Chapter 4) rather than C# interfaces:

```python
from typing import Protocol

class Notifier(Protocol):
    def notify(self, recipient: str, message: str) -> None: ...

class ConsoleNotifier:                       # doesn't inherit from Notifier
    def notify(self, recipient: str, message: str) -> None:
        print(f"[{recipient}] {message}")

def escalate(ticket: Ticket, notifier: Notifier) -> None:   # ConsoleNotifier fits structurally
    notifier.notify(ticket.assignee or "lead", f"{ticket.id} escalated")
```

---

## 7. Type hints

Type hints (`def f(x: int) -> str`) are **not enforced at run time**. They're checked by
external tools (**mypy**, **pyright**, or newer fast checkers), used by editors for
completion, and by libraries like **Pydantic** for run-time validation (Chapter 3).

```python
def find(tickets: list[Ticket], ticket_id: str) -> Ticket | None: ...
def first_or[T](items: list[T], default: T) -> T: ...          # generics (3.12+ syntax)
type TicketMap = dict[str, Ticket]                              # type alias (3.12+)
```

Treat type hints like C#'s nullable reference types (Book I, Chapter 2): a compile-time aid
you should use everywhere in non-trivial code, checked in CI, while remembering the runtime
doesn't enforce them.

---

## 8. Errors and context managers

```python
try:
    ticket = load(ticket_id)
except FileNotFoundError:
    ticket = None
except (ValueError, KeyError) as e:
    raise TicketLoadError(f"Bad data for {ticket_id}") from e   # like InnerException
else:
    print("loaded")              # runs if no exception
finally:
    cleanup()
```

- Python uses exceptions more liberally than C#: "easier to ask forgiveness than permission"
  (EAFP) is idiomatic (`try: d[key] except KeyError:` rather than checking first).
- **Context managers** (`with`) are `using`:

```python
with open("tickets.csv", encoding="utf-8") as f:     # file closed automatically
    for line in f:
        ...
```

---

## 9. Concurrency: async, threads and the GIL

```python
import asyncio
import httpx

async def fetch_ticket(client: httpx.AsyncClient, ticket_id: str) -> dict:
    r = await client.get(f"/api/tickets/{ticket_id}")
    r.raise_for_status()
    return r.json()

async def main() -> None:
    async with httpx.AsyncClient(base_url="https://beacon.example.com", timeout=10) as client:
        tickets = await asyncio.gather(*(fetch_ticket(client, i) for i in ["T-1", "T-2", "T-3"]))
        print([t["title"] for t in tickets])

asyncio.run(main())
```

`async`/`await` and `asyncio.gather` map directly to C#'s `async`/`await` and `Task.WhenAll`
(Book I, Chapter 9), with an event loop like JavaScript's (Book V, Chapter 2): one thread,
cooperative scheduling. Blocking calls (like `requests.get` or `time.sleep`) inside async code
block the whole loop.

The **Global Interpreter Lock (GIL)** in CPython allows only one thread to execute Python
bytecode at a time. Threads still help for I/O (the GIL is released while waiting), but not
for CPU-bound Python code; use `multiprocessing`/`concurrent.futures.ProcessPoolExecutor`, or
libraries whose heavy work runs in C/Rust/GPU code outside the GIL.

> **🔄 Current (as of October 2026):** CPython offers an officially supported **free-threaded
> build** (no GIL) since Python 3.14, still optional and with some library compatibility
> caveats. Most production deployments still use the default build. Check before relying on
> parallel threads for CPU-bound Python code.

---

## 10. In practice: a Beacon ticket model in Python

A small, idiomatic module you'll build on in Chapters 2 and 3, mirroring Book I's domain:

```python
# beacon_tools/models.py
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum


class Priority(str, Enum):
    LOW = "Low"
    NORMAL = "Normal"
    HIGH = "High"
    URGENT = "Urgent"

    @property
    def response_target(self) -> timedelta:
        return {
            Priority.URGENT: timedelta(hours=1),
            Priority.HIGH: timedelta(hours=4),
            Priority.NORMAL: timedelta(days=1),
            Priority.LOW: timedelta(days=3),
        }[self]


ACTIVE = frozenset({"Open", "InProgress"})


@dataclass(slots=True)
class Ticket:
    id: str
    title: str
    priority: Priority
    status: str
    created_at: datetime
    assignee: str | None = None
    comment_count: int = 0

    @property
    def is_active(self) -> bool:
        return self.status in ACTIVE

    def is_breaching(self, now: datetime | None = None) -> bool:
        now = now or datetime.now(timezone.utc)
        return self.is_active and now - self.created_at > self.priority.response_target


def workload_by_agent(tickets: list[Ticket], now: datetime) -> dict[str, tuple[int, int]]:
    """Active and breaching counts per assignee (Book I, Chapter 7's report, in Python)."""
    result: dict[str, tuple[int, int]] = {}
    for t in tickets:
        if not (t.is_active and t.assignee):
            continue
        active, breaching = result.get(t.assignee, (0, 0))
        result[t.assignee] = (active + 1, breaching + int(t.is_breaching(now)))
    return dict(sorted(result.items(), key=lambda kv: (-kv[1][1], -kv[1][0])))
```

Compare with the C# versions: the same ideas (an enum with behavior, a computed property, a
pure report function with `now` passed in for testability), expressed with Python's tools.

---

## 11. What can go wrong (for C# developers especially)

- **Mutable default arguments.**
- **Naive datetimes** (no time zone) mixed with aware ones, causing exceptions or wrong times.
- **`==` vs `is`**: use `is` only for `None` (and singletons); `==` for values.
- **Assuming type hints are enforced** at run time.
- **Blocking calls inside async code.**
- **Expecting threads to speed up CPU-bound Python.**
- **Shadowing built-ins** (`list = [...]`, `id = ...`, `type = ...`).
- **Floating point for money** (`decimal.Decimal` instead).
- **Mixing tabs and spaces**, or inconsistent indentation (let a formatter handle it).

---

## 12. How an experienced engineer thinks about this

- **Map concepts, then learn idioms.** Write Python like Python (comprehensions, dataclasses,
  context managers), not like C# with different syntax.
- **Use type hints and a type checker** for anything beyond a throwaway script.
- **Python is glue and orchestration**; heavy computation happens in native libraries.
- **Choose the language for the ecosystem**: Python for AI/data tooling, C# for Beacon's core
  services, connected through APIs.

---

## 13. Check yourself

**Questions**

1. How do Python's typing and execution models differ from C#'s?
2. What's the problem with mutable default arguments, and how do you avoid it?
3. Translate a LINQ `Where`/`Select`/`ToDictionary` chain into comprehensions.
4. What does `@dataclass(frozen=True)` give you, and what's its C# analogue?
5. How is `Protocol` different from a C# interface?
6. Are type hints enforced at run time? What checks them?
7. What is the GIL, and when does it matter?

**Exercises**

1. Port Book I's `TriageQueue` to Python using `heapq`.
2. Write `workload_by_agent` using `collections.defaultdict` and compare readability.
3. Add type hints to a script and run `mypy --strict` or `pyright` on it; fix the findings.
4. Fetch ten tickets concurrently from Beacon's API with `httpx.AsyncClient` and
   `asyncio.gather`, with a timeout.

**Interview-style questions**

- "What are the main differences between Python and C#?"
- "What's a list comprehension? A generator?"
- "How does concurrency work in Python?"

---

## 14. Going deeper

- [The Python Tutorial](https://docs.python.org/3/tutorial/) (official).
- Luciano Ramalho, *Fluent Python* — the best book on idiomatic Python.
- [PEP 8 — Style Guide](https://peps.python.org/pep-0008/) and
  [typing documentation](https://typing.python.org/)

**Next:** [Chapter 2 — Environments and Packages](02-environments-and-packages.md) sets up
Python projects properly, avoiding the "works on my machine" problems Python is notorious for.
