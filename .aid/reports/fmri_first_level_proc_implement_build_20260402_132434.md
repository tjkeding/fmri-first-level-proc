<implement_report>
  <meta project="fmri_first_level_proc" mode="implement" submodule="build" timestamp="2026-04-02T17:45:00Z" />
  <spec_ref>fmri_first_level_proc_implement_plan_20260402_132434.md</spec_ref>
  <changes_applied>
    <change id="C1" status="done">
      <files_modified>
        <file path="INPUT_SPECIFICATION.md" lines_changed="4" />
      </files_modified>
      <notes>Added column order contract [tx,ty,tz,rx,ry,rz,...] and units (mm for translations, degrees for rotations) with warning about radian underweighting.</notes>
    </change>
    <change id="C2" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="7" />
      </files_modified>
      <notes>Added Notes section to prepare_motion_file() docstring documenting degree assumption.</notes>
    </change>
    <change id="C3" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="75" />
      </files_modified>
      <notes>New gen_min_outlier_epi() function added between ROI Extraction and QC Summary sections. Uses 3dToutcount + np.argmin + 3dbucket. Full docstring, skip-if-exists, error handling.</notes>
    </change>
    <change id="C4" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_act_first_level.py" lines_changed="4" />
      </files_modified>
      <notes>Import added; call placed after remove_previous block, before censor generation. Uses label="" for single-scan task output.</notes>
    </change>
    <change id="C5" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="4" />
      </files_modified>
      <notes>Import added; call placed after connectivity validation, before censor generation. Uses label="" for single-scan task output.</notes>
    </change>
    <change id="C6" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="5" />
      </files_modified>
      <notes>Import added; per-run loop placed after remove_previous/validation, before QC data. Uses label=f"run{i+1}".</notes>
    </change>
    <change id="C7" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/first_level_config.py" lines_changed="24" />
      </files_modified>
      <notes>Config validation: extract_raw_ptseries requires extract_out_file_pre for all three types. Namespace: extract_raw_ptseries bool set for all three types (True/False). task_act edge case: when extract=False but extract_raw_ptseries=True, extract_out_file_pre is preserved from extraction block.</notes>
    </change>
    <change id="C8" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_act_first_level.py" lines_changed="18" />
      </files_modified>
      <notes>Raw ptseries block placed after extraction validation, before min-outlier EPI. Validates template_path != None/WB, calls validate_template + extract_roi_stats, writes CSV with skip-if-exists. CLI argument added.</notes>
    </change>
    <change id="C9" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="18" />
      </files_modified>
      <notes>Same pattern as C8. Placed after connectivity validation, before min-outlier EPI. CLI argument added.</notes>
    </change>
    <change id="C10" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="20" />
      </files_modified>
      <notes>Per-run extraction loop. Template validated once on first scan, then reused. Output: {extract_out_file_pre}_run{N}_raw_ptseries.csv. Placed after min-outlier EPI, before QC summary. CLI argument added.</notes>
    </change>
    <change id="C11" status="done">
      <files_modified>
        <file path="example_config.yaml" lines_changed="3" />
      </files_modified>
      <notes>extract_raw_ptseries: false added to all three analysis type extraction blocks with descriptive comments.</notes>
    </change>
    <change id="C12" status="done">
      <files_modified>
        <file path="INPUT_SPECIFICATION.md" lines_changed="20" />
      </files_modified>
      <notes>Documented extract_raw_ptseries in all three analysis type extraction sections. Added min_outlier_epi and raw_ptseries output files to all three Output Files subsections.</notes>
    </change>
    <change id="C13" status="done">
      <files_modified>
        <file path="fmri_first_level_proc/__init__.py" lines_changed="1" />
        <file path="pyproject.toml" lines_changed="1" />
        <file path="fmri_first_level_proc/first_level_utils.py" lines_changed="2" />
        <file path="fmri_first_level_proc/first_level_config.py" lines_changed="2" />
        <file path="fmri_first_level_proc/task_act_first_level.py" lines_changed="2" />
        <file path="fmri_first_level_proc/task_conn_first_level.py" lines_changed="2" />
        <file path="fmri_first_level_proc/rest_conn_first_level.py" lines_changed="2" />
      </files_modified>
      <notes>Version bumped from 2.3.1 to 2.4.0 in all 7 files. Last-updated dates set to 04/02/26.</notes>
    </change>
  </changes_applied>
  <summary>
    <total_changes>13</total_changes>
    <completed>13</completed>
  </summary>
  <next_steps>Recommended: run /test to validate all changes. Key areas to cover:
    - gen_min_outlier_epi: mock 3dToutcount/3dbucket, verify argmin logic, skip-if-exists, error paths
    - extract_raw_ptseries: config validation (all 3 types), namespace building, pipeline integration (template validation, skip-if-exists, per-run for rest_conn)
    - Motion units: docstring presence (no functional change to test)
    - Config edge cases: extract_raw_ptseries=true with missing extract_out_file_pre; extract_raw_ptseries=true with extract=false (task_act edge case)
  </next_steps>
</implement_report>
