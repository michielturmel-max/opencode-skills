---
name: zero-dep-python-tool
description: Use when building a dependency-free Python CLI or service - stdlib only, SQLite storage, argparse entry points, unittest/pytest-free tests, dist tarball, systemd service and timer.
---

# Zero-dependency Python tool

House style from `pricewatch` (CLI + plugin) and `wiskunde3mavo` (single-file
service). Stdlib only: no pip, no virtualenv, no pyproject.

## Layout

```
bin/<name>              executable launcher (shebang, sys.path bootstrap)
src/<name>/             package modules (src-layout, never installed)
test/                   tests + fixtures/ + cli.test.sh + run-all.sh
systemd/<name>.service  oneshot unit;  <name>.timer  user timer
dist.sh                 builds dist/<name>-<version>.tar.gz
install.sh              installs to ~/.config/omarchy/plugins/<name>
AGENTS.md               build/test commands + domain rules, ~50 lines
README.md               quick start, command table, paths, "Grenzen"
manifest.json           {"id","version"} - single version source for dist
```

Tiny tool = one `main.py` + `tests/` beside it. Launcher finds the package
without installation:

```python
import os, sys
HERE = os.path.dirname(os.path.realpath(__file__))
for candidate in (os.path.join(os.path.dirname(HERE), "src"), os.path.join(HERE, "..", "src")):
    if os.path.isdir(os.path.join(candidate, "pricewatch")):
        sys.path.insert(0, os.path.abspath(candidate)); break
from pricewatch.cli import main  # noqa: E402
raise SystemExit(main())
```

## Style

Every module: `from __future__ import annotations`, type hints on all
functions, stdlib imports. Identifiers English, human-facing strings Dutch
(or the tool's language). Docstrings/comments explain the *why* only, never
restate the line above. Money as `float`/`REAL` is fine at consumer price
magnitudes. Config via env vars with defaults at the top of the file, each
documented in a README table, e.g.
`OPENCODE_CLI_BIN = os.getenv("OPENCODE_CLI_BIN", "opencode").strip() or "opencode"`.

## CLI (argparse)

One `build_parser()`; subparsers `dest="command", required=True`; each ends
`p.set_defaults(func=handler)`; handlers are `(store, args) -> int`. Global
`--db`/`--version` on the root, `--json` per command. Validate everything
*before* writing: a rejected command leaves no half row.

```python
def die(message: str, code: int = 1) -> NoReturn:
    print(f"pricewatch: {message}", file=sys.stderr)
    raise SystemExit(code)
```

`emit(payload, as_json)` prints `json.dumps(payload, ensure_ascii=False,
default=str)`. `main()` parses, opens the store and calls
`args.func(store, args)` in `try/except KeyError: die(...)` /
`except BrokenPipeError: return 0` / `finally: store.close()`.

## SQLite storage

One file per tool. Module-level `SCHEMA` of `CREATE TABLE IF NOT EXISTS`,
`executescript` on every open. Money REAL, timestamps INTEGER unix seconds,
ids TEXT (slugified name, 4-hex uuid suffix on collision).

```python
SCHEMA_VERSION = 2
self.db = sqlite3.connect(self.path, timeout=15.0)
self.db.row_factory = sqlite3.Row
self.db.execute("PRAGMA journal_mode=WAL")   # panel reads while timer writes
self.db.execute("PRAGMA foreign_keys=ON")
self.db.executescript(SCHEMA)
```

Version in `PRAGMA user_version`; upgrades additive-only (`ALTER TABLE ...
ADD COLUMN`), never drop tables - history cannot be re-fetched. `commit()`
after every write; `resolve()` accepts id *or* name; `Store` is a context
manager; unknown update fields raise `ValueError`. A failed source never
erases the previous good value.

## State and cache paths

`default_home()` = `$PRICEWATCH_HOME` override (tests), else
`$XDG_DATA_HOME` or `~/.local/share`, then `/pricewatch`. Side-state lives
next to the db: `cooldown.json` (`indent=1, sort_keys=True`, load errors
ignored) and a `check.lock` held with a non-blocking `fcntl.flock(LOCK_EX |
LOCK_NB)` in a `@contextmanager` that yields True/False; the second run says
so and stops.

## HTTP (urllib only)

`urllib.request` + gzip/deflate decode, browser-ish User-Agent, per-host
throttle, retries with `1.5 * (attempt + 1)` backoff. Exceptions the caller
records instead of crashing: `FetchError` (retry later), `Blocked(FetchError)`
(bot wall/403/429), `NotFound(FetchError)` (404/410).

## Tests

Two offline layers, no pytest:

1. `unittest`: files self-bootstrap the path
   (`sys.path.insert(0, os.path.join(os.path.dirname(HERE), "src"))`),
   `TestCase` classes, `tempfile.TemporaryDirectory` in setUp/tearDown,
   fixtures = saved real HTML pages in `test/fixtures/`.
2. End-to-end CLI in bash (`test/cli.test.sh`): `set -uo pipefail`,
   `mktemp -d` + `export PRICEWATCH_HOME="$WORK/state"`, tiny
   `ok`/`no`/`assert_eq`/`assert_contains`/`assert_fails` helpers, JSON via
   `python3 -c "import json,sys; ..."`. Network stubbed by monkeypatching
   `urllib.request.urlopen` / `subprocess.run` with logging fakes; or plain
   assert-scripts run as `python tests/test_grok.py`.

```bash
python3 -m unittest discover -s test -v   # python suite
bash test/cli.test.sh                     # CLI suite
./test/run-all.sh                         # both; skips missing tools, exits on first failure
PRICEWATCH_HOME=/tmp/pw ./bin/pricewatch check --timeout 25  # live, isolated
```

## dist tarball

`dist.sh`: version from `manifest.json`, stage in `mktemp -d` with
`trap 'rm -rf "$STAGE"' EXIT`, copy only runtime files (`manifest.json
install.sh README.md LICENSE bin src systemd`), strip `__pycache__`/`*.pyc`,
`chmod +x`, then `tar -czf dist/$NAME.tar.gz -C "$STAGE" "$NAME"`,
`sha256sum` (sed-trimmed) and the unpack command. Tests and fixtures stay home.

## systemd (user units)

Service is oneshot with `%h` paths, journal output and an explicit timeout:
`Type=oneshot`, `ExecStart=%h/.config/omarchy/plugins/<n>/bin/<n> check
--notify --timeout 25`, `TimeoutStartSec=180`. Timer: `OnBootSec=4min`,
`OnUnitActiveSec=30min`, `AccuracySec=1min`, `Persistent=false` (a closed
laptop need not catch up), never on the hour. Install to
`~/.config/systemd/user` via `install.sh timer` (`systemctl --user
daemon-reload && enable --now`). Never install, start units or notify without
the user asking; `./install.sh` is their last step.

## AGENTS.md / README

AGENTS.md: build/test commands, style, domain rules (a parser without a
fixture is not done), boundaries. README: quick start, command table,
env-var table, "Waar staat het" paths, "Ontwikkelen" with exact test/build
commands, "Grenzen" for what the tool does not promise.
