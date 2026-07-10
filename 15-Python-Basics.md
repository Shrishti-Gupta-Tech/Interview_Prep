# 15. Python Basics — Deep Dive (Nice to Have)

> **Interview positioning:** *"I have strong expertise in Java and Spring Boot, and I know Python fundamentals well. I'm comfortable picking up Python for scripts, tests, or where the project requires it."*

---

## 15.0 Why Learn Python?

Even if Java is your primary language:
- **Scripts & automation** — quick task runners
- **Testing tools** — many CI helpers, load testers use Python
- **Data / ML** — everything from Pandas to PyTorch
- **DevOps** — Ansible, cloud SDKs, glue code
- **Interviews** — some companies allow Python for coding rounds

### Python vs Java — 30-second summary

| | Java | Python |
|-|------|--------|
| Typing | Static (compile-time) | Dynamic (runtime) |
| Execution | JVM bytecode | Interpreted (CPython) |
| Verbose? | Verbose | Concise |
| Speed | Fast | Slower (but Cython, PyPy for perf) |
| Package manager | Maven/Gradle | pip / poetry |
| Concurrency | Threads (real parallelism) | Threads (GIL limits) + asyncio + multiprocessing |
| Enterprise use | Huge (backend) | Huge (data/ML, scripting) |

---

## 15.1 Setup & REPL

```bash
python --version              # 3.10+
python                         # REPL
python script.py               # run
python -m venv venv            # create virtual env
source venv/bin/activate       # activate (Linux/Mac)
venv\Scripts\activate          # activate (Windows)
pip install requests
pip freeze > requirements.txt
pip install -r requirements.txt
```

### Hello world
```python
# hello.py
print("Hello, World!")

if __name__ == "__main__":
    print("script started")
```

Run: `python hello.py`

---

## 15.2 Variables & Data Types

Python is dynamically typed — no need to declare types.

```python
x = 10                # int
y = 3.14              # float
name = "Alice"        # str
flag = True           # bool
empty = None          # NoneType (like null)

# Type check
type(x)               # <class 'int'>
isinstance(x, int)    # True

# Type conversion
int("42")             # 42
float("3.14")         # 3.14
str(10)               # "10"
bool(0)               # False
bool("")              # False
bool([])              # False
list("abc")           # ['a', 'b', 'c']
```

**Falsy values:** `False`, `0`, `""`, `None`, `[]`, `{}`, `()`.

### Type hints (optional, Python 3.5+)

```python
def add(a: int, b: int) -> int:
    return a + b

name: str = "Alice"
scores: list[int] = [95, 88, 76]
```

Great for IDE support & mypy checks. Not enforced at runtime.

---

## 15.3 Strings — Powerful & Immutable

```python
s = "Hello, World!"

# Basics
len(s)                  # 13
s.upper()               # "HELLO, WORLD!"
s.lower()               # "hello, world!"
s.title()               # "Hello, World!"
s.strip()               # remove whitespace ends
s.lstrip("H")           # remove H from left
s.replace("H", "J")     # "Jello, World!"
s.split(",")            # ['Hello', ' World!']
"-".join(["a","b"])     # "a-b"
s.startswith("Hello")   # True
s.endswith("!")         # True
"o" in s                # True
s.count("o")            # 2
s.find("o")             # 4 (first index)
s.index("o")            # like find, but raises if missing

# Indexing & Slicing (immutable — creates new string)
s[0]                    # 'H'
s[-1]                   # '!'  (last)
s[0:5]                  # 'Hello'
s[7:]                   # 'World!'
s[:5]                   # 'Hello'
s[::-1]                 # reverse: '!dlroW ,olleH'
s[::2]                  # every 2nd char

# String formatting
name = "Alice"; age = 25
f"Hello {name}, age {age}"                    # f-string (preferred)
f"pi = {3.14159:.2f}"                          # "pi = 3.14"
"Hello {}, age {}".format(name, age)          # .format
"Hello %s, age %d" % (name, age)              # % (old)

# Multi-line
text = """line 1
line 2
line 3"""
```

---

## 15.4 Lists — Ordered, Mutable

Like Java's `ArrayList`.

```python
nums = [1, 2, 3]
mixed = [1, "two", 3.0, [4, 5]]        # types can mix

# Access
nums[0]                # 1
nums[-1]               # 3
nums[1:3]              # [2, 3]

# Mutate
nums.append(4)         # [1,2,3,4]
nums.insert(0, 0)      # [0,1,2,3,4]
nums.remove(2)         # removes first 2 → [0,1,3,4]
nums.pop()             # 4 (removes & returns last) → [0,1,3]
nums.pop(0)            # 0
nums[1] = 99           # replace
del nums[0]

# Query
len(nums)
5 in nums              # True/False
nums.index(3)
nums.count(3)

# Ordering
sorted(nums)           # returns NEW sorted list
nums.sort()            # in-place
nums.sort(reverse=True)
nums.sort(key=lambda x: -x)
nums.reverse()

# Concatenation
[1, 2] + [3, 4]        # [1,2,3,4]
[0] * 5                # [0,0,0,0,0]

# List comprehension — Pythonic!
squares = [x*x for x in range(5)]              # [0,1,4,9,16]
evens = [x for x in range(10) if x % 2 == 0]   # [0,2,4,6,8]
matrix = [[i*j for j in range(3)] for i in range(3)]

# Copy
copy1 = nums[:]                # shallow copy
copy2 = list(nums)
import copy
deep = copy.deepcopy(nested_list)
```

---

## 15.5 Tuples — Immutable

Like a `record` in Java, but simpler.

```python
t = (1, 2, 3)
t[0]                    # 1
len(t)
# t[0] = 99             # ❌ TypeError

# Single-element tuple needs comma
single = (1,)

# Packing / unpacking
a, b, c = t             # a=1, b=2, c=3
a, *rest = t            # a=1, rest=[2,3]

# Named tuples (structured)
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x                     # 3
```

Use tuples for fixed data, dict keys (must be immutable).

---

## 15.6 Dictionaries — Key-Value

Like Java's `HashMap`. Preserves insertion order (Python 3.7+).

```python
d = {"name": "Alice", "age": 25}

# Access
d["name"]              # "Alice"
d.get("email")         # None (safer than d["email"])
d.get("email", "n/a")  # default

# Mutate
d["email"] = "a@x.com"
del d["age"]

# Query
"name" in d
len(d)
list(d.keys())
list(d.values())
list(d.items())

# Iteration
for k, v in d.items():
    print(k, v)

# Dict comprehension
sq = {x: x*x for x in range(5)}                # {0:0, 1:1, 2:4, 3:9, 4:16}

# Merge (Python 3.9+)
a = {"a": 1}
b = {"b": 2}
merged = a | b                                 # {"a": 1, "b": 2}
a |= b                                          # in-place

# Default values
from collections import defaultdict
counts = defaultdict(int)
for word in ["a","b","a"]: counts[word] += 1   # no KeyError

# Counter
from collections import Counter
Counter("mississippi")                          # {'i':4, 's':4, 'p':2, 'm':1}
```

---

## 15.7 Sets — Unique, Unordered

Like Java's `HashSet`.

```python
s = {1, 2, 3}
empty = set()          # NOT {} — that's an empty dict!

s.add(4)
s.remove(2)            # raises KeyError if missing
s.discard(2)           # safe (no error)
2 in s

# Set operations
a = {1, 2, 3}
b = {2, 3, 4}
a | b                  # union {1,2,3,4}
a & b                  # intersection {2,3}
a - b                  # difference {1}
a ^ b                  # symmetric diff {1,4}

# From list — dedupe
list(set([1,1,2,3,3]))

# Set comprehension
{x*x for x in range(5)}
```

---

## 15.8 Control Flow

### If-elif-else

```python
if x > 0:
    print("positive")
elif x == 0:
    print("zero")
else:
    print("negative")

# Ternary
label = "pos" if x > 0 else "neg"
```

### For loops

```python
for i in range(5):                # 0..4
    print(i)

for i in range(1, 10, 2):         # 1, 3, 5, 7, 9
    print(i)

for i, v in enumerate(["a","b","c"]):
    print(i, v)                    # 0 a, 1 b, 2 c

for k, v in {"a": 1, "b": 2}.items():
    print(k, v)

# Parallel iteration
names = ["Alice", "Bob"]
ages = [25, 30]
for name, age in zip(names, ages):
    print(name, age)
```

### While
```python
n = 5
while n > 0:
    print(n)
    n -= 1

# break, continue, pass
for i in range(10):
    if i == 5: break
    if i % 2 == 0: continue
    print(i)
```

### match (Python 3.10+ — pattern matching)

```python
def status_message(code):
    match code:
        case 200: return "OK"
        case 404: return "Not Found"
        case 500 | 502 | 503: return "Server Error"
        case _: return "Unknown"
```

---

## 15.9 Functions

```python
def greet(name, greeting="Hello"):     # default args
    """Docstring — appears in help(greet)."""
    return f"{greeting}, {name}"

greet("Alice")                          # "Hello, Alice"
greet("Bob", greeting="Hi")             # keyword arg

# *args (variable positional) & **kwargs (variable keyword)
def foo(*args, **kwargs):
    print(args)         # tuple
    print(kwargs)       # dict

foo(1, 2, 3, name="A", age=10)
# args=(1,2,3), kwargs={"name":"A","age":10}

# Unpacking on call
nums = [1, 2, 3]
def add(a, b, c): return a + b + c
add(*nums)               # 6

info = {"a": 1, "b": 2, "c": 3}
add(**info)              # 6

# Lambda (single expression)
sq = lambda x: x * x
sq(5)                    # 25

# Higher-order
list(map(lambda x: x*2, [1,2,3]))        # [2,4,6]
list(filter(lambda x: x > 1, [1,2,3]))   # [2,3]

from functools import reduce
reduce(lambda a, b: a + b, [1,2,3,4])    # 10
```

---

## 15.10 OOP in Python

```python
class Animal:
    kingdom = "Animalia"              # class variable (shared)

    def __init__(self, name, age):    # constructor
        self.name = name              # instance variable
        self.age = age

    def speak(self):                  # instance method
        return "Some sound"

    def __str__(self):                # like toString
        return f"Animal({self.name})"

    def __repr__(self):               # for debugging
        return f"Animal(name={self.name!r}, age={self.age})"

    def __eq__(self, other):          # equality
        return isinstance(other, Animal) and self.name == other.name

    def __hash__(self):
        return hash(self.name)

# Inheritance
class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)   # parent constructor
        self.breed = breed

    def speak(self):                  # override
        return "Woof!"

    def fetch(self):
        return f"{self.name} fetches"

# Usage
d = Dog("Rex", 3, "Labrador")
print(d.speak())        # Woof!
print(d)                # Animal(Rex)  (uses __str__)
isinstance(d, Animal)   # True

# Encapsulation convention
class Account:
    def __init__(self):
        self.owner = "public"        # public
        self._internal = "protected" # convention: leading _
        self.__secret = "private"    # name-mangled to _Account__secret

# Class methods & static methods
class MathUtils:
    @classmethod
    def create(cls):                  # gets class
        return cls()

    @staticmethod
    def add(a, b):                    # no self, no cls
        return a + b

MathUtils.add(1, 2)                   # 3
```

### Dataclasses (Python 3.7+) — like Java records

```python
from dataclasses import dataclass

@dataclass
class Book:
    title: str
    author: str
    price: float = 0.0                # default

b = Book("Java", "Bloch", 500)
print(b)                              # Book(title='Java', author='Bloch', price=500)
b == Book("Java", "Bloch", 500)       # True — __eq__ auto-generated
```

### Properties (getters/setters, but Pythonic)

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32

t = Temperature(25)
t.celsius          # 25  (uses getter, no ())
t.celsius = 30     # uses setter
t.fahrenheit       # 86.0  (computed, read-only)
```

---

## 15.11 Exception Handling

```python
try:
    x = 10 / 0
except ZeroDivisionError as e:
    print("zero", e)
except (ValueError, TypeError) as e:
    print("multi", e)
except Exception as e:
    print("catch-all")
else:
    print("no exception")            # runs if no exception
finally:
    print("always runs")

# Raising
def check_age(age):
    if age < 0:
        raise ValueError("Age must be positive")

# Custom exception
class InsufficientFundsError(Exception):
    def __init__(self, balance, requested):
        super().__init__(f"Balance {balance} < {requested}")
        self.balance = balance
        self.requested = requested
```

Python has NO checked exceptions — everything is unchecked.

---

## 15.12 File I/O

```python
# Write
with open("data.txt", "w") as f:      # 'with' auto-closes (like try-with-resources)
    f.write("Line 1\n")
    f.writelines(["Line 2\n", "Line 3\n"])

# Read
with open("data.txt", "r") as f:
    content = f.read()                # entire file as string
    # or
    for line in f:
        print(line.rstrip())          # line by line (memory-friendly)

# Read all lines
with open("data.txt") as f:
    lines = f.readlines()             # list of lines

# Append
with open("data.txt", "a") as f:
    f.write("added\n")

# Binary
with open("img.png", "rb") as f:
    data = f.read()

# JSON
import json
data = {"name": "Alice", "age": 25}

with open("out.json", "w") as f:
    json.dump(data, f, indent=2)

with open("out.json") as f:
    loaded = json.load(f)

# String ↔ JSON
s = json.dumps(data)
d = json.loads(s)
```

**Modes:** `r` (read), `w` (write, truncates), `a` (append), `b` (binary), `+` (r/w).

---

## 15.13 Modules & Packages

```python
# Import
import math
math.sqrt(16)

from math import sqrt, pi
sqrt(16)

from math import sqrt as s
s(16)

import numpy as np                     # alias

# Custom module
# mymath.py
def square(x): return x * x

# main.py
import mymath
mymath.square(5)

# Package (folder with __init__.py; optional in Python 3.3+)
mypackage/
├── __init__.py
├── mod1.py
└── mod2.py

# Standard library gems
import os                              # OS operations
import sys                              # system
import re                               # regex
import datetime
import json
import random
import collections
import itertools
import functools
import pathlib
```

---

## 15.14 Useful Standard Library

### datetime

```python
from datetime import datetime, date, timedelta, timezone

now = datetime.now()
today = date.today()
tomorrow = today + timedelta(days=1)
utc = datetime.now(timezone.utc)

now.strftime("%Y-%m-%d %H:%M:%S")
datetime.strptime("2025-01-15", "%Y-%m-%d")
```

### pathlib (modern paths)

```python
from pathlib import Path

p = Path("data") / "input.txt"
p.exists()
p.is_file()
p.read_text()
p.write_text("hello")
Path(".").glob("*.py")                 # iterator
Path("./logs").mkdir(exist_ok=True)
```

### os / environment

```python
import os
os.getcwd()
os.environ.get("PATH")
os.environ["MY_VAR"] = "value"
os.listdir(".")
```

### collections — essential

```python
from collections import Counter, defaultdict, deque, OrderedDict, namedtuple

Counter("hello")                       # {'l':2, 'h':1, 'e':1, 'o':1}
Counter([1,2,2,3]).most_common(2)      # [(2,2),(1,1)]

d = defaultdict(list)
d["fruit"].append("apple")             # no KeyError

q = deque([1,2,3])                     # fast O(1) at both ends
q.appendleft(0); q.append(4); q.popleft()
```

### itertools — iteration helpers

```python
from itertools import chain, combinations, permutations, product, groupby

list(chain([1,2], [3,4]))                          # [1,2,3,4]
list(combinations([1,2,3], 2))                     # [(1,2),(1,3),(2,3)]
list(permutations([1,2,3], 2))                     # [(1,2),(1,3),(2,1),...]
list(product([1,2], ['a','b']))                    # [(1,'a'),(1,'b'),(2,'a'),(2,'b')]
```

### functools

```python
from functools import lru_cache, reduce, partial

@lru_cache(maxsize=None)                # memoization!
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

reduce(lambda a, b: a + b, [1,2,3,4])   # 10

add = lambda a, b: a + b
add10 = partial(add, 10)                # partial application
add10(5)                                 # 15
```

### requests — HTTP (not stdlib but ubiquitous)

```python
import requests

r = requests.get("https://api.github.com/users/octocat")
r.status_code                          # 200
r.json()                                # dict
r.text                                  # string

data = {"title": "New"}
r = requests.post(url, json=data, headers={"Authorization": "Bearer x"})
```

---

## 15.15 Comprehensions & Generators

### List / dict / set comprehensions

```python
squares = [x*x for x in range(10)]
evens = [x for x in nums if x % 2 == 0]
sq_dict = {x: x*x for x in range(5)}
unique_lens = {len(w) for w in words}
```

### Generators — lazy, memory-efficient

```python
# Generator function
def count_up_to(n):
    i = 0
    while i < n:
        yield i                        # produces values one at a time
        i += 1

for x in count_up_to(5): print(x)      # 0,1,2,3,4

# Generator expression (parens instead of brackets)
gen = (x*x for x in range(1_000_000))  # doesn't create a huge list
sum(gen)
```

Use generators for large sequences — no memory explosion.

---

## 15.16 Decorators

Functions that wrap other functions. Metaprogramming FTW.

```python
def log(fn):
    def wrapper(*args, **kwargs):
        print(f"Calling {fn.__name__} with {args} {kwargs}")
        result = fn(*args, **kwargs)
        print(f"→ {result}")
        return result
    return wrapper

@log
def add(a, b):
    return a + b

add(2, 3)
# Calling add with (2, 3) {}
# → 5
```

Real-world: `@functools.lru_cache`, `@dataclass`, `@app.route` in Flask, `@staticmethod`, `@classmethod`, `@property`.

### Timing decorator

```python
import time, functools

def timeit(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        print(f"{fn.__name__} took {time.perf_counter() - start:.3f}s")
        return result
    return wrapper

@timeit
def slow():
    time.sleep(1)
    return "done"
```

---

## 15.17 Context Managers — `with` statement

```python
with open("data.txt") as f:            # auto-closes on exit
    data = f.read()

# Custom context manager
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.time()
    yield
    print(f"Elapsed: {time.time() - start:.3f}s")

with timer():
    time.sleep(1)                       # Elapsed: 1.001s
```

---

## 15.18 Common Utilities

```python
# zip
list(zip([1,2,3], ["a","b","c"]))     # [(1,'a'),(2,'b'),(3,'c')]

# enumerate
for i, v in enumerate(["a","b"]): print(i, v)

# any / all
any([False, True, False])              # True
all([True, True])                       # True

# sorted with key
users = [{"name": "Alice", "age": 30}, {"name": "Bob", "age": 25}]
sorted(users, key=lambda u: u["age"])  # sort by age
sorted(users, key=lambda u: u["age"], reverse=True)

# min/max
max([1,2,3])
max(users, key=lambda u: u["age"])     # {"name": "Alice", ...}

# print — flexible
print("a", "b", "c", sep="-", end="!\n")   # a-b-c!
```

---

## 15.19 Python Concurrency (Quick Overview)

Due to the **GIL (Global Interpreter Lock)**, only one thread executes Python bytecode at a time in CPython.

- **Threading** — good for I/O-bound (network, disk)
- **Multiprocessing** — for CPU-bound (bypasses GIL)
- **asyncio** — event loop for high-concurrency I/O

```python
# Threading
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=4) as ex:
    results = list(ex.map(fetch_url, urls))

# Multiprocessing
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as ex:
    results = list(ex.map(cpu_task, data))

# asyncio
import asyncio
async def fetch(url): ...

async def main():
    urls = [...]
    results = await asyncio.gather(*(fetch(u) for u in urls))

asyncio.run(main())
```

---

## 15.20 Testing — pytest

```python
# test_calc.py
import pytest

def add(a, b): return a + b

def test_add():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, 1) == 0

@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (5, 5, 10),
    (-1, 1, 0),
])
def test_add_multi(a, b, expected):
    assert add(a, b) == expected

def test_raises():
    with pytest.raises(ValueError):
        int("not a number")
```

Run: `pytest`

---

## 15.21 Common Pythonic Idioms

```python
# Swap
a, b = b, a

# Chained comparisons
if 0 < x < 100: ...

# Dict get with default
count = d.get("key", 0) + 1

# Enumerate over pairs
for i, item in enumerate(items):
    ...

# List conditional
result = [x if x > 0 else 0 for x in nums]

# Reversed iteration
for x in reversed(list):
    ...

# Sum with generator (no intermediate list)
sum(x*x for x in range(1_000_000))

# Truthiness idioms
if not lst: print("empty")
if not user: print("no user")

# Walrus operator (Python 3.8+)
while (line := file.readline()):
    process(line)
```

---

## 15.22 Common Python Interview Questions

1. **List vs Tuple?**  
   List mutable, tuple immutable. Tuples can be dict keys.

2. **What is PEP 8?**  
   Python style guide (indent 4 spaces, snake_case, etc.).

3. **What is the GIL?**  
   Global Interpreter Lock — only one thread executes bytecode at a time in CPython. Real parallelism needs multiprocessing.

4. **`is` vs `==`?**  
   `is` — identity (same object in memory). `==` — equality.

5. **Deep vs shallow copy?**  
   `copy.copy()` — shallow (nested objects shared). `copy.deepcopy()` — recursive copy.

6. **What are decorators?**  
   Functions that wrap other functions. Used for logging, caching, auth, timing.

7. **What are generators?**  
   Functions with `yield` — lazy, memory-efficient sequences.

8. **`*args` and `**kwargs`?**  
   Variable positional args (tuple) / keyword args (dict).

9. **List comprehension vs generator expression?**  
   Comprehension creates list in memory; gen expr is lazy.

10. **How is Python interpreted?**  
    Source → bytecode (.pyc) → interpreted by PVM (Python Virtual Machine).

11. **What are dunder methods?**  
    Double-underscore methods like `__init__`, `__str__`, `__eq__`. Define object behavior.

12. **Difference between `@staticmethod` and `@classmethod`?**  
    `staticmethod`: no self/cls, plain function in class. `classmethod`: receives class as `cls`, useful for alternative constructors.

13. **What is `if __name__ == "__main__"`?**  
    Runs the block only if script is executed directly, not when imported as a module.

14. **Mutable default arg gotcha?**  
    ```python
    def bad(items=[]):    # ❌ same list reused across calls!
        items.append(1)
        return items
    def good(items=None): # ✅
        if items is None: items = []
        ...
    ```

15. **What is `pip` and `venv`?**  
    pip = package installer. venv = isolated virtual environment for dependencies.

16. **`__str__` vs `__repr__`?**  
    `__str__` — user-friendly. `__repr__` — for developers/debug (should be unambiguous).

17. **What is a Python namespace?**  
    Mapping from names to objects. Local, enclosing, global, built-in (LEGB rule).

18. **What is monkey patching?**  
    Modifying a class or module at runtime (add/replace methods). Powerful but risky.

19. **How to iterate a dictionary sorted by value?**  
    `for k, v in sorted(d.items(), key=lambda x: x[1]):`

20. **What are Python's data structures for common tasks?**  
    - Fast lookup: `dict` / `set`
    - Ordered: `list`
    - Fixed order + immutable: `tuple`
    - Priority: `heapq`
    - FIFO/LIFO: `collections.deque`
    - Frequency: `collections.Counter`

---

## 15.23 Java vs Python — Side-by-Side

| Concept | Java | Python |
|---------|------|--------|
| Print | `System.out.println("hi")` | `print("hi")` |
| For loop | `for (int i=0; i<n; i++)` | `for i in range(n):` |
| Null | `null` | `None` |
| Array/List | `ArrayList<Integer>` | `list` |
| Map | `HashMap<K,V>` | `dict` |
| Set | `HashSet<T>` | `set` |
| Immutable list | `List.of(1,2,3)` | `(1,2,3)` (tuple) |
| String | `String s = "x"` | `s = "x"` |
| Concatenate | `sb.append()` | `"".join([...])` or f-string |
| Type declaration | `int x = 5` | `x = 5` (or `x: int = 5`) |
| Interface | `interface X { }` | Duck typing / ABC |
| Static method | `static void x() {}` | `@staticmethod` |
| Try-with-resources | `try (...)` | `with ...:` |
| Braces | `{}` | Indentation |
| Ternary | `a > 0 ? "+" : "-"` | `"+" if a > 0 else "-"` |
| Lambda | `x -> x * 2` | `lambda x: x * 2` |

---

## 15.24 When to Use Python (vs Java)

**Reach for Python when:**
- Small scripts, automation, glue code
- Data analysis, ML, notebooks
- Rapid prototyping
- Web scraping
- DevOps tools

**Stick with Java when:**
- Large-scale enterprise backends
- High-throughput services (better JIT, threading)
- Team knows Java, has Java infrastructure
- Type-safety matters for large codebases (though Python has mypy)

---

## 15.25 Cheat Sheet

```
Everything is an object (even functions & classes)
Dynamic typing; type hints optional
Lists mutable, tuples immutable, sets unique, dicts key-value
List comprehensions are Pythonic: [x*x for x in range(10)]
Generators (yield) for lazy sequences
Dunder methods (__init__, __str__, __eq__) define object behavior
@dataclass for record-like classes
GIL: use asyncio/threading for I/O, multiprocessing for CPU
with ... = try-with-resources; @contextmanager for custom
pip + venv for env management
pytest for testing
Always use `if __name__ == "__main__":` in scripts
```

---

## Practical Assignments

1. Solve a couple of LeetCode problems in Python — appreciate the conciseness.
2. Write a script that reads a JSON file, filters entries, and writes a CSV.
3. Convert a Java class to Python (`@dataclass` version).
4. Write a decorator that logs function calls with args & duration.
5. Use `requests` to fetch data from a public API and pretty-print with `json`.
6. Use `pytest` to write parametrized tests for a small function.

That's enough Python to handle interview questions confidently and dabble in real projects. 🐍