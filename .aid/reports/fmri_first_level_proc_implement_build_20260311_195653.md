<?xml version="1.0" encoding="UTF-8"?>
<implement_report>
  <meta project="fmri_first_level_proc" mode="implement" submodule="build" timestamp="2026-03-11T19:56:53Z" />
  <spec_ref>/Volumes/Backup Plus/Yale Research Faculty/projects/fmri-first-level-proc/fmri_first_level_proc_implement_plan_20260311_195653.md</spec_ref>
  <changes_applied>

    <change id="C3" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="~15" />
      </files_modified>
      <notes>
        Restored trailing ' (AFNI transpose) on the -input argument to 3dTproject in
        notch_filter_motion(). After 3dTproject executes, added a 1dtranspose call to
        restore rows=TRs, cols=params orientation, followed by os.replace() to atomically
        overwrite the output path. Updated the comment block to explain the full
        transpose round-trip rationale. No deviations from spec.
      </notes>
    </change>

    <change id="C1" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="~50" />
      </files_modified>
      <notes>
        Replaced monolithic get_stim_data() with two functions: read_stim_data() (read +
        validate only, returns sorted DataFrame) and write_stim_onset_files() (writes onset
        files, returns beta_cond_order as list of str in first-appearance order). The
        beta_cond_order return value is the key artifact that eliminates the np.unique()
        ordering ambiguity — it records the exact order in which conditions were appended
        to beta_onsets.txt, which must match the 3dLSS sub-brick layout.
      </notes>
    </change>

    <change id="C2" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="~20" />
      </files_modified>
      <notes>
        Updated run() to call read_stim_data() first, then trial-survival filtering,
        then write_stim_onset_files() with filtered cond_beta_labels (BUG-002 fix).
        Threaded beta_cond_order into gen_beta_series() (BUG-001 fix). Replaced
        np.unique(stim_data['CONDITION']) with stim_data['CONDITION'].unique() in
        gen_design_matrix() for both the -num_stimts count and the nuisance condition
        loop. Updated gen_beta_series() signature (added beta_cond_order parameter) and
        loop body (removed inner if guard, corrected indentation to match new structure).
        Docstring references to get_stim_data() updated to read_stim_data() in both
        gen_design_matrix() and gen_beta_series(). No deviations from spec.
      </notes>
    </change>

    <change id="C4" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_act_first_level.py" lines_changed="~30" />
      </files_modified>
      <notes>
        Replaced get_stim_data() with read_stim_data() + write_stim_onset_files() in
        task_act_first_level.py, mirroring the task_conn pattern. Updated run() to call
        read_stim_data() first and write_stim_onset_files() after the full filtering
        block (trial-survival, contrast drop, extraction label drop). task_act's
        write_stim_onset_files() returns None (no beta_cond_order needed). Docstring
        reference in run_first_level() updated.
      </notes>
    </change>

  </changes_applied>
  <summary>
    <total_changes>4</total_changes>
    <completed>4</completed>
  </summary>
  <next_steps>
    Recommended: run /test to validate all changes. Priority test items per brainstorm:
    (P0) BUG-001 test: timing DataFrame with non-alphabetical first-appearance order;
         verify beta_cond_order from write_stim_onset_files() and sub-brick indices in
         gen_beta_series() map correctly.
    (P0) BUG-003 test: synthetic TRs×6 motion file through notch_filter_motion();
         verify output shape is TRs×6 and values are numerically reasonable.
    (P1) BUG-002 test: timing data with 0–1 surviving trials for some conditions;
         verify dropped conditions receive individual onset files, beta_onsets.txt
         contains only surviving conditions.
    (P2) task_act harmonization test: verify task_act onset files are written only
         after condition filtering.
  </next_steps>
</implement_report>
