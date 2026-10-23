# Python Type Checking, From the API Down

How much checking data needs depends on where it came from and who you trust to have produced it. Data from the wire is untrusted and gets parsed at runtime. Data from a programmer in the same process is trusted and gets checked statically, plus a few runtime checks for mistakes types can't express. This guide walks the layers of an ngstats-style project from the outside in.

## The layers at a glance

| Layer | What arrives | Trust | Mechanism | A failure becomes |
| --- | --- | --- | --- | --- |
| 1. API boundary | JSON bodies, query params, env vars | None | Pydantic models | HTTP 422 with every error listed |
| 2. API handlers | Validated pydantic objects | Full | Static types, explicit translation | HTTP error mapped from library exceptions |
| 3. Library public surface | Whatever a programmer passes | Mostly | Static hints + `py.typed`, coercion, value checks | `ValueError` / numpy error |
| 4. Library internals | Coerced arrays | Full | Static types only | Type-checker error at build time |
| 5. Untyped third-party edge | scipy / statsmodels results | Partial | Wrap, normalize, `cast` sparingly | Contained `Any` |

Cross-cutting all five: pyright or mypy in strict mode, run in CI as if it were the compiler.

## Layer 1: The API boundary — parse untrusted input

Pydantic's real subject is **data crossing into your program from outside**, not user-defined types as such. That outside data includes JSON request bodies, config files, environment variables, message queue payloads, and rows from a database or CSV. Pydantic takes that loosely typed input and hands you a typed, trusted Python object, or a structured list of everything wrong with it. The models you define are just how you describe the shape you expect. The point is the boundary.

```python
from typing import Literal
from pydantic import BaseModel, Field, model_validator

class PairedTTestRequest(BaseModel):
    a: list[float] = Field(min_length=2)
    b: list[float] = Field(min_length=2)
    alternative: Literal["two-sided", "less", "greater"] = "two-sided"

    @model_validator(mode="after")
    def same_length(self) -> "PairedTTestRequest":
        if len(self.a) != len(self.b):
            raise ValueError("a and b must have the same length for a paired test")
        return self
```

With FastAPI, declaring this as a handler parameter is enough. Bad input never reaches your code, and the client gets a 422 listing every problem with its location. The same models generate the JSON Schema that becomes your OpenAPI document, so the validation rules and the published spec can't drift apart.

Environment variables and config are also boundary data. Use `pydantic-settings` for them:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="NGSTATS_API_")
    log_level: Literal["debug", "info", "warning"] = "info"
    max_sample_size: int = 1_000_000
```

A container started with `NGSTATS_API_MAX_SAMPLE_SIZE=abc` then fails at startup with a clear message, rather than at 3 a.m. on the first large request.

## Layer 2: API handlers — translate between worlds

Handlers receive validated objects, so they do no validation of their own. Their job is translation in both directions:

- **Inbound:** pydantic request model → plain library call.
- **Outbound:** library result → pydantic response model. This is where serialization policy lives: converting numpy scalars to `float`, deciding what NaN or inf becomes in JSON (JSON has no NaN), flattening tuples, choosing which fields to expose.
- **Errors:** library `ValueError` → an HTTP error with a useful message.

```python
@app.post("/v1/ttest/paired", response_model=TestResultResponse)
def paired_ttest(req: PairedTTestRequest) -> TestResultResponse:
    try:
        result = ngstats.ttest_rel(req.a, req.b, alternative=req.alternative)
    except ValueError as e:
        raise HTTPException(status_code=422, detail=str(e))
    return TestResultResponse.from_result(result)
```

Handlers are ordinary typed Python, checked statically like everything else.

**On duplicated checks:** the length check above also exists in the library. That's deliberate. The library's check is authoritative because the library is used outside the API too. The API repeats only the rules worth publishing in the OpenAPI schema and reporting in pydantic's structured format. Anything else the API leaves to the library and translates the resulting `ValueError`.

## Layer 3: The library's public surface — trusted callers, static contract

Here the caller is a programmer in the same process. Type checking is mostly static:

- **Annotate the public API fully and ship a `py.typed` marker.** That empty file in the package tells mypy, pyright, and IDEs that your library's hints are meant to be trusted. Their tooling then flags `ttest("abc", b)` while they type, at zero runtime cost. Your job is to provide good hints, and their job is to run a checker.
- **Accept broadly, return precisely.** Annotate inputs as `numpy.typing.ArrayLike` and outputs as concrete types such as a frozen dataclass of floats.
- **Coerce rather than police types.** Call `np.asarray(a, dtype=float)` at the top, and lists, tuples, Series, and arrays all just work. Garbage fails right there with numpy's own error. This is the duck-typing convention numpy and scipy follow, and `isinstance` checks are generally avoided.
- **Hand-code runtime checks only for what types can't express,** especially mistakes that would otherwise produce a wrong answer silently rather than crash:
    - `alpha` not between 0 and 1
    - fewer than 2 observations
    - 2-D input where 1-D is expected
    - NaNs present (raise, drop, or follow a `nan_policy` argument, as scipy does)
    - unequal lengths for paired tests

    Raise `ValueError` with a clear message.

```python
from typing import Literal
import numpy as np
from numpy.typing import ArrayLike, NDArray

Alternative = Literal["two-sided", "less", "greater"]

def ttest_rel(a: ArrayLike, b: ArrayLike, *, alternative: Alternative = "two-sided") -> TestResult:
    x = _as_1d_float(a, "a")
    y = _as_1d_float(b, "b")
    if x.size != y.size:
        raise ValueError(f"paired test needs equal lengths, got {x.size} and {y.size}")
    return _ttest_rel_impl(x, y, alternative)

def _as_1d_float(data: ArrayLike, name: str) -> NDArray[np.float64]:
    arr = np.asarray(data, dtype=np.float64)
    if arr.ndim != 1:
        raise ValueError(f"{name} must be 1-D, got shape {arr.shape}")
    if arr.size < 2:
        raise ValueError(f"{name} needs at least 2 observations, got {arr.size}")
    if np.isnan(arr).any():
        raise ValueError(f"{name} contains NaN")
    return arr
```

**Not pydantic at this layer.** The in-process caller is trusted, the data is arrays (which pydantic handles poorly), validation in hot loops such as bootstraps costs real time, and every library dependency becomes a version constraint on everyone who installs you. Heavy dependencies belong to applications, not libraries.

Pydantic does have `@validate_call`, which validates function arguments at runtime. It's a side feature, and the problem it addresses is normally handled by type hints plus a checker before the code ever runs.

## Layer 4: Library internals — fully static

Once `_as_1d_float` has returned, every internal function receives `NDArray[np.float64]` and can rely on it. No further runtime checking. Use the type toolbox to make signatures self-documenting:

- `Literal[...]` for string options instead of bare `str`
- Frozen dataclasses for result and value objects
- `Final` for constants
- `@overload` where the return type depends on the arguments
- `NewType` when two floats mean different things
- `Protocol` when a function needs "anything with these methods": structural typing, which is duck typing written down and checked

Use `assert` only for internal invariants that indicate a bug in your own code, never for validating caller input, since asserts can be disabled with `python -O`.

## Layer 5: The untyped edge — scipy and statsmodels

scipy and statsmodels are only partly typed, so `Any` leaks in from them. Contain it right where it enters:

```python
from scipy import stats

def _ttest_rel_impl(x: NDArray[np.float64], y: NDArray[np.float64], alternative: Alternative) -> TestResult:
    res = stats.ttest_rel(x, y, alternative=alternative)
    statistic = float(res.statistic)   # normalize numpy scalars / Any to plain float
    p_value = float(res.pvalue)
    ...
```

Converting to plain Python types immediately both normalizes the values and turns `Any` back into something the checker understands. Where you truly can't express a type, use `cast()` or `# type: ignore[code]`, keep them few, keep them at this edge, and comment each one. ngstats as a whole is a typed facade over loosely typed libraries, so this layer is the reason it exists.

## Tooling across all layers

- **Pyright or mypy in strict mode** in pre-commit and CI. A type error fails the build, like a compiler with `-Wall -Werror`.
- **Ruff's `ANN` rules** flag missing annotations.
- **`beartype` in the test suite (optional).** A decorator that checks hints at runtime cheaply. Enabling it only under test surfaces violations without production cost.
- **Tests** cover what the checker can't: value rules, the untyped edge, and the API's error translation.

## Where each kind of mistake gets caught

| Mistake | Caught at | By |
| --- | --- | --- |
| Client sends `"a": "oops"` | Layer 1 | Pydantic → 422 |
| Container misconfigured env var | Layer 1 | pydantic-settings, at startup |
| Client sends a sample containing NaN | Layer 3 | Library `ValueError` → handler → 422 |
| Programmer calls `ttest_rel("abc", b)` | Before running | Their IDE / type checker, via `py.typed` |
| Programmer passes a 2-D array | Layer 3 | `_as_1d_float` → `ValueError` |
| You return a numpy scalar where `float` is declared | Build time | Your type checker, plus the Layer 5 normalization |
| scipy changes a return attribute | Test time | Your tests at Layer 5 |
