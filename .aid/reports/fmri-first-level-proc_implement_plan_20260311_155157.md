<?xml version="1.0" encoding="UTF-8"?>
<implement_plan>
  <meta project="fmri-first-level-proc" mode="implement" submodule="plan" timestamp="2026-03-11T19:51:57Z" />
  <input_reports>
    <report path="<local_path>/fmri-first-level-proc_brainstorm_20260311_194852.md" mode="brainstorm" key_items="2" />
  </input_reports>
  <changes>
    <change id="C1" priority="P0" source_item="T1/A1">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>
        Remove trailing single-quote (AFNI transpose operator) from the -input argument
        in notch_filter_motion() on line 671. Update the comment on line 667 to remove
        the now-incorrect reference to the transpose operator.

        Root cause: commit b450597 standardized intermediate motion files to .1D extension.
        AFNI reads .1D files as row-major (N-rows x 6-cols) by default — the transpose
        operator is not needed and counterproductively produces a 6-rows x N-cols matrix,
        causing all downstream DOF checks to fail.
      </description>
      <spec>
        Line 667 comment:
          OLD: "# 3dTproject on 1D file: use \' (AFNI transpose) so rows=timepoints, cols=params"
          NEW: "# 3dTproject on .1D file: rows=timepoints, cols=motion params (no transpose needed)"

        Line 671 f-string:
          OLD: "-input", f"{motion_6col}'"
          NEW: "-input", f"{motion_6col}"
      </spec>
      <dependencies>none</dependencies>
      <risk>low - single-character removal; no logic change; .1D format confirmed row-major in AFNI</risk>
      <rollback>Restore the trailing single-quote on line 671</rollback>
    </change>

    <change id="C2" priority="P1" source_item="T2/A2">
      <file path="fmri_first_level_proc/task_conn_first_level.py" action="modify" />
      <description>
        Reorder the contrast parsing and condition-drop logic in task_conn_first_level.py
        to mirror the proven task_act pattern:

        1. Move valid_contrast_functions() call to before the condition-drop block,
           passing original_conds (full condition list before any filtering).
        2. Assign the parsed result directly to args.contrast_functions.
        3. Keep the existing drop block unchanged — it already operates on cont["CONDS"]
           which is now valid (parsed dicts instead of raw strings).
        4. Remove the second (redundant) valid_contrast_functions() call at line 557.
        5. Pass args.contrast_functions directly to gen_conn_contrasts() instead of parsed_contrasts.
        6. Also update the contrast guard block prior to the drop block to validate contrast
           function/label co-presence (mirroring task_act lines 388-395), and place the
           valid_contrast_functions() call inside this guard block.

        Current broken ordering (task_conn):
          line 485: original_conds = list(args.cond_beta_labels)
          line 486: args.cond_beta_labels = [filtered list]
          line 496-508: DROP BLOCK — accesses cont["CONDS"] on raw strings → TypeError
          ...
          line 557: valid_contrast_functions() — never reached on drop

        Correct ordering (mirroring task_act):
          Parse contrasts with original_conds (before any filtering)
          → args.contrast_functions = parsed dicts
          → DROP BLOCK — cont["CONDS"] is now valid
          → remove redundant second parse
      </description>
      <spec>
        Before the trial-survival block (around line 480), add a contrast guard and
        parse block:

          # Contrast-related checks
          if args.contrast_labels is not None:
              if args.contrast_functions is None:
                  logger.error("--contrast_functions must exist if --contrast_labels exist.")
                  sys.exit(1)
          else:
              if args.contrast_functions is not None:
                  logger.error("--contrast_labels must exist if --contrast_functions exist.")
                  sys.exit(1)

        After original_conds is assigned (line 485) but before the drop block:
          if args.contrast_functions is not None and args.contrast_labels is not None:
              args.contrast_functions = valid_contrast_functions(
                  args.contrast_functions, args.contrast_labels,
                  original_conds, logger=logger)

        Remove lines 555-565 (the second valid_contrast_functions() call and its guard):
          OLD:
            if args.contrast_functions is not None and args.contrast_labels is not None:
                parsed_contrasts = valid_contrast_functions(...)
                if args.calc_conn is not None:
                    gen_conn_contrasts(bseries_out, parsed_contrasts, ...)
                    qc_data["contrast_labels"] = args.contrast_labels
                else:
                    logger.warning(...)
          NEW:
            if args.contrast_functions is not None and args.contrast_labels is not None:
                if args.calc_conn is not None:
                    gen_conn_contrasts(bseries_out, args.contrast_functions, ...)
                    qc_data["contrast_labels"] = args.contrast_labels
                else:
                    logger.warning(...)
      </spec>
      <dependencies>none</dependencies>
      <risk>medium - structural reorder; but mirrors proven task_act pattern exactly</risk>
      <rollback>Revert to original ordering (move parse back to line 557, restore raw string access in drop block)</rollback>
    </change>
  </changes>
  <execution_order>C1, C2</execution_order>
</implement_plan>
