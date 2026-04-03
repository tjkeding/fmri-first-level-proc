---
name: v2.4.0 brainstorm — motion units, min-outlier EPI, raw ptseries
description: Brainstorm report for three stakeholder-driven changes targeting v2.4.0 release
type: project
---

# Brainstorm Report — v2.4.0 Stakeholder Changes (2026-04-02)

## Overview

Three changes requested based on stakeholder feedback:
1. Standardize rotation parameters to degrees throughout the pipeline
2. Output a minimum-outlier representative EPI frame per scan for alignment QC
3. Extract pre-regression parcellated time series (raw ptseries)

---

## Change 1: Motion Rotation Units — Degrees Throughout

### Decision
- fmri-first-level-proc expects, processes, and outputs **degrees** exclusively
- No auto-detection, no conversion within the pipeline
- Column order contract: `[tx, ty, tz, rx, ry, rz, ...]` (translations first, rotations second; translations in mm, rotations in degrees)
- Must be explicitly documented in `INPUT_SPECIFICATION.md`

### Scope in fmri-first-level-proc
- Update `INPUT_SPECIFICATION.md` with explicit unit and column order specification
- Update `prepare_motion_file()` docstring to state degrees assumption
- No functional code changes needed — the pipeline already passes through motion values without conversion

### Deferred to Upstream
- Remove `np.deg2rad()` call in `extract_motion_regressors()`
- Update rotation unit detection messaging
- Update tests

### Impact on FD Calculation
With degrees passed to AFNI's `1d_tool.py -censor_motion` (Euclidean norm, no radius scaling), at head radius ~57.3 mm, 1° ≈ 1 mm of arc — AFNI's intended convention. Previously, passing radians underweighted rotational contribution to FD by ~57x.

---

## Change 2: Minimum-Outlier Representative EPI Frame

### Decision
- Mandatory output, no config toggle
- Single actual TR (not an aggregate like mean/median)
- Uses AFNI's `3dToutcount -automask -fraction` to identify the TR with the lowest outlier fraction
- Extracts that volume with `3dbucket`

### Implementation Plan
- New shared utility: `gen_min_outlier_epi(scan_path, out_dir, out_file_pre, label, logger)` in `first_level_utils.py`
- Task (task_act, task_conn): one frame → `{out_file_pre}_min_outlier_epi.nii.gz`
- Rest (rest_conn): one per run → `{out_file_pre}_run{N}_min_outlier_epi.nii.gz`
- Placement: early in each pipeline's `run()`, after argument validation, before regression/filtering
- Skip-if-exists logic consistent with rest of pipeline

---

## Change 3: Pre-Regression Parcellated Time Series

### Decision
- Separate toggle per modality: `extract_raw_ptseries: true/false` under `extraction:` config block
- CSV output via existing `extract_roi_stats()` infrastructure
- Requires `template_path`, `average_type`, `extract_out_file_pre` (same validation as existing extraction)
- Placed before regression, after template validation
- Both task_act and task_conn support the toggle; skip-if-exists prevents redundant extraction when both are run on the same scan/template

### Output Naming

| Modality | Output File |
|----------|-------------|
| rest_conn | `{extract_out_file_pre}_run{N}_raw_ptseries.csv` (per-run, not concatenated) |
| task_conn | `{extract_out_file_pre}_raw_ptseries.csv` (single file) |
| task_act | `{extract_out_file_pre}_raw_ptseries.csv` (single file) |

### Redundancy Note
task_act and task_conn raw ptseries are identical when run on the same scan with the same template/average_type. Skip-if-exists handles this — whichever pipeline runs first produces the file, the other skips.

---

## Assumptions Approved by Stakeholder
1. `[tx, ty, tz, rx, ry, rz]` is the only supported motion column order
2. `extract_raw_ptseries` lives under the existing `extraction:` config block
3. Min-outlier EPI uses AFNI `-automask` (no user-supplied mask required)
4. Both task_act and task_conn support `extract_raw_ptseries` with skip-if-exists dedup
