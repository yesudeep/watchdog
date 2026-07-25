# Code review style guide

watchdog is a filesystem-monitoring library with a very large installed base. It runs on
Linux, macOS, BSD and Windows, it is threaded, and it ships a C extension module. Most
real defects here fall into a handful of recurring categories, listed below in the order
we care about them.

Prefer a small number of high-value comments to exhaustive coverage.

## What not to comment on

These are already enforced mechanically; flagging them wastes everyone's time.

- **Formatting.** `ruff format` is authoritative (double quotes, 4-space indent,
  120-column lines). Never suggest reformatting.
- **Lint rules already in CI.** `ruff` runs with `extend-select = ["ALL"]` and an explicit
  ignore list in `pyproject.toml`. If a rule is in that ignore list, the omission is
  deliberate.
- **Missing type annotations on new public functions.** mypy runs with
  `disallow_untyped_defs`, `disallow_incomplete_defs` and `disallow_untyped_calls`; CI
  catches these.
- **Docstring style.** The `D` rules are disabled on purpose.
- Stylistic rewrites that do not change behaviour.

## Python version floor

**The minimum supported version is Python 3.9** (`python_requires=">=3.9"`,
`target-version = "py39"`). This is the most frequently missed constraint in this
repository. Flag any use of:

- `match` statements (3.10+)
- `X | Y` union syntax evaluated at runtime — it is fine in annotations because modules
  use `from __future__ import annotations`, but not in `isinstance()`, `cast()`,
  `TypeAlias` values, or any other runtime position
- `itertools.pairwise` (3.10+), `typing.Self` (3.11+), `asyncio.TaskGroup` (3.11+),
  `enum.StrEnum` (3.11+), `datetime.UTC` (3.11+)
- `dict`/`list` generics in runtime positions without the `__future__` import
- Keyword-only `dataclasses` features from 3.10+ (`slots=`, `kw_only=`)

## Cross-platform parity

Event emitters are platform-specific: `inotify` (Linux), `fsevents` (macOS), `kqueue`
(BSD), `read_directory_changes`/`winapi` (Windows), and `polling` (portable fallback).

- A behavioural change to one backend usually needs a matching change to the others, or
  an explicit note in the pull request explaining why it is platform-specific. Flag
  changes that silently make backends diverge.
- Changes to shared code in `observers/api.py`, `events.py` or `utils/` affect every
  platform. Reason about all of them, not just the contributor's.
- Platform detection goes through `watchdog.utils.platform`, not ad-hoc `sys.platform`
  checks.
- Windows path semantics (case-insensitivity, `\\?\` prefixes, path length limits) and
  macOS normalisation (NFD vs NFC in filenames) are recurring sources of bugs.

## Path handling: str and bytes

Watches may be scheduled with either `str` or `bytes` paths, and **events must preserve
the type the caller used**. See `observers/fsevents.py` and `observers/inotify.py` for the
established pattern.

- Flag any code that assumes `str`, or that decodes without re-encoding on the way out.
- Use `os.fsdecode`/`os.fsencode`, never `.decode()`/`.encode()` with a hardcoded codec —
  filenames are not guaranteed to be valid UTF-8 on POSIX.
- Do not use `str()` on a path that may be `bytes`; it produces `b'...'`.

## Threading and lifecycle

Observers, emitters and the `BaseThread` helper in `utils/__init__.py` all run as daemon
threads.

- Flag shared mutable state accessed without the relevant lock. `BaseObserver` guards its
  handler/watch bookkeeping with `self._lock` (an `RLock`).
- Flag `start()`/`stop()`/`join()` sequences that can deadlock, double-start, or leave a
  thread running after `stop()`.
- Watch for races between a watch being unscheduled and an in-flight event referencing it.
- Callbacks fire on emitter threads, not the caller's thread. Anything touching state
  owned by another thread needs synchronisation.
- Daemon threads do not run cleanup at interpreter exit; do not rely on `finally` for
  releasing OS resources at shutdown.

## Resource ownership

Each backend holds OS resources that leak silently if mishandled, and leaks here are
long-lived because the library is used by daemons and dev servers that run for days.

- inotify: watch descriptors and the inotify fd
- FSEvents: `FSEventStreamRef`, `CFRunLoopRef` and their retain/release pairing
- Windows: directory `HANDLE`s
- kqueue: kqueue fd and the open fd held per watched file — this backend hits per-process
  fd limits on large trees, so flag anything that increases fds per watch

Flag acquisition without a matching release on every path, including the error paths.

## The C extension

`src/watchdog_fsevents.c` is the only memory-unsafe code we ship, so review it strictly.

- **Reference counting**: every `Py_INCREF` needs its `Py_DECREF`; check for leaks on
  early-return and error paths. Confirm whether each C API call returns a new or borrowed
  reference before assuming.
- **Error handling**: a NULL return from a C API call must be checked and propagated. Do
  not return NULL without an exception set, and do not return non-NULL with one set.
- **Allocation**: check every allocation for NULL before writing through it. Prefer
  `PyMem_Malloc`/`PyMem_Free` and keep allocator pairs matched.
- **The GIL**: callbacks arrive on CoreFoundation run-loop threads. Any Python C API call
  from those threads needs the GIL held (`PyGILState_Ensure`/`Release`). Flag Python API
  use where the GIL state is unclear.
- Note that `src/pythoncapi_compat.h` is vendored and must not be edited.

## Security

- `watchdog/tricks/` executes user-supplied commands via `subprocess`. Data derived from
  the filesystem — filenames above all — is **untrusted input**: it may contain shell
  metacharacters, newlines, or anything except `/` and NUL. Flag any path where such data
  reaches a shell, and any new `shell=True`.
- `watchmedo` loads YAML configuration. It must stay `yaml.safe_load`.
- `watchmedo` can import classes named in a config file. Treat any widening of that
  mechanism as a security-relevant change.
- Flag symlink-following and TOCTOU patterns in path traversal — watched directories are
  frequently not under the operator's control.

## Public API compatibility

This library has a very large number of downstream dependants, so accidental breakage is
expensive.

- Flag removed or renamed public names, changed method signatures, changed event class
  hierarchies in `events.py`, and changes to the `observers/api.py` contract.
- Flag changes to event ordering or coalescing semantics; downstream code depends on them
  in ways that are hard to see from inside a diff.
- New public API should be additive and keyword-argument based where practical.
- `utils/backwards_compat.py` exists to preserve old import paths. Do not break them.

## Tests

- Tests use pytest, live in `tests/`, and bare `assert` is correct there.
- Filesystem-event tests are inherently timing-sensitive. Flag new `time.sleep()`-based
  synchronisation; prefer the existing helpers in `tests/utils.py` and the polling
  patterns already used in the suite. Note that `flaky` is available for genuinely
  unavoidable cases.
- A bug fix should come with a regression test that fails without the fix.
- Tests that only pass on the contributor's platform should be marked with the
  appropriate skip conditions.
