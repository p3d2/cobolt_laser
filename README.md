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

## Which file to use

| File | Status |
|------|--------|
| **`cobolt_laser_notebook.ipynb`** | **Current.** Fixed rewrite: one connection for the whole notebook, absolute scheduling with deadlines set after the setup round-trips, scheduling error and command latency logged separately, missed-deadline count, priority/timer boost that is restored and *reported*, timestamped CSVs with a metadata header that never overwrite. |
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
