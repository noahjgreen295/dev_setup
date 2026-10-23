# Typing Numerical Python

The scientific Python stack is typed unevenly. numpy ships detailed type information of its own. pandas and scipy have separate stub packages. statsmodels has nothing. Even where types exist, precision drains away as values pass through arithmetic, indexing and reductions. Writing a strictly typed library on top of this stack means knowing what the checker can see, how precise it is, and where it stops, and then drawing your own typed boundary around the rest.

This essay is a field guide for that, aimed at ngstats. Every `reveal_type` result quoted below came from actually running pyright 1.1.414 against numpy 2.5.3, pandas 3.0.5 with pandas-stubs, scipy 1.18.1 with scipy-stubs, and statsmodels 0.15.0, in October 2026. These stubs change quickly, so check your own versions with `reveal_type` before relying on any specific result.

```
 ┌───────────────────────────────────────────────────────┐
 │  ngstats public API       fully typed, py.typed       │
 ├───────────────────────────────────────────────────────┤
 │  ngstats internals        NDArray[np.float64], float  │
 ├─────────────┬─────────────┬─────────────┬─────────────┤
 │  numpy      │  pandas     │  scipy      │ statsmodels │
 │  ships own  │  pandas-    │  scipy-     │  no types   │
 │  types      │  stubs      │  stubs      │             │
 │  █████████  │  ██████     │  ███████    │  ░          │
 └─────────────┴─────────────┴─────────────┴─────────────┘
        how much of each library the checker can see
```

## Who ships types, and how to get them

| Library | Type information | Install | What pyright sees |
| --- | --- | --- | --- |
| numpy | Ships its own (`py.typed`) | Nothing extra | Detailed: dtypes, and shapes in some places |
| pandas | Separate `pandas-stubs`, maintained by pandas-dev | `uv add --dev pandas-stubs` | `Series[float]`, but DataFrame columns are untyped |
| scipy | Separate `scipy-stubs`, "the official type stubs for SciPy" | `uv add --dev scipy-stubs` | Typed result objects, e.g. `TtestResult[float64, float64]` |
| statsmodels | None | Nothing available | Strict pyright: "Stub file not found," partially unknown types |

Stub packages go in the **dev** dependency group. Only the type checker needs them, so the people who install ngstats don't pay for them. Users who type-check their own code install the stubs themselves.

mypy is stricter than pyright here. For a package with neither `py.typed` nor stubs, mypy reports a missing-stubs error and treats the module as untyped. pyright infers what it can from the library's source and then, in strict mode, complains about everything it couldn't work out.

## numpy: two annotations do most of the work

You need two names from `numpy.typing`:

- **`ArrayLike`** for inputs: "anything numpy can turn into an array."
- **`NDArray[np.float64]`** for arrays you hold and return.

`ArrayLike` is a union of protocols covering buffers, objects with `__array__`, nested sequences and scalars. Lists, tuples, Series and arrays all pass, and a `dict` is rejected. One surprise: **a `str` passes**, because a string is a sequence. `np.asarray("abc", dtype=np.float64)` then fails at runtime with `ValueError: could not convert string to float: 'abc'`, and `np.asarray("1.5", dtype=np.float64)` quietly succeeds. `ArrayLike` documents intent. The real gatekeeping is still coercion plus value checks at the top of each public function.

### Pass `np.float64`, not `float`

This was the most useful finding while researching this essay:

```python
np.asarray([1, 2, 3], dtype=np.float64)   # ndarray[tuple[int], dtype[float64]]
np.asarray([1, 2, 3], dtype=float)        # ndarray[tuple[int], dtype[Any]]
```

At runtime the two are identical. To the type checker, `dtype=float` throws away the dtype and leaves `Any`, which then spreads through everything computed from that array. Write `dtype=np.float64` everywhere you coerce.

### Precision drains through operations

Start with `x`, a 1-D float64 array, and watch what pyright knows after each step:

| Expression | Revealed type |
| --- | --- |
| `x` | `ndarray[tuple[int], dtype[float64]]` |
| `x + 1` | `ndarray[tuple[int], dtype[float64]]` |
| `x * 2.5` | `ndarray[tuple[Any, ...], dtype[float64]]`: shape lost |
| `np.sqrt(x)` | `ndarray[tuple[Any, ...], dtype[Any]]`: dtype lost too |
| `x[0]` | `Any`: everything lost |
| `x.mean()`, `x.std()`, `x.sum()` | `float64` |
| `np.isnan(x).any()` | `numpy.bool` |

```
 x ──────────▶ x * 2.5 ──────────▶ np.sqrt(...) ──────────▶ x[0]
 shape ✓       shape ✗             shape ✗                  Any
 dtype ✓       dtype ✓             dtype ✗
```

The lesson: **don't chase precise types for intermediate values.** Annotate function signatures, let the checker hold each function to its declared return type, and re-establish precision where it matters, either with `np.asarray(..., dtype=np.float64)` or with an explicit annotation (`r: NDArray[np.float64] = np.sqrt(x)` is accepted, because `Any` is assignable to anything).

### Shapes are a bonus, not a contract

Recent numpy stubs track shape in some places. A 1-D `asarray` reveals `tuple[int]`, and `np.zeros((3, 4))` reveals `tuple[Literal[3], Literal[4]]`. But the `NDArray` alias accepts any shape, many operations drop the shape, and there's no way to say "the same length as the other argument." Treat shape information as a pleasant extra and enforce shape at runtime, as `_as_1d_float` does with its `ndim` check.

If you want shapes written into signatures for readers, the third-party `jaxtyping` package supports annotations like `Float[np.ndarray, "n"]`, and it can check them at runtime when paired with `beartype`. pyright doesn't verify those shapes statically.

## numpy scalars versus Python floats

Reductions return numpy scalars, not Python floats. How much that matters depends on which scalar:

| Value | `isinstance(v, float)` | `json.dumps(v)` |
| --- | --- | --- |
| `np.float64(1.5)` | `True` | `1.5` |
| `np.float32(1.5)` | `False` | `TypeError` |
| `np.int64(3)` | `False` | `TypeError` |
| `np.bool_(True)` | `False` | `TypeError` |
| `float("nan")` | `True` | `NaN`, which is not valid JSON |

`np.float64` subclasses `float`, both at runtime and in the stubs, so a function declared `-> float` may return `x.mean()` without complaint. The other numpy scalars don't, and they break JSON serialization. NaN is a trap of a different kind: Python's `json` module emits a bare `NaN` by default, which is not valid JSON, so many clients will reject the response.

Practical rule: **convert with `float()`, `int()` and `bool()` where values leave the numerical core**, meaning in the constructor arguments of your result dataclasses. Then decide NaN policy once, at the API layer.

## pandas with pandas-stubs

With pandas-stubs installed:

```python
s = pd.Series([1.0, 2.0])     # Series[float]
s.mean()                      # float
df["a"]                       # Series[Any]
df.to_numpy()                 # ndarray[tuple[int, int], dtype[Any]]
```

`Series` is generic over its element type, but **DataFrame columns are untyped**: `df["a"]` is `Series[Any]` whatever the column holds. There's no static way to declare "a DataFrame with a float column called `a`."

Conversion has a sharp edge:

```python
def from_series(s: "pd.Series[float]") -> NDArray[np.float64]:
    return s.to_numpy(dtype=np.float64)   # error: floating[Any] is not float64


def from_series(s: "pd.Series[float]") -> NDArray[np.float64]:
    return np.asarray(s, dtype=np.float64)   # OK
```

`to_numpy` is typed as returning `floating[Any]` even when you pass `dtype=np.float64`. `np.asarray` with an explicit dtype gives the checker what you want, so the coercion step at the top of each public function is also the typing fix.

The quotes around `"pd.Series[float]"` keep the subscript out of runtime evaluation. `from __future__ import annotations` at the top of the module does the same for every annotation in the file.

For ngstats, keep pandas out of public signatures. Accept `ArrayLike` (a Series qualifies) and coerce.

## scipy with scipy-stubs

scipy-stubs gives result objects real types:

```python
res = stats.ttest_ind(x, y, equal_var=False)   # TtestResult[float64, float64]
res.statistic                                  # float64
res.pvalue                                     # float64
stats.mannwhitneyu(x, y).pvalue                # float64
```

Still wrap the values in `float()` before they go into your results, for the scalar reasons above. Without the stubs, pyright falls back to inferring from scipy's source, and strict mode reports a stream of unknown types.

## statsmodels: build a typed edge

statsmodels ships no type information. In strict mode, pyright reports:

```
error: Stub file not found for "statsmodels.api" (reportMissingTypeStubs)
error: Type of "fit" is partially unknown
error: Type of "add_constant" is partially unknown
```

and `model.params` comes back as `Any`. The fix is containment: **one module touches statsmodels, relaxes the strict rules for itself only, and exposes plain typed functions.**

```python
# pyright: reportMissingTypeStubs=false, reportUnknownMemberType=false, reportUnknownVariableType=false, reportUnknownArgumentType=false
"""ngstats/_statsmodels_edge.py: the only module in ngstats that imports statsmodels."""
import numpy as np
from numpy.typing import NDArray
import statsmodels.api as sm


def ols_slope(x: NDArray[np.float64], y: NDArray[np.float64]) -> tuple[float, float]:
    """Return (slope, p-value) for y ~ 1 + x."""
    model = sm.OLS(y, sm.add_constant(x)).fit()
    return float(model.params[1]), float(model.pvalues[1])
```

The `# pyright:` comment on the first line turns those rules off for this file only. Everywhere else, strict mode applies, and callers see `ols_slope(...) -> tuple[float, float]` with nothing unknown leaking through.

import-linter can keep the boundary from eroding. This contract fails if any other ngstats module imports statsmodels:

```toml
[tool.importlinter]
root_package = "ngstats"
include_external_packages = true

[[tool.importlinter.contracts]]
name = "Only the edge module imports statsmodels"
type = "forbidden"
source_modules = ["ngstats"]
forbidden_modules = ["statsmodels"]
ignore_imports = ["ngstats._statsmodels_edge -> statsmodels"]
```

A test run with a stray `import statsmodels` in another module reported `ngstats.naughty -> statsmodels (l.1)` and failed, as intended.

## `Any` is silent

Strict mode flags *unknown* types, the ones pyright couldn't infer. It does not flag *explicit* `Any` coming from stubs. This passes strict mode with no complaint:

```python
def first(x: NDArray[np.float64]) -> float:
    return x[0]          # x[0] is Any, so nothing is checked
```

Three tools help:

- **`reveal_type(expr)`** is the debugger for types. Drop it in while writing and pyright prints what it actually knows. `from typing import reveal_type` makes it importable at runtime too.
- **`pyright --verifytypes ngstats`** checks your installed package's public API and reports which exported symbols have missing or partially unknown types. It's a completeness score for what you ship with `py.typed`.
- **Ruff's `ANN401`** bans `Any` in your own annotations, so at least you never introduce it on purpose.

## A recipe for ngstats

1. Annotate public inputs as `ArrayLike` and outputs as plain Python types or frozen dataclasses of them.
2. Coerce at the top of every public function with `np.asarray(data, dtype=np.float64)`: `np.float64`, never `float`. Check `ndim`, size and NaNs right there.
3. Annotate internals with `NDArray[np.float64]`. Don't chase precise types for intermediate values.
4. Convert numpy scalars with `float()`, `int()` and `bool()` before they enter result objects.
5. Install `pandas-stubs` and `scipy-stubs` as dev dependencies.
6. Confine statsmodels to one edge module with relaxed rules, and guard it with an import-linter contract.
7. Use `reveal_type` freely while writing, `pyright --verifytypes` before releasing, and `ANN401` to keep `Any` out of your own code.

## Conclusion

Numerical Python will never be as precisely typed as a Java codebase. Array shapes, DataFrame columns and dtypes after arbitrary operations are beyond what today's checkers can follow. That's not a reason to give up on typing the library. It's a reason to put the precision where it pays off: at the public surface, where the checker can hold every function to a clear contract, and at the edges, where untyped code is wrapped so its uncertainty doesn't spread. Inside those walls, let the checker be approximate and let runtime checks handle what types can't say.

## Sources

- [numpy.typing, NumPy documentation](https://numpy.org/doc/stable/reference/typing.html)
- [pandas-stubs on GitHub](https://github.com/pandas-dev/pandas-stubs)
- [scipy-stubs on GitHub](https://github.com/scipy/scipy-stubs)
- [Pyright: comments and file-level configuration](https://microsoft.github.io/pyright/#/comments)
- [Pyright: typed libraries and `--verifytypes`](https://microsoft.github.io/pyright/#/typed-libraries)
- [jaxtyping on GitHub](https://github.com/patrick-kidger/jaxtyping)
- [Import Linter: Forbidden contracts](https://import-linter.readthedocs.io/en/latest/contract_types/forbidden/)
