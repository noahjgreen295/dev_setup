# Layered Python Architecture with import-linter

C, C++, and Java give a program structure through the language itself. Header files declare the interface, `private` marks what is not part of it, and package rules say who may depend on whom. Python has none of that built in. Any module can import any other, and nothing fails when a convention is broken. The structure still matters, but it lives in code review and memory unless a tool enforces it.

This essay covers the layered architecture an ngstats-style project should have, and how import-linter turns that architecture from a convention into a contract that CI checks on every commit.

## 1. What a layer is

A layered architecture arranges modules in tiers. The rule is simple: a module may import from layers below it, never from layers above.

```
     ┌──────────────────────────────┐
     │         higher layer         │   may import ↓
     ├──────────────────────────────┤
     │         middle layer         │   may import ↓
     ├──────────────────────────────┤
     │         lower layer          │   knows nothing above it
     └──────────────────────────────┘
```

The payoff:

- **Lower layers are reusable and testable on their own.** They don't drag the upper layers along.
- **Changes flow in one direction.** Editing a high layer can't break a low one.
- **The code has a reading order.** Start at the bottom and each layer is built only from what you've already read.
- **No import cycles.** In Python a cycle is a real bug, not just a style problem. It shows up as `ImportError: cannot import name ... (most likely due to a circular import)`, often only when modules are imported in a particular order.

## 2. Two scales of layering in ngstats

The project has layers at two scales.

**System scale.** Each tier talks only to the one beneath it.

```
 ┌────────────────────────────────────────────────┐
 │  frontend        separate container            │
 └───────────────────────┬────────────────────────┘
                         │  HTTP / JSON
 ┌───────────────────────▼────────────────────────┐
 │  ngstats_api     FastAPI app, pydantic models  │
 └───────────────────────┬────────────────────────┘
                         │  Python calls
 ┌───────────────────────▼────────────────────────┐
 │  ngstats         the library                   │
 └───────────────────────┬────────────────────────┘
                         │  Python calls
 ┌───────────────────────▼────────────────────────┐
 │  numpy · scipy · statsmodels                   │
 └────────────────────────────────────────────────┘
```

The frontend never imports Python at all, since it only speaks HTTP. The rule that matters in code is the middle one: `ngstats_api` imports `ngstats`, and `ngstats` must never import `ngstats_api`, `fastapi`, or `pydantic`. The library has to stay usable by someone who has never heard of the API.

**Library scale.** Inside `ngstats`, the subpackages form their own stack.

```
 ngstats
 ┌──────────────────────────────────────────────────┐
 │  inference     _means | _proportions | ...       │  hypothesis tests
 ├──────────────────────────────────────────────────┤
 │  effectsize                                      │  Cohen's d, etc.
 ├──────────────────────────────────────────────────┤
 │  results       HypothesisTestResult, ...         │  frozen dataclasses
 ├──────────────────────────────────────────────────┤
 │  _validation   coercion and value checks         │  np.asarray, NaN checks
 └──────────────────────────────────────────────────┘
```

A t-test in `inference` may compute an effect size from `effectsize`, package it as a `results` object, and validate its inputs with `_validation`. None of the reverse directions make sense. A result object that imports a hypothesis test is a design mistake, and ideally the build should reject it.

These layers line up with the trust layers in `python_type_checking.md`. Pydantic parses untrusted input at the top of `ngstats_api`. The library's public functions coerce and value-check through `_validation`. Everything below that is checked statically.

## 3. Why Python needs a tool for this

Python offers conventions for structure:

- **Underscore modules and names.** `_means.py` is private by convention.
- **`__all__` in each `__init__.py`.** This is the explicit export list, the closest thing Python has to a header file.

Neither convention is enforced. A developer in a hurry can write `from ngstats.inference._means import _pooled_variance` inside `ngstats.results`, and everything runs. The architecture erodes one convenient import at a time, and nobody notices until a cycle appears or the library can't be installed without FastAPI.

The fix is to state the architecture in configuration and have the build check it.

## 4. What import-linter is

import-linter is a separate tool from ruff, maintained by David Seddon (`seddonym/import-linter` on GitHub) and not by Astral, the company behind ruff. Ruff mostly checks one file at a time. import-linter builds the import graph for your whole package and checks it against architecture rules you declare, which it calls **contracts**. If any import breaks a contract, CI fails.

Setup: add it with `uv add --dev import-linter` and run it with `lint-imports`, next to ruff in pre-commit or CI. You declare contracts in a `.importlinter` file, or under `[tool.importlinter]` in `pyproject.toml` so all your config stays in one file.

## 5. The contract types

| Contract | Rule | ngstats example |
| --- | --- | --- |
| **Forbidden** | One set of modules may never import another | `ngstats` may not import `fastapi`, `pydantic`, or `ngstats_api` |
| **Protected** | A module can only be imported by modules on an allow-list | Only `ngstats.inference` may import `ngstats.inference._means` |
| **Layers** | Higher layers may import lower ones, never the reverse | `inference` above `effectsize` above `results` |
| **Independence** | Sibling modules may not depend on each other | `_means`, `_proportions`, `_nonparametric` stay separate |
| **Acyclic siblings** | No import cycles between sibling modules | Siblings may depend on each other, but only in one direction |
| **Custom** | You write your own contract type | Rarely needed |

### 5.1 Forbidden: keep the library free of the API

```
   ngstats ──✗──▶ fastapi
   ngstats ──✗──▶ pydantic
   ngstats ──✗──▶ ngstats_api
   ngstats ──✓──▶ numpy, scipy, statsmodels
```

```toml
[[tool.importlinter.contracts]]
name = "Library stays free of web dependencies"
type = "forbidden"
source_modules = ["ngstats"]
forbidden_modules = ["fastapi", "pydantic", "ngstats_api"]
```

Two details from the docs. First, forbidding an external package requires `include_external_packages = true` in the top-level `[tool.importlinter]` table. Without it, external packages aren't in the graph. Second, only root-level external names can be forbidden: `pydantic` works, `pydantic.fields` does not.

### 5.2 Layers: the library's internal stack

```toml
[[tool.importlinter.contracts]]
name = "ngstats internal layering"
type = "layers"
containers = ["ngstats"]
layers = [
    "inference",
    "effectsize",
    "results",
    "_validation",
]
exhaustive = true
```

Layers are listed **highest first**. With `containers` set, names are relative to the container, so `"inference"` means `ngstats.inference`. `exhaustive = true` makes the contract fail if someone adds a new subpackage to `ngstats` without placing it in a layer. That forces a decision about where new code belongs.

A layer entry can also hold siblings. `"_means | _proportions"` puts two modules in the same layer and forbids them from importing each other. `"_means : _proportions"` puts them in the same layer but lets them import each other. Don't mix `|` and `:` on one line.

### 5.3 Protected: `private` at the module level

```
   ngstats.inference (__init__.py) ──✓──▶ ngstats.inference._means
   ngstats.effectsize              ──✗──▶ ngstats.inference._means
   ngstats.effectsize              ──✓──▶ ngstats.inference        (public surface)
```

```toml
[[tool.importlinter.contracts]]
name = "Inference internals are private"
type = "protected"
protected_modules = [
    "ngstats.inference._means",
    "ngstats.inference._proportions",
]
allowed_importers = ["ngstats.inference"]
```

This is the closest Python gets to Java's `private`. The only door into `_means` is `ngstats.inference/__init__.py`, which re-exports the public names and lists them in `__all__`. Modules under an allowed importer count as allowed too, so code inside `ngstats.inference` can still use its own internals.

### 5.4 Independence: siblings that stay apart

```
     _means      _proportions      _nonparametric
        │   ✗ ◀──────┼──────▶ ✗        │
        │            │                 │
        ▼            ▼                 ▼
     ┌──────────────────────────────────────┐
     │  results · _validation  (shared)     │
     └──────────────────────────────────────┘
```

```toml
[[tool.importlinter.contracts]]
name = "Test families are independent"
type = "independence"
modules = [
    "ngstats.inference._means",
    "ngstats.inference._proportions",
    "ngstats.inference._nonparametric",
]
```

When two test families need the same helper, the helper moves down into a shared layer rather than one family importing the other. Independent siblings can be understood, tested, and deleted one at a time.

### 5.5 Acyclic siblings and custom contracts

**Acyclic siblings** is the relaxed form of independence. Siblings may depend on each other, but never in a cycle. Use it when strict independence is too strong.

**Custom** contracts are Python classes you write for rules the built-in types can't express. A project this size shouldn't need one.

## 6. A complete configuration

The library's `pyproject.toml`:

```toml
[tool.importlinter]
root_package = "ngstats"
include_external_packages = true
exclude_type_checking_imports = true

[[tool.importlinter.contracts]]
name = "Library stays free of web dependencies"
type = "forbidden"
source_modules = ["ngstats"]
forbidden_modules = ["fastapi", "pydantic", "ngstats_api"]

[[tool.importlinter.contracts]]
name = "ngstats internal layering"
type = "layers"
containers = ["ngstats"]
layers = [
    "inference",
    "effectsize",
    "results",
    "_validation",
]
exhaustive = true

[[tool.importlinter.contracts]]
name = "Inference internals are private"
type = "protected"
protected_modules = [
    "ngstats.inference._means",
    "ngstats.inference._proportions",
]
allowed_importers = ["ngstats.inference"]

[[tool.importlinter.contracts]]
name = "Test families are independent"
type = "independence"
modules = [
    "ngstats.inference._means",
    "ngstats.inference._proportions",
    "ngstats.inference._nonparametric",
]
```

`exclude_type_checking_imports = true` ignores imports under `if TYPE_CHECKING:`. Those exist only for annotations and never run, so they shouldn't count against the architecture.

**The system scale, in a uv workspace.** If the library and API live in one repo, the workspace root can declare both packages and state the system layering directly:

```toml
[tool.importlinter]
root_packages = ["ngstats", "ngstats_api"]

[[tool.importlinter.contracts]]
name = "API sits above the library"
type = "layers"
layers = ["ngstats_api", "ngstats"]
```

## 7. Running it

```bash
uv add --dev import-linter
uv run lint-imports
```

`lint-imports` reads the config, builds the graph, and reports each contract as kept or broken. For a broken contract it shows the import chain responsible, and it exits nonzero so CI fails.

As a pre-commit hook, run it through uv so it uses the project's environment:

```yaml
- repo: local
  hooks:
    - id: import-linter
      name: import-linter
      entry: uv run lint-imports
      language: system
      pass_filenames: false
      types: [python]
```

`pass_filenames: false` matters. import-linter checks the whole graph, not individual files.

## 8. Escape hatches

Real code sometimes needs an exception. import-linter offers a few, and they deserve the same discipline as `# type: ignore`: few, narrow, and each with a comment saying why.

- **`ignore_imports`** on any contract exempts specific imports, written as `"importer -> imported"`.
- **`allow_indirect_imports = true`** on a forbidden contract only flags direct imports, not chains through other modules.
- **`exclude_type_checking_imports`** at the top level, covered above.

If the exceptions list keeps growing, the declared architecture is wrong. Fix the layers rather than piling up exemptions.

## 9. Where import-linter fits among the other tools

| Mechanism | Scope | What it declares or enforces |
| --- | --- | --- |
| Underscore modules and names | Convention | "Not part of the contract" |
| `__all__` | Each package | The public export list |
| ruff `PLC2701` (preview) | Each file | Importing another module's private names |
| ruff `TID251` (banned-api) | Each file | Banning modules or APIs by name |
| pyright / mypy | Whole program | Types, private attribute access, what a module re-exports |
| import-linter | Whole import graph | Layers, allow-lists, independence, forbidden dependencies |

**How it relates to ruff:** ruff's flake8-tidy-imports rules can ban modules by name, which covers simple cases like "never import pydantic here." Ruff can't express layers or allow-lists, though. Those need import-linter.

## 10. The Java developer's translation table

| Java / C++ | Python equivalent |
| --- | --- |
| Header file / public interface | `__init__.py` re-exports + `__all__` |
| `private` class or package-private module | Underscore module + protected contract |
| Package dependency rules (or C++ include discipline) | Layers and forbidden contracts |
| Compiler rejects the violation | `lint-imports` fails CI |

None of these stops a determined developer, but neither did `private` in Java, since reflection gets around it. What they provide is the same thing the keywords did: a declared structure, visible in one place, checked on every build.

## Sources

- [Import Linter documentation](https://import-linter.readthedocs.io/en/stable/)
- [Import Linter: Configure](https://import-linter.readthedocs.io/en/latest/get_started/configure/)
- [Import Linter: Forbidden contracts](https://import-linter.readthedocs.io/en/latest/contract_types/forbidden/)
- [Import Linter: Layers contracts](https://import-linter.readthedocs.io/en/latest/contract_types/layers/)
- [Import Linter: Protected contracts](https://import-linter.readthedocs.io/en/latest/contract_types/protected/)
- [Ruff rule PLC2701: import-private-name](https://docs.astral.sh/ruff/rules/import-private-name/)
