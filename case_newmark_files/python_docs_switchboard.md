# The Python Docs Switchboard

Which Python documentation tools to use, routed by what you are shipping. Companion to [the Python Toolchain Stack](python_toolchain_stack.md). State as of October 2026.

For the styled version, open [python_docs_switchboard.html](python_docs_switchboard.html) in a browser.

**Status key:** **PICK** = modern default · alt = solid alternative · ⚠️ edge = maintained, but teetering, too new, or overtaken

## The quote

> “So if you’re building a library, Sphinx or MkDocs is the tool; if you’re building a web service, OpenAPI is.”

- **Decrypt:** Mostly right. The split is real: a library documents its Python API for people who import it, while a web service documents an HTTP contract for people who call it over the wire. Different readers, different tools.
- **Patch 1:** The MkDocs half got complicated in 2026. Material for MkDocs, the theme nearly everyone used it with, went into maintenance mode, and its team is building a successor called Zensical that is still pre-1.0.
- **Patch 2:** It skips command-line tools, where the first documentation anyone reads is --help.

---

## LIB · Library / package (quote: covered)

Code other people import. Readers need install steps, tutorials, and an API reference generated from your docstrings so it never drifts from the code.

**Pipeline:** write: Markdown (MyST) → extract: autodoc + napoleon → build: Sphinx → theme: Furo → host: Read the Docs

**Take:** Sphinx is the safe pick for a new library right now: stable, versioned, and MyST lets you write it all in Markdown. MkDocs + Material is still friendlier and perfectly fine for existing sites; just know it's in maintenance mode while Zensical matures.

### Sphinx side

| Tool | Status | Notes |
|---|---|---|
| Sphinx | **PICK** | The Python standard since 2008 (CPython's own docs). Cross-references, versioning, PDF/ePub output, a huge extension ecosystem. 9.x is current. |
| MyST-Parser | **PICK** | Write Sphinx docs in Markdown instead of reStructuredText, with full access to directives and roles. |
| autodoc + napoleon | **PICK** | Pull API docs from docstrings; napoleon reads Google and NumPy style. Pair with sphinx-autodoc-typehints. |
| sphinx-autoapi | alt | Builds the API reference by parsing source instead of importing it. Safer for packages with heavy or side-effecting imports. |
| Furo | **PICK** | Clean, fast, accessible theme with dark mode. The common default for new projects. |
| pydata-sphinx-theme | alt | The theme of NumPy, pandas and much of the scientific stack. Pick it if your library lives in that world. |

### MkDocs side

| Tool | Status | Notes |
|---|---|---|
| MkDocs + Material | ⚠️ edge | The friendliest docs setup in Python for years. Material entered maintenance mode in early 2026: critical and security fixes only, for at least 12 months. |
| mkdocstrings | alt | API reference for MkDocs, powered by griffe (static analysis, no import needed). Still actively developed. |
| Zensical | ⚠️ edge | Material's successor from the same team; reads existing mkdocs.yml. Promising, but still 0.0.x. |
| MkDocs 2.0 | ⚠️ edge | A ground-up rewrite of core MkDocs, in development with no release date. Expect breaking changes for plugins. |

### Minimalist

| Tool | Status | Notes |
|---|---|---|
| pdoc | alt | Zero-config API docs straight from docstrings and type hints. Ideal for small libraries and internal packages; no narrative pages. |
| pydoctor | alt | API-reference generator from the Twisted project. Solid, but rarely chosen outside that ecosystem. |

---

## API · Web service (quote: covered)

Code other people call over HTTP. Readers need every endpoint, request and response schema, auth flow and error, in a form machines can also read to generate clients.

**Pipeline:** write: type hints + Pydantic → extract: FastAPI → spec: OpenAPI 3.1 → render: Swagger UI / Scalar → lint: Redocly CLI

**Take:** Never write the OpenAPI file by hand if your framework can emit it. Generate it from your type hints and Pydantic models, so the docs and the code are the same source.

### Spec

| Tool | Status | Notes |
|---|---|---|
| OpenAPI 3.1 | **PICK** | The current workhorse. Aligned with JSON Schema, supported by essentially every generator, renderer and client tool. |
| OpenAPI 3.2 | ⚠️ edge | Released Sept 2025: streaming (SSE, JSON Lines), nested tags, a QUERY method, OAuth device flow. Tooling support is still catching up. |

### Emit it from your framework

| Tool | Status | Notes |
|---|---|---|
| FastAPI | **PICK** | Generates the spec automatically from type hints and Pydantic models, and serves Swagger UI and ReDoc out of the box. |
| Litestar | alt | Same idea as FastAPI, with built-in OpenAPI and several bundled renderers. |
| Django Ninja | alt | FastAPI-style typed API layer for Django, with built-in OpenAPI. |
| drf-spectacular | **PICK** | The standard OpenAPI 3 generator for Django REST Framework. |
| APIFlask / flask-smorest | alt | OpenAPI for Flask apps, both built on the apispec library. |

### Render & check

| Tool | Status | Notes |
|---|---|---|
| Swagger UI | **PICK** | The interactive try-it-out console. Bundled with most frameworks. |
| ReDoc | alt | Three-panel reference layout. Better for reading than for poking at endpoints. |
| Scalar | alt | Modern renderer with a built-in API client and good dark mode. Increasingly the nicer-looking default. |
| Redocly CLI / Spectral | alt | Lint the spec in CI for missing descriptions, inconsistent naming and breaking changes. |
| openapi-python-client | alt | Generates a typed Python client from the spec. A good test of whether your spec is complete. |

### Not REST?

| Tool | Status | Notes |
|---|---|---|
| AsyncAPI | alt | OpenAPI's sibling for event-driven services: Kafka, MQTT, WebSockets. |
| GraphQL schema | alt | Self-documenting through introspection; Strawberry generates it from type hints. Explore with GraphiQL. |
| gRPC .proto | alt | The .proto files are the contract; protoc-gen-doc renders them to HTML or Markdown. |

---

## CLI · Command-line tool (quote: silent)

Code people run from a terminal. The first documentation they see is --help, so that's where the effort goes first.

**Pipeline:** write: Typer help text → render: sphinx-click / mkdocs-click → publish: README + docs site

**Take:** Write good help text in Typer or Click, then render it into your docs site with a plugin rather than copying it by hand.

| Tool | Status | Notes |
|---|---|---|
| Typer | **PICK** | Builds the CLI and its --help output from type hints and docstrings. Rich formatting built in. |
| Click | alt | The library under Typer. More explicit, same documentation story. |
| sphinx-click / mkdocs-click | **PICK** | Render the real --help output of every command into your docs, so the two never drift. |
| argparse | alt | Stdlib. Fine for small scripts; help output is plain. |
| README | **PICK** | For most CLI tools a strong README with install, a quickstart and examples is most of the docs anyone reads. |

---

## Shared ground

Applies on every route. The generator renders the docs; these decide whether there is anything good to render.

### Docstring style

| Tool | Status | Notes |
|---|---|---|
| Google style | **PICK** | Readable as plain text, supported by napoleon and mkdocstrings. The usual default. |
| NumPy style | **PICK** | The scientific stack's convention. Longer, very structured; use it if your users live there. |
| reST field lists | ⚠️ edge | Sphinx's native :param: style. Still works, but hard to read in source. |

### Docstring checks

| Tool | Status | Notes |
|---|---|---|
| Ruff D + DOC rules | **PICK** | pydocstyle and pydoclint checks, reimplemented in Ruff. Catches missing and malformed docstrings in the same pass as linting. |
| interrogate | alt | Reports docstring coverage as a percentage. Useful as a CI gate. |
| doctest | alt | Run the examples in your docstrings as tests (pytest --doctest-modules), so examples stay correct. |

### Structure

| Tool | Status | Notes |
|---|---|---|
| Diátaxis | **PICK** | Split docs into tutorials, how-to guides, reference and explanation. The most useful idea in technical writing in years. |

### Hosting

| Tool | Status | Notes |
|---|---|---|
| Read the Docs | **PICK** | Versioned docs per release, PR previews, search. Free for open source. |
| GitHub Pages | alt | Build in GitHub Actions and publish. Simple; versioning is up to you. |
| llms.txt | ⚠️ edge | An emerging convention for an AI-readable index of your docs; plugins exist for Sphinx and MkDocs. Cheap to add, not yet a standard. |

## Dog Solitude

The scrapyard. Dead, removed or abandoned tools, by route.

| Code | Route | Obsolete |
|---|---|---|
| LIB | Library / package | Epydoc, HappyDoc, recommonmark, m2r |
| API | Web service | Swagger 2.0 spec, django-rest-swagger, flask-restplus |
| ALL | Shared ground | pydocstyle, darglint |

## Sources

- [Material for MkDocs reference (pydevtools)](https://pydevtools.com/handbook/reference/mkdocs-material/)
- [Zensical release listing](https://simple-repository.app.cern.ch/project/zensical)
- [OpenAPI 3.2 announcement (OpenAPI Initiative)](https://www.openapis.org/?p=4078)
