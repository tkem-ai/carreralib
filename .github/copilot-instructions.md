# carreralib

Python interface to Carrera® DIGITAL 124/132 slotcar systems, connected via
serial (cable) or Bluetooth LE (Carrera AppConnect®). A pure-Python library
with a small `curses` demo race management system (RMS). Requires
Python >= 3.10; CI also runs on free-threaded (`3.13t`, `3.14t`) and PyPy
interpreters.

This file follows the [AGENTS.md](https://agents.md/) conventions: plain
Markdown, grouped by topic, with commands that can be run as written.
Instructions given explicitly in a task, issue or chat prompt take
precedence over this file.

## Project layout

- `src/carreralib/` — the library package (`src` layout)
  - `__init__.py` — public exports (`ControlUnit`, `connection`, `protocol`)
    and `__version__`
  - `cu.py` — `ControlUnit`, the main public API for talking to a Control Unit
  - `protocol.py` — low-level wire protocol: `pack`/`unpack`, `chksum`
  - `connection.py` — abstract `Connection` base class, `open()` and `scan()`
  - `serial.py` — serial (pyserial) connection; internal, undocumented
  - `ble.py` — Bluetooth LE (bleak) connection; internal, undocumented
  - `fw.py` — firmware version/update CLI (`python -m carreralib.fw`); internal
  - `__main__.py` — demo RMS CLI (`python -m carreralib`); internal
  - `*.pyi` — type stubs for the documented public modules only
    (`__init__`, `cu`, `connection`, `protocol`)
  - `py.typed` — PEP 561 marker so type checkers pick up the stubs
- `tests/` — `unittest`-style tests, run with `pytest`
- `docs/` — Sphinx documentation (reStructuredText, `index.rst` only)
- `CHANGELOG.rst` — release notes, newest first
- `pyproject.toml` — packaging, `ruff` and `pyright` configuration
- `tox.ini` — the single source of truth for all checks

## Setup commands

- Install the check runner: `pip install tox`
- Install the package for ad-hoc work: `pip install -e .`
  (pulls in `bleak` and `pyserial`; `windows-curses` on Windows)
- Tools used by the tox environments (`pytest`, `pytest-cov`, `ruff`,
  `pyright`, `sphinx`) are installed by tox itself; install them manually
  only if you want to run a tool outside tox.

## Build and test commands

Run everything CI runs with a single command: `tox`. Individual
environments:

- `tox -e py` — run tests via `pytest` with coverage
- `tox -e pyright` — type-check with pyright (`typeCheckingMode = "standard"`)
- `tox -e ruff` — lint with `ruff check`
- `tox -e ruff-format` — check formatting (`ruff format --diff`, check only)
- `tox -e docs` — build the Sphinx HTML docs with warnings as errors (`-W`)
- `tox -e doctest` — run doctests in `docs/` (also `-W`)

Before considering a change complete, run `tox` (or at least `tox -e
py,pyright,ruff,ruff-format`) and make sure it is green. `tox -e
ruff-format` only reports; run `ruff format` to actually reformat files.

## Code style

- Formatting and linting are enforced by `ruff`; configuration lives in
  `[tool.ruff.lint]` in `pyproject.toml`. Do not hand-format around it.
- Target Python 3.10 syntax and stdlib. Do not use features newer than 3.10
  (e.g. PEP 695 `type` statements or generics syntax, `ExceptionGroup`,
  `tomllib`). Keep the code free of `typing` imports in `.py` files; types
  live in the `.pyi` stubs.
- Public API is exported from `carreralib/__init__.py`; keep `__all__` and
  the matching `__init__.pyi` in sync when adding public symbols.
- Docstrings use reST/Sphinx style (see `cu.py`), including grid tables for
  namedtuple field documentation. Follow this for new public classes and
  functions. Only the public modules are rendered by `sphinx.ext.autodoc`.
- Response types (`ControlUnit.Status`, `ControlUnit.Timer`) are
  `collections.namedtuple` subclasses with `__slots__ = ()`.
- When adding or changing a public symbol in `__init__.py`, `cu.py`,
  `connection.py` or `protocol.py`, update the matching `.pyi` stub in the
  same directory in the same change. `serial.py`, `ble.py`, `fw.py` and
  `__main__.py` are internal and intentionally have no stubs; do not add any.
- Prefer `%`-style or `%r`-style lazy formatting in `logging` calls
  (`logger.debug("Sending %r", buf)`), never string concatenation.
- Module-level `logger = logging.getLogger(__name__)` is the logging pattern.

## Testing instructions

- Tests live in `tests/` and use the standard library `unittest` module
  (`unittest.TestCase` subclasses with `test_*` methods), not bare
  `pytest`-style functions. Follow this pattern for new tests.
- Hardware (serial cable, Bluetooth LE Control Unit) is never available in
  CI or agent environments. Do not write tests that need a device. For
  `ControlUnit` logic, use a small fake `Connection` subclass that records
  `send()` calls and returns canned `recv()` responses.
- `serial.py`, `ble.py`, `fw.py` and `__main__.py` cannot be exercised
  against real devices here; changes to them must be kept minimal, reviewed
  carefully, and covered by `protocol.py`/`cu.py` unit tests where possible.
- Add or extend a test for every bug fix and every new public behaviour.

## Documentation

- `docs/index.rst` is the only documentation page; the API section is
  generated from docstrings with `autodoc`, so public docstrings are
  user-facing documentation.
- Sphinx runs with `-W` (warnings are errors). Malformed reST, broken
  cross-references or unknown roles will break `tox -e docs`.
- Code examples in `README.rst` and `docs/index.rst` show real Control Unit
  output; do not "fix" their values to look tidier.

## Commit and pull request instructions

- Commit messages are a single imperative sentence ending with a period,
  e.g. `Add type stubs for public API.` Reference issues as `Fix #18: ...`.
- Every user-visible change gets a bullet in `CHANGELOG.rst` under the
  topmost (unreleased or upcoming) version heading, using the existing
  format: `X.Y.Z YYYY-MM-DD` heading underlined with `-`, one blank line
  between bullets, code in double backticks.
- The package version is `__version__` in `src/carreralib/__init__.py`
  (read by setuptools via `[tool.setuptools.dynamic]`). Only bump it in a
  dedicated release commit, not as part of a feature or fix.
- Do not add runtime dependencies without a strong reason; the current set
  (`bleak`, `pyserial`, `windows-curses` on Windows) is intentionally small.
- Do not change the CI matrix, `dependabot.yml` or `.readthedocs.yaml`
  unless the task is specifically about them.
- Keep the diff focused on the task. Do not reformat, rename or
  reorganise unrelated code.

## Security and safety considerations

- Firmware updates (`fw.py`, `ControlUnit.fwu_*`) can brick real hardware.
  Treat any change to the firmware path as high risk, keep the existing
  warnings in the docs, and never add firmware files to the repository.
- `connection.open()` passes anything that does not look like a BLE
  address to `serial.serial_for_url()`, which also accepts network URLs
  (`socket://`, `rfc2217://`). Device strings are operator-supplied; do not
  introduce code paths that build them from untrusted input.
- This is not an official Carrera® product; do not imply official
  affiliation in docs, docstrings or code comments. Carrera® and Carrera
  AppConnect® are registered trademarks of Carrera Toys GmbH.
