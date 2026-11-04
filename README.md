# cobolt

Control scripts for the Cobolt 808 nm laser (serial `19122`, model
`0808-06-01-0120-100`, 100 mW head) — blink protocols over the serial link plus
timing characterisation. Runs on **Windows** (`COM8`); the priority/timer helpers
are no-ops elsewhere.

**Run the hardware notebook on Windows**, where the laser is (`COM8`). WSL2 cannot
see Windows COM ports without `usbipd-win` passing the USB device through.
`pycobolt` is GitHub-only (not on PyPI) and needs Python ≥ 3.10 — the old
notebook's 3.9.7 kernel is no longer supported.

`pyproject.toml`, `uv.lock` and `.python-version` (3.12) are committed, so both
sides install the identical set with one command:

```bash
uv sync                                  # creates .venv from uv.lock
uv run jupyter lab cobolt_laser_notebook.ipynb
```

Windows needs [uv](https://docs.astral.sh/uv/) (`winget install --id=astral-sh.uv -e`)
and **Git for Windows** (uv shells out to git for the `pycobolt` dependency).
PowerShell blocks `.venv\Scripts\activate.ps1` by default — either
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, or skip activation and use
`uv run …`.

Work from a **native** path on each OS (`C:\Users\silvap1\projects\cobolt` on
Windows, `~/projects/cobolt` in WSL) and sync with git. Do not run Windows Python
from `\\wsl.localhost\…`: 9P is slow and it leaves a Windows `.venv` inside the
Linux tree.

No uv? `pip install -r requirements.txt && jupyter lab cobolt_laser_notebook.ipynb`.

## Notebook workflow

Run §1–§4 once (defaults → helpers → connect → routine), then re-run the cells in
**§5** as often as you like. Each run is one call:

```python
run(power_mw=20, on_s=1.0, off_s=1.0, n_cycles=10, tag="20mW_1s")
run(on_s=0.2)                      # everything else comes from the §1 defaults
```

`run()` clamps the power to `MAX_POWER_MW`, refuses to fire if the laser is
disconnected or the interlock is open, shows a **live per-cycle progress bar**,
prints mean/p95/max for scheduling error and command latency, writes its own
timestamped CSV to `logs/`, plots the run, and appends a row to `RUNS`.
Afterwards: `summary_table()` compares every run so far, `replot(2)` re-draws an
earlier one.

The progress bar is timing-aware: it is rendered *before* the schedule starts, and
it switches itself off (with a printed reason) when the period is under 20 ms or
when the first render is too slow for the OFF dwell — so it can never silently
degrade the numbers you are measuring.

**Troubleshooting.** If `import serial` or `import pycobolt` fails while
`uv pip list` still shows them, the venv lost package files but kept their
metadata (typical after Windows-side uv touches `.venv` over `\\wsl.localhost\…`).
Repair with `rm -rf .venv && uv sync`.

## Which file to use

| File | Status |
|------|--------|
| **`cobolt_laser_notebook.ipynb`** | **Current.** Fixed rewrite: one connection for the whole notebook, per-run `run(...)` calls with a live progress bar and a comparison table, absolute scheduling with deadlines set after the setup round-trips, scheduling error and command latency logged separately, missed-deadline count, priority/timer boost that is restored and *reported*, timestamped CSVs with a metadata header that never overwrite. |
| `cobolt_laser_blink_notebook.ipynb` | Superseded, kept for reference. Does not run as saved: cell 10 calls `laser_routine`, cell 8 defines `laser_routine_tight`; stored outputs come from an older version. |
| `Routines.ipynb` | First `pycobolt` attempt (long irradiation + cycles, `np.diff` histogram). |
| `Cycles.ipynb`, `cobolt_blink_tracking.ipynb` | Raw `pyserial` with short commands (`!p`/`!e`/`!d`) — rejected by this firmware (`Syntax error: illegal command`). |
| `Untitled.ipynb` | Unrelated Dash/NetworkX prototype. |

## Data

- `logs/` — output of the current notebook: `#`-comment metadata header, then
  `cycle,phase,planned_s,late_s,cmd_s` (seconds; read with
  `pandas.read_csv(path, comment="#")`).
- `*.csv` in the root (`test.csv`, `m15*.csv`, `multiply_*.csv`) — **legacy, not
  comparable.** They use an `actual_s` column from an earlier metric; `test.csv`
  even holds negative values that the current definition cannot produce. Treat
  them as a different measurement or re-run the sweeps.

## Caveats

All logged numbers are **software** timings — when Python sent the serial command
and how long it took to return. They are *not* optical pulse durations. Serial
toggling costs ~9–17 ms per edge, so ~20 % error on a 50 ms pulse; for sub-ms
precision drive the laser's digital/analogue **modulation input** from a function
generator or DAQ instead, and validate with a photodiode.
