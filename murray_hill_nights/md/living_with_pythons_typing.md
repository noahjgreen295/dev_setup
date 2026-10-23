# Living With Python's Typing

If you learned to program in C, C++ or Java, Python's attitude toward types can feel like a betrayal. You spent years building habits that kept whole categories of bugs out of your programs: declare everything, let the compiler check it, trust nothing that hasn't been checked. Then you arrive in a language where a function will cheerfully accept a string where it asked for an integer, and the community calls that a feature. The discomfort is reasonable. It comes from instincts that served you well for a long time, and you don't have to throw them away. You have to learn which of them still apply, which have a Python equivalent, and which were answers to problems Python solves another way.

## Shedding the Java frame of mind

### What you're losing, and what you're not

It helps to separate three things a statically typed language gave you, because they don't all go away:

- **Documentation.** A signature like `double mean(List<Double> xs)` tells the reader what goes in and what comes out. Python keeps this completely. Type hints say the same thing, and IDEs show them on hover.
- **Early error detection.** The compiler caught type mismatches before the program ran. Python keeps most of this too, but you have to opt in by running a type checker as part of the build.
- **A guarantee.** If it compiled, the types matched everywhere, including in code written by other people. This is the part that really goes. Python's checking is local to code that's annotated and checked, and the language itself guarantees nothing.

Most of the unease comes from that third item, and it's worth naming plainly so it doesn't color everything else. You still get documentation, and you still get early errors. What you give up is the assurance that the whole world is typed. In its place you get boundaries you control, and the rest of this essay is about drawing them.

### Two philosophies

The difference isn't that one language cares about correctness and the other doesn't. They locate correctness in different places.

**The Java and C++ philosophy: the compiler is the gatekeeper.** Types are a contract enforced before anything runs. A class *is* something because it declares what it is: it extends this, implements that. Design happens up front, in the type hierarchy, and the goal is to make incorrect programs impossible to write. Flexibility is something you ask for explicitly, through generics, interfaces and templates, and pay for in ceremony. This is **nominal** typing: identity comes from declared names.

**The Python philosophy: behavior over identity, and consenting adults.** An object is acceptable if it can do what's asked of it. Whether it inherits from the right class doesn't matter. That's duck typing: if it walks like a duck and quacks like a duck, treat it as a duck. The language trusts the programmer ("we're all consenting adults here," in the community's own phrase): it marks private things with an underscore rather than locking them, and it lets you replace almost anything at runtime. Design happens incrementally, and correctness comes from tests, readable code and, increasingly, optional static checking layered on top. This is **structural** typing in spirit: identity comes from shape.

Two well-known Python sayings capture the difference in practice:

- **EAFP: "Easier to ask forgiveness than permission."** Try the operation and handle the exception if it fails, rather than checking everything first. Java code tends toward the opposite style, LBYL ("look before you leap"): check `instanceof`, check `null`, then act.
- **"Explicit is better than implicit"** and **"Readability counts"**, from *The Zen of Python* (`import this`). Python values code a person can read at a glance, and treats boilerplate as a cost, not a safety feature.

| | Java / C++ | Python |
| --- | --- | --- |
| Where correctness lives | The compiler | Tests, readable code, an optional checker |
| What makes an object acceptable | Its declared type | What it can do |
| Privacy | Enforced (`private`) | Conventional (`_name`) |
| Error style | Look before you leap | Ask forgiveness |
| Flexibility | Requested explicitly, with ceremony | The default |
| Design | Up front, in the hierarchy | Incremental, in small functions |

Modern typed Python is a synthesis of the two. Type hints, protocols and strict checkers bring back most of the compiler's early warnings while keeping the behavioral view of objects. You're not being asked to give up on rigor. You're being asked to apply it in different places.

### Mind exercises

Instincts change through repetition, not argument. These are small, concrete exercises for retraining them. Each takes minutes.

1. **Ask "what does it need to do?" instead of "what is it?"** Before writing a parameter's type, list the operations the function performs on it. If it only iterates, the type is `Iterable[float]`, not `list[float]`. If it only calls `.mean()`, it's a protocol with one method. Do this for every function you write for a week.
2. **Delete an `isinstance` check.** Find one in your code (or write the one your Java instinct wants) and replace it with coercion (`np.asarray`) or with simply attempting the operation. Notice what fails, and how clearly. Usually the failure is clear enough without the guard.
3. **Translate one Java interface into a `Protocol`.** Take an interface you've written many times, say a `Comparable` or a `DataSource`, and write it as a Python protocol. Then write a class that satisfies it without mentioning it. The moment the checker accepts that class is the moment structural typing stops feeling loose.
4. **Rewrite one LBYL block as EAFP.** Turn `if key in d: x = d[key] else: ...` into `try: x = d[key] except KeyError: ...`, or better, `d.get(key, default)`. Compare which version reads more clearly.
5. **Write first, annotate second.** Write a small function with no hints, get it working, then add annotations and run the checker. This reverses the Java order, and it shows that types in Python are a description of working code, not a precondition for writing it.
6. **Ask the checker what it believes.** Spend an afternoon sprinkling `reveal_type()` through real code. Seeing what's inferred, and where it gives up and says `Unknown`, builds an accurate sense of what the tool knows, which is the basis for trusting it the right amount.
7. **Read some well-typed Python.** Browse the source of a modern typed library, or the stubs in typeshed for a standard-library module you use. Seeing idiomatic annotations written by people who think in Python is faster than deriving the style yourself.
8. **Name the instinct when it fires.** When you feel the urge to guard, declare or lock something down, pause and ask which of the three things above you're protecting: documentation, early errors, or a guarantee. If it's one of the first two, Python has a tool for it. If it's the third, decide where the boundary is and check there.

None of these asks you to stop caring about types. They ask you to locate the care differently. The remaining sections are the practical toolkit for doing that.

## Technical

Modern Python can give a C, C++, or Java developer most of what they miss about strong typing, as long as type checking is treated as part of the build rather than a suggestion.

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
