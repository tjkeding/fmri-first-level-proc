<implement_plan>
  <meta project="fmri_first_level_proc" mode="implement" submodule="plan" timestamp="2026-04-02T17:24:34Z" />
  <input_reports>
    <report path="<local_path>/fmri_first_level_proc_brainstorm_20260402.md" mode="brainstorm" key_items="3" />
  </input_reports>

  <!-- ===================================================================== -->
  <!-- CHANGE 1: Motion Rotation Units - Degrees-Only Contract               -->
  <!-- ===================================================================== -->
  <changes>
    <change id="C1" priority="P1" source_item="Change 1: Motion Rotation Units">
      <file path="INPUT_SPECIFICATION.md" action="modify" />
      <description>
        Update the "Motion Regressor File" section of INPUT_SPECIFICATION.md to
        explicitly document the unit and column-order contract:
        - Column order: [tx, ty, tz, rx, ry, rz, ...] (translations first, rotations second)
        - Units: translations in mm, rotations in DEGREES
        - This is the only supported column order
        - No auto-detection or conversion is performed by the pipeline
      </description>
      <spec>
        In the "### Motion Regressor File" section (~line 341), after the "Columns"
        bullet, insert a new sub-section or expand the existing text:
        - Add a "Units and Column Order" sub-bullet under "Columns" specifying:
          tx, ty, tz (mm), rx, ry, rz (degrees)
        - Add a warning that rotations MUST be in degrees (radians will cause
          incorrect FD computation: ~57x underweighting of rotational contribution)
        - State that the pipeline passes motion values through without conversion
      </spec>
      <dependencies>none</dependencies>
      <risk>low - documentation only, no functional change</risk>
      <rollback>Revert the text edit in INPUT_SPECIFICATION.md</rollback>
    </change>

    <change id="C2" priority="P1" source_item="Change 1: Motion Rotation Units">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>
        Update the prepare_motion_file() docstring to explicitly state the
        degrees assumption for rotation parameters. No functional code change.
      </description>
      <spec>
        In the docstring of prepare_motion_file() (~line 71-96), add a "Notes"
        section stating:
        - Rotation parameters are assumed to be in degrees
        - The pipeline passes motion values through to AFNI without unit conversion
        - Column order contract: [tx, ty, tz, rx, ry, rz, ...]
      </spec>
      <dependencies>none</dependencies>
      <risk>low - docstring only</risk>
      <rollback>Revert docstring edit</rollback>
    </change>

    <!-- ===================================================================== -->
    <!-- CHANGE 2: Minimum-Outlier Representative EPI Frame                    -->
    <!-- ===================================================================== -->
    <change id="C3" priority="P0" source_item="Change 2: Minimum-Outlier Representative EPI Frame">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>
        Add new shared utility function gen_min_outlier_epi() that:
        1. Runs 3dToutcount -automask -fraction on the input scan
        2. Identifies the TR index with the lowest outlier fraction
        3. Extracts that single volume with 3dbucket
        4. Returns the path to the output file
        Includes skip-if-exists logic and full logging.
      </description>
      <spec>
        Function signature:
          def gen_min_outlier_epi(scan_path, out_dir, out_file_pre, label, logger):
              """Extract the minimum-outlier EPI frame from a 4D scan.
              ...
              Parameters
              ----------
              scan_path : str
                  Path to the 4D NIfTI scan.
              out_dir : str
                  Output directory.
              out_file_pre : str
                  Output filename prefix.
              label : str
                  Descriptive label for the output file (e.g., "" for task,
                  "run1" for rest_conn).
              logger : logging.Logger

              Returns
              -------
              str
                  Path to the extracted single-volume NIfTI.
              """

        Implementation:
        - Build output filename:
          - If label is empty/None: f"{out_file_pre}_min_outlier_epi.nii.gz"
          - If label is set: f"{out_file_pre}_{label}_min_outlier_epi.nii.gz"
        - out_path = os.path.join(out_dir, output_filename)
        - Skip-if-exists: if os.path.exists(out_path): log and return out_path
        - Run: 3dToutcount -automask -fraction -quiet {scan_path}
          via run_afni_command with capture_output=True
        - Parse stdout: one float per line (one per TR)
        - np.argmin() to find the TR index with lowest outlier fraction
        - Log the chosen TR index and its outlier fraction
        - Run: 3dbucket -prefix {out_path} {scan_path}[{min_idx}]
          via run_afni_command
        - Verify output exists, log success/error
        - Return out_path

        Place this function after extract_roi_stats() and before
        write_qc_summary(), in the "General Utilities" region of the file.
        Actually, to maintain logical grouping, place it in a new section
        "# Min-Outlier EPI" between the "ROI Extraction" and "QC Summary"
        sections (i.e., after line ~1114, before line ~1116).
      </spec>
      <dependencies>none</dependencies>
      <risk>medium - new AFNI command usage; tested pattern (3dToutcount, 3dbucket)
             is standard AFNI; numpy argmin is deterministic</risk>
      <rollback>Remove the function from first_level_utils.py</rollback>
    </change>

    <change id="C4" priority="P0" source_item="Change 2: Minimum-Outlier Representative EPI Frame">
      <file path="fmri_first_level_proc/task_act_first_level.py" action="modify" />
      <description>
        Add gen_min_outlier_epi() call to the task_act run() function.
        Placement: early in run(), after argument validation / remove_previous
        but before censor file generation. Uses label="" for single-scan task.
        Import gen_min_outlier_epi from first_level_utils.
      </description>
      <spec>
        1. Add gen_min_outlier_epi to the import block from .first_level_utils
        2. In run(), insert call after the "if args.remove_previous" block
           and before "# Generate censor file from motion regressors":
           gen_min_outlier_epi(args.scan_path, args.out_dir, args.out_file_pre,
                               "", logger)
      </spec>
      <dependencies>C3</dependencies>
      <risk>low - simple function call insertion at a natural placement point</risk>
      <rollback>Remove the import and function call</rollback>
    </change>

    <change id="C5" priority="P0" source_item="Change 2: Minimum-Outlier Representative EPI Frame">
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <description>
        Add gen_min_outlier_epi() call to the task_conn run() function.
        Same placement pattern as task_act. Uses label="" for single-scan task.
        Import gen_min_outlier_epi from first_level_utils.
      </description>
      <spec>
        1. Add gen_min_outlier_epi to the import block from .first_level_utils
        2. In run(), insert call after the "if args.remove_previous" block
           and before "# Make sure optional args exist if --extract_pbseries":
           gen_min_outlier_epi(args.scan_path, args.out_dir, args.out_file_pre,
                               "", logger)
      </spec>
      <dependencies>C3</dependencies>
      <risk>low - simple function call insertion</risk>
      <rollback>Remove the import and function call</rollback>
    </change>

    <change id="C6" priority="P0" source_item="Change 2: Minimum-Outlier Representative EPI Frame">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>
        Add per-run gen_min_outlier_epi() calls to the rest_conn run() function.
        Placement: early in run(), after argument validation / remove_previous
        but before gen_residual_ts(). Uses label=f"run{i+1}" for per-run output.
        Import gen_min_outlier_epi from first_level_utils.
      </description>
      <spec>
        1. Add gen_min_outlier_epi to the import block from .first_level_utils
        2. In run(), insert a loop after the "if args.remove_previous" block
           and before "# Make sure optional args exist if --extract_ptseries":
           for i, scan in enumerate(args.scan_paths):
               gen_min_outlier_epi(scan, args.out_dir, args.out_file_pre,
                                   f"run{i+1}", logger)
      </spec>
      <dependencies>C3</dependencies>
      <risk>low - simple loop with function call</risk>
      <rollback>Remove the import and loop</rollback>
    </change>

    <!-- ===================================================================== -->
    <!-- CHANGE 3: Pre-Regression Raw Parcellated Time Series                  -->
    <!-- ===================================================================== -->
    <change id="C7" priority="P0" source_item="Change 3: Pre-Regression Raw Ptseries">
      <file path="fmri_first_level_proc/first_level_config.py" action="modify" />
      <description>
        Add config validation and namespace building for the new
        extract_raw_ptseries toggle under the extraction block.
        Must be supported for all three analysis types.
      </description>
      <spec>
        1. In validate_config(), in the extraction validation block:
           - For task_act: add check: if ext.get("extract_raw_ptseries", False)
             then "extract_out_file_pre" must be present
           - For task_conn: add check: if ext.get("extract_raw_ptseries", False)
             then "extract_out_file_pre" must be present
           - For rest_conn: add check: if ext.get("extract_raw_ptseries", False)
             then "extract_out_file_pre" must be present

        2. In build_namespace():
           - For task_act (extraction block):
             Add ns.extract_raw_ptseries = bool(extraction.get("extract_raw_ptseries", False))
           - For task_conn (extraction block):
             Add ns.extract_raw_ptseries = bool(extraction.get("extract_raw_ptseries", False))
           - For rest_conn (extraction block):
             Add ns.extract_raw_ptseries = bool(extraction.get("extract_raw_ptseries", False))
           - For all three: in the else branch (extraction is None), set
             ns.extract_raw_ptseries = False
      </spec>
      <dependencies>none</dependencies>
      <risk>low - pattern follows existing extraction toggle logic exactly</risk>
      <rollback>Remove the added validation checks and namespace attributes</rollback>
    </change>

    <change id="C8" priority="P0" source_item="Change 3: Pre-Regression Raw Ptseries">
      <file path="fmri_first_level_proc/task_act_first_level.py" action="modify" />
      <description>
        Add raw ptseries extraction logic to the task_act run() function.
        When extract_raw_ptseries is true, extract ROI stats from the raw
        (pre-regression) scan using the validated template. Place before
        regression (after template validation, before censor generation).
      </description>
      <spec>
        1. In run(), after template validation block (around line 520-526) and
           before "# Generate censor file from motion regressors":
           - Add block:
             if getattr(args, 'extract_raw_ptseries', False):
                 raw_out = os.path.join(args.out_dir,
                     f"{args.extract_out_file_pre}_raw_ptseries.csv")
                 if not os.path.exists(raw_out):
                     if args.template_path is None or args.template_path == "WB":
                         logger.error("extract_raw_ptseries requires a valid template_path.")
                         sys.exit(1)
                     # Validate template for extraction if not already done
                     validated_tpl = validate_template(args.scan_path, args.template_path,
                         args.out_dir, args.force_diff_atlas, conn_type="extract", logger=logger)
                     raw_df = extract_roi_stats(args.scan_path, validated_tpl,
                         args.average_type, logger=logger)
                     raw_df.to_csv(raw_out)
                     if os.path.exists(raw_out):
                         logger.info("Extracted raw (pre-regression) ptseries to %s.", raw_out)
                     else:
                         logger.error("Failed to extract raw ptseries.")
                         sys.exit(1)
                 else:
                     logger.info("Raw ptseries already exists: %s", raw_out)

           Note: template_path will already be validated by the extraction block
           above when extract=true, so we reuse args.template_path directly. The
           validate_template call is a safety net for the case where
           extract_raw_ptseries=true but extract=false (template may not have
           been validated yet). Actually, the simpler approach: just ensure
           template validation happens if extract_raw_ptseries is true.
           Refine: check if args.template_path is already validated or "WB".

        2. Add --extract_raw_ptseries to the CLI argparse block (main()):
           parser.add_argument("--extract_raw_ptseries", action='store_true',
               default=False, required=False,
               help="Extract pre-regression parcellated time series.")
      </spec>
      <dependencies>C7</dependencies>
      <risk>medium - template validation path needs care (already validated vs not);
             extract_roi_stats is proven infrastructure; skip-if-exists is safe</risk>
      <rollback>Remove the added block and CLI argument</rollback>
    </change>

    <change id="C9" priority="P0" source_item="Change 3: Pre-Regression Raw Ptseries">
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <description>
        Add raw ptseries extraction logic to the task_conn run() function.
        Same output file name as task_act (skip-if-exists handles dedup).
        Place before regression (after template validation, before censor).
      </description>
      <spec>
        1. In run(), after the connectivity validation block and before
           "# Generate censor file from motion regressors":
           - Add block (same pattern as task_act C8):
             if getattr(args, 'extract_raw_ptseries', False):
                 # Need template and extraction prefix
                 if args.extract_out_file_pre is None:
                     logger.error("extract_raw_ptseries requires extract_out_file_pre.")
                     sys.exit(1)
                 raw_out = os.path.join(args.out_dir,
                     f"{args.extract_out_file_pre}_raw_ptseries.csv")
                 if not os.path.exists(raw_out):
                     if args.template_path is None:
                         logger.error("extract_raw_ptseries requires a valid template_path.")
                         sys.exit(1)
                     validated_tpl = validate_template(args.scan_path, args.template_path,
                         args.out_dir, args.force_diff_atlas, conn_type="extract", logger=logger)
                     raw_df = extract_roi_stats(args.scan_path, validated_tpl,
                         args.average_type, logger=logger)
                     raw_df.to_csv(raw_out)
                     if os.path.exists(raw_out):
                         logger.info("Extracted raw (pre-regression) ptseries to %s.", raw_out)
                     else:
                         logger.error("Failed to extract raw ptseries.")
                         sys.exit(1)
                 else:
                     logger.info("Raw ptseries already exists: %s", raw_out)

        2. Add --extract_raw_ptseries to CLI argparse (main()):
           parser.add_argument("--extract_raw_ptseries", action='store_true',
               default=False, required=False,
               help="Extract pre-regression parcellated time series.")
      </spec>
      <dependencies>C7</dependencies>
      <risk>medium - same template validation considerations as C8</risk>
      <rollback>Remove the added block and CLI argument</rollback>
    </change>

    <change id="C10" priority="P0" source_item="Change 3: Pre-Regression Raw Ptseries">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>
        Add raw ptseries extraction logic to the rest_conn run() function.
        Per-run output: {extract_out_file_pre}_run{N}_raw_ptseries.csv.
        Place before regression (after template validation, before gen_residual_ts).
      </description>
      <spec>
        1. In run(), after the connectivity validation block and before
           "# Build QC summary data":
           - Add block:
             if getattr(args, 'extract_raw_ptseries', False):
                 if args.extract_out_file_pre is None:
                     logger.error("extract_raw_ptseries requires extract_out_file_pre.")
                     sys.exit(1)
                 if args.template_path is None:
                     logger.error("extract_raw_ptseries requires a valid template_path.")
                     sys.exit(1)
                 validated_tpl = validate_template(args.scan_paths[0], args.template_path,
                     args.out_dir, args.force_diff_atlas, conn_type="extract", logger=logger)
                 for i, scan in enumerate(args.scan_paths):
                     raw_out = os.path.join(args.out_dir,
                         f"{args.extract_out_file_pre}_run{i+1}_raw_ptseries.csv")
                     if not os.path.exists(raw_out):
                         raw_df = extract_roi_stats(scan, validated_tpl,
                             args.average_type, logger=logger)
                         raw_df.to_csv(raw_out)
                         if os.path.exists(raw_out):
                             logger.info("Extracted raw ptseries for run %d to %s.", i+1, raw_out)
                         else:
                             logger.error("Failed to extract raw ptseries for run %d.", i+1)
                             sys.exit(1)
                     else:
                         logger.info("Raw ptseries for run %d already exists: %s", i+1, raw_out)

        2. Add --extract_raw_ptseries to CLI argparse (main()):
           parser.add_argument("--extract_raw_ptseries", action='store_true',
               default=False, required=False,
               help="Extract pre-regression parcellated time series (per-run).")
      </spec>
      <dependencies>C7</dependencies>
      <risk>medium - per-run extraction with template validated once on first scan</risk>
      <rollback>Remove the added block and CLI argument</rollback>
    </change>

    <!-- ===================================================================== -->
    <!-- CHANGE: Config and documentation updates                               -->
    <!-- ===================================================================== -->
    <change id="C11" priority="P1" source_item="All changes">
      <file path="example_config.yaml" action="modify" />
      <description>
        Add extract_raw_ptseries to the extraction block of all three analysis
        type examples. Also add a comment about motion units.
      </description>
      <spec>
        1. In task_act extraction block: add extract_raw_ptseries: false with comment
        2. In task_conn extraction block: add extract_raw_ptseries: false with comment
        3. In rest_conn extraction block: add extract_raw_ptseries: false with comment
      </spec>
      <dependencies>C7</dependencies>
      <risk>low - documentation config example</risk>
      <rollback>Revert edits</rollback>
    </change>

    <change id="C12" priority="P1" source_item="All changes">
      <file path="INPUT_SPECIFICATION.md" action="modify" />
      <description>
        Document the extract_raw_ptseries option and min-outlier EPI output
        in INPUT_SPECIFICATION.md. Add output file descriptions.
      </description>
      <spec>
        1. In each analysis type section, add extract_raw_ptseries to the
           extraction parameter table
        2. In the Output Files section, add min_outlier_epi and raw_ptseries
           output descriptions
      </spec>
      <dependencies>C1, C3, C7</dependencies>
      <risk>low - documentation only</risk>
      <rollback>Revert edits</rollback>
    </change>

    <change id="C13" priority="P2" source_item="All changes">
      <file path="fmri_first_level_proc/__init__.py" action="modify" />
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <file path="fmri_first_level_proc/first_level_config.py" action="modify" />
      <file path="fmri_first_level_proc/task_act_first_level.py" action="modify" />
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <file path="pyproject.toml" action="modify" />
      <description>
        Bump version to 2.4.0 across all files that contain version strings.
        Update "Last updated" dates in file headers.
      </description>
      <spec>
        - __init__.py: __version__ = "2.4.0"
        - pyproject.toml: version = "2.4.0"
        - first_level_utils.py header: Version: 2.4.0, Last updated: 04/02/26
        - first_level_config.py header: Version: 2.4.0, Last updated: 04/02/26
        - task_act_first_level.py header: Version: 2.4.0, Last updated: 04/02/26
        - task_conn_first_level.py header: Version: 2.4.0, Last updated: 04/02/26
        - rest_conn_first_level.py header: Version: 2.4.0, Last updated: 04/02/26
      </spec>
      <dependencies>C1-C12</dependencies>
      <risk>low - version bump only</risk>
      <rollback>Revert version strings to 2.3.1</rollback>
    </change>
  </changes>

  <execution_order>C1, C2, C3, C4, C5, C6, C7, C8, C9, C10, C11, C12, C13</execution_order>

  <!-- Notes:
    - C1-C2 (motion units): Documentation-only, no dependencies
    - C3 (gen_min_outlier_epi utility): Must precede C4-C6 (pipeline integrations)
    - C7 (config validation): Must precede C8-C10 (pipeline integrations)
    - C4-C6 and C8-C10 are independent of each other (different features)
    - C11-C12 (docs): depend on feature changes being defined
    - C13 (version bump): last, after all functional changes
  -->
</implement_plan>
