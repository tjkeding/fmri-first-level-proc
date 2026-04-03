# CLAUDE.md


## Project Overview:
This is a general purpose framework for doing first-level (within-participant post-processing including regression) analyses for fMRI data. Activation first-levels are available for task-based fMRI data and functional connectivity first-levels are available for both task-based and resting-state fMRI data. Written in Python and heavily utilizes AFNI.

This package is pip-installable from GitHub:
```
pip install git+https://github.com/tjkeding/fmri-first-level-proc.git
```

## Project Structure:
```
fmri_first_level_proc/               # Repository root
├── pyproject.toml                    # pip-installable packaging config
├── .gitignore                        # Ignored files when pushing to GitHub
├── README.md                         # GitHub README.md (primarily for future users)
├── INPUT_SPECIFICATION.toml          # GitHub input spec file (primarily for future LLMs)
├── fmri_first_level_proc/            # Python package
│   ├── __init__.py                   # Package init with public API exports
│   ├── first_level_utils.py          # Shared utilities (logging, AFNI wrappers, ROI extraction, connectivity)
│   ├── first_level_config.py         # YAML config loader and validator
│   ├── run_first_level.py            # Config-driven dispatch runner
│   ├── task_act_first_level.py       # Task activation analysis pipeline
│   ├── task_conn_first_level.py      # Task beta series / connectivity pipeline
│   └── rest_conn_first_level.py      # Resting-state residual / connectivity pipeline
├── environment.yaml                  # Conda environment specification
├── example_config.yaml               # Example YAML config for the package
├── example_subj.timing.csv           # Example task timing file
```

## Project Impact:
*   Meant as a free-standing, pip-installable Python package to make first-level processing of fMRI data easy to access, implement, and document
*   Users should not need to make changes to the package, so it should be robust enough to handle the vast majority of typical use cases
*   Will be called within a broader preprocessing pipeline so inputs and outputs can be standardized rigorously
