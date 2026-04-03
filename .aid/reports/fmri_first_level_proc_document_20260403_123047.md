<document_report>
  <meta project="fmri_first_level_proc" mode="document" timestamp="2026-04-03T12:30:47Z" />
  <files_updated>
    <file path="fmri_first_level_proc/first_level_utils.py" changes="Added gen_min_outlier_epi to module-level docstring capability list">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_act_first_level.py" changes="Updated run() docstring: expanded ordered steps from 13 to 16 to include extract_raw_ptseries (step 4), gen_min_outlier_epi (step 5), and renumbered downstream steps accordingly">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_conn_first_level.py" changes="Updated run() docstring: expanded ordered steps from 17 to 20 to include extract_raw_ptseries (step 4), gen_min_outlier_epi (step 5), and renumbered downstream steps accordingly">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/rest_conn_first_level.py" changes="Updated run() docstring: expanded ordered steps from 9 to 11 to include gen_min_outlier_epi per run (step 5) and extract_raw_ptseries per run (step 6)">
      <type>docstring</type>
    </file>
    <file path="README.md" changes="Added motion degrees convention, min-outlier EPI output, and extract_raw_ptseries toggle to Key Features">
      <type>readme</type>
    </file>
    <file path="AID_LOG.md" changes="Created new AID_LOG.md per AID Framework (Weaver, 2025)">
      <type>aid_log</type>
    </file>
  </files_updated>
  <aid_log>
    <status>created</status>
    <sections_modified>All sections 1-7 (new file)</sections_modified>
  </aid_log>
  <coverage>
    <public_functions_documented>67/67</public_functions_documented>
    <classes_documented>0/0</classes_documented>
    <modules_with_docstrings>5/5</modules_with_docstrings>
  </coverage>
  <summary>
    Documentation updated for v2.4.0. All three new features are reflected:
    (1) gen_min_outlier_epi() added to first_level_utils module docstring; all three pipeline run() docstrings updated to include the mandatory gen_min_outlier_epi step and the optional extract_raw_ptseries step in their ordered execution sequences.
    (2) motion_deriv_degree was already documented in INPUT_SPECIFICATION.md and example_config.yaml; the prepare_motion_file() docstring already specifies degrees-only convention. No additional changes needed.
    (3) extract_raw_ptseries was already present in INPUT_SPECIFICATION.md and example_config.yaml from implementation. No additional changes needed.
    README.md updated with concise mention of all three features. AID_LOG.md created per disclosure framework requirements.
  </summary>
</document_report>
