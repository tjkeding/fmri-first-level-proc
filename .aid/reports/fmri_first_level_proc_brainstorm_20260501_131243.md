<brainstorm_report>
  <meta project="fmri_first_level_proc" mode="brainstorm" timestamp="2026-05-01T17:12:43Z" />

  <context_files>
    <file path="ciric_et_al_bandpass_implementation.md" relevance="User-provided pseudoalgorithm for the new sequenced bandpass path. Source of the six-step procedure (confounds, NTRP interpolation, 3dBandpass, nuisance regression)." />
    <file path="fmri_first_level_proc/rest_conn_first_level.py" relevance="Current resting-state pipeline. Target of the new use_sequenced_bandpass branch. gen_residual_ts (lines 107-243) is the locus of the architectural change." />
    <file path="fmri_first_level_proc/first_level_utils.py" relevance="Shared utilities. Existing notch_filter_motion (line 668) provides the apostrophe-transpose pattern to be replicated for new 1D filtering helpers. compute_tissue_derivative (line 734) is the upstream of the derivative regressors." />
    <file path="fmri_first_level_proc/first_level_config.py" relevance="YAML config loader and validator. Target for use_sequenced_bandpass key (BOOL_KEYS list, rest_conn-specific validation, build_namespace)." />
  </context_files>

  <topics>

    <topic id="T1" title="Fidelity to Ciric specification: interpolation method, polort, censor placement">
      <summary>The sandbox markdown's pseudoalgorithm deviates from the canonical Ciric et al. (2017/2018) specification on three points: (a) AFNI NTRP linear interpolation rather than Lomb-Scargle frequency-domain surrogate interpolation; (b) polort -1 (no detrending) rather than polort 2 (mean + linear + quadratic); (c) censoring applied at the interpolation step rather than as a final-residual re-censor. Decision is the level of fidelity to assume for the implementation.</summary>
      <research>
        Ciric et al. (2017, NeuroImage; PMID 28302591) and Ciric et al. (2018, Nature Protocols; PMID 30446748) prescribe (via xcpEngine reference implementation): TMP-REG ordering (filter then regress); frequency-domain surrogate interpolation for censored TRs (Lomb-Scargle / Power 2014 spectral reconstruction); polort 2 detrending applied uniformly; final-residual re-censoring [R1]. AFNI's 3dTproject -cenmode NTRP implements piecewise linear interpolation with constant-fill at run boundaries; AFNI developer (Reynolds) explicitly cautioned that NTRP inflates autocorrelation in the output time series, a known artifact relevant to downstream connectivity [R3]. Power et al. (2014) demonstrated that uninterpolated censored frames produce Gibbs-like ringing in adjacent uncensored TRs through bandpass impulse response; Lomb-Scargle interpolation reduces this artifact more effectively than linear interpolation [R2].
      </research>
      <approaches>
        <approach id="A1" label="Markdown verbatim" feasibility="high" risk="med">
          <description>Implement the sandbox pseudoalgorithm exactly as written: NTRP linear interpolation, polort -1 throughout, no final-residual re-censor. Documented as Ciric-inspired rather than Ciric-exact.</description>
          <pros>AFNI-only implementation. Simple. Internally consistent with current pipeline's polort -1 convention.</pros>
          <cons>Residuals contain interpolated values at originally-censored TRs, contaminating downstream connectivity correlations. NTRP autocorrelation inflation passes through unchecked. Reviewer-defensibility concern if option named after Ciric does not match Ciric.</cons>
          <statistical_considerations>Linear interpolation is a defensible approximation of frequency-domain surrogate interpolation in many AFNI-based pipelines (CBIG benchmarks comparable performance). The principal weakness is the censored-TR contamination of connectivity, not the interpolation method itself.</statistical_considerations>
        </approach>
        <approach id="A2" label="Canonical Ciric" feasibility="med" risk="med">
          <description>Lomb-Scargle frequency-domain surrogate interpolation (scipy.signal.lombscargle or astropy.timeseries), polort 2, final-residual re-censor.</description>
          <pros>Faithful to published specification. Defensible by direct citation.</pros>
          <cons>Adds scipy as runtime dependency where currently only numpy/pandas required. Several hundred lines of new custom code (Lomb-Scargle reconstruction wrapper). Higher test surface.</cons>
        </approach>
        <approach id="A3" label="Markdown + polort 2" feasibility="high" risk="med">
          <description>NTRP linear interpolation, polort 2 to match Ciric detrending, no final-residual re-censor.</description>
          <pros>Single-line change relative to A1; modest defensibility gain.</pros>
          <cons>Inconsistent with rest_conn's polort -1 convention; censored-TR contamination unaddressed.</cons>
        </approach>
        <approach id="A4" label="Markdown + final-residual re-censor (CHOSEN)" feasibility="high" risk="low">
          <description>NTRP linear interpolation, polort -1 throughout, plus a final post-regression step that zeros the residuals at originally-censored TRs.</description>
          <pros>AFNI-only. Internally consistent with current pipeline's polort convention. Addresses the most concrete data-quality concern (interpolated values contaminating connectivity) without taking on Lomb-Scargle complexity. Documents as Ciric-inspired with linear-interpolation approximation.</pros>
          <cons>Inherits NTRP autocorrelation-inflation caveat (R3, Reynolds AFNI message-board guidance); bandpass impulse response on linearly-interpolated boundaries less faithful than frequency-domain surrogate.</cons>
        </approach>
      </approaches>
      <decision status="decided" chosen="A4">
        Implement the markdown pseudoalgorithm verbatim (NTRP linear interpolation, polort -1) with one explicit addition: a final post-regression step that zeros the residuals at originally-censored TRs. The config flag is renamed to use_sequenced_bandpass to honestly describe the architecture rather than claim Ciric fidelity.
      </decision>
    </topic>

    <topic id="T2" title="Mechanism of final-residual re-censor at step 6">
      <summary>A4 commits to producing residuals with zeros at originally-censored TRs (matching the current pipeline's output convention). Mechanism choice: pass -censor + -cenmode flags to step 6's 3dTproject call, or apply a post-step zeroing operation.</summary>
      <research>
        AFNI 3dTproject -cenmode ZERO replaces censored values with zeros BEFORE projection. With -ort regressors that are themselves continuous (NTRP-interpolated + bandpassed in the sequenced path), zeroing only the BOLD reintroduces a coefficient-estimation bias because nuisance regressors retain non-zero values at the same TRs. -cenmode KILL shortens output, breaking concatenation alignment. -cenmode NTRP at step 6 is a no-op since BOLD is already interpolated [R3]. Applying censoring as a post-step (3dcalc -expr 'a*b' with the censor.1D as the second operand) is the cleanest separation of concerns, since 3dcalc accepts 1D files and broadcasts across voxels along the time axis [R5 / AFNI standard].
      </research>
      <decision status="decided" chosen="B3">
        Step 6 is 3dTproject with -ort regressors and no -censor flag. A 3dcalc post-step zeroes the residual at originally-censored TRs by element-wise multiplication with the censor.1D file, broadcast across voxels along the time axis. This preserves clean coefficient estimation on the continuous filtered signals and isolates the censoring as a transparent output-masking step.
      </decision>
    </topic>

    <topic id="T3" title="Bandpass tool choice for sequenced path">
      <summary>Markdown specifies 3dBandpass for both BOLD and nuisance bandpass steps. AFNI's recommendation of 3dTproject over 3dBandpass applies specifically to the simultaneous-regress-and-filter case, not as a general deprecation. Choice between 3dBandpass and 3dTproject -bandpass for the sequenced path's standalone bandpass step.</summary>
      <research>
        3dBandpass implements FFT-based brick-wall filtering with zero-padding; 3dTproject -bandpass implements regression-based filtering by projecting out sine/cosine basis vectors at stopband frequencies [R4]. The two are mathematically non-equivalent in general but produce essentially equivalent output for typical resting-state run lengths (200-600 TRs) [R4]. AFNI's recommendation to use 3dTproject over 3dBandpass is contextual: it applies to the "do bandpass and regression in one step" use case, not as a deprecation. 3dBandpass remains supported, is used internally by 3dTproject, and shares its core algorithm. Both tools offer -nodetrend to disable polynomial detrending at this step.
      </research>
      <decision status="decided" chosen="C1">
        Use 3dBandpass -nodetrend for both the BOLD bandpass (step 4) and the nuisance regressor bandpass (step 5). Algorithm consistency across BOLD and regressors; markdown-faithful; no AFNI-deprecation concern (3dBandpass remains supported and is used internally by 3dTproject).
      </decision>
    </topic>

    <topic id="T4" title="Nuisance regressor filtering pattern (1D handling)">
      <summary>Each nuisance 1D file (motion, CSF, WM, optional GS, optional derivatives) must be processed through (a) censor-interpolation via 3dTproject -cenmode NTRP and (b) bandpass via 3dBandpass. Both AFNI tools follow the standard 1D convention (rows=voxels, cols=timepoints) and require the apostrophe-transpose round-trip for files written with rows=TRs.</summary>
      <decision status="decided" chosen="D1">
        Add two shared utilities to first_level_utils.py: censor_interpolate_1d_afni() and bandpass_filter_1d_afni(). Both wrap their respective AFNI command in the apostrophe-transpose round-trip pattern established by notch_filter_motion. Each 1D nuisance regressor flows through censor_interpolate_1d_afni -> bandpass_filter_1d_afni in the sequenced backend. The BOLD path uses inline 3dTproject (censor-interp) and 3dBandpass calls without transpose (volumetric input).
      </decision>
    </topic>

    <topic id="T5" title="Implementation architecture inside rest_conn">
      <summary>Simultaneous and sequenced paths share approximately 60 lines of setup (motion notch-filter, censor file generation, motion preparation, DOF check, run-level skip-on-low-DOF, concatenation, optional cleanup) and diverge only in per-run residual generation. Sub-decisions on tissue-derivative ordering and intermediate-file management.</summary>
      <decision status="decided" chosen="E4">
        Refactor gen_residual_ts so that motion preparation, censor file generation, DOF computation, run-level skip logic, concatenation, and cleanup remain in the function body. Extract per-run residual generation to two private helpers: _generate_run_residual_simultaneous (preserving the current 3dTproject -ort + -bandpass + -censor command) and _generate_run_residual_sequenced (implementing the six-step Ciric-inspired flow plus the post-step 3dcalc zero). Dispatch via if args.use_sequenced_bandpass.
      </decision>

      <subdecision id="S5a" title="Tissue-derivative computation order">
        <decision status="decided" chosen="S5a-1">Derivatives-on-raw, then filter both base and derivative regressors through the same NTRP-interp + 3dBandpass pipeline.</decision>
      </subdecision>

      <subdecision id="S5b" title="Intermediate-file management">
        <decision status="decided" chosen="S5b-ii">Write per-run intermediates to {out_dir}/_sequenced_intermediates/{run_label}/. Cleanup is run-scoped (clean each run's intermediates as soon as that run's final residual is produced). Cleanup behavior is tied to the existing --keep_run_res_dtseries flag: when True, intermediates are retained; when False, they are removed.</decision>
      </subdecision>
    </topic>

  </topics>

  <action_items>
    <item priority="P0" target_mode="implement" description="Add use_sequenced_bandpass key to first_level_config.py (BOOL_KEYS, rest_conn validation, build_namespace)." />
    <item priority="P0" target_mode="implement" description="Add CLI flag --use_sequenced_bandpass to rest_conn_first_level.py argparse." />
    <item priority="P0" target_mode="implement" description="Implement censor_interpolate_1d_afni() in first_level_utils.py (3dTproject -cenmode NTRP + 1dtranspose round-trip)." />
    <item priority="P0" target_mode="implement" description="Implement bandpass_filter_1d_afni() in first_level_utils.py (3dBandpass -nodetrend + 1dtranspose round-trip)." />
    <item priority="P0" target_mode="implement" description="Refactor gen_residual_ts per E4 hybrid: shared setup retained, per-run residual generation dispatched to two backends." />
    <item priority="P0" target_mode="implement" description="Implement _generate_run_residual_simultaneous as direct extraction of current per-run logic." />
    <item priority="P0" target_mode="implement" description="Implement _generate_run_residual_sequenced (six-step flow + 3dcalc post-step + intermediate-dir lifecycle)." />
    <item priority="P0" target_mode="implement" description="Update example_config.yaml to demonstrate use_sequenced_bandpass under rest_conn." />
    <item priority="P1" target_mode="test" description="Unit tests for censor_interpolate_1d_afni and bandpass_filter_1d_afni." />
    <item priority="P1" target_mode="test" description="Integration tests for _generate_run_residual_sequenced." />
    <item priority="P1" target_mode="test" description="Dispatch tests for refactored gen_residual_ts." />
    <item priority="P1" target_mode="test" description="Config-validation tests for use_sequenced_bandpass." />
    <item priority="P1" target_mode="test" description="Regression tests confirming bitwise-identical simultaneous-path output with flag False." />
    <item priority="P2" target_mode="document" description="Update rest_conn_first_level.py module docstring and gen_residual_ts docstring to describe the two paths." />
    <item priority="P2" target_mode="document" description="Update INPUT_SPECIFICATION.md to document the use_sequenced_bandpass key." />
    <item priority="P2" target_mode="document" description="Update README.md key features to mention the optional sequenced bandpass mode." />
    <item priority="P2" target_mode="document" description="Add docstrings to the two new utilities (censor_interpolate_1d_afni, bandpass_filter_1d_afni)." />
  </action_items>

  <next_steps>
    Recommended downstream mode: /implement plan. The brainstorm decisions are concrete enough to translate directly into a tech spec for the build phase. Plan should: (a) sequence the implementation as utilities-first; (b) identify the exact line ranges in first_level_config.py and rest_conn_first_level.py that change; (c) specify the intermediate-directory naming convention and cleanup semantics in code-level detail; (d) propose a versioning bump (likely v2.5.0 given the new public config option). After /implement plan is finalized, /test design should produce the regression-test specification, then /implement build executes per spec.
  </next_steps>

</brainstorm_report>
