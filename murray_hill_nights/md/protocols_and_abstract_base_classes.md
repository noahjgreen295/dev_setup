# Protocols and Abstract Base Classes

Python has two ways to say "this kind of object." An **abstract base class** (ABC) defines a family by inheritance. A class belongs because it says so, by subclassing the ABC. In return it can inherit shared code, and Python refuses to instantiate it until the abstract methods are filled in. A **protocol** defines a family by shape. A class belongs if it has the right methods and attributes, whether or not it has ever heard of the protocol, and a type checker verifies the fit before the code runs.

The first is **nominal** typing: membership by name. The second is **structural** typing: membership by structure. Java's `abstract class` is nominal. A Java `interface` is nominal too, because you must write `implements`. A protocol is closer to an interface you satisfy without declaring it, which is how Go interfaces and C++20 concepts work.

```
   NOMINAL (ABC)                         STRUCTURAL (Protocol)

        ┌──────────┐                          ┌──────────────┐
        │ BaseTest │                          │ SupportsTest │
        └──────────┘                          │  name: str   │
          ▲      ▲                            │  run(a, b)   │
 inherits │      │ inherits                   └──────────────┘
   ┌──────┴───┐ ┌┴────────────┐                ▲      ▲      ▲
   │WelchTTest│ │MannWhitneyU │        matches ┊      ┊      ┊ matches
   └──────────┘ └─────────────┘     ┌──────────┐ ┌────────┐ ┌──────────┐
                                    │WelchTTest│ │ MyTest │ │ LegacyT  │
                                    └──────────┘ └────────┘ └──────────┘
```

A quick word on naming: Python has no `@abstractclass` decorator. A class is abstract when it inherits from `abc.ABC` and marks at least one method with `@abstractmethod`.

## Abstract base classes

### What they are

`abc` has been in the standard library since 2007–2008 (PEP 3119, Python 2.6 and 3.0). You inherit from `ABC`, mark the methods subclasses must supply, and write any shared logic as ordinary methods.

The classic use is the **template method**: the base class owns the algorithm, and subclasses fill in one step.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

import numpy as np
from numpy.typing import ArrayLike, NDArray
from scipy import stats


@dataclass(frozen=True, slots=True)
class TestResult:
    method: str
    statistic: float
    p_value: float


class BaseTest(ABC):
    name: str

    def run(self, a: ArrayLike, b: ArrayLike) -> TestResult:
        x = _as_1d_float(a, "a")
        y = _as_1d_float(b, "b")
        statistic, p_value = self._compute(x, y)
        return TestResult(method=self.name, statistic=statistic, p_value=p_value)

    @abstractmethod
    def _compute(
        self, x: NDArray[np.float64], y: NDArray[np.float64]
    ) -> tuple[float, float]: ...


class WelchTTest(BaseTest):
    name = "Welch t-test"

    def _compute(
        self, x: NDArray[np.float64], y: NDArray[np.float64]
    ) -> tuple[float, float]:
        res = stats.ttest_ind(x, y, equal_var=False)
        return float(res.statistic), float(res.pvalue)


class MannWhitneyU(BaseTest):
    name = "Mann-Whitney U"

    def _compute(
        self, x: NDArray[np.float64], y: NDArray[np.float64]
    ) -> tuple[float, float]:
        res = stats.mannwhitneyu(x, y)
        return float(res.statistic), float(res.pvalue)
```

(`_as_1d_float` is the coercion helper from `python_type_checking.md`.)

Validation, packaging and naming live in one place. Each subclass writes only the statistics.

### What they enforce

The enforcement happens at runtime and needs no tooling:

```python
class Forgetful(BaseTest):
    name = "oops"

Forgetful()   # TypeError: Can't instantiate abstract class Forgetful
              # without an implementation for abstract method '_compute'
```

The error comes at instantiation, not at definition and not when the missing method is finally called. That's early enough to catch in any test that builds the object.

Abstract properties work too: stack `@property` on top of `@abstractmethod`.

### Two lesser-known features

**`register()` creates virtual subclasses.** `BaseTest.register(LegacyTest)` makes `isinstance(LegacyTest(), BaseTest)` return `True` without inheritance. Python checks nothing when you do this, so it's a promise, not a verification. It's the ABC world's escape hatch for types you can't make inherit.

**Some standard ABCs are already structural.** `collections.abc.Iterable`, `Sized`, `Hashable` and a few others define a `__subclasshook__` that looks for the right dunder method. A class with an `__iter__` method counts as `Iterable` for `isinstance` even if it never inherited from it. The ABC world had shape-based checks all along, for a handful of one-method interfaces.

### Limits

- **Implementers must inherit from you.** Their code now imports your package and sits in your hierarchy.
- **Types you don't own can't join** except through `register()`, which verifies nothing.
- **Inheritance couples.** Changing the base class changes every subclass, which is the point when you own them all and a hazard when you don't.

## Protocols

### What they are

`typing.Protocol` arrived in Python 3.8 (PEP 544, 2019). You write a class listing the methods and attributes you require, with `...` as the bodies:

```python
from collections.abc import Sequence
from typing import Protocol


class SupportsTest(Protocol):
    name: str

    def run(self, a: ArrayLike, b: ArrayLike) -> TestResult: ...


def run_batch(
    tests: Sequence[SupportsTest], a: ArrayLike, b: ArrayLike
) -> list[TestResult]:
    return [t.run(a, b) for t in tests]
```

`WelchTTest` and `MannWhitneyU` satisfy `SupportsTest` automatically. So does a user's class that has never imported `BaseTest`:

```python
class PermutationTest:              # no base class at all
    name = "permutation test"

    def run(self, a: ArrayLike, b: ArrayLike) -> TestResult:
        ...
```

Pass it to `run_batch` and pyright is satisfied. Change `run`'s signature so it no longer matches, and pyright reports the mismatch at the call site, before anything runs.

### What they enforce

At runtime, nothing. A protocol class is close to inert, and passing a non-matching object raises no error until a missing method is actually called. The value of a protocol is entirely in what pyright or mypy does with it. That's why it's easy to go years without meeting one: without a type checker in the build, it has nothing to offer.

**`@runtime_checkable`** adds limited `isinstance` support:

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class SupportsMean(Protocol):
    def mean(self) -> float: ...


isinstance(np.array([1.0, 2.0]), SupportsMean)   # True
isinstance([1.0, 2.0], SupportsMean)             # False: list has no .mean()
```

It checks only that the members exist, not their signatures or return types, and it's slower than an ordinary `isinstance`. Use it for dispatch, not as validation.

### Lesser-known features

**Protocols can carry default code, for explicit subclasses only.** A class may subclass a protocol on purpose (`class WelchTTest(SupportsTest): ...`). It then inherits any method bodies the protocol defines, and the checker verifies the fit at the class definition. Implicit implementers get the contract without the code.

**Data attributes count.** `name: str` in the protocol means implementers must have a `name` attribute of type `str`.

**The standard library is full of protocols.** `typing.SupportsFloat`, `SupportsInt`, `SupportsIndex` and `SupportsAbs` are protocols. In typeshed, the type stubs that pyright and mypy use, `Iterable`, `Sized`, `Container` and friends are declared as protocols too. You've been using structural types every time you annotated a parameter as `Iterable[float]`.

### Limits

- **No runtime enforcement** for users who don't run a type checker.
- **No shared implementation** for implicit implementers.
- **Accidental matches are possible.** Anything with the right shape qualifies, even if its `run` means something unrelated. In practice this rarely causes trouble, because the names and signatures are specific.

## More examples

### Side by side

| | Abstract base class | Protocol |
| --- | --- | --- |
| Membership | Declared by inheriting (nominal) | Implied by shape (structural) |
| Checked by | Python, at instantiation | pyright / mypy, before running |
| Useful without a type checker | Yes | Barely (`@runtime_checkable` only) |
| Shares code with implementers | Yes | Only with explicit subclasses |
| Fits types you don't own | Only via `register()`, unchecked | Yes |
| Introduced | Python 2.6 / 3.0 (PEP 3119) | Python 3.8 (PEP 544) |
| Closest outside analogy | Java `abstract class` | Go interface, C++20 concept |

### Both in one design

The ngstats example above already uses both, and that's the usual pattern:

- **`BaseTest` (ABC)** is for the people *writing* tests inside ngstats. It shares the validate-compute-package code and refuses to instantiate an incomplete test.
- **`SupportsTest` (Protocol)** is for the code *consuming* tests: `run_batch`, a multiple-comparison helper, the API layer. It accepts ngstats' own tests and anyone else's.

Writers inherit from the ABC. Consumers annotate with the protocol. Neither side constrains the other.

### A callback protocol

A protocol with `__call__` describes a function's shape, so plain functions satisfy it:

```python
class PValueAdjuster(Protocol):
    def __call__(self, pvalues: Sequence[float], /) -> list[float]: ...


def bonferroni(pvalues: Sequence[float], /) -> list[float]:
    m = len(pvalues)
    return [min(1.0, p * m) for p in pvalues]


def adjusted_pvalues(
    results: Sequence[TestResult], adjust: PValueAdjuster = bonferroni
) -> list[float]:
    return adjust([r.p_value for r in results])
```

The `/` makes the parameter positional-only. Without it, the checker also requires the parameter *name* to match, so a user's `def holm(ps: Sequence[float])` would be rejected for calling it `ps`. For a simple signature, `Callable[[Sequence[float]], list[float]]` does the same job. A callback protocol is better when you need keyword arguments, overloads, or a named type that reads well in docs.

### Accepting arrays without naming numpy

```python
class SupportsMeanAndStd(Protocol):
    def mean(self) -> float: ...
    def std(self) -> float: ...


def zscore_of_mean(x: SupportsMeanAndStd, mu: float) -> float:
    return (x.mean() - mu) / x.std()
```

The signature says exactly what the function uses. It doesn't import numpy or pandas, and it doesn't claim to need more than two methods. How well numpy arrays and pandas Series match depends on how their own type stubs declare the return types, so check what pyright says in practice. In ngstats you'll usually take `ArrayLike` and coerce with `np.asarray` instead, but the pattern is useful wherever coercion doesn't make sense.

### When an ABC is the only right answer

Suppose ngstats grows a plug-in system where third parties register new tests by name, and the registry instantiates them. Most plug-in authors won't run a type checker. Here the ABC's runtime enforcement matters: a plug-in missing `_compute` fails loudly at registration, not halfway through someone's analysis.

## Conclusion

Neither replaces the other. They answer different questions:

- **"What does this function need from its argument?"** That's a protocol. It describes the minimum, accepts types you don't own, and costs nothing at runtime.
- **"How do I build a family of classes that share code and can't be left incomplete?"** That's an ABC. It gives inheritance, a template for subclasses, and runtime enforcement that works without any tooling.

A practical default: annotate parameters with protocols (or the standard ones, like `Iterable`) and reach for an ABC only when you own the implementations and want them to share code. With a strict type checker in the build, as ngstats will have, protocols cover most of what interfaces gave you in Java. ABCs stay for the cases where inheritance is the point.

## Sources

- [typing.Protocol, Python documentation](https://docs.python.org/3/library/typing.html#typing.Protocol)
- [abc — Abstract Base Classes, Python documentation](https://docs.python.org/3/library/abc.html)
- [PEP 544: Protocols: Structural subtyping (static duck typing)](https://peps.python.org/pep-0544/)
- [PEP 3119: Introducing Abstract Base Classes](https://peps.python.org/pep-3119/)
