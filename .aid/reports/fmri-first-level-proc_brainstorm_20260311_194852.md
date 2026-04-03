<brainstorm_report>
  <meta project="fmri-first-level-proc" mode="brainstorm" timestamp="2026-03-11T19:48:52Z" />
  <context_files>
    <file path="FIX_fromOrch.txt" relevance="Primary input — external test run bug report identifying BUG-001 and BUG-002" />
    <file path="fmri_first_level_proc/first_level_utils.py" relevance="Contains notch_filter_motion() (BUG-001 line 671), valid_contrast_functions() (contrast parsing), compute_dof()" />
    <file path="fmri_first_level_proc/task_act_first_level.py" relevance="Reference implementation for correct contrast parse-then-drop ordering (lines 397, 462-473)" />
    <file path="fmri_first_level_proc/task_conn_first_level.py" relevance="Contains BUG-002 (drop logic at 496-508 before parse at 557); gen_conn_contrasts() contrast consumption" />
    <file path="fmri_first_level_proc/rest_conn_first_level.py" relevance="Contains run-level failure handling (gen_residual_ts lines 106-223); sole caller of notch_filter_motion()" />
    <file path="fmri_first_level_proc/first_level_config.py" relevance="YAML config validation and namespace construction for contrast functions" />
    <file path="tests/test_contrast_parser.py" relevance="Existing test coverage for valid_contrast_functions() — 30 tests including adversarial inputs" />
    <file path="example_config.yaml" relevance="Reference YAML config showing contrast specification for both task_act and task_conn" />
  </context_files>
  <topics>
    <topic id="T1" title="BUG-001: Trailing transpose operator in notch_filter_motion() causes universal rest_conn failure">
      <summary>
        The AFNI transpose operator (single-quote) on first_level_utils.py:671 was necessary when
        intermediate motion files used the .txt extension (AFNI reads .txt as column-major). Commit
        b450597 renamed all intermediate motion files to .1D (AFNI reads as row-major by default),
        making the transpose counterproductive. The double-transpose produces a 6-row x N-column
        matrix instead of N-row x 6-column. Downstream, the censor file has 6 entries, DOF computes
        as negative, and the skip-low-DOF logic fires for every run in every session — 100% rest_conn
        failure rate for any subject using notch_filter_band.

        Full provenance:
        - Pre-v2.1.0 (4945e38): "-input", f"{motion_6col}\\'" — escaped backslash + transpose on .txt
        - v2.1.0 (29782b7): "-input", f"{motion_6col}'" — removed backslash, kept transpose on .txt (correct)
        - Post-v2.1.0 (b450597): .1D rename — transpose now double-transposes .1D files (broken)

        notch_filter_motion() is only called from rest_conn_first_level.py:121. The input is always
        .1D at the point of use (prepare_motion_file() inside notch_filter_motion() always outputs .1D).
        No conditional logic is needed.

        The regression test mentioned in session history (test_notch_filter_no_backslash) no longer
        exists in the test suite. There is zero test coverage for notch_filter_motion().
      </summary>
      <approaches>
        <approach id="A1" label="Remove trailing single-quote" feasibility="high" risk="low">
          <description>Delete the trailing single-quote from the f-string on line 671, changing
            f"{motion_6col}'" to f"{motion_6col}". Single-character fix.</description>
          <pros>Minimal diff; directly addresses root cause; no conditional logic needed since all
            call paths produce .1D files.</pros>
          <cons>None identified.</cons>
        </approach>
      </approaches>
      <decision status="decided" chosen="A1">
        The fix is unambiguous. The transpose operator is no longer needed after the .1D extension
        standardization. A dimensionality-checking regression test (assert output_rows == input_rows)
        should be added to prevent future regressions — the previous test only checked for the
        backslash character, not functional correctness.
      </decision>
    </topic>

    <topic id="T2" title="BUG-002: task_conn condition-drop logic accesses unparsed contrast strings">
      <summary>
        In task_conn_first_level.py, the condition-drop block (lines 496-508) accesses
        cont["CONDS"] on raw contrast strings before valid_contrast_functions() parses them
        into dicts at line 557. This causes TypeError when conditions are dropped due to
        insufficient surviving trials.

        In task_act_first_level.py, the ordering is correct: parse at line 397, drop at lines
        462-473. The task_conn pipeline must be reordered to match.

        Two fix options were evaluated:
        - Option A (chosen): Move valid_contrast_functions() before the drop block; parse with
          original_conds (the full condition list before filtering); assign result back to
          args.contrast_functions; remove redundant second call at line 557.
        - Option B (rejected): Move drop block after line 557 — rejected because
          valid_contrast_functions() calls sys.exit(1) when a contrast references a condition
          not in cond_list. After condition filtering, dropped conditions are removed from
          cond_beta_labels, causing valid_contrast_functions() to fatally reject contrasts
          referencing those dropped conditions.

        Option A mirrors the proven task_act pattern and eliminates the redundant parse call.
      </summary>
      <approaches>
        <approach id="A2" label="Parse-first, then drop (mirror task_act)" feasibility="high" risk="low">
          <description>
            1. Call valid_contrast_functions() with original_conds (before condition filtering)
            2. Assign parsed result back to args.contrast_functions
            3. Run the existing drop block on parsed dicts (cont["CONDS"] now valid)
            4. Remove the second valid_contrast_functions() call at line 557
            5. Pass args.contrast_functions directly to gen_conn_contrasts()
          </description>
          <pros>Mirrors proven task_act pattern; eliminates redundant parsing; consistent
            codebase; all downstream consumers receive parsed dicts.</pros>
          <cons>Slightly larger diff than Option B, but structurally cleaner.</cons>
        </approach>
        <approach id="A3" label="Move drop block after second parse call" feasibility="low" risk="high">
          <description>Move the drop block to after line 557 and operate on parsed_contrasts.</description>
          <pros>Smaller diff.</pros>
          <cons>valid_contrast_functions() would sys.exit(1) when contrasts reference dropped
            conditions not in the filtered cond_beta_labels. Fatal failure on the exact scenario
            the drop logic is designed to handle gracefully.</cons>
        </approach>
      </approaches>
      <decision status="decided" chosen="A2">
        Option A is the only viable approach. Option B causes fatal exits in the exact scenario
        requiring graceful handling. The implementation should follow the task_act pattern exactly:
        parse with full condition list, then filter parsed results.
      </decision>
    </topic>

    <topic id="T3" title="Contrast function validation audit (activation and connectivity)">
      <summary>
        Full lifecycle audit of contrast functions from user input through AFNI execution.

        Entry paths: CLI (valid_string_list splits comma-delimited) and YAML config
        (build_namespace reads list[str] from config block). Both produce list[str] or None.

        Config validation (validate_config): checks functions/labels length parity and
        co-presence. Does NOT validate equation syntax or condition references (deferred to
        runtime — appropriate since config validation runs before timing data is loaded).

        Runtime parsing (valid_contrast_functions): regex-based parser with _FULL_RE grammar
        validation, condition reference check against cond_list, coefficient float validation,
        and zero-coefficient warning. 30 tests in test_contrast_parser.py including adversarial
        inputs (injection attempts, malformed syntax, edge cases).

        Leading negative coefficients: confirmed correct through full trace. Regex captures
        sign as part of coefficient string. SYM equation builder (task_act) and 3dcalc
        expression builder (task_conn gen_conn_contrasts) both preserve sign fidelity.
        Covered by test_leading_negative, test_leading_negative_decimal, test_leading_negative_sym.

        Downstream consumption verified in both pipelines:
        - task_act: SYM equations for 3dDeconvolve GLTs
        - task_conn: parcellated (compute_matrix_contrast) and seed-to-voxel (3dcalc)

        Edge cases verified:
        - All contrasts dropped → empty list; task_act truthiness check skips; task_conn
          iterates empty zip (no-op). Both correct.
        - None vs empty list: None is falsy, skipped by all guard clauses. Correct.
        - Non-zero-sum coefficients: not validated (deliberate — AFNI accepts weighted means).
          Could add warning as future quality-of-life enhancement, deferred.

        No additional bugs found. The only issue is BUG-002 (ordering in task_conn).
      </summary>
      <approaches>
        <approach id="A4" label="No changes needed beyond BUG-002 fix" feasibility="high" risk="low">
          <description>The contrast validation pipeline is robust. The BUG-002 fix (T2/A2)
            addresses the only identified issue. No additional changes required.</description>
          <pros>Avoids unnecessary code churn; existing test coverage is comprehensive.</pros>
          <cons>None. Non-zero-sum warning deferred as low-priority enhancement.</cons>
        </approach>
      </approaches>
      <decision status="decided" chosen="A4">
        Contrast validation is sound. BUG-002 fix resolves the only issue. Future enhancement
        (non-zero-sum warning) deferred — not a correctness issue.
      </decision>
    </topic>

    <topic id="T4" title="Resting-state run-level failure handling verification">
      <summary>
        The gen_residual_ts() function in rest_conn_first_level.py has four layers of
        run-level failure protection:

        Layer 1 — Pre-flight DOF skip (lines 151-153): Runs with insufficient DOF (computed
        via compute_dof with exit_on_error=False) are skipped before 3dTproject is called.

        Layer 2 — 3dTproject exception handling (lines 184-188): Exceptions from
        run_afni_command are caught; the run is skipped and the loop continues.

        Layer 3 — Output existence check (lines 190-196): If the output file doesn't exist
        after 3dTproject, the run is not added to completed_runs. The loop proceeds.

        Layer 4 — All-runs-failed guard (lines 198-200): Only if every run fails does the
        pipeline abort with sys.exit(1). If at least one run succeeds, concatenation proceeds
        with completed_runs only.

        Concatenation (line 203): 3dTcat receives only completed_runs paths.

        Architecture is sound. The universal failure observed in the integration test run was
        caused by BUG-001 corrupting notch_filter_motion output, which corrupted the censor
        file, which made every run's DOF negative, triggering Layer 1 for all runs, which
        triggered Layer 4. This is correct defensive behavior given corrupted input — the
        failure handling worked as designed; the input was wrong.

        Once BUG-001 is fixed, the censor files will have correct row counts, DOF will be
        positive for healthy runs, and the skip logic will only trigger for genuinely
        low-DOF runs.
      </summary>
      <approaches>
        <approach id="A5" label="No changes needed" feasibility="high" risk="low">
          <description>The run-level failure handling architecture is correct. BUG-001 fix
            restores correct inputs to the DOF check. No changes to rest_conn logic needed.</description>
          <pros>Avoids unnecessary changes to validated defensive logic.</pros>
          <cons>None identified.</cons>
        </approach>
      </approaches>
      <decision status="decided" chosen="A5">
        Run-level failure handling is architecturally sound. The observed universal failure was
        a symptom of BUG-001 (corrupted notch filter output), not a flaw in the handling logic.
        Fixing BUG-001 resolves the symptom without requiring changes to rest_conn.
      </decision>
    </topic>
  </topics>
  <action_items>
    <item priority="P0" target_mode="implement" description="BUG-001: Remove trailing single-quote from first_level_utils.py:671 (notch_filter_motion -input argument). Update comment on line 667." />
    <item priority="P0" target_mode="test" description="BUG-001: Add regression test for notch_filter_motion() verifying output_rows == input_rows with .1D input." />
    <item priority="P1" target_mode="implement" description="BUG-002: In task_conn_first_level.py, move valid_contrast_functions() call before the condition-drop block. Parse with original_conds. Assign result to args.contrast_functions. Remove redundant parse call at line 557. Pass args.contrast_functions to gen_conn_contrasts()." />
    <item priority="P1" target_mode="test" description="BUG-002: Add integration test for task_conn with dropped conditions + contrasts — verify graceful handling (contrasts referencing dropped conditions skipped, others computed correctly)." />
  </action_items>
  <next_steps>
    Proceed to /implement mode to apply both fixes. BUG-001 first (P0, single-character fix with
    comment update), then BUG-002 (P1, structural reorder mirroring task_act). Follow with /test
    mode to add regression tests for both bugs and verify the full 164+ test suite passes.
  </next_steps>
</brainstorm_report>
