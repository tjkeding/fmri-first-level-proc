# Publish Report — fmri_first_level_proc v2.3.1

**Date:** 2026-03-13
**Version:** 2.3.1 (patch)
**Commit:** 4f7caf1
**Tag:** v2.3.1
**Branch:** main
**Remote:** origin/main

## Change Summary

Single patch fix in `rest_conn_first_level.py`:

- **File:** `fmri_first_level_proc/rest_conn_first_level.py`, line 197
- **Change:** `-polort 2` to `-polort -1` in `gen_residual_ts()` 3dTproject call
- **Rationale:** When bandpass filtering is active, polynomial detrending is redundant; `-polort -1` disables it per field consensus.

## Version Bump

All 8 version-bearing files updated from 2.3.0 to 2.3.1:

| File | Field |
|------|-------|
| `pyproject.toml` | `version = "2.3.1"` |
| `fmri_first_level_proc/__init__.py` | `__version__ = "2.3.1"` |
| `fmri_first_level_proc/first_level_utils.py` | header `# Version: 2.3.1` |
| `fmri_first_level_proc/first_level_config.py` | header `# Version: 2.3.1` |
| `fmri_first_level_proc/run_first_level.py` | header `# Version: 2.3.1` |
| `fmri_first_level_proc/task_act_first_level.py` | header `# Version: 2.3.1` |
| `fmri_first_level_proc/task_conn_first_level.py` | header `# Version: 2.3.1` |
| `fmri_first_level_proc/rest_conn_first_level.py` | header `# Version: 2.3.1` |

## Pre-publish Checks

- **Commit message:** Clean — no attribution trailers or tool-use references.
- **Tag message:** Clean — `v2.3.1: set polort -1 when bandpass filtering is active in rest_conn`.
- **.gitignore:** Whitelist intact; no sensitive files staged.
- **Tests:** Not re-run this session (manual single-flag edit); last known state 305/305 pass (v2.3.0).

## Git Operations

1. `git add` — 8 files staged
2. `git commit` — 4f7caf1
3. `git tag -a v2.3.1`
4. `git push origin main --tags` — pushed commit and tag
