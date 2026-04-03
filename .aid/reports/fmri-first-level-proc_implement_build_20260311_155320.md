<?xml version="1.0" encoding="UTF-8"?>
<implement_report>
  <meta project="fmri-first-level-proc" mode="implement" submodule="build" timestamp="2026-03-11T19:53:20Z" />
  <spec_ref><local_path>/fmri-first-level-proc_implement_plan_20260311_155157.md</spec_ref>
  <changes_applied>
    <change id="C1" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="2" />
      </files_modified>
      <tests_run>0</tests_run>
      <tests_passed>0</tests_passed>
      <notes>
        Line 667: Updated comment from "3dTproject on 1D file: use \' (AFNI transpose) so rows=timepoints, cols=params"
        to "3dTproject on .1D file: rows=timepoints, cols=motion params (no transpose needed)".
        Line 671: Removed trailing single-quote from f"{motion_6col}'" → f"{motion_6col}".
        No tests currently exist for notch_filter_motion() — regression test addition deferred to /test mode
        as specified in the brainstorm action items (P0, target_mode=test).
      </notes>
    </change>
    <change id="C2" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="19" />
      </files_modified>
      <tests_run>0</tests_run>
      <tests_passed>0</tests_passed>
      <notes>
        Three distinct sub-changes applied:
        1. Added co-presence guard block for --contrast_functions / --contrast_labels (lines 481-489),
           mirroring task_act_first_level.py lines 388-395.
        2. Added parse-with-original-conds block (lines 497-503): valid_contrast_functions() now called
           after original_conds is assigned but before args.cond_beta_labels is filtered. This ensures
           the drop block below receives parsed dicts (cont["CONDS"] valid) rather than raw strings.
        3. Removed redundant second valid_contrast_functions() call (formerly lines 556-565). The
           connectivity contrasts block (lines 574-580) now passes args.contrast_functions directly
           to gen_conn_contrasts() — already parsed and already filtered by the drop block.
        Integration test for dropped conditions + contrasts deferred to /test mode as specified in the
        brainstorm action items (P1, target_mode=test).
      </notes>
    </change>
  </changes_applied>
  <summary>
    <total_changes>2</total_changes>
    <completed>2</completed>
    <tests_passing>N/A — no existing tests cover notch_filter_motion(); task_conn integration tests
      exist but were not run (implement mode does not run tests per mode constraints)</tests_passing>
  </summary>
  <next_steps>
    Recommended: run /test to add regression tests for both bugs and validate the full test suite.
    Specifically:
    - BUG-001 (P0): Add test for notch_filter_motion() verifying output_rows == input_rows with .1D input.
    - BUG-002 (P1): Add integration test for task_conn with dropped conditions + contrasts verifying
      graceful handling (contrasts referencing dropped conditions skipped, others computed correctly).
    - Confirm all 164+ existing tests still pass.
  </next_steps>
</implement_report>
