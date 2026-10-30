# General code review

Review of the `carreralib` codebase (correctness, types, style, docs,
packaging, CI configuration, security). Last updated 2026-10-30 against
`master` at `894f88f` (v2.0.0 plus dependency bumps). Baseline: the full
`tox` run (`py`, `docs`, `doctest`, `pyright`, `ruff`, `ruff-format`) is
green; findings below are things the automated checks do not catch. See
`.github/copilot-instructions.md` for the conventions these findings refer to.

Findings from the previous review that are still open are kept and marked
**still open**; nothing from the previous review has been fixed yet.

## Bugs

### High: crash on unrecognized CU response in the demo CLI (still open)

[__main__.py](../src/carreralib/__main__.py#L111-L120) in `RMS.run()`:

```python
data = self.cu.poll()
...
elif isinstance(data, ControlUnit.Timer):
    self.handle_timer(data)
else:
    self.logger.warning("Unknown data from CU: " + data)
```

`ControlUnit.poll()` ([cu.py](../src/carreralib/cu.py#L130)) can legitimately
return `None` (`return None  # TODO: raise?`). When that happens here,
`"..." + data` raises `TypeError: can only concatenate str (not "NoneType")
to str`, crashing the whole `curses` UI. The `data == last` short-circuit
only hides this while `poll()` keeps returning the same value across
iterations. Fix: use lazy `%r` formatting instead of `+`, i.e.
`self.logger.warning("Unknown data from CU: %r", data)`, as every other
logging call in the package already does.

### Medium: BLE connect timeout leaks a running background thread (still open)

[ble.py](../src/carreralib/ble.py#L86-L90), `BLEConnection.__init__`:

```python
def __init__(self, address, timeout=1.0):
    self.__thread = t = BleakThread(address)
    self.__timeout = timeout
    t.start()
```

`BleakThread.start()` raises `TimeoutError` if the BLE connection isn't
established in time, but by then the thread is already running `main()`
(which has no timeout of its own around `BleakClient(...)` connect). If
`__init__` raises, the partially constructed `BLEConnection` is discarded,
but the background thread is not: it keeps trying to connect indefinitely
with no way to stop it. Worse, `Connection.__del__` then calls `close()`,
which calls `BleakThread.stop()` → `send()`, which reads `self.__loop`
([ble.py](../src/carreralib/ble.py#L41)) before `main()` has assigned it
([ble.py](../src/carreralib/ble.py#L59)), so an `AttributeError` is printed
as "Exception ignored in `__del__`". Consider passing a timeout into
`BleakClient`, stopping/joining the thread before re-raising, and
initialising `self.__loop = None` in `BleakThread.__init__`.

### Medium: `ChecksumError` is not a `ProtocolError`

[protocol.py](../src/carreralib/protocol.py#L4-L9) documents `ProtocolError`
as "the base class of all protocol exceptions", but `ChecksumError`
derives directly from `Exception`:

```python
class ProtocolError(Exception):
    """The base class of all protocol exceptions."""

class ChecksumError(Exception):
    """Raised when a checksum is wrong."""
```

Callers who follow the docstring and `except protocol.ProtocolError` will
not catch checksum failures. The stub
[protocol.pyi](../src/carreralib/protocol.pyi#L3-L4) mirrors the current
(wrong) hierarchy, so it has to be changed together with the
implementation. Changing the base class is backwards compatible for
existing `except ChecksumError` handlers.

### Low: `__del__` noise when a `SerialConnection` fails to open

[connection.py](../src/carreralib/connection.py#L20-L21) unconditionally
calls `self.close()` from `__del__`. If `serial_for_url()` raises inside
[`SerialConnection.__init__`](../src/carreralib/serial.py#L7-L11) (bad
device path, port in use), `self.__serial` was never assigned, so `close()`
raises `AttributeError`, which Python reports as "Exception ignored in:
`<function Connection.__del__>`" on stderr right after the real error. A
`getattr(self, "_SerialConnection__serial", None)` guard in `close()` (or
assigning the attribute to `None` first) removes the noise.

### Low: wrong exception types in the BLE thread

- `notify()` in [ble.py](../src/carreralib/ble.py#L64-L69) catches
  `asyncio.QueueFull`, but `self.__input` is a `queue.Queue`, whose
  `put_nowait()` raises `queue.Full`. The queue is unbounded so neither is
  ever raised today; the handler is dead code with the wrong type.
- `BleakThread.send()` ([ble.py](../src/carreralib/ble.py#L47-L48)) returns
  `f.result(timeout=timeout)`, which raises `concurrent.futures.TimeoutError`
  (the builtin `TimeoutError` on 3.11+), not `carreralib.connection.TimeoutError`
  like `recv()` does. Callers catching the package's `TimeoutError` on
  `send()` will miss it.

### Low: `RMS.reset()` can spin forever

[__main__.py](../src/carreralib/__main__.py#L78-L82) loops
`while not isinstance(status, ControlUnit.Status)` to drain pending timer
messages. If `poll()` returns `None` (see the first finding), the loop
never terminates. Breaking out on `None` (or raising once `poll()` does)
would make the two problems disappear together.

## Design / maintainability notes

- `Connection.send()`'s offset/size bounds-checking
  ([serial.py](../src/carreralib/serial.py#L28-L38),
  [ble.py](../src/carreralib/ble.py#L110-L120)) is duplicated verbatim in
  both connection implementations, and a third copy lives in
  [`protocol.chksum()`](../src/carreralib/protocol.py#L14-L24). Factoring
  it into a shared helper (e.g. on the `Connection` base class) would avoid
  drift between the copies. (still open)
- `BleakThread.send()` ([ble.py](../src/carreralib/ble.py#L39-L51)) silently
  swallows a `RuntimeError` when the loop isn't running instead of logging
  it; failures there would be invisible to callers/logs. (still open)
- `ControlUnit.request()` ([cu.py](../src/carreralib/cu.py#L136-L150))
  retries `recv()` in an unbounded `while True` loop whenever a reply
  doesn't start with the expected command byte. This relies entirely on
  the underlying connection's own timeout to bound the loop. Likely fine
  given the hardware use case, but worth a comment noting the assumption.
  (still open)
- `protocol.pack`/`protocol.unpack`
  ([protocol.py](../src/carreralib/protocol.py#L45),
  [protocol.py](../src/carreralib/protocol.py#L64)) still carry
  `# TODO: check all args used` / `# TODO: check all buf used`: extra
  arguments or trailing buffer bytes are silently accepted. (still open)
- The checksum-offset behaviour of the `C` format specifier (a leading
  count in `"2xC"` means "start the checksum at buffer offset 2") is only
  explained in [docs/index.rst](../docs/index.rst) and by the test cases in
  [test_protocol.py](../tests/test_protocol.py). A short comment near
  `_pack_C`/`_unpack_C` in `protocol.py` would help readers of the code.
  (still open)
- `pyright` reports two warnings and three informations in
  [__main__.py](../src/carreralib/__main__.py#L167-L171) (`leader` possibly
  unbound, `self.start` possibly `None`). They are false positives by
  construction (`pos == 1` always runs first and sets both), but they are
  the only diagnostics left in the whole project; initialising
  `leader = None` before the loop and asserting `self.start is not None`
  would make `tox -e pyright` completely silent.
- Docstring typos in public, autodoc-rendered text: "fule mode"
  ([cu.py](../src/carreralib/cu.py#L37)), "milleseconds"
  ([cu.py](../src/carreralib/cu.py#L59)), "too small a message"
  ([connection.py](../src/carreralib/connection.py#L6), missing "for"),
  "byte data rom an object" ([connection.py](../src/carreralib/connection.py#L34)).

## Configuration notes

- **Ruff ignore list is inert.** `[tool.ruff.lint]` in
  [pyproject.toml](../pyproject.toml) sets `ignore = ["B008", "UP024",
  "UP031"]` but no `select`/`extend-select`, so only ruff's default rules
  (`E4`, `E7`, `E9`, `F`) run and the `B`/`UP` ignores never apply. The
  ignores were clearly curated against `B` and `UP` (those three are
  exactly the rules that fire on this code base), so the intent was
  probably `extend-select = ["B", "UP"]`. Enabling them additionally
  surfaces `B905` (two `zip()` calls without `strict=` in `__main__.py`)
  and `B904` (a `raise` inside `except` without `from` in `ble.py`); either
  add those to the ignore list or fix them when turning the rules on.
- **Dependabot targets a branch that does not exist.**
  [dependabot.yml](dependabot.yml) sets `target-branch: "develop"`, but the
  `tkem-ai/carreralib` repository only has `master`. Dependabot cannot open
  pull requests against a missing branch and will log an error instead.
  Either create `develop` or drop the `target-branch` key so updates land
  on the default branch. (If this repository mirrors `tkem/carreralib`,
  which `pyproject.toml` still points to, verify the branch exists there.)
- `[build-system]` requires `setuptools-scm >= 8.2`, but the version comes
  from `__version__` via `[tool.setuptools.dynamic]`, not from git tags.
  `setuptools-scm` then only contributes its git-based file finder for
  sdists, which overlaps with the explicit `MANIFEST.in`. Fine if
  intentional; otherwise the build requirement can be dropped.

## Documentation notes

- [docs/conf.py](../docs/conf.py#L17) defines `intersphinx_mapping` but
  `sphinx.ext.intersphinx` is not in `extensions`, so the mapping is dead
  and `:class:` references to stdlib types (e.g.
  `collections.namedtuple` in the `Status`/`Timer` docstrings) render as
  plain text instead of links. Add the extension (and expect
  `tox -e docs` to need network access to fetch the inventory, or vendor
  it).
- [docs/index.rst](../docs/index.rst#L6-L7) still introduces the library as
  connecting "via a serial (cable) connection" only, and the `ControlUnit`
  API note says `device` "should name a serial port". The README and the
  Demo RMS section already cover Bluetooth LE / AppConnect® MAC addresses;
  the API section should say the same.
- The bottom border of the `pack()` format table in
  [docs/index.rst](../docs/index.rst) ends in a stray `+-+`. docutils
  tolerates it (the `-W` build passes), but it is a cosmetic slip worth
  fixing while editing that table.

## Type stubs (`.pyi`)

All four public stub files (`connection.pyi`, `cu.pyi`, `protocol.pyi`,
`__init__.pyi`) were re-checked line by line against their implementations:
signatures, return types, defaults and re-exports match, and `pyright`,
`ruff check` and `ruff format` are clean against them. The only
discrepancy is the inherited one noted above: `ChecksumError` should
subclass `ProtocolError` in both the stub and the implementation.

## Security notes (informational, not vulnerabilities in this context)

- [connection.py](../src/carreralib/connection.py#L42-L51)'s `open()` picks
  a transport based on the shape of the `device` string, ultimately calling
  `serial.serial_for_url()` for anything that isn't BLE-address-shaped.
  `serial_for_url` supports handlers like `socket://host:port` and
  `rfc2217://`, so an attacker-controlled `device` string could make the
  process open arbitrary outbound TCP connections. Not an issue for the
  documented use case (operator-supplied local device strings), but
  relevant if `device` were ever sourced from untrusted/remote input.
- [fw.py](../src/carreralib/fw.py#L43-L59) streams an operator-supplied
  file straight to the Control Unit's firmware update command with no
  validation beyond stripping quotes and blank lines. The docs already
  warn that this can destroy hardware; the code should keep failing loudly
  rather than ever retrying or "recovering" silently.

## Test coverage gaps

`tox -e py` reports 27% total coverage: `protocol.py` 78%, `connection.py`
50%, `cu.py` 41%, and `serial.py`/`ble.py`/`fw.py`/`__main__.py` at 0%
(expected: hardware/CLI, cannot run here). Only `tests/test_protocol.py`
exists (three test methods). Candidates for additional *pure-logic* unit
tests that need no hardware, using a small fake `Connection` subclass:

- `ControlUnit.poll()` response parsing (`Status` vs `Timer` vs `None`,
  including the two-branch checksum fallback for `?:`-prefixed replies).
- `ControlUnit.request()` skipping unexpected messages and stopping on an
  empty response.
- `ControlUnit.setword`/`setlap`/`setpos` argument validation
  (`ValueError` on out-of-range `word`/`address`/`value`/`repeat`/`position`).
- `ControlUnit.version()`'s `None` fallback when `request()` returns an
  empty response.
- `ControlUnit.fwu_write()` block splitting when `max_fwu_block_size` is set.
- `protocol.pack`/`unpack` error paths (`ValueError` for bad format
  characters and out-of-range values, `ChecksumError`, `chksum` bounds
  checks); these are the uncovered lines in `protocol.py`.
- `connection.open()` dispatch on BLE-address-shaped vs serial device
  strings (can be tested by monkeypatching the lazily imported classes).

## Summary

The core protocol/type-stub layer (`protocol.py`, `connection.py`, `cu.py`
and their `.pyi` stubs) is solid, and the full `tox` suite is green on
Python 3.13. The open items are, in priority order: the `__main__.py`
`None` crash (and the related `RMS.reset()` spin), the BLE thread leak on
connect timeout, `ChecksumError` not deriving from `ProtocolError`, the
inert ruff ignore list, and the Dependabot `develop` target branch. The
remaining notes are small cleanups and documentation drift.
