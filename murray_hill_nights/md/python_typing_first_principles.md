# Python Typing from First Principles

In C, C++ and Java the type system belongs to the compiler. A type error stops the build, and a program that compiled is guaranteed to agree with its own declarations. Python's type system works differently in one fundamental way, and nearly every surprise follows from it: **Python itself ignores type hints.** They are read and enforced by a separate tool that you run alongside your code, much as you'd run a linter.

This essay covers that model and the everyday vocabulary, written for someone arriving from statically typed languages. It's the foundation for the rest of this folder. The examples were checked with Python 3.13, pyright 1.1.414 and mypy in October 2026.

## 1. A type hint is an annotation, not a declaration

```python
def double(x: int) -> int:
    return x * 2

rate: float = 0.05
```

That syntax attaches annotations to the parameter, the return value and the variable. Python evaluates them, stores them (`double.__annotations__` is `{'x': <class 'int'>, 'return': <class 'int'>}`), and otherwise does nothing with them. So this runs without complaint:

```python
print(double("ab"))   # prints "abab"
```

The hint says `int`, the call passes a `str`, and Python happily repeats the string. Nothing checks the annotation at runtime unless you add a library that does.

The checking happens on a separate path:

```
                      ┌──────────────────────┐
                ┌────▶│  type checker        │──▶  errors, before running
                │     │  pyright / mypy      │     (in the IDE, CI, pre-commit)
 ┌───────────┐  │     └──────────────────────┘
 │ your .py  │──┤
 │ with hints│  │     ┌──────────────────────┐
 └───────────┘  └────▶│  Python interpreter  │──▶  runs the program,
                      │  ignores the hints   │     hints and all
                      └──────────────────────┘
```

Run pyright over the same file and it reports:

```
error: Argument of type "Literal['ab']" cannot be assigned to parameter "x"
       of type "int" in function "double"
```

Your program and your type checker are two different consumers of the same source. The checker gives you most of what a compiler's type pass gave you. The interpreter never sees any of it.

Since Python 3.14, annotations aren't even evaluated when a function or class is defined. They're computed lazily, only if something asks for them. That's an implementation detail for most code, but it underlines the point: annotations are metadata.

## 2. Who does the checking

| Checker | From | Notes |
| --- | --- | --- |
| **pyright** | Microsoft | Fast, strict, thorough. Powers Pylance, the Python language server in VS Code, so its errors appear as you type. |
| **mypy** | The original, now a Python community project | The reference implementation that much of the typing ecosystem grew up around. |
| **Pyrefly** | Meta | Newer, written in Rust. Reached 1.0 in May 2026. |
| **ty** | Astral (makers of ruff and uv) | Newer, written in Rust. Still 0.0.x as of late 2026. |

They mostly agree, but not always: the same code can pass one checker and fail another. Pick one per project and make it part of the build. For ngstats that's pyright, which you already get in VS Code through Pylance.

## 3. Gradual typing: unannotated code is unchecked code

Python lets you type a codebase partially. That's called **gradual typing**, and the mechanism is the special type **`Any`**. `Any` is compatible with everything in both directions: you can pass anything where `Any` is expected and use an `Any` value anywhere. Checking simply stops at it.

Unannotated code is effectively `Any`:

```python
def untyped(x):
    return x.nonexistent_method()

y = untyped(3)     # pyright: y is "Unknown"   mypy: y is "Any"
```

Neither checker complains about calling a method that doesn't exist on an `int`, because `x` has no declared type to check against. pyright's name for an inferred-but-unknowable type is `Unknown`, which is its way of distinguishing "you didn't say" from an explicit `Any`.

The two checkers also differ on what they look at:

```python
def untyped(x):
    return 1 + "a"
```

pyright reports the `int + str` error anyway. **mypy, by default, doesn't check the bodies of unannotated functions at all.** It reports nothing until you annotate the function or run `mypy --strict`, which also flags `Function is missing a type annotation`.

This is why every strict setup turns on "strict mode": it treats missing annotations and unknown types as errors, so `Any` can't spread silently. A project without strict mode can look fully checked while large parts of it aren't.

## 4. The everyday vocabulary

### Built-in and container types

```python
count: int
ratio: float           # also accepts an int
name: str
ok: bool
raw: bytes

values: list[float]
lookup: dict[str, int]
pair: tuple[str, float]        # exactly two items, in that order
samples: tuple[float, ...]     # any length, all floats
seen: set[str]
```

`list` means a list, `tuple[str, float]` means a fixed-shape record, and `tuple[float, ...]` means an immutable sequence of any length. That last spelling surprises most people the first time.

### Accept broadly, return precisely

For parameters, prefer the abstract types from `collections.abc`:

```python
from collections.abc import Iterable, Mapping, Sequence

def mean(xs: Sequence[float]) -> float: ...       # list, tuple, ... all fine
def total(xs: Iterable[float]) -> float: ...      # even a generator
def describe(cfg: Mapping[str, str]) -> str: ...  # dict or any read-only mapping
```

For return values, be concrete (`list[float]`, `dict[str, int]`) so callers know exactly what they get. It's the same advice as programming to interfaces in Java.

### Unions and `None`

```python
def parse(s: str) -> int | float: ...
def find(name: str) -> int | None: ...
```

`X | None` means "an X or nothing," and the checker makes you handle the `None`:

```python
def length(s: str | None) -> int:
    return len(s)          # error: "None" is incompatible with "Sized"

def length(s: str | None) -> int:
    if s is None:
        return 0
    return len(s)          # fine: s is narrowed to str here
```

That second version shows **narrowing**: after `if s is None: return`, the checker knows `s` is a `str`. `isinstance` checks, truthiness tests and early returns all narrow. This is the closest Python gets to eliminating null-pointer bugs, and it's one of the biggest payoffs of typing.

### `Literal`

```python
from typing import Literal

Alternative = Literal["two-sided", "less", "greater"]

def ttest(a: Sequence[float], b: Sequence[float], alternative: Alternative = "two-sided") -> float: ...

ttest(a, b, alternative="both")   # error: "both" is not one of the allowed values
```

A `Literal` restricts a string (or int, or bool) to specific values. It's a lightweight enum, and it reads well in signatures.

### `Callable`

```python
from collections.abc import Callable

def apply(f: Callable[[float], float], xs: list[float]) -> list[float]:
    return [f(x) for x in xs]
```

`Callable[[arg types], return type]` describes a function. For anything fancier, such as keyword arguments, see callback protocols in `protocols_and_abstract_base_classes.md`.

### Type aliases

```python
type Vector = list[float]       # Python 3.12+

def norm(v: Vector) -> float: ...
```

An alias is a name for a type. It doesn't create a new type: a `Vector` is a `list[float]`. Use `typing.NewType` when you want the checker to keep two otherwise identical types apart.

### A first look at generics

```python
def first[T](xs: Sequence[T]) -> T:      # Python 3.12+
    return xs[0]

first([1, 2, 3])    # int
first("abc")        # str
```

`T` is a type variable: whatever element type goes in comes out. This is Java's `<T> T first(List<T> xs)` with lighter syntax. Generics deserve an essay of their own. For now it's enough to read them.

### `Final`

```python
from typing import Final

DEFAULT_ALPHA: Final = 0.05      # the checker rejects reassignment
```

## 5. Old spelling versus new

Python's typing syntax got simpler over several releases, and older code and tutorials still use the old forms. Both are valid. Write the new ones.

| Old | New | Since |
| --- | --- | --- |
| `from typing import List, Dict` … `List[int]` | `list[int]` | 3.9 (PEP 585) |
| `Optional[str]`, `Union[int, str]` | `str \| None`, `int \| str` | 3.10 (PEP 604) |
| `Vector: TypeAlias = list[float]` | `type Vector = list[float]` | 3.12 (PEP 695) |
| `T = TypeVar("T")` then `def f(x: T) -> T` | `def f[T](x: T) -> T` | 3.12 (PEP 695) |
| `from typing import Callable, Sequence` | `from collections.abc import Callable, Sequence` | 3.9 |

You'll also see `from __future__ import annotations` at the top of older modules. It made annotations lazy before Python 3.14 made that the default.

## 6. Inference: you annotate less than you think

The checker infers the types of local variables, so you rarely annotate them:

```python
counts = {"a": 1}     # dict[str, int], inferred
nums = []             # pyright: list[Unknown]   mypy: error, "Need type annotation"
nums: list[float] = []
```

Annotate function signatures, class attributes and module-level constants. Annotate a local only when inference can't work it out, typically an empty container.

**`reveal_type(expr)`** is how you ask the checker what it thinks. Put it in your code and pyright or mypy prints the inferred type. Since Python 3.11 you can import it from `typing`, so the line also runs, printing the runtime type. It's the single most useful debugging tool in typed Python.

## 7. Where types come from

The checker needs type information for everything your code touches:

- **Your own annotations.**
- **Libraries that ship types.** A package containing an empty `py.typed` file is telling checkers "my inline annotations are meant to be used." numpy does this, and ngstats will too.
- **Stub files (`.pyi`).** These are signatures only, no implementation. They're used when a library isn't annotated itself. They come from two places:
    - **typeshed**, a community repository of stubs for the entire standard library and some third-party packages. It's bundled inside every checker, which is why `len`, `open` and `json.dumps` are already typed.
    - **Separate stub packages**, such as `pandas-stubs` or `types-requests`, installed as dev dependencies.
- **Nothing.** For an untyped library with no stubs, everything it returns is `Any` or `Unknown`. `typing_numerical_python.md` covers how to fence that off.

## 8. What the checker can and can't promise

Python's type checking is **unsound by design**: code can pass the checker and still hit a type error at runtime. The gaps are known, and each has a standard answer:

| Gap | Answer | Where it's covered |
| --- | --- | --- |
| `Any` from unannotated code or untyped libraries | Strict mode, typed edges around untyped libraries | `typing_numerical_python.md` |
| Escape hatches: `cast()`, `# type: ignore` | Use few, keep them at boundaries, comment each one | `living_with_pythons_typing.md` |
| Data from outside: JSON, env vars, files | Parse it at the boundary with pydantic | `python_type_checking.md` |
| Callers who don't run a checker | Runtime value checks on your public functions | `python_type_checking.md` |
| Architecture: who may import whom | import-linter | `layered_python_architecture_with_import_linter.md` |

Within those limits, a strictly checked codebase catches the large majority of type errors before the code runs, and the annotations document every signature as they go.

## 9. Translation table for the C, C++ and Java programmer

| C / C++ / Java | Python |
| --- | --- |
| The compiler checks types | A separate checker (pyright, mypy) checks types; the interpreter ignores hints |
| A type error fails the build | Only if you make the checker part of the build (pre-commit, CI) |
| Every variable must be declared | Annotate signatures; locals are inferred |
| `null` / `nullptr` | `None`, opted into with `X \| None` and enforced by narrowing |
| `enum` | `enum.Enum`, or `Literal[...]` for simple string choices |
| Generics `<T>` / templates | `def f[T](...)`, `class Box[T]: ...` |
| Interfaces | `typing.Protocol` (implicit) or `abc.ABC` (explicit) |
| Header files | Stub files (`.pyi`), plus `__all__` for the export list |
| `void` | `None` as a return type |
| Casting | `typing.cast(T, x)`, which only changes what the checker believes and does nothing at runtime |
| `Object` | `object` (safe, requires narrowing) versus `Any` (unchecked) |

That last row is worth remembering. `object` is the honest "could be anything": the checker makes you narrow it before use. `Any` switches checking off. Reach for `object` when you mean "anything," and treat every `Any` as a gap.

## Suggested reading order for this folder

1. **Python Typing from First Principles**: this essay
2. **Living With Python's Typing**: technique and temperament
3. **Python Type Checking, From the API Down**: who checks what, and where
4. **Protocols and Abstract Base Classes**: interfaces, two ways
5. **Typing Numerical Python**: numpy, pandas, and where the checker gives up
6. **Layered Python Architecture with import-linter**: enforcing structure

## Sources

- [typing — Support for type hints, Python documentation](https://docs.python.org/3/library/typing.html)
- [What's New in Python 3.14: deferred evaluation of annotations](https://docs.python.org/3/whatsnew/3.14.html)
- [PEP 484: Type Hints](https://peps.python.org/pep-0484/)
- [PEP 585](https://peps.python.org/pep-0585/), [PEP 604](https://peps.python.org/pep-0604/), [PEP 695](https://peps.python.org/pep-0695/)
- [mypy documentation: untyped definitions and strict mode](https://mypy.readthedocs.io/en/stable/command_line.html)
- [Pyright documentation](https://microsoft.github.io/pyright/)
- [typeshed on GitHub](https://github.com/python/typeshed)
- [ty vs Pyrefly, pydevtools.com (September 2026)](https://pydevtools.com/handbook/explanation/ty-vs-pyrefly/)
