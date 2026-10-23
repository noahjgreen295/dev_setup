# Obsolete Python Practices (and What to Use Instead)

Condensed from "7 'Outdated' Python Practices You Need to Stop Using Right Now in 2026" by chris (Medium, Apr 19, 2026).

None of these old patterns will break your code. Python is backward-compatible, so nothing forces you to modernize; the cost shows up later as unreadable code, swallowed errors, and tooling friction. You don't need a big refactor: adopt one change the next time you touch a file.

---

## 1. `os.path` → `pathlib`

`pathlib` (Python 3.4+) treats paths as objects instead of strings passed to scattered functions.

Old:

```python
import os

file_path = os.path.join("data", "raw", "input.csv")
if os.path.exists(file_path):
    file_name = os.path.basename(file_path)
    file_ext = os.path.splitext(file_path)[1]
    dir_name = os.path.dirname(file_path)
```

New:

```python
from pathlib import Path

file_path = Path("data") / "raw" / "input.csv"
if file_path.exists():
    file_name = file_path.name        # "input.csv"
    file_ext = file_path.suffix       # ".csv"
    dir_name = file_path.parent       # Path("data/raw")
```

Also handy: `file_path.read_text()`, `Path(".").glob("**/*.py")`, and cross-platform paths for free.

**Still use `os` for:** low-level file descriptors, some permission handling, or tools that require string paths.

---

## 2. `typing.List` / `Dict` / `Optional` / `Union` → built-in generics and `|`

Since 3.9, built-in collections accept subscripts. Since 3.10, `X | Y` replaces `Union`, and `X | None` replaces `Optional`.

Old:

```python
from typing import Any, List, Dict, Optional, Tuple, Union

def process_records(
    items: List[Dict[str, Any]],
    limit: Optional[int] = None
) -> Tuple[List[str], int]:
    ...

def fetch(key: Union[str, int]) -> Optional[str]:
    ...
```

New:

```python
from typing import Any

def process_records(
    items: list[dict[str, Any]],
    limit: int | None = None
) -> tuple[list[str], int]:
    ...

def fetch(key: str | int) -> str | None:
    ...
```

On 3.8 or earlier, `from __future__ import annotations` enables this syntax in annotations. See PEP 604.

---

## 3. `setup.py` → `pyproject.toml`

`setup.py` is executable code, so tools can't read its metadata without running it. `pyproject.toml` is declarative and is the standard for pip, uv, build, hatch, Hatchling, PDM, Flit, and setuptools.

Old:

```python
from setuptools import setup, find_packages

setup(
    name="my-package",
    version="0.1.0",
    packages=find_packages(),
    install_requires=[
        "requests>=2.28",
        "pydantic>=2.0",
    ],
    python_requires=">=3.10",
)
```

New:

```toml
[project]
name = "my-package"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.28",
    "pydantic>=2.0",
]

[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```

If you ship `setup.py` + `setup.cfg` + `MANIFEST.in`, consolidating into `pyproject.toml` shrinks your config considerably.

---

## 4. Broad `except` → specific exceptions

A blanket `except Exception` hides what failed and where.

Old:

```python
try:
    result = fetch_data(url)
    parsed = json.loads(result)
    save_to_db(parsed)
except Exception:
    print("Something went wrong")
```

New:

```python
import json
import logging

logger = logging.getLogger(__name__)

try:
    result = fetch_data(url)
except ConnectionError as e:
    logger.error("Network request failed: %s", e)
    raise

try:
    parsed = json.loads(result)
except json.JSONDecodeError as e:
    logger.error("Invalid JSON in response: %s", e)
    raise ValueError("Malformed API response") from e

save_to_db(parsed)
```

Each failure gets its own handling and log message. For multiple simultaneous exceptions (common in async code), see Exception Groups and `except*` (3.11+).

**Rule of thumb:** if your `except` doesn't name the specific exception class you're guarding against, you're probably catching too much.

---

## 5. Hand-written data classes → `@dataclass` or Pydantic

Old (all boilerplate):

```python
class User:
    def __init__(self, name: str, email: str, age: int, active: bool = True):
        self.name = name
        self.email = email
        self.age = age
        self.active = active

    def __repr__(self):
        return f"User(name={self.name!r}, email={self.email!r}, age={self.age}, active={self.active})"

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return (self.name, self.email, self.age, self.active) == (
            other.name, other.email, other.age, other.active
        )
```

New, with `dataclasses` (stdlib, 3.7+):

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    email: str
    age: int
    active: bool = True
```

You get `__init__`, `__repr__`, and `__eq__`. Options: `frozen=True` (immutable), `slots=True` (3.10+, memory), `order=True` (comparisons).

New, with Pydantic (when you need validation):

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    name: str
    email: EmailStr
    age: int
    active: bool = True
```

Adds runtime validation, JSON/dict serialization, and schema generation.

**Which one:** internal data that never touches outside input → `dataclass`. Data at a boundary (API requests/responses, config parsing, forms) → Pydantic.

---

## 6. `%` and `.format()` → f-strings

Old:

```python
name = "Alice"
score = 95.678
print("Hello %s, your score is %.2f%%" % (name, score))
print("Hello {}, your score is {:.2f}%".format(name, score))
```

New (3.6+):

```python
print(f"Hello {name}, your score is {score:.2f}%")
```

Newer f-string features:

```python
# Python 3.12+: nested quotes work fine
items = ["apple", "banana", "cherry"]
print(f"Items: {", ".join(items)}")

# Debug shorthand (3.8+): the = sign prints both expression and value
threshold = 42
print(f"{threshold = }")  # Output: threshold = 42
```

**Still use `.format()` for:** templates defined ahead of time and filled in later (f-strings evaluate immediately).

```python
# Template defined once, used many times
template = "Hello {name}, welcome to {place}"
print(template.format(name="Bob", place="Paris"))
```

---

## 7. Long `if`/`elif` chains → structural pattern matching

`match`/`case` (3.10+, PEP 634) is more than a switch statement: it checks structure and binds variables in one step.

Old:

```python
def handle_command(command: dict):
    action = command.get("action")
    if action == "move":
        x = command.get("x", 0)
        y = command.get("y", 0)
        move_player(x, y)
    elif action == "attack":
        target = command.get("target")
        if target:
            attack(target)
    elif action == "heal":
        amount = command.get("amount", 10)
        heal_player(amount)
    else:
        print(f"Unknown action: {action}")
```

New:

```python
def handle_command(command: dict):
    match command:
        case {"action": "move", "x": x, "y": y}:
            move_player(x, y)
        case {"action": "move"}:
            move_player(0, 0)
        case {"action": "attack", "target": str(target)}:
            attack(target)
        case {"action": "heal", "amount": int(amount)}:
            heal_player(amount)
        case {"action": "heal"}:
            heal_player(10)
        case _:
            print(f"Unknown command: {command}")
```

`{"action": "attack", "target": str(target)}` confirms the key exists, confirms the value is a string, and assigns it to `target`.

With nested structures and class instances, including guard clauses:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

@dataclass
class Rectangle:
    origin: Point
    width: float
    height: float

def describe_shape(shape):
    match shape:
        case Circle(center=Point(x=0, y=0), radius=r):
            print(f"Circle at origin with radius {r}")
        case Circle(center=Point(x=x, y=y), radius=r) if r > 100:
            print(f"Large circle at ({x}, {y})")
        case Rectangle(width=w, height=h) if w == h:
            print(f"Square with side {w}")
        case Rectangle(origin=Point(x=x, y=y)):
            print(f"Rectangle at ({x}, {y})")
```

**Best for:** complex data structures, command parsing, AST traversal, state machines. Not a replacement for every `if`.
