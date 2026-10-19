# The Python Toolchain Stack

A layered map of the modern Python toolchain, read like the OSI model: the editor at the top (L9) sits on the layers below it, all the way down to the language itself (L0). Each layer names the current default pick, then the other live options. State as of October 2026.

For the styled, interactive version, open [python_toolchain_stack.html](python_toolchain_stack.html) in a browser.

**Status key:** **PICK** = modern default · alt = solid alternative · ⚠️ edge = maintained, but teetering or superseded · ◆×N = tool appears in N layers

## Default path, top to bottom

VS Code → Pylance → Ruff → Pyright → pytest → uv init + pyproject.toml → uv → uv python → CPython 3.15

---

## L9 · Editor / IDE

Where you type. Mostly a host for the layers below: the IDE's smarts come from the language server it runs.

**Pick:** VS Code (Positron for notebook-heavy data science)

| Tool | Status | Notes |
|---|---|---|
| VS Code | **PICK** | Free, huge extension ecosystem, first-class Pylance and Jupyter support. |
| PyCharm | alt | Best-in-class refactoring and debugger. Brings its own analysis engine instead of an LSP. One unified product with a free tier. |
| Positron | alt | Posit's data-science IDE built on the VS Code core. Variables pane, plots, R and Python side by side. |
| Cursor | alt | VS Code fork with AI built in. Can't use Pylance (license is VS Code only), so pair it with basedpyright or ty. |
| Zed | alt | Native, very fast editor. Speaks LSP, so ty, basedpyright or Pyrefly plug in directly. |
| JupyterLab | alt | Notebook-first. Still the lingua franca for exploratory analysis and sharing results. |

---

## L8 · Language server

Completions, go-to-definition, hover types, rename. Most of these are type checkers wearing an LSP hat, which is why they reappear one layer down.

**Pick:** Pylance in VS Code; basedpyright or ty elsewhere

| Tool | Status | Notes |
|---|---|---|
| Pylance | **PICK** | Microsoft's closed-source LSP for VS Code, powered by the Pyright engine. |
| basedpyright ◆×2 | alt | Open fork of Pyright that restores Pylance-only features. The go-to for Cursor, Neovim and other non-Microsoft editors. |
| Pyrefly (LSP) ◆×2 | alt | Meta's Rust checker, 1.0 since May 2026. Ships its own VS Code extension and language server. |
| ty server ◆×2 | ⚠️ edge | Astral's Rust checker as an LSP. Incremental updates are extremely fast, but still beta. |
| Ruff server ◆×2 | alt | Ruff has a built-in LSP for lint diagnostics and fixes. Runs alongside the type-aware server, not instead of it. |

---

## L7 · Lint & format

Mechanical correctness and one consistent code style, enforced in the editor, pre-commit and CI.

**Pick:** Ruff (lint + format, one binary)

| Tool | Status | Notes |
|---|---|---|
| Ruff ◆×2 | **PICK** | Rust linter and formatter. Replaces flake8, isort, pyupgrade, most plugins, and Black in one tool. |
| Black | ⚠️ edge | The formatter that ended style debates. Still maintained, but Ruff format matches it nearly line for line. |
| Pylint | alt | Slower but deeper checks (design smells, some cross-module issues). Still maintained. |
| flake8 | ⚠️ edge | Maintained, but Ruff reimplements its rules much faster. Mostly seen in older configs. |
| “The Rest” | ⚠️ edge | pyflakes, pycodestyle, isort, pyupgrade, autoflake, Bandit, Prospector. All still maintained, but Ruff reimplements nearly all of their rules, so there's little reason to add them to a new project. |

---

## L6 · Type checking

Type hints checked two ways: statically before the code runs, and at runtime at the edges where outside data comes in. The static side is the fastest-moving part of the stack.

**Pick:** Pyright statically (Pyrefly if speed on a big repo matters); Pydantic at runtime

### Static (before it runs)

| Tool | Status | Notes |
|---|---|---|
| Pyright | **PICK** | Strict, fast enough, and identical to the diagnostics Pylance shows in the editor, so CI and IDE never disagree. |
| Pyrefly ◆×2 | alt | Meta's Rust checker. Stable 1.0 (May 2026), much faster than mypy and Pyright, strong inference, migration tooling. |
| mypy | alt | The reference implementation. Slowest, but still the most common in existing codebases and has plugins (Django, Pydantic). |
| ty ◆×2 | ⚠️ edge | Astral's checker. Fastest of all and great diagnostics, but 0.0.x beta and narrower rule coverage. |
| basedpyright ◆×2 | alt | Same engine as Pyright with stricter defaults and extra rules. Fine CLI choice too. |

### Runtime (while it runs)

| Tool | Status | Notes |
|---|---|---|
| Pydantic | **PICK** | Validates and parses data against your type hints at runtime: API payloads, configs, LLM output. Rust core, powers FastAPI. Static checkers understand its models. |
| msgspec | alt | Much faster serialization and validation for JSON and MessagePack. Fewer features than Pydantic; great on hot paths. |
| attrs / dataclasses | alt | Typed data classes without validation. dataclasses is stdlib; attrs adds validators and slots. |
| beartype | alt | Decorator that checks function arguments and returns against their hints at call time, with near-zero overhead. |

---

## L5 · Testing

Runtime verification. One clear winner here, plus a few companions.

**Pick:** pytest

| Tool | Status | Notes |
|---|---|---|
| pytest | **PICK** | Plain asserts, fixtures, parametrize, and a plugin for everything. The de facto standard. |
| coverage.py / pytest-cov | alt | Line and branch coverage. Pairs with pytest. |
| Hypothesis | alt | Property-based testing. Especially good for numerical and quant code with edge cases you won't think of. |
| nox / tox | alt | Run the suite across Python versions and dependency sets. nox is configured in Python; tox in config files. |

---

## L4 · Scaffold, build & publish

How a project starts, how it declares metadata, and how it becomes a wheel. pyproject.toml is the hub of all of it.

**Pick:** uv init + pyproject.toml, hatchling or uv_build backend

### Scaffold

| Tool | Status | Notes |
|---|---|---|
| uv init ◆×3 | **PICK** | Creates a minimal pyproject.toml project in one command. Enough for most work. |
| Copier | alt | Template engine that can push template updates into projects already generated from it. |
| cookiecutter | ⚠️ edge | The original project-template tool. Maintained but slow-moving, and one-shot only. |

### Declare & build

| Tool | Status | Notes |
|---|---|---|
| pyproject.toml | **PICK** | The standard (PEP 517/518/621). Metadata, dependencies and tool config all live here. |
| hatchling / uv_build | **PICK** | Modern build backends. uv_build is Astral's pure-Python-package backend; hatchling is the PyPA-adjacent default. |
| setuptools | alt | Still the most-used backend and fine when configured via pyproject.toml. Needed for compiled extensions. |
| setup.py | ⚠️ edge | Legacy entry point. Direct invocation is deprecated; keep only if you need imperative build logic. |
| poetry-core ◆×2 | alt | Poetry's backend. Comes along if you use Poetry. |

### Publish

| Tool | Status | Notes |
|---|---|---|
| uv publish ◆×3 | **PICK** | Uploads to PyPI with trusted publishing support. |
| twine | alt | The classic uploader. Still fine. |

---

## L3 · Dependencies & environments

Resolve, lock and install packages into an isolated environment. The layer most transformed in the last few years.

**Pick:** uv

### Project managers

| Tool | Status | Notes |
|---|---|---|
| uv ◆×3 | **PICK** | Resolver, lockfile, venv manager, tool runner. 10 to 100x faster than pip. Also appears in L4 and L2. |
| Poetry ◆×2 | alt | Mature, 2.x now uses standard PEP 621 metadata. Slower than uv but stable and well understood. |
| pixi ◆×2 | alt | Modern conda-ecosystem manager with lockfiles. The pick when you need non-Python binaries (CUDA, GDAL, compilers). |
| conda ◆×2 | alt | Still the standard in many data-science and HPC shops. Handles native libraries pip can't. |
| PDM / Hatch | alt | Standards-first project managers. Fine, but uv has absorbed most of their audience. |

### Installers & isolation

| Tool | Status | Notes |
|---|---|---|
| pip | alt | The baseline installer bundled with Python. Fine for one-offs and Docker layers; no lockfile or project management. |
| venv | alt | Stdlib virtual environments. What uv creates under the hood. |
| virtualenv | ⚠️ edge | Faster, older sibling of venv. Maintained, mostly used indirectly by tox, Hatch and pipx. |

---

## L2 · Python version manager

Which interpreters are installed and which one a project uses.

**Pick:** uv python

| Tool | Status | Notes |
|---|---|---|
| uv python ◆×3 | **PICK** | Downloads standalone CPython (and PyPy) builds on demand; pins per project via .python-version. |
| pyenv | ⚠️ edge | Compiles Python from source behind shell shims. Maintained, but slower and fussier than uv. |
| conda ◆×2 | alt | Python is just another package in a conda env, alongside its native dependencies. |
| pixi ◆×2 | alt | Same idea as conda, with project lockfiles. |
| mise | alt | Polyglot version manager (Python, Node, Go...). Good if you juggle several languages. |
| py install (Windows) | alt | The new Python install manager on Windows; python.org now documents installing 3.15 this way. |

---

## L1 · Interpreter / runtime

The program that actually executes your code.

**Pick:** CPython 3.15

| Tool | Status | Notes |
|---|---|---|
| CPython 3.15 | **PICK** | The reference implementation. 3.15 (Oct 9 2026) adds explicit lazy imports, frozendict, sentinel, UTF-8 by default and a faster JIT. |
| CPython free-threaded | alt | The no-GIL build (3.14t / 3.15t). Real parallel threads; check that your C extensions support it. |
| PyPy | ⚠️ edge | JIT-compiled Python, often several times faster on pure-Python loops. Trails CPython by a few versions; C-extension friction. |
| GraalPy | alt | Oracle's JVM-based Python. Niche: Java interop and polyglot apps. |
| MicroPython | alt | Python for microcontrollers. Different world, same syntax. |

---

## L0 · The language

Not a tool but the contract every layer above implements: the language reference, the typing spec, and the packaging PEPs.

**Pick:** Python 3.15 language + typing spec

| Tool | Status | Notes |
|---|---|---|
| Language reference | **PICK** | Syntax and semantics, evolved through PEPs. 3.15 headliner: explicit lazy imports (PEP 810). |
| Typing spec | **PICK** | The shared rulebook that mypy, Pyright, Pyrefly and ty are all graded against for conformance. |
| Packaging PEPs | **PICK** | 517/518 (build), 621 (metadata), 751 (pylock.toml). Why tools in L2–L4 are interchangeable. |

---

## On the edge

- **setup.py**: setuptools still reads it, but running `python setup.py install/upload` is deprecated. Metadata belongs in pyproject.toml now; keep setup.py only for custom build logic.
- **Black**: still released and widely used, but Ruff's formatter is a near drop-in and runs in the same binary as the linter. New projects rarely start with Black.
- **pyenv**: maintained, but `uv python install` does the same job faster with no shell shims.
- **virtualenv**: maintained by PyPA and still used inside tox, Hatch and pipx. You rarely call it directly anymore; stdlib venv or `uv venv` covers it.
- **cookiecutter**: maintained but slow-moving. Copier can re-apply template updates to existing projects, which cookiecutter can't.
- **PyPy**: alive (8.0 released Sep 2026) but trails CPython by several versions and has friction with C extensions. Good for pure-Python hot loops, not a default.
- **ty**: the opposite case: new, fast, and still 0.0.x beta. Worth watching, not yet worth gating CI on.

## The Astral factor

uv, Ruff and ty all come from Astral, which OpenAI agreed to acquire in March 2026. All three remain permissively licensed (MIT / Apache 2.0) and open source, so forking stays possible. The practical hedge: keep your config in standard pyproject.toml fields and commit your lockfile, and switching tools later stays cheap.

## Left out on purpose

- **Jython**, **IronPython**: stalled on Python 2.7-era compatibility.
- **Cython**: a compiler for extensions, not a runtime layer.

## Dog Solitude

The scrapyard. Dead, removed or abandoned tools, by layer.

| Code | Short | Layer | Obsolete |
|---|---|---|---|
| L9 | interface | Editor / IDE | Atom, Komodo IDE, Light Table, Ninja-IDE |
| L8 | intelligence | Language server | Microsoft Python Language Server, pyls (Palantir), Kite |
| L7 | style | Lint & format | PyChecker, pep8, pep257, pydocstyle |
| L6 | types | Type checking | pytype, Pydantic v1 |
| L5 | verify | Testing | nose, pytest-runner, setup.py test |
| L4 | project | Scaffold, build & publish | distutils, distutils2, distribute, eggs (.egg), bdist_wininst, setup.py upload, PasteScript |
| L3 | deps | Dependencies & environments | easy_install, pyvenv, pip legacy resolver, PyPM |
| L2 | versions | Python version manager | pythonbrew |
| L1 | runtime | Interpreter / runtime | Python 2.7, CPython 3.9 and older, Stackless Python, Unladen Swallow, Psyco, Pyjion, Pyston |
| L0 | spec | The language | Python 2 syntax, asyncore, asynchat, imp, lib2to3 / 2to3, PEP 594 dead batteries (cgi, telnetlib, crypt, nntplib...) |

## Sources

- [ty vs Pyrefly (pydevtools)](https://pydevtools.com/handbook/explanation/ty-vs-pyrefly/)
- [Pyrefly v1.0.0 release](https://github.com/facebook/pyrefly/releases/tag/1.0.0)
- [Python 3.15.0rc3 release page](https://www.python.org/downloads/release/python-3150rc3/)
- [PyPy v7.3.23 release notes](https://doc.pypy.org/release-v7.3.23.html)
- [OpenAI to Acquire Astral (pydevtools)](https://pydevtools.com/blog/openai-acquires-astral/)
