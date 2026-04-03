<document_report>
  <meta project="fmri_first_level_proc" mode="document" timestamp="2026-03-12T12:22:42Z" />
  <files_updated>
    <file path="fmri_first_level_proc/__init__.py" changes="Updated __version__ from '2.1.0' to '2.2.0' to match pyproject.toml and git tag v2.2.0.">
      <type>inline_comment</type>
    </file>
    <file path="fmri_first_level_proc/run_first_level.py" changes="(1) Updated version header from '2.0 / 02/17/26' to '2.2 / 03/11/26'. (2) Corrected usage examples from 'python run_first_level.py' to 'run-first-level' (pip-installed CLI entry point). (3) Added docstring to main().">
      <type>inline_comment</type>
    </file>
    <file path="fmri_first_level_proc/first_level_utils.py" changes="Updated module-level docstring to clarify the stimulus I/O read/write split: 'Stimulus timing file I/O (read/validate phase: read_and_validate_stim_data; write phase: write_onset_file)'.">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_act_first_level.py" changes="(1) Removed '--censor_path' from the module-level REQUIREMENTS docstring — it is NOT a user-provided input; it is generated internally. Replaced with '--fd_threshold' with a note that the censor file is generated from motion_path. (2) Expanded run() docstring to document all 13 pipeline steps in order, including the read/write split and contrast parse-then-drop ordering.">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/task_conn_first_level.py" changes="(1) Removed '--censor_path' from the module-level REQUIREMENTS docstring for the same reason as task_act. (2) Expanded run() docstring to document all 17 pipeline steps in order, including: parse-before-drop for contrasts, write_stim_onset_files returning beta_cond_order, and beta_cond_order threading into gen_beta_series().">
      <type>docstring</type>
    </file>
    <file path="fmri_first_level_proc/rest_conn_first_level.py" changes="Expanded run() docstring from a one-liner to a detailed 9-step pipeline sequence, documenting the per-run notch filter, censor file generation, DOF-based run skipping, and concatenation flow.">
      <type>docstring</type>
    </file>
    <file path="example_config.yaml" changes="Corrected usage examples from 'python run_first_level.py' to 'run-first-level' (pip-installed CLI entry point).">
      <type>inline_comment</type>
    </file>
  </files_updated>
  <coverage>
    <public_functions_documented>67/67</public_functions_documented>
    <classes_documented>0/0</classes_documented>
    <modules_with_docstrings>6/6</modules_with_docstrings>
  </coverage>
  <summary>
    Documentation audit for the v2.2.0 codebase (post read/write split refactor and notch-filter fix).

    Critical corrections made:
    1. FACTUAL ERROR (both task pipelines): '--censor_path' was listed as a required user input in the module-level REQUIREMENTS docstrings of task_act_first_level.py and task_conn_first_level.py. The censor file has never been a user-provided input — it is generated internally from --motion_path and --fd_threshold. This has been corrected in both files.
    2. VERSION MISMATCH: __init__.py reported version 2.1.0 while pyproject.toml and the git tag are at 2.2.0. Fixed to 2.2.0.
    3. STALE CLI INVOCATION: run_first_level.py header and example_config.yaml used 'python run_first_level.py ...' instead of the correct pip-installed entry point 'run-first-level ...'. Fixed in both files. README.md and INPUT_SPECIFICATION.md were already correct.
    4. VERSION HEADER: run_first_level.py header said 'Version: 2.0, Last updated: 02/17/26'. Updated to match all other modules (2.2, 03/11/26).

    Documentation enrichments made:
    - run() docstrings in all three pipeline modules (task_act, task_conn, rest_conn) expanded from one-liners to detailed numbered pipeline sequences that document the full execution order including the v2.3.0 read/write split, parse-then-drop ordering, beta_cond_order threading, and per-run DOF-based skipping.
    - main() in run_first_level.py received a docstring (was undocumented).
    - first_level_utils.py module docstring now explicitly names the read and write functions for stimulus I/O.

    No changes were made to README.md, INPUT_SPECIFICATION.md, or example_config.yaml content (beyond the CLI invocation fix) — these were already accurate and complete for the current v2.2.0 state.
  </summary>
</document_report>
