# Git guide — this fork

Personal cheat sheet for working in this fork of
`vanga/indian-supreme-court-judgments`.

## Remote layout

| Remote     | Fetch URL                                                         | Push URL                        | Use                  |
| ---------- | ----------------------------------------------------------------- | ------------------------------- | -------------------- |
| `origin`   | `https://github.com/Xieta-AI/indian-supreme-court-judgments.git`  | (same)                          | Your fork — push here |
| `upstream` | `https://github.com/vanga/indian-supreme-court-judgments.git`     | `DISABLE_PUSH_TO_UPSTREAM`      | Public project — fetch only, pushes **hard-disabled** |

The upstream push URL was deliberately set to an invalid value so any
accidental `git push upstream` fails fast. Do not "fix" it.

Verify any time:

```bash
git remote -v
git branch -vv         # shows which remote/branch main tracks
```

## Refresh from upstream (pull in new public commits)

```bash
git fetch upstream
git checkout main
git merge upstream/main          # merge commit preserves both histories
git push origin main             # publish merged result to your fork
```

Preview incoming changes before merging:

```bash
git fetch upstream
git log --oneline main..upstream/main   # commits you're about to pull in
git diff main...upstream/main           # the actual diff
```

Optional alias:

```bash
git config alias.refresh '!git fetch upstream && git merge upstream/main'
# then:  git refresh
```

## Day-to-day

```bash
git status -sb                   # quick status + branch
git add <files>                  # stage specific files, avoid `git add -A`
git commit -m "..."
git push                         # pushes to origin (your fork) by default
```

For risky / experimental work:

```bash
git checkout -b feature/xyz
# ...commit...
git checkout main
git merge feature/xyz
git branch -d feature/xyz
```

## Local-only files (secrets, machine config)

Pick by scope:

- **Everyone forking this repo should ignore it** → add to `.gitignore`
  (committed).
- **Only your machine** → add to `.git/info/exclude` (not tracked).
- **File is tracked upstream but you want local edits uncommitted** →
  `git update-index --skip-worktree <file>`
  (undo: `git update-index --no-skip-worktree <file>`).

Already ignored by this repo: `.env`, `.venv/`, `sc_data/`, `packages/`,
`local_sc_judgments_data/`, `captcha-tmp/`, `captcha-failures/`,
`temp-files/`, runtime state JSONs. See `.gitignore`.

## Conflict during refresh merge

Unlikely since your changes are additive, but if it happens:

```bash
git status                       # lists conflicted files
# edit each file, remove <<<<<<< / ======= / >>>>>>> markers
git add <resolved-files>
git commit                       # completes the merge
git push origin main
```

To abort instead:

```bash
git merge --abort
```

## Undo / recover

| What                                                  | Command                                     |
| ----------------------------------------------------- | ------------------------------------------- |
| Unstage a file                                        | `git restore --staged <file>`               |
| Discard local edits in a file                         | `git restore <file>`                        |
| Amend last commit (only if NOT pushed)                | `git commit --amend`                        |
| Move main back one commit, keep changes staged        | `git reset --soft HEAD~1`                   |
| See what was there before any ref move                | `git reflog`                                |

Avoid `git reset --hard` and `git push --force` on `main` — they destroy
work. If you truly need them, double-check the target first.

## Guardrails

- Never push to any remote that isn't yours. `upstream` is hard-disabled
  above; don't re-enable it or add other third-party push targets. If you
  ever want to contribute back to vanga's project, push to your own fork
  and open the PR on the GitHub web UI — no pushing to their repo needed.
- Don't force-push `main`. You chose merge (not rebase) so there's no need.
- Don't skip hooks (`--no-verify`) — fix the hook failure instead.
- `prune/` scripts and `dataset_sizes.csv` are special (see `AGENTS.md`) —
  don't edit `dataset_sizes.csv` by hand or invoke `prune/` from normal code.

## S3 upload posture (this fork)

This fork is configured to stay local. Three things to know:

1. **Default flow is local.** `python download.py` uses
   `local_only=True` on the `S3ArchiveManager` (`download.py:317`) — files
   land under `./sc_data/` and nothing is uploaded.

2. **`--sync-s3` and `--sync-s3-fill` upload to vanga's public bucket**
   (`s3://indian-supreme-court-judgments`). Don't pass these flags unless
   you've arranged to write to a bucket you actually own. Running them
   against vanga's bucket from this fork would either fail on AWS auth
   or — worse, if you have creds — publish to their dataset.

3. **GitHub Actions on this fork are disabled** via the Actions settings
   page (`https://github.com/Xieta-AI/indian-supreme-court-judgments/settings/actions`).
   The upstream workflows `.github/workflows/update-s3-data.yml` and
   `.github/workflows/fill-s3.yml` are still present in the repo (so
   upstream refreshes don't conflict), but they can't run. If you ever
   re-enable Actions, neuter or delete those two files first.

## Scraper politeness (this fork)

`--max_workers` default is **1** (changed from upstream's 5) in:
- `download.py` (CLI arg + `run()` signature)
- `sync_s3_fill.py` (`sync_s3_fill_gaps()` signature)

Keep it at 1 unless you have a specific reason. Per `AGENTS.md` and
`README.md`, being gentle to `ecourts.gov.in` is an explicit project value.

## Useful reads

- Workflow plan: `C:\Users\direc\.claude\plans\i-downloaded-this-repo-velvety-prism.md`
- Project rules: `AGENTS.md` (root)
- Dataset layout / conventions: `README.md` (root)
