# Environments and Packages

Python's packaging story has a reputation, captured by a famous XKCD comic showing a tangle
of Python installations, package managers and paths. Much of that reputation is historical:
modern tools have made Python environments nearly as predictable as .NET projects. But you
need to understand the model underneath, or you'll install packages into the wrong place,
break your operating system's Python, or ship code that only works on your machine.

This chapter explains interpreters, virtual environments, packages and lock files, and sets
up Beacon's Python tooling with **uv**, the fast, modern project manager that has become the
default for new projects.

---

## 1. The problem: one machine, many Pythons and many dependency sets

On a typical machine:

- The OS may ship its own Python, which system tools depend on.
- You may need several Python versions (3.12 for one project, 3.14 for another).
- Each project needs its own set of packages at specific versions: one needs `pydantic` 1.x,
  another 2.x.

In .NET, each project's dependencies are resolved per project and the runtime is selected via
the target framework (Book I, Chapter 1). Python's default behavior is different: `pip
install` installs into **whichever Python environment is active**, globally by default. That's
the root of most Python environment problems.

---

## 2. The mental model: interpreter + environment + packages

```text
 Python interpreter (e.g. CPython 3.14 at /usr/bin/python3.14)
   └─ site-packages: where installed packages live for this environment
 Virtual environment (.venv/ inside your project)
   ├─ a link to an interpreter
   └─ its own, isolated site-packages
```

A **virtual environment** (venv) is a directory containing a pointer to a Python interpreter
and its own `site-packages`. Activating it (or running its `python` directly) makes `import`
use that project's packages only. **One virtual environment per project** is the rule, like
each .NET project having its own dependency graph.

### Package sources and formats

- **PyPI** (Python Package Index) is the public registry, like nuget.org.
- **Wheels** (`.whl`) are pre-built packages, possibly containing compiled native code for
  specific platforms (e.g. `numpy-…-cp314-manylinux_x86_64.whl`). Installing a wheel is fast.
- **Source distributions** (`.tar.gz`) must be built on install, which may need a C compiler.
  Most popular packages ship wheels for common platforms.

### Dependency declaration

Modern Python projects declare dependencies in **`pyproject.toml`** (the equivalent of a
`.csproj`):

```toml
[project]
name = "beacon-tools"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.8",
    "typer>=0.12",
]

[dependency-groups]
dev = ["pytest>=8", "ruff>=0.6", "mypy>=1.11"]
```

Version specifiers: `>=2.8` (at least), `~=2.8` (compatible: ≥2.8, <3.0), `==2.8.1` (exact).
`pyproject.toml` lists **direct** dependencies with ranges; a **lock file** records the exact
resolved versions of everything (including transitive dependencies), for reproducible installs,
the same distinction as `package.json` and `pnpm-lock.yaml` (Book V, Chapter 3).

---

## 3. Tools: pip, venv and uv

### The classic toolchain

```bash
python3 -m venv .venv                   # create an environment
source .venv/bin/activate               # activate (Windows: .venv\Scripts\activate)
pip install httpx pydantic              # install into the venv
pip freeze > requirements.txt           # pin current versions (a crude lock file)
pip install -r requirements.txt         # reproduce
deactivate
```

It works, but it's slow, `requirements.txt` mixes direct and transitive dependencies, and
managing Python versions themselves needs another tool (pyenv).

### uv: the modern default

**uv** (from Astral, written in Rust) replaces pip, venv, pip-tools, pipx and pyenv with one
very fast tool:

```bash
# install uv once (see docs for Windows)
curl -LsSf https://astral.sh/uv/install.sh | sh

uv python install 3.14                 # install a Python version (no system Python needed)
uv init beacon-tools                   # new project with pyproject.toml
cd beacon-tools
uv add httpx pydantic typer            # add dependencies (updates pyproject.toml and uv.lock)
uv add --dev pytest ruff mypy          # dev dependencies
uv run python -m beacon_tools          # run inside the project's environment (created automatically)
uv run pytest                          # run tools from the environment
uv sync --locked                       # install exactly what uv.lock says (CI)
uv lock --upgrade-package httpx        # upgrade one dependency deliberately
uvx ruff check .                       # run a tool in an ephemeral environment (like npx / dotnet tool exec)
```

What uv gives you, mapped to .NET and Node concepts:

| Need | .NET | Node | uv |
|---|---|---|---|
| Runtime version | Target framework, SDK | `.nvmrc` / volta | `requires-python`, `.python-version`, `uv python install` |
| Dependencies | `<PackageReference>` | `package.json` | `pyproject.toml` |
| Lock file | `packages.lock.json` | `pnpm-lock.yaml` | `uv.lock` |
| Restore/install | `dotnet restore` | `pnpm install --frozen-lockfile` | `uv sync --locked` |
| Run in project context | `dotnet run` | `pnpm exec` | `uv run` |
| Global tools | `dotnet tool` | `npx` | `uvx` / `uv tool install` |

> **🔄 Current (as of October 2026):** uv has become the most widely recommended tool for new
> Python projects. Poetry, PDM and Hatch remain in use and follow the same `pyproject.toml`
> standards. pip and venv remain the universal baseline every Python installation has.

### Conda

**Conda** (and its faster implementation, mamba) manages environments that include
**non-Python** dependencies (CUDA libraries, C/Fortran scientific libraries) from the
conda-forge channel. It's common in data science and ML research. For application
development and AI API work, uv with PyPI wheels is usually simpler.

---

## 4. Project layout and quality tools

```text
beacon-tools/
  pyproject.toml
  uv.lock
  .python-version
  src/
    beacon_tools/
      __init__.py
      __main__.py          # `python -m beacon_tools`
      models.py
      client.py
      cli.py
  tests/
    test_models.py
```

The **`src/` layout** prevents accidentally importing the package from the working directory
instead of the installed environment, a subtle source of "works locally, fails in CI" bugs.

### Ruff, type checking and pytest

```toml
# pyproject.toml (excerpt)
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "ASYNC", "S"]   # pycodestyle, pyflakes, isort, bugbear, pyupgrade, simplify, async, security

[tool.mypy]
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

- **Ruff**: an extremely fast linter **and** formatter (replacing flake8, isort, Black and more).
  The equivalent of ESLint + Prettier (Book V, Chapter 3) or Roslyn analyzers + `dotnet format`.
- **mypy** / **pyright**: static type checking (Chapter 1's type hints).
- **pytest**: the standard test framework, with plain `assert` statements and fixtures:

```python
# tests/test_models.py
from datetime import datetime, timedelta, timezone
from beacon_tools.models import Priority, Ticket

NOW = datetime(2026, 10, 7, 12, tzinfo=timezone.utc)

def make(priority=Priority.URGENT, age=timedelta(minutes=30), status="Open") -> Ticket:
    return Ticket(id="T-1", title="VPN", priority=priority, status=status, created_at=NOW - age)

def test_urgent_ticket_breaches_after_one_hour():
    assert not make(age=timedelta(minutes=60)).is_breaching(NOW)
    assert make(age=timedelta(minutes=61)).is_breaching(NOW)

def test_resolved_tickets_never_breach():
    assert not make(status="Resolved", age=timedelta(days=30)).is_breaching(NOW)
```

The same boundary tests as Book I, Chapter 14, in Python.

---

## 5. Supply-chain safety

PyPI has the same risks as npm (Book V, Chapter 3): typosquatting, malicious packages,
compromised maintainers, and packages that run code at install time (source distributions
execute build scripts).

Defenses:

- **Lock files** (`uv.lock`) and `uv sync --locked` in CI.
- **Prefer wheels**; be wary of packages that only ship source distributions.
- **Audit**: `uvx pip-audit` (or GitHub's dependency scanning) for known vulnerabilities.
- **Check that packages exist and are the real thing**, especially names suggested by AI
  assistants (hallucinated package names are registered by attackers: Book XI, Chapter 11).
- **Private indexes** (Azure Artifacts feeds) for internal packages, with index configuration
  that prevents dependency confusion.

---

## 6. Running Python in production

- **Containers**: `python:3.14-slim` base images or distroless variants; install with
  `uv sync --locked --no-dev` in a build stage; run as non-root (Book VIII, Chapters 4 and 6).

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.14-bookworm-slim AS build
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project
COPY src/ src/
RUN uv sync --locked --no-dev

FROM python:3.14-slim-bookworm
WORKDIR /app
COPY --from=build /app /app
ENV PATH="/app/.venv/bin:$PATH"
USER 1000
ENTRYPOINT ["python", "-m", "beacon_tools"]
```

- **Web services**: FastAPI (or Flask/Django) on an ASGI server like Uvicorn, behind the same
  kinds of proxies and platforms as .NET apps.
- **Serverless**: Azure Functions supports Python; Container Apps jobs run Python containers.
- **Notebooks** (Jupyter) for exploration and data analysis, not for production code paths.

---

## 7. In practice: Beacon's Python tooling project

Beacon gets a small Python project, `tools/beacon-tools`, in the monorepo for AI evaluation
scripts, data exports and automation (Chapter 3 and Book XI):

```bash
cd tools
uv init --package beacon-tools --python 3.14
cd beacon-tools
uv add httpx pydantic typer rich
uv add --dev pytest ruff mypy respx
uv run pytest
uv run ruff check . && uv run ruff format --check .
uv run mypy src
```

CI job (next to the .NET and web jobs from Books II and V):

```yaml
  python-tools:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: tools/beacon-tools } }
    steps:
      - uses: actions/checkout@<pinned-sha>
      - uses: astral-sh/setup-uv@<pinned-sha>
      - run: uv sync --locked
      - run: uv run ruff check . && uv run ruff format --check .
      - run: uv run mypy src
      - run: uv run pytest
      - run: uvx pip-audit
```

Every developer and CI run now uses the same Python version (from `.python-version`), the same
locked packages, and the same quality gates, which is the .NET-like predictability you're used to.

---

## 8. What can go wrong

- **Installing into the system Python** (with `sudo pip`), breaking OS tools.
- **No virtual environment**, so projects share and break each other's packages.
- **No lock file**, so CI and production install different versions than you tested.
- **"It works in my notebook"**: code depending on notebook state and cell execution order.
- **Native dependencies** missing in slim containers (compilers, system libraries).
- **Untrusted packages**, especially ones suggested by AI tools.
- **Mixing tools** (pip, conda, uv) in one environment.

---

## 9. How an experienced engineer thinks about this

- **One isolated environment per project**, with a pinned interpreter version.
- **Declare direct dependencies; lock everything**; install from the lock in CI.
- **Use modern, fast tooling** (uv, Ruff) to make the right thing the easy thing.
- **Same quality gates as any other code**: lint, format, types, tests, audit.

---

## 10. Check yourself

**Questions**

1. What is a virtual environment, and why use one per project?
2. What's the difference between `pyproject.toml` dependencies and a lock file?
3. What are wheels and source distributions?
4. What does `uv run` do? `uv sync --locked`?
5. Why use the `src/` layout?
6. What do Ruff, mypy and pytest do?
7. What supply-chain risks apply to PyPI, and how do you mitigate them?

**Exercises**

1. Create `beacon-tools` with uv, add dependencies, and inspect `uv.lock`.
2. Write pytest tests for `workload_by_agent` from Chapter 1.
3. Build the Docker image from section 6 and run it as a non-root user.
4. Run `uvx pip-audit` and resolve any findings.

**Interview-style questions**

- "How do you manage dependencies and environments in Python?"
- "How do you make Python builds reproducible?"

---

## 11. Going deeper

- [uv documentation](https://docs.astral.sh/uv/)
- [Python Packaging User Guide](https://packaging.python.org/)
- [Ruff documentation](https://docs.astral.sh/ruff/) and [pytest documentation](https://docs.pytest.org/)

**Next:** [Chapter 3 — Practical Python](03-practical-python.md) writes real tools: API
clients, data processing and automation scripts for Beacon.
