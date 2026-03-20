---
title: uv - The Python Package Manager You've Been Waiting For
date: 2026-03-20
draft: false
tags: ["python", "uv", "packaging", "poetry", "pip", "fastapi"]
categories: ["python"]
description: "A complete guide to uv, the tool that replaces pip, poetry, pyenv, and pipx in a single command. Learn in practice how to create projects, manage dependencies, Python versions, and global tools — dramatically faster."
cover: uv-python-package-manager.png
---

> **Level:** beginner to intermediate  
> **Prerequisites:** basic knowledge of Python and the command line

---

## The Problem uv Solves

If you've worked with Python for any amount of time, you've probably encountered something like this:

```bash
# To start a "simple" project, you needed:
pip install pipx          # to safely install global tools
pipx install poetry       # to manage dependencies
poetry self add poetry-plugin-shell  # to activate the venv via poetry
poetry python install 3.12           # to install the Python version
poetry new my-project     # to create the project
poetry shell              # to activate the environment
poetry add fastapi        # finally, install what you actually wanted
```

Before writing a single line of code, you'd already had to install **3 separate tools** (`pip`, `pipx`, `poetry`) and run **7 commands**. And that's not counting `pyenv` for managing multiple Python versions, `pip-tools` for locking dependencies, or `virtualenv` for creating virtual environments.

**uv** was built to fix this. A single tool, written in Rust, that replaces all of the above — and does it much faster.

---

## What is uv?

**uv** is a Python package and project manager developed by [Astral](https://astral.sh) — the same company behind [Ruff](https://github.com/astral-sh/ruff), the fastest Python linter/formatter available.

In one sentence: **uv is to Python what Cargo is to Rust** — a unified tool that handles everything in the project lifecycle.

### What it replaces

| Old tool | Purpose | uv equivalent |
|---|---|---|
| `pip` | Install packages | `uv pip install` |
| `pip-tools` | Lock dependencies | `uv pip compile` |
| `virtualenv` / `venv` | Create virtual environments | `uv venv` |
| `pyenv` | Manage Python versions | `uv python install` |
| `poetry` | Manage projects and deps | `uv init` + `uv add` |
| `pipx` | Isolated global tools | `uv tool install` / `uvx` |
| `twine` | Publish packages to PyPI | `uv publish` |

---

## Why Is It So Fast?

uv is written in **Rust** and uses several techniques that make it 10 to 100x faster than `pip`:

- **Global package cache**: packages downloaded once are cached and reused across all projects via hard links — no file copying.
- **Parallel dependency resolution**: resolves the dependency graph in parallel, not sequentially.
- **No Python overhead**: being written in Rust, it doesn't need Python to start — the process launches in milliseconds.

To put it concretely: installing [Trio](https://trio.readthedocs.io/)'s dependencies takes **~11ms** with a warm cache in uv, versus **~5s** with pip.

---

## Installation

```bash
# macOS and Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Via pip
pip install uv

# Via Homebrew (macOS)
brew install uv
```

After installing, reload your shell:

```bash
source ~/.bashrc   # or ~/.zshrc
```

Verify the installation:

```bash
uv --version
# uv 0.5.x (...)
```

---

## Core Concepts

Before diving into commands, two central concepts are worth understanding:

### 1. pyproject.toml — the heart of the project

`pyproject.toml` is the modern Python standard (PEP 517/518) for configuring projects. uv reads and writes it automatically. This is where your dependencies, Python version, scripts, and more are declared.

```toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.9.0",
]
```

### 2. uv.lock — the lockfile

`uv.lock` is generated automatically and records the **exact** versions of all dependencies (including transitive ones). This guarantees that every team member and CI server installs exactly the same packages.

> **Tip:** commit `uv.lock` to git. Never edit it manually.

---

## Practical Guide: From Zero to Project

### Creating a new project

```bash
# Option 1: create the folder and initialize
uv init my-project
cd my-project

# Option 2: initialize in the current folder
mkdir my-project && cd my-project
uv init
```

Generated structure:

```
my-project/
├── main.py           ← example entry point
├── pyproject.toml    ← project configuration
├── README.md
└── .python-version   ← pinned Python version
```

### Installing a specific Python version

```bash
# List available versions
uv python list

# Install a version
uv python install 3.13

# Pin the version for this project (creates/updates .python-version)
uv python pin 3.13
```

### Adding dependencies

```bash
# Main dependency
uv add fastapi

# With extras (equivalent to pip install fastapi[standard])
uv add "fastapi[standard]"

# Development dependency
uv add --dev pytest ruff

# Specific version range
uv add "sqlalchemy>=2.0,<3.0"
```

uv updates `pyproject.toml` and `uv.lock` automatically.

### Installing existing dependencies

```bash
# Sync the environment with pyproject.toml / uv.lock
uv sync

# Without dev dependencies (useful in production)
uv sync --no-dev
```

### Removing dependencies

```bash
uv remove fastapi
```

### Running commands in the virtual environment

```bash
# Run the application
uv run fastapi dev my_project/app.py

# Run tests
uv run pytest

# Run a script
uv run python script.py

# Open the interactive interpreter
uv run python
```

> **You don't need to activate the venv manually.** `uv run` handles it for you.

If you prefer to activate the venv for a terminal session:

```bash
source .venv/bin/activate   # Linux/macOS
.venv\Scripts\activate      # Windows
```

---

## Full Example: FastAPI Project

Let's build a FastAPI project from scratch with uv:

```bash
# 1. Create the project
uv init fast-zero
cd fast-zero

# 2. Create the Python package
mkdir fast_zero
touch fast_zero/__init__.py

# 3. Install dependencies
uv add "fastapi[standard]"
uv add --dev pytest pytest-cov ruff taskipy

# 4. Run (after creating app.py as shown below)
uv run fastapi dev fast_zero/app.py
```

Before running, create the file `fast_zero/app.py` with the following content:

```python
from fastapi import FastAPI

app = FastAPI()


@app.get('/')
def read_root():
    return {'message': 'Hello, World!'}
```

Configuring `pyproject.toml`:

```toml
# ─────────────────────────────────────────────
# [project] — metadata and main dependencies
# Essential project information and packages
# required to run in production.
# ─────────────────────────────────────────────
[project]
name = "fast-zero"          # project name (kebab-case by convention)
version = "0.1.0"           # current project version
requires-python = ">=3.12"  # minimum accepted Python version
dependencies = [
    # fastapi with the [standard] group, which includes uvicorn, httpx
    # and other useful extras for development and production
    "fastapi[standard]>=0.115.0",
]

# ─────────────────────────────────────────────
# [dependency-groups] — development dependencies
# Packages used only during development
# (tests, linting, formatting). Not included in production.
# ─────────────────────────────────────────────
[dependency-groups]
dev = [
    "pytest>=8.0.0",      # testing framework
    "pytest-cov>=6.0.0",  # plugin to measure test coverage
    "ruff>=0.9.0",        # linter and code formatter
    "taskipy>=1.14.0",    # task runner — shortcuts for project commands
]

# ─────────────────────────────────────────────
# [tool.ruff] — global Ruff configuration
# Defines behaviors that apply to both
# the linter and the formatter.
# ─────────────────────────────────────────────
[tool.ruff]
line-length = 79             # maximum characters per line (PEP 8 standard)
extend-exclude = ['migrations']  # ignores the migrations folder (auto-generated
                                 # code — we don't want ruff to touch it)

# ─────────────────────────────────────────────
# [tool.ruff.lint] — linter configuration
# Defines which best-practice rules ruff
# will check in the code.
# ─────────────────────────────────────────────
[tool.ruff.lint]
preview = true  # enables rules still in experimental phase
select = [
    'I',   # isort — checks alphabetical ordering of imports
    'F',   # pyflakes — detects common errors (unused variables, unnecessary imports)
    'E',   # pycodestyle errors — code style errors
    'W',   # pycodestyle warnings — code style warnings
    'PL',  # pylint — general Python best practices
    'PT',  # flake8-pytest — best practices specific to pytest tests
]

# ─────────────────────────────────────────────
# [tool.ruff.format] — formatter configuration
# Defines the automatic code formatting style.
# ─────────────────────────────────────────────
[tool.ruff.format]
preview = true          # enables experimental formatting behaviors
quote-style = 'single'  # uses single quotes instead of double (team preference)

# ─────────────────────────────────────────────
# [tool.pytest.ini_options] — pytest configuration
# Defines how pytest behaves when running tests.
# ─────────────────────────────────────────────
[tool.pytest.ini_options]
pythonpath = "."           # adds the project root to PYTHONPATH, enabling
                           # imports like "from fast_zero.app import app"
addopts = '-p no:warnings' # suppresses warnings from external libraries to
                           # keep test output clean

# ─────────────────────────────────────────────
# [tool.taskipy.tasks] — command shortcuts
# Defines tasks runnable with "uv run task <name>".
# Saves you from memorizing long flags and paths.
# ─────────────────────────────────────────────
[tool.taskipy.tasks]
lint   = 'ruff check'                        # checks best practices in the code
format = 'ruff format'                       # formats the code automatically
run    = 'fastapi dev fast_zero/app.py'      # starts the development server
test   = 'pytest -s -x --cov=fast_zero -vv' # runs tests with:
                                             #   -s: shows prints during tests
                                             #   -x: stops on first failure
                                             #   --cov=fast_zero: measures package coverage
                                             #   -vv: verbose output
```

Running the tasks:

```bash
uv run task test
uv run task lint
uv run task run
```

---

## Global Tools (replaces pipx)

uv also manages global tools — those you use in the terminal but aren't tied to any specific project.

```bash
# Install a tool globally
uv tool install ruff
uv tool install httpie
uv tool install black

# Run without installing (temporary environment, equivalent to pipx run)
uvx ruff check .
uvx black --check .

# List installed tools
uv tool list

# Upgrade
uv tool upgrade ruff

# Remove
uv tool uninstall black
```

---

## Scripts with Inline Dependencies

A lesser-known but very useful feature: uv can run **Python scripts with dependencies declared in the file itself**, without needing a full project.

```python
# script.py
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx",
#   "rich",
# ]
# ///

import httpx
from rich import print

response = httpx.get("https://api.github.com/repos/astral-sh/uv")
print(response.json()["stargazers_count"])
```

```bash
uv run script.py
# Automatically installs deps in an isolated env and runs the script
```

This is great for utility scripts, automations, and sharing code without setting up an entire project.

---

## Command Reference Table

| Command | What it does |
|---|---|
| **Project** | |
| `uv init my-project` | Creates folder and initializes project |
| `uv init` | Initializes project in current folder |
| `uv init --lib` | Creates a library layout (with `src/`) |
| **Dependencies** | |
| `uv add fastapi` | Adds a main dependency |
| `uv add "fastapi[standard]"` | Adds with extras |
| `uv add --dev pytest` | Adds as a dev dependency |
| `uv remove fastapi` | Removes a dependency |
| `uv sync` | Installs everything from `pyproject.toml` (= `poetry install`) |
| `uv sync --no-dev` | Installs production dependencies only |
| `uv lock` | Generates/updates `uv.lock` without installing |
| **Running** | |
| `uv run <command>` | Runs in the project's virtual environment |
| `uv run --with httpx script.py` | Runs with a temporary extra package |
| **Global tools** | |
| `uvx ruff check .` | Runs a tool without installing (= `pipx run`) |
| `uv tool install ruff` | Installs a tool globally (= `pipx install`) |
| `uv tool uninstall ruff` | Removes a global tool |
| `uv tool list` | Lists installed global tools |
| `uv tool upgrade ruff` | Upgrades a global tool |
| **Python versions** | |
| `uv python install 3.13` | Downloads and installs Python 3.13 |
| `uv python list` | Lists available/installed versions |
| `uv python pin 3.12` | Pins version in `.python-version` |
| `uv python uninstall 3.11` | Removes an installed version |
| **Virtual env and pip** | |
| `uv venv` | Creates `.venv/` in current folder |
| `uv venv --python 3.12` | Creates venv with specific version |
| `uv pip install fastapi` | Installs a package (pip-compatible interface) |
| `uv pip freeze` | Lists installed packages |
| `uv pip compile pyproject.toml -o requirements.txt` | Generates a locked `requirements.txt` |

---

## uv vs. Alternatives

### uv vs. pip + venv

| | pip + venv | uv |
|---|---|---|
| Speed | Slow | 10–100x faster |
| Lockfile | None (needs pip-tools) | Native `uv.lock` |
| Manages Python | No | Yes |
| Single tool | No (multiple tools) | Yes |
| Learning curve | Low | Low |

**When to use pip + venv:** legacy projects, environments where uv can't be installed.

---

### uv vs. Poetry

| | Poetry | uv |
|---|---|---|
| Speed | Moderate | Much faster |
| Manages Python | Yes (via plugin) | Yes (native) |
| Lockfile format | `poetry.lock` (proprietary) | `uv.lock` (PEP standard) |
| Publish to PyPI | Yes | Yes (`uv publish`) |
| Shell plugin | Needs installation | `uv run` makes it unnecessary |
| Maturity | High (since 2018) | Growing fast (2024+) |
| pip compatibility | Partial | Full (pip interface) |

**When to use Poetry:** teams with heavy Poetry history, projects that depend on specific Poetry plugins.

---

### uv vs. Conda

| | Conda | uv |
|---|---|---|
| Non-Python packages | Yes (C, Fortran, CUDA) | No |
| Speed | Slow | Much faster |
| Data science use | Excellent | Adequate |
| Installation size | Large | Small |
| Focus | Data science / ML | General Python development |

**When to use Conda:** data science with heavy native dependencies (GPU TensorFlow, CUDA, geospatial packages).

---

## Advantages of uv

- **Exceptional speed**: dependency resolution and installation in fractions of a second
- **Single tool**: replaces pip, poetry, pyenv, pipx, and more
- **Smart global cache**: saves disk space with hard links
- **Standard lockfile**: compatible with the modern Python ecosystem
- **No circular dependency**: doesn't need Python to run
- **Full pip compatibility**: migrate gradually without breaking anything
- **Actively maintained**: by Astral, with corporate backing

---

## Disadvantages and Limitations

- **Relatively new**: launched in 2024; less track record than pip/poetry
- **No non-Python packages**: no support for C, CUDA, etc. (Conda's domain)
- **Frequent changes**: the API is still evolving; some commands changed between versions
- **uv.lock ≠ poetry.lock**: migrations require conversion
- **Smaller plugin ecosystem**: Poetry has mature plugins that uv doesn't yet replicate

---

## Migrating from Poetry to uv

If you have an existing Poetry project and want to migrate:

```bash
# 1. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. In your project directory, sync
# uv can read Poetry's pyproject.toml natively
uv sync

# 3. Swap your day-to-day commands
# poetry run pytest   →   uv run pytest
# poetry add X        →   uv add X
# poetry shell        →   source .venv/bin/activate  (or just use uv run)

# 4. (Optional) Generate uv.lock and remove poetry.lock
uv lock
git rm poetry.lock
git add uv.lock
```

> `uv sync` can read `pyproject.toml` files generated by Poetry. The migration is typically straightforward for most projects.

---

## Day-to-Day Tips

### Pass environment variables when running

```bash
DATABASE_URL=postgresql://localhost/dev uv run python manage.py migrate
```

### Working with .env files

```bash
# uv doesn't read .env automatically — use python-dotenv or direnv
uv add --dev python-dotenv
```

### Check what's installed

```bash
uv pip list
uv pip show fastapi
```

### Upgrade all dependencies

```bash
uv lock --upgrade
uv sync
```

### Export requirements.txt (for legacy deployments)

```bash
uv pip compile pyproject.toml -o requirements.txt
# or, for production only:
uv export --no-dev -o requirements.txt
```

### Run a one-off command without a project

```bash
uv run --with fastapi --with uvicorn python -c "
import uvicorn
from fastapi import FastAPI
app = FastAPI()

@app.get('/')
def root(): return {'ok': True}

uvicorn.run(app)
"
```

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run tests
        run: uv run pytest
```

The `--frozen` flag ensures `uv.lock` won't be updated during CI — if there's a divergence, the build fails, which is the desired behavior.

### Docker

```dockerfile
FROM python:3.13-slim

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copy lockfile and install dependencies (cacheable layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Copy code
COPY . .

CMD ["uv", "run", "fastapi", "run", "fast_zero/app.py", "--port", "8000"]
```

---

## Conclusion

uv is probably the most significant improvement to the Python development experience in years. It doesn't reinvent the wheel — it takes all the tools we were already using and unifies them into a coherent interface that's dramatically faster and easier to use.

If you're starting a new project today, **use uv**. If you have existing projects with Poetry or pip, the migration is straightforward and worth the effort.

The Python ecosystem finally has the tool it deserved.

---

## Additional Resources

- [uv official documentation](https://docs.astral.sh/uv/)
- [Migration guide: Poetry → uv](https://docs.astral.sh/uv/guides/migration/)
- [FastAPI integration guide](https://docs.astral.sh/uv/guides/integration/fastapi/)
- [GitHub repository](https://github.com/astral-sh/uv)
- [Speed benchmarks](https://github.com/astral-sh/uv/blob/main/BENCHMARKS.md)
