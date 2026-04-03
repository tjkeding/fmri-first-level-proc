<test_report>
  <meta project="fmri_first_level_proc" mode="test" timestamp="2026-04-03T00:00:00Z" />
  <pre_design_run>
    <total>351</total>
    <passed>351</passed>
    <failed>0</failed>
    <errors>0</errors>
    <coverage_pct>not measured (no coverage plugin)</coverage_pct>
    <failures>
    </failures>
  </pre_design_run>
  <design_phase>
    <tests_created>104</tests_created>
    <tests_modified>0</tests_modified>
    <files_created>
      <file path="tests/test_v240_coverage_gaps.py" test_count="83" coverage_target="Utility functions: prepare_motion_file, compute_tissue_derivative, compute_dof, build_decon_base_command, build_conn_output_path, check_trial_survival, write_qc_summary, remove_files_from_dir, validate_onset_file_format, sanitize_filename_label, argparse type validators, log helpers, validate_extract_options, validate_connectivity_options, run_first_level dispatch" />
      <file path="tests/test_v240_pipeline_integration.py" test_count="21" coverage_target="Pipeline-level integration: gen_min_outlier_epi invocation per pipeline, motion_deriv_degree column count, QC summary JSON output, args.copy() mutation safety, include_motion_derivs column count, tissue derivative invocation, path length validation, GS_paths GSR warning, remove_previous flag, all-conditions-dropped abort" />
    </files_created>
    <design_rationale>
      Analysis of the existing 351 tests revealed that while config validation, contrast parsing,
      connectivity, sub-brick ordering, and v2.3.0 regression tests were thorough, significant
      gaps existed in:
      
      1. Core utility function unit tests: prepare_motion_file (column truncation, 1D reshape,
         error paths), compute_tissue_derivative (derivative correctness, multi-dim rejection,
         constant signal), compute_dof (exit_on_error flag, boundary values, unreadable censor),
         build_decon_base_command (all regressor combinations).
      
      2. Exhaustive parametric coverage: build_conn_output_path had no test for all 8
         fishZ/pcorr/calc_conn combinations. validate_onset_file_format had no direct tests.
         sanitize_filename_label edge cases (empty, special-only, consecutive underscores).
      
      3. Validation helper functions: validate_extract_options and validate_connectivity_options
         had no direct unit tests for their error paths.
      
      4. Pipeline-level v2.4.0 integration: gen_min_outlier_epi was tested in isolation but
         not verified as being correctly invoked (correct label, correct scan path) within
         each pipeline's run() function. motion_deriv_degree column-count propagation in
         rest_conn was untested. QC summary JSON output was untested. args.copy() mutation
         safety was assumed but never verified.
      
      5. General safety: remove_files_from_dir safety guards (root/home rejection) were
         untested. argparse type validators had no direct unit tests.
      
      Notable discovery during testing: compute_tissue_derivative correctly rejects
      single-timepoint files (np.loadtxt returns 0-dim array, failing ndim==1 check).
      This is appropriate defensive behavior documented in the test.
    </design_rationale>
  </design_phase>
  <post_design_run>
    <total>455</total>
    <passed>455</passed>
    <failed>0</failed>
    <errors>0</errors>
    <coverage_pct>not measured (no coverage plugin)</coverage_pct>
    <failures>
    </failures>
  </post_design_run>
  <summary>
    <all_passing>true</all_passing>
    <recommendation>proceed_to_document</recommendation>
  </summary>
  <action_items>
    <item priority="P2" target_mode="implement" description="Consider adding a guard in compute_tissue_derivative to handle single-timepoint inputs with a clearer error message (currently fails with 'must be single-column, got shape ()' which is technically correct but could be confusing to users)." />
  </action_items>
</test_report>
