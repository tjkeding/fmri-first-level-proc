<?xml version="1.0" encoding="UTF-8"?>
<implement_plan>
  <meta project="fmri_first_level_proc" mode="implement" submodule="plan" timestamp="2026-03-11T19:56:53Z" />
  <input_reports>
    <report path="<local_path>/fmri_first_level_proc_brainstorm_20260311_220000.md" mode="brainstorm" key_items="4" />
  </input_reports>
  <changes>

    <change id="C1" priority="P0" source_item="T2/A2 BUG-002 + T1/A1 BUG-001">
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <description>
        Split get_stim_data() into read_stim_data() + write_stim_onset_files().
        read_stim_data() reads, validates, and returns the sorted DataFrame (no file I/O).
        write_stim_onset_files() writes onset files and returns beta_cond_order — a list
        of conditions in the order their onsets were appended to beta_onsets.txt (first-appearance
        order, matching the sub-brick layout 3dLSS will produce). This addresses BUG-002
        (dropped conditions never get individual onset files when get_stim_data runs before
        trial-survival filtering) and establishes the foundation for BUG-001 fix.
      </description>
      <spec>
        NEW: read_stim_data(args, logger) -> pd.DataFrame
          - Calls read_and_validate_stim_data(args.task_timing_path, args.cond_beta_labels, logger=logger)
          - Returns sorted_df (same as current get_stim_data start)
          - No onset file I/O

        NEW: write_stim_onset_files(stim_data, args, logger) -> list[str]
          - Accepts the already-read stim_data DataFrame and current args.cond_beta_labels
            (which may have had conditions dropped by trial-survival filtering)
          - Iterates stim_data['CONDITION'].unique() in first-appearance order (pandas .unique())
          - For cond NOT in args.cond_beta_labels: writes individual _concat_{cond}_onsets.txt
          - For cond IN args.cond_beta_labels: accumulates onsets/durations AND appends cond to
            beta_cond_order list
          - Writes combined beta_onsets.txt from accumulated onsets
          - Returns beta_cond_order (list of str, first-appearance order of beta conditions)

        REMOVE: get_stim_data() — replaced by read_stim_data() + write_stim_onset_files()
      </spec>
      <dependencies>none</dependencies>
      <risk>medium - API change to internal functions; run() call sites must be updated (C2)</risk>
      <rollback>Revert task_conn_first_level.py to previous get_stim_data() monolith</rollback>
    </change>

    <change id="C2" priority="P0" source_item="T2/A2 BUG-002 + T1/A1 BUG-001">
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <description>
        Reorder run() to: read_stim_data first → trial-survival + condition filtering →
        write_stim_onset_files with filtered labels. Thread beta_cond_order through
        gen_design_matrix() and gen_beta_series().
      </description>
      <spec>
        In run():
          Line ~558: Replace `stim_data = get_stim_data(args, logger)` with:
            stim_data = read_stim_data(args, logger)
          After trial-survival filtering and condition dropping (current lines ~584-606):
            beta_cond_order = write_stim_onset_files(stim_data, args, logger)
          Pass beta_cond_order to gen_design_matrix() and gen_beta_series() as additional argument.

        gen_design_matrix(stim_data, args, logger) — no signature change needed for BUG-001
          (gen_design_matrix does not use sub-brick ordering; np.unique() there determines
           nuisance regressor order for 3dDeconvolve, which does not affect beta series extraction).
          HOWEVER: the brainstorm decision states to eliminate np.unique() from gen_design_matrix
          line 216 and use stim_data['CONDITION'].unique() with filter instead — apply this as well.
          Update line 212 -num_stimts calculation: use len(stim_data['CONDITION'].unique()) instead of np.unique.
          Update loop at line 216: change np.unique(stim_data['CONDITION']) to stim_data['CONDITION'].unique()
          (filter to cond not in args.cond_beta_labels remains the same).

        gen_beta_series(stim_data, args, beta_cond_order, logger) — add beta_cond_order param:
          Replace `for curr_cond in np.unique(stim_data['CONDITION']):` (line ~296)
          with `for curr_cond in beta_cond_order:` — iterate in the same order onsets were
          appended to beta_onsets.txt so total_used correctly tracks sub-brick offsets.
          Remove the inner `if curr_cond in args.cond_beta_labels:` guard — beta_cond_order
          already contains only beta conditions, no filtering needed.
      </spec>
      <dependencies>C1</dependencies>
      <risk>medium - changes call signatures; must update all call sites in run()</risk>
      <rollback>Revert run(), gen_design_matrix(), gen_beta_series() signatures and body</rollback>
    </change>

    <change id="C3" priority="P0" source_item="T3/A2 BUG-003">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>
        Fix notch_filter_motion() in first_level_utils.py. Restore trailing ' (AFNI transpose)
        on the -input argument so 3dTproject sees rows=params, cols=TRs (as AFNI expects for .1D
        files of shape TRs×params). After 3dTproject output is written, run AFNI's 1dtranspose
        on the output to convert from rows=params, cols=TRs back to rows=TRs, cols=params.
        This keeps the full operation in the AFNI ecosystem and avoids numpy I/O edge cases.
      </description>
      <spec>
        In notch_filter_motion():

        1. Change -input argument from:
             "-input", f"{motion_6col}",
           to:
             "-input", f"{motion_6col}'",
           (trailing apostrophe = AFNI in-line transpose operator)

        2. After run_afni_command(cmd, ...) and the existence check, add:
             # 3dTproject writes rows=params, cols=TRs; 1dtranspose restores rows=TRs, cols=params
             transposed_path = out_path + ".transposed.1D"
             transpose_cmd = ["1dtranspose", out_path, transposed_path]
             run_afni_command(transpose_cmd, description=f"1dtranspose notch filter {label}", logger=logger)
             if not os.path.exists(transposed_path):
                 logger.error("1dtranspose failed for notch-filtered motion: %s", out_path)
                 sys.exit(1)
             os.replace(transposed_path, out_path)
             logger.info("Notch-filtered motion parameters (rows=TRs) saved to %s", out_path)

        3. Update the comment on line 690 to reflect the corrected orientation logic.

        Note: 3dTproject with ' on input will output rows=params,cols=TRs. 1dtranspose
        converts to rows=TRs,cols=params. os.replace atomically renames so out_path
        always contains the final rows=TRs orientation.
      </spec>
      <dependencies>none</dependencies>
      <risk>low - self-contained in one function; corrects orientation to match downstream usage</risk>
      <rollback>Remove ' from -input, remove 1dtranspose block</rollback>
    </change>

    <change id="C4" priority="P2" source_item="T4/A1 task_act harmonization">
      <file path="fmri_first_level_proc/task_act_first_level.py" action="modify" />
      <description>
        Harmonize task_act_first_level.py to use the same read-first/write-after-filtering pattern.
        Split task_act's get_stim_data() into read_stim_data() + write_stim_onset_files().
        Reorder run() to call read first, perform trial-survival filtering, then write onset files.
        This prevents a hypothetical future breakage and makes both pipelines architecturally symmetric.
      </description>
      <spec>
        NEW: read_stim_data(args, logger) -> pd.DataFrame
          - Calls read_and_validate_stim_data(args.task_timing_path, args.cond_labels, logger=logger)
          - Returns sorted_df; no file I/O

        NEW: write_stim_onset_files(stim_data, args, logger)
          - Iterates stim_data['CONDITION'].unique()
          - For cond in args.cond_labels: writes _concat_{cond}_onsets.txt
          - Returns None (task_act does not need beta_cond_order)

        REMOVE: get_stim_data() — replaced by the two functions above

        In run() [task_act]:
          Replace `stim_data = get_stim_data(args, logger)` (line ~505) with:
            stim_data = read_stim_data(args, logger)
          After trial-survival filtering (after lines ~511-513):
            write_stim_onset_files(stim_data, args, logger)

        Note: task_act's run_first_level() references onset files on disk already written by
        write_stim_onset_files, so no signature change is needed for run_first_level().
      </spec>
      <dependencies>none (independent of C1-C3)</dependencies>
      <risk>low - task_act works correctly today; this is a defensive refactor for consistency</risk>
      <rollback>Revert task_act_first_level.py to monolithic get_stim_data()</rollback>
    </change>

  </changes>
  <execution_order>C3, C1, C2, C4</execution_order>
</implement_plan>
