<implement_plan>
  <meta project="fmri_first_level_proc" mode="implement" submodule="plan" timestamp="2026-05-01T13:21:01Z" />

  <input_reports>
    <report path="fmri_first_level_proc_brainstorm_20260501_131243.md" mode="brainstorm" key_items="10" />
  </input_reports>

  <changes>

    <change id="C1" priority="P0" source_item="action_item[1]">
      <file path="fmri_first_level_proc/first_level_config.py" action="modify" />
      <description>Plumb the new use_sequenced_bandpass boolean key through config validation and Namespace construction so that downstream rest_conn execution sees the flag from YAML configs.</description>
      <spec>
1. Line 74: extend BOOL_KEYS tuple to include "use_sequenced_bandpass". New value:
   BOOL_KEYS = ("remove_previous", "include_motion_derivs", "use_tissue_derivs", "censor_prev_tr", "use_sequenced_bandpass")
2. No new entry needed in REQUIRED_KEYS["rest_conn"] (line 62-63): the key is optional with default False.
3. validate_config rest_conn block: the existing _merge_global_into_block BOOL_KEYS loop handles None -> False defaulting automatically once the key is in BOOL_KEYS.
4. build_namespace rest_conn branch (line 574 onward, immediately after the ns.use_tissue_derivs line at 584): add
   ns.use_sequenced_bandpass = bool(merged_block.get("use_sequenced_bandpass", False))
      </spec>
      <risk>low</risk>
    </change>

    <change id="C2" priority="P0" source_item="action_item[2]">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>Add the --use_sequenced_bandpass CLI flag to the rest_conn argparse block.</description>
      <spec>
In main() argparse block, after --use_tissue_derivs definition, add:
parser.add_argument("--use_sequenced_bandpass", action='store_true', default=False, required=False,
                    help="Use sequenced denoising path...")

Update the module-level docstring INPUTS section to mention --use_sequenced_bandpass.
      </spec>
      <risk>low</risk>
    </change>

    <change id="C3" priority="P0" source_item="action_item[3]">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>Add censor_interpolate_1d_afni() shared utility implementing the apostrophe-transpose round-trip pattern around 3dTproject -cenmode NTRP for 1D regressor files. Mirrors the structure of notch_filter_motion.</description>
      <spec>
Insert a new function immediately after notch_filter_motion (after line 732, before compute_tissue_derivative at 734).
Signature: def censor_interpolate_1d_afni(input_path, tr, censor_path, out_dir, out_prefix, label, logger):
Behavior: Build AFNI command with -polort -1, -dt, -input file.1D' (apostrophe-transpose), -censor, -cenmode NTRP, -prefix; run; verify; apply 1dtranspose round-trip; os.replace; log; return out_path.
NumPy-style docstring with Parameters, Returns, and one-paragraph summary covering AFNI command, 1D-orientation handling, and intended use.
      </spec>
      <risk>low-medium</risk>
    </change>

    <change id="C4" priority="P0" source_item="action_item[4]">
      <file path="fmri_first_level_proc/first_level_utils.py" action="modify" />
      <description>Add bandpass_filter_1d_afni() shared utility wrapping 3dBandpass -nodetrend with the same apostrophe-transpose round-trip.</description>
      <spec>
Insert immediately after censor_interpolate_1d_afni (from C3), before compute_tissue_derivative.
Signature: def bandpass_filter_1d_afni(input_path, tr, bandpass, out_dir, out_prefix, label, logger):
Behavior: Build 3dBandpass command with -nodetrend, -dt, -prefix, positional fbot/ftop, apostrophe-suffixed dataset path; run; verify; 1dtranspose round-trip; os.replace; log; return out_path.
NumPy-style docstring covering the -nodetrend/polort -1 alignment, transpose dance, and post-interpolation filtering intent.
      </spec>
      <risk>low-medium</risk>
    </change>

    <change id="C5" priority="P0" source_item="action_item[6]">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>Extract the existing per-run residual generation logic into _generate_run_residual_simultaneous() so the simultaneous path becomes a named, dispatchable backend (E4 hybrid). Behavior must be bitwise-identical to the v2.4.0-post1 simultaneous path.</description>
      <spec>
Add new private helper at module level, immediately before gen_residual_ts.
Signature: def _generate_run_residual_simultaneous(run_idx, run_scan, censor_path, prepared_motion, args, logger):
Body matches prior inline 3dTproject construction verbatim including argument order (-input, -dt, -censor, -cenmode ZERO, -ort motion, -ort CSF, -ort WM, optional GS, optional derivatives, -polort -1, optional -bandpass, -prefix), early-return on existing output, exception-to-None semantics.
Returns: run_out path on success or None on 3dTproject failure (caller treats None as skip-this-run via continue).
      </spec>
      <risk>medium - bitwise-identical extraction is required for regression test compatibility.</risk>
    </change>

    <change id="C6" priority="P0" source_item="action_item[7,8,9]">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>Add _generate_run_residual_sequenced() implementing the six-step Ciric-inspired flow plus 3dcalc post-step plus intermediate-directory lifecycle.</description>
      <spec>
Add new function immediately after _generate_run_residual_simultaneous.
Add imports: censor_interpolate_1d_afni, bandpass_filter_1d_afni from .first_level_utils; import shutil to stdlib block.
Signature: def _generate_run_residual_sequenced(run_idx, run_scan, censor_path, prepared_motion, args, logger):
Six-step flow with intermediate-dir creation/cleanup:
  Step 2-3: 3dTproject -polort -1 -censor X.1D -cenmode NTRP on BOLD (interpolated dtseries).
  Step 4: 3dBandpass -nodetrend on interpolated BOLD.
  Step 5: per-regressor (motion, CSF, WM, optional GS, optional tissue derivatives) NTRP-interp + bandpass via the new utilities. Tissue derivatives computed on RAW signal then routed through the same filter pipeline (S5a-1).
  Step 6: 3dTproject -polort -1 -ort filtered_motion -ort filtered_CSF [...] (no -bandpass, no -censor) on filtered BOLD.
  Post-step: 3dcalc -a residual.nii.gz -b censor.1D -expr 'a*b' to zero residuals at originally-censored TRs.
Intermediate-dir lifecycle: create {args.out_dir}/_sequenced_intermediates/run{N}/ at start; remove on success when not args.keep_run_res_dtseries; preserve on failure regardless of flag.
      </spec>
      <dependencies>C3, C4</dependencies>
      <risk>high</risk>
    </change>

    <change id="C7" priority="P0" source_item="action_item[5]">
      <file path="fmri_first_level_proc/rest_conn_first_level.py" action="modify" />
      <description>Refactor gen_residual_ts per E4 hybrid: retain shared setup; replace inline per-run 3dTproject construction with single dispatch line selecting between simultaneous and sequenced backends.</description>
      <spec>
Replace lines 175-216 of the per-run loop body with:
  if args.use_sequenced_bandpass:
      run_out = _generate_run_residual_sequenced(i, run_scan, censor_paths[i], prepared_motion[i], args, logger)
  else:
      run_out = _generate_run_residual_simultaneous(i, run_scan, censor_paths[i], prepared_motion[i], args, logger)
  if run_out is None:
      continue
  completed_runs.append(run_out)

Surrounding setup and teardown unchanged. DOF pre-flight check is identical for both paths (sequenced step-6 ort count matches simultaneous ort count).
Update gen_residual_ts docstring to describe dispatch and the two backends.
      </spec>
      <dependencies>C5, C6</dependencies>
      <risk>medium</risk>
    </change>

    <change id="C8" priority="P0" source_item="action_item[10]">
      <file path="example_config.yaml" action="modify" />
      <description>Document the new use_sequenced_bandpass option under the rest_conn block of the example config.</description>
      <spec>
In rest_conn block, immediately after use_tissue_derivs line, add:
    use_sequenced_bandpass: false  # true = use sequenced denoising path (Ciric-inspired)...
      </spec>
      <dependencies>C1</dependencies>
      <risk>low</risk>
    </change>

  </changes>

  <execution_order>C1, C2, C3, C4, C5, C6, C7, C8</execution_order>

</implement_plan>
