# Living With Python's Typing

Modern Python can give a C, C++, or Java developer most of what they miss about strong typing, as long as type checking is treated as part of the build rather than a suggestion.

## Technical

**Run the checker as if it were the compiler.** Use pyright or mypy in strict mode, wired into pre-commit and CI, and treat a type error as a failed build. Python then behaves like a compiled language with `-Wall -Werror`, except the compile step is separate from running. Ruff's `ANN` rules flag missing annotations so nothing slips through untyped.

**Use `Protocol` for duck typing with a contract.** A `Protocol` is an interface that a class satisfies by having the right methods, with no inheritance declared. It is structural typing: duck typing written down and checked statically. You get a self-documenting interface without forcing callers into your class hierarchy.

**Accept broadly, return precisely.** In ngstats, annotate inputs as `numpy.typing.ArrayLike`, which documents "anything array-ish." Annotate internals and outputs as `NDArray[np.float64]` or concrete types. The signature still tells the reader exactly what is acceptable.

**Use the rest of the type toolbox:**

- `Literal["two-sided", "less", "greater"]` instead of bare `str`
- Frozen dataclasses for value objects
- `Final` for constants
- `@overload` where the return type depends on the arguments
- `NewType` when two floats mean different things

**Optional runtime enforcement in tests.** `beartype` is a decorator that checks hints at runtime cheaply. Some people enable it only in the test suite, so violations surface in tests without costing anything in production.

**Wall off untyped code.** scipy and statsmodels are only partly typed, so `Any` will leak in from them. Wrap them at the edge and expose fully typed signatures. That is literally what ngstats is: a typed facade over loosely typed libraries.

## Psychological

**You have done duck typing before.** C++ templates are compile-time duck typing: a template works with any type that has the operations it uses, and nothing declares that up front. C++20 concepts were added to write those requirements down, which is exactly what `Protocol` does for Python. Python's version is structural typing checked by a separate tool instead of the compiler. That is a different arrangement from what you know, not an absence of typing.

**The types have not gone away.** Enforcing them became optional, and you choose to enforce them. In your own code, Python can be nearly as strict as Java. What you give up is the guarantee that other people's code is typed, and that is a boundary problem you handle by wrapping.

**Expect the checker to be unsound sometimes, and don't take it personally.** There will be `# type: ignore` lines and `cast()` calls at the edges where untyped libraries come in. Keep them few, keep them at the boundary, and add a comment explaining each one. Tests cover the remaining gap, which is the Python community's real answer to "how do you trust this."

**The convention bending you most is not checking types at runtime.** The C/Java instinct says guard every input. The Python instinct is to coerce what you can (`np.asarray`), validate what affects correctness, and let static checking handle the rest. Once that split is clear, most of the discomfort goes away.
