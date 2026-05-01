<implement_report>
  <meta project="fmri_first_level_proc" mode="implement" submodule="build" timestamp="2026-05-01T13:42:21Z" />
  <spec_ref>fmri_first_level_proc_implement_plan_20260501_132101.md</spec_ref>

  <changes_applied>

    <change id="C1" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_config.py" lines_changed="2" />
      </files_modified>
      <notes>BOOL_KEYS tuple at line 74 extended to include the new boolean key. build_namespace rest_conn branch at line 585 now reads the merged block value with default False. _merge_global_into_block requires no edit because its existing BOOL_KEYS loop handles None-to-False defaulting automatically.</notes>
    </change>

    <change id="C2" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="4" />
      </files_modified>
      <notes>argparse store_true flag added at line 727 immediately after the existing --use_tissue_derivs definition. Module-level docstring INPUTS optional bullet inserted at line 57.</notes>
    </change>

    <change id="C3" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="68" />
      </files_modified>
      <notes>censor_interpolate_1d_afni inserted at line 734, immediately after notch_filter_motion. Function constructs the AFNI command with -polort -1, -dt, apostrophe-suffixed -input, -censor, -cenmode NTRP, -prefix; performs the 1dtranspose round-trip with os.replace to restore rows=TRs orientation. NumPy-style docstring with Parameters, Returns, and one-paragraph summary.</notes>
    </change>

    <change id="C4" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="71" />
      </files_modified>
      <notes>bandpass_filter_1d_afni inserted at line 802, immediately after censor_interpolate_1d_afni and before compute_tissue_derivative. Function builds 3dBandpass command with -nodetrend, -dt, -prefix, then positional fbot/ftop and apostrophe-suffixed dataset path; performs the 1dtranspose round-trip. NumPy-style docstring covers the -nodetrend/polort -1 alignment, transpose dance, and post-interpolation filtering intent.</notes>
    </change>

    <change id="C5" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="70" />
      </files_modified>
      <notes>_generate_run_residual_simultaneous inserted at line 111 as a module-level private helper, immediately before gen_residual_ts. Body matches the prior inline 3dTproject construction verbatim. The inline 3dTproject construction inside gen_residual_ts is intentionally untouched at this stage; the dispatch refactor that replaces it is a later change.</notes>
    </change>

    <change id="C6" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="163" />
      </files_modified>
      <notes>Added import shutil at line 81. Added censor_interpolate_1d_afni and bandpass_filter_1d_afni to the from .first_level_utils import block at lines 107-108. _generate_run_residual_sequenced inserted at line 181. Function implements the full six-step Ciric-inspired pipeline: NTRP-interpolate BOLD, 3dBandpass BOLD, per-regressor NTRP-interp + 3dBandpass via the new utilities, nuisance regression on filtered BOLD with filtered orts, post-step 3dcalc zeroing of originally-censored TRs, plus a finally block whose cleanup gates on (success and not args.keep_run_res_dtseries). Tissue derivatives are computed on raw signals and routed through the same filter pipeline as base regressors.</notes>
    </change>

    <change id="C7" status="done" user_decision="n/a">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="14" />
      </files_modified>
      <notes>Inline 3dTproject construction inside gen_residual_ts replaced with a dispatch block at lines 418-430 that selects between _generate_run_residual_sequenced and _generate_run_residual_simultaneous based on args.use_sequenced_bandpass and converts a None return into a continue. Updated the n_regressors comment to document both denoising-path rationales. Appended a paragraph to the gen_residual_ts docstring describing the dispatch.</notes>
    </change>

    <change id="C8" status="done" user_decision="n/a">
      <files_modified>
        <file path="example_config.yaml" lines_changed="1" />
      </files_modified>
      <notes>Added one line to the rest_conn block immediately after the use_tissue_derivs line. Default value false matches the BOOL_KEYS default-False semantics.</notes>
    </change>

  </changes_applied>

  <summary>
    <total_changes>8</total_changes>
    <completed>8</completed>
    <skipped>0</skipped>
    <blocked>0</blocked>
  </summary>

  <next_steps>Recommended: run /test to design the regression suite covering the new sequenced denoising path. The brainstorm report enumerates six P1 test design items: unit tests for the two new 1D-AFNI utilities, integration tests for the sequenced backend, dispatch tests for the refactored gen_residual_ts, config-validation tests for the new boolean key, and a regression test confirming bitwise-identical output with the new flag set False against v2.4.0-post1. Three P2 documentation tasks remain for /document mode after testing validates the implementation.</next_steps>

</implement_report>
