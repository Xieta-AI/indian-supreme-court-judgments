# Local download — Supreme Court judgments for a date range

How to scrape SC judgments for a given date range to your own machine.
No AWS, no S3, no credentials. Just files on disk.

## One-time setup

From the repo root:

```bash
uv sync                 # install Python 3.13 deps into .venv/
source .venv/bin/activate   # on Git Bash / macOS / Linux
# or: .venv\Scripts\activate  on Windows cmd/PowerShell
```

## The command

```bash
python download.py \
  --start_date 2024-01-01 \
  --end_date   2024-01-31 \
  --max_workers 1
```

- `--start_date` / `--end_date` are inclusive, in `YYYY-MM-DD`.
- `--max_workers 1` forces serial scraping (be gentle to `ecourts.gov.in`,
  per `AGENTS.md` / `README.md`). Pass it explicitly — the upstream default
  is still 5.
- **Do not pass `--sync-s3` or `--sync-s3-fill`.** Those are the only flags
  that write to S3. Leave them off and the run is 100% local.

## Where output lands

| Path                          | What's in it                                                   |
| ----------------------------- | -------------------------------------------------------------- |
| `./sc_data/<year>/`           | Per-year folders with metadata JSON and downloaded PDFs        |
| `./sc_data/<year>/*.tar`      | Packaged tar archives (unless you pass `--no-package`)         |
| `./sc_track.json`             | Tracking state — last processed date, used for resume          |
| `./captcha-tmp/`              | Temp captcha images (ignored by git)                           |
| `./captcha-failures/`         | Captcha images the solver couldn't crack (ignored by git)      |
| `./temp-files/`               | Misc scratch (ignored by git)                                  |

All of the above are in `.gitignore`, so none of it will accidentally get
committed to your fork.

## What to expect during a run

- Requests hit `https://scr.sci.gov.in` one at a time (serial, because
  `--max_workers=1`). It's slow by design.
- Each request goes through a captcha solver (`src/captcha_solver/`).
  Failures land in `./captcha-failures/` for inspection; the scraper
  retries automatically.
- Log lines are color-coded (green = INFO, yellow = WARN, red = ERROR).
- `Ctrl+C` is safe — the scraper writes `sc_track.json` after each chunk,
  so you can re-run the same command to resume.

## Resume after interruption

Just re-run. If you omit `--start_date`, the scraper reads the last date
from `sc_track.json` and picks up from there:

```bash
python download.py --max_workers 1
```

If you want to force a specific start regardless of state:

```bash
python download.py \
  --start_date 2024-02-01 \
  --end_date   2024-02-28 \
  --max_workers 1
```

## Flag reference (local-only subset)

| Flag            | Default       | Use                                                        |
| --------------- | ------------- | ---------------------------------------------------------- |
| `--start_date`  | (from state)  | Inclusive start, `YYYY-MM-DD`                              |
| `--end_date`    | today         | Inclusive end, `YYYY-MM-DD`                                |
| `--day_step`    | `1`           | Days per request chunk — leave at 1 unless you know why    |
| `--max_workers` | `5` upstream  | **Pass `1`** for kindness + predictable serial behavior    |
| `--no-package`  | off           | Skip tar packaging; leave individual JSON/PDFs per record  |

**Do not use:**

- `--sync-s3` — downloads new data **and uploads to S3** (vanga's bucket).
- `--sync-s3-fill` — same thing, historical gap-filling.

Both are upstream's production flags for maintaining the public dataset.
They are irrelevant for local use and will try to write to a bucket you
don't own.

## Confirming the run stayed local

After a run, spot-check:

```bash
ls sc_data/                      # per-year folders with your data
cat sc_track.json                # last_date should match your --end_date
```

If you want belt-and-suspenders: watch network activity during the run
(e.g. Wireshark / Resource Monitor) — traffic should only go to
`scr.sci.gov.in`. No `s3.amazonaws.com` or `s3.*.amazonaws.com` should
appear.

## Quick recipes

Tiny smoke test (one day):

```bash
python download.py --start_date 2024-01-02 --end_date 2024-01-02 --max_workers 1
```

One full year:

```bash
python download.py --start_date 2023-01-01 --end_date 2023-12-31 --max_workers 1
```

Everything from 1950 to today (long — expect days to weeks at 1 worker):

```bash
python download.py --start_date 1950-01-01 --max_workers 1
```

## Troubleshooting

- **Captcha keeps failing** → check `./captcha-failures/` images. The
  solver is in `src/captcha_solver/`; worst case, delete the failure
  folder and re-run.
- **`sc_track.json` shows a wrong date** → delete it to reset all state,
  then re-run with explicit `--start_date`.
- **Run hangs** → `Ctrl+C`, then re-run (state is saved per chunk).
- **Got `AccessDenied` or AWS errors** → you passed `--sync-s3` by
  mistake. Remove it and re-run.

## See also

- `README.md` (root) — project overview and S3 dataset layout
- `AGENTS.md` (root) — project rules incl. the ecourts kindness rule
- `gs_code/git_guide.md` — fork / git workflow for this checkout
