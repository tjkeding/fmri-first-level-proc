<document_report>
  <meta project="fmri_first_level_proc" mode="document" timestamp="2026-03-11T20:07:25Z" />
  <files_updated>
    <file path="fmri_first_level_proc/first_level_utils.py" changes="Added module-level Python docstring summarizing all utility categories; updated version header to 2.2 / 03/11/26; expanded compute_dof docstring to document exit_on_error parameter; expanded build_decon_base_command docstring with full Parameters/Returns; expanded format_netcorr_mat docstring with matrix section lookup logic">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/first_level_config.py" changes="Added module-level Python docstring describing public API, config schema overview, and validation rules; updated version header to 2.2 / 03/11/26">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/rest_conn_first_level.py" changes="Added docstrings to gen_residual_ts (per-run DOF skip logic, 3dTproject dispatch, concatenation), gen_ptseries (3dROIstats extraction), gen_conn (parcellated/seed-to-voxel dispatch); updated version header to 2.2 / 03/11/26">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_conn_first_level.py" changes="Added docstrings to get_stim_data, gen_design_matrix, gen_beta_series, gen_pbseries, gen_conn; updated version header to 2.2 / 03/11/26">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_act_first_level.py" changes="Added docstrings to valid_extract_labels, get_stim_data, run_first_level, extract_effects (T-to-Z conversion logic documented); updated version header to 2.2 / 03/11/26">
      <type>docstring</type>
    </file>
  </files_updated>
  <coverage>
    <public_functions_documented>64/64</public_functions_documented>
    <classes_documented>0/0</classes_documented>
    <modules_with_docstrings>5/5</modules_with_docstrings>
  </coverage>
  <summary>All five source modules now have 100% function docstring coverage (64/64) and module-level Python docstrings (5/5). Version headers updated to 2.2 / 03/11/26 across all modules to reflect post-2.1.0 commits (skip-low-DOF logic, warn-and-skip for low-trial conditions, .1D standardization). README.md and INPUT_SPECIFICATION.md were not modified per instructions. No functional code was changed.</summary>
</document_report>
