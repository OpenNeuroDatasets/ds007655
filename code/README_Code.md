# MorseEEG-ATP — Companion Analysis Code

This directory contains MATLAB and Python scripts for working with the **MorseEEG-ATP** auditory-sequence EEG and behavioral dataset. The workflow covers acquisition-file conversion, behavioral analysis, EEG preprocessing, spectral feature extraction, event-marker-locked EEG visualization, group-level spectral statistics, and exploratory cross-participant classification.

The scripts are designed to work with the BIDS-organized dataset. **Users of the released BIDS data do not need to run Step0**; Step0 requires acquisition-system exports and behavioral source files that are separate from the distributed BIDS recordings.

## 1. Workflow

| Step | Script | Function | Main inputs |
| --- | --- | --- | --- |
| 0 | `step0_hdf5_to_bids.m` | Convert g.tec recorder HDF5 and behavioral MAT files to raw BIDS-organized EEG and behavior | Acquisition HDF5 files and E-Prime behavioral MAT files |
| 1 | `step1_behavior_analysis.m` | Cross-task behavioral summaries, agreement measures, and Strong/Weak grouping | `participants.tsv` |
| 2 | `step2_preprocess.m` | EEG re-referencing, filtering, epoching, and trial quality control | BrainVision EEG, event TSV, and behavioral TSV files |
| 3 | `step3_extract_PSD.m` | Extract trial-level power spectral density (PSD) features | Step2 preprocessed EEG |
| 4 | `step4_ERP_analysis.m` | Produce event-marker-locked EEG waveforms and scalp maps | Step2 preprocessed EEG and channel locations |
| 5 | `step5_PSD_ranksum_FDR.m` | Compare participant-level PSD between behavioral groups | Step1 grouping and Step3 PSD outputs |
| 6 | `step6_PSD_LOSO_classification.py` | Run the Python/PyTorch leave-one-subject-out (LOSO) PSD classification benchmarks | Step1 grouping and Step3 PSD outputs |

For an existing BIDS dataset, run **Step1 and Step2**, then **Step3**. Step4 uses Step2 results; Steps5 and 6 use Step1 and Step3 results. The scripts can be run in numerical order:

```text
Step1 -> Step2 -> Step3 -> Step4 -> Step5 -> Step6 (Python)
```

To create a raw BIDS-organized dataset from acquisition exports, run **Step0** first and then the applicable analysis steps.

## 2. Software requirements

MATLAB Steps0–5 were prepared for the following reference software environment:

- Windows
- MATLAB R2022b
- EEGLAB v2024.0

Step6 runs with **Python 3**, NumPy, SciPy, scikit-learn, PyTorch and Matplotlib. Exact Python package versions are not specified; cross-version numerical equivalence has not been established. See [`ENVIRONMENT_DEPENDENCIES.md`](ENVIRONMENT_DEPENDENCIES.md) for software requirements. BrainVision export/import functions must be available in the EEGLAB installation.

## 3. Directory structure and paths

Place the MATLAB scripts and the Python Step6 script under `<dataset_root>/code/`. The following is a representative layout; not every participant or dataset file is shown.

```text
<dataset_root>/
├── dataset_description.json
├── participants.tsv
├── participants.json
├── task-dotcount_events.json
├── task-firstlast_events.json
├── task-dotcount_beh.json
├── task-firstlast_beh.json
├── stimuli/
│   ├── audio/
│   └── stimulus_label_mapping.tsv
├── sub-01/
│   ├── beh/
│   │   ├── sub-01_task-dotcount_beh.tsv
│   │   └── sub-01_task-firstlast_beh.tsv
│   └── eeg/
│       ├── sub-01_task-dotcount_eeg.vhdr
│       ├── sub-01_task-dotcount_eeg.vmrk
│       ├── sub-01_task-dotcount_eeg.eeg
│       ├── sub-01_task-dotcount_events.tsv
│       ├── sub-01_task-dotcount_channels.tsv
│       └── sub-01_task-dotcount_eeg.json
├── sub-02/
├── ...
├── derivatives/
└── code/
    ├── step0_hdf5_to_bids.m
    ├── step1_behavior_analysis.m
    ├── step2_preprocess.m
    ├── step3_extract_PSD.m
    ├── step4_ERP_analysis.m
    ├── step5_PSD_ranksum_FDR.m
    ├── step6_PSD_LOSO_classification.py
    ├── eloc62stroop.loc
    ├── README_Code.md
    └── ENVIRONMENT_DEPENDENCIES.md
```

Steps1–6 determine the dataset root from the parent directory of the script's `code/` folder. Alternatively, set the environment variable before running the scripts:

```matlab
setenv('MORSEEEG_BIDS_ROOT', '<dataset-root>');
```

The environment variable is useful when the scripts are stored outside the dataset's `code/` directory. Use the canonical filename **`step5_PSD_ranksum_FDR.m`** in the public code directory.

## 4. Step0 — HDF5 and behavioral MAT conversion

**Script:** `step0_hdf5_to_bids.m`  
**Purpose:** Generate raw BIDS-organized EEG and behavioral files from g.tec HDF5 recordings and E-Prime-derived MAT files.

Configure the source/output locations in Section A1 of the script:

```matlab
paths.exp1.eeg      = '';
paths.exp1.behavior = '';
paths.exp2.eeg      = '';
paths.exp2.behavior = '';
paths.bidsRoot      = '';
paths.stimuliSource = '';  % optional
paths.stimulusExcel = '';  % optional
```

The input MAT files must contain an `outputCell` variable. The HDF5 reader accesses the acquisition datasets `/RawData/Samples`, `/RawData/AcquisitionTaskDescription`, `/AsynchronData/Time`, `/AsynchronData/Value`, and `/AsynchronData/TypeID`. The script includes the input-file sorting and HDF5 reading functions; no separate reader script is needed.

Step0 produces BrainVision EEG (`.vhdr`, `.vmrk`, `.eeg`), associated `channels.tsv`, `events.tsv` and `_eeg.json` files, trial-level `sub-XX/beh/*_beh.tsv` files, task-level event/behavior JSON dictionaries, `participants.tsv` and `participants.json`, and top-level dataset metadata. If stimulus materials are configured, the script also copies the audio files and creates a stimulus-label mapping.

EEG and behavioral source files are paired according to naturally sorted filenames. **Check participant pairing, channel order, the 156 formal trial events per task, and event-to-behavior correspondence before using newly converted recordings.** The generated `channels.tsv` marks channels `good` by default; known channel-quality exceptions must be considered when interpreting raw recordings. Step0 produces raw data only; minimally preprocessed data are generated by Step2.

## 5. Step1 — Behavioral analysis and grouping

**Input:** `<dataset_root>/participants.tsv`  
**Output:** `derivatives/behavior/`

Step1 uses each participant's mean accuracy and reaction time in the `dotcount` and `firstlast` tasks. It computes paired behavioral summaries, Pearson and Spearman correlations, and intraclass correlation coefficients. It also constructs performance-based Strong/Weak groupings using accuracy and response-time components.

The candidate accuracy weights are `0.5`, `0.6`, `0.7`, `0.8`, `0.9`, and `1.0`. The grouping used by Steps5 and 6 is the **fourth solution**, with accuracy weight **0.8**; for the full dataset this specifies **16 Strong and 16 Weak participants**. The grouping arrays contain row indices into the matched participant list, and `commonSubIDs.mat` provides the participant-ID mapping.

Key outputs include:

```text
derivatives/behavior/
├── matched_subjects_exp1_exp2.tsv
├── Exp1_vs_Exp2_minimal_evidence.mat
├── top_week_sub_ACC_RT.mat
├── labelinf_ACC_RT.mat
├── weight_list.mat
├── fusedScore_all.mat
├── sortedIdx_all.mat
├── commonSubIDs.mat
├── difference_analysis_results.mat
├── Fig1_exp1_exp2_agreement.pdf / .tif
└── Fig3_weight_dependent_behavioral_separation.pdf / .tif
```

Step1 uses participant-level summaries; it does not generate separate trial-level or CPM-block behavioral statistics.

## 6. Step2 — EEG preprocessing

**Inputs:** BrainVision EEG files, corresponding `*_events.tsv`, and behavioral `*_beh.tsv` files containing `stimutoRT_time`.  
**Output:** `derivatives/preprocessed/`

Step2 processes both tasks using the following settings:

- EEG sampling rate: **600 Hz**.
- For sub-13 and sub-14, channel 56 is replaced by the mean of channels 50, 55, and 57; for sub-15, channel 43 is replaced by the mean of channels 33, 34, 42, and 52. These substitutions are applied by participant ID.
- Average re-referencing across the 62 EEG channels.
- EEGLAB `eegfilt` bandpass filtering at **1–40 Hz** with a filter-order argument of **100**.
- Variable-length epochs from the stimulus event marker to the response endpoint recorded in the behavioral data.
- Baseline subtraction using the **−200 to 0 ms** pre-marker interval.
- Rejection of an epoch if its absolute amplitude exceeds **200 µV** in any channel.

Each output `.mat` file contains `sub_preData`, a three-block cell array of retained trials; each trial is a **62 × time samples** EEG matrix. The files are named `sub-XX_task-<task>_desc-preproc_eeg.mat`. Step2 also writes `derivatives/preprocessed/dataset_description.json`.

Retained-trial arrays are organized by block; they do **not** independently contain the original trial indexes or event labels. Refer to the raw event and behavioral files when performing trial-specific analyses.

## 7. Step3 — PSD feature extraction

**Input:** `derivatives/preprocessed/*_desc-preproc_eeg.mat`  
**Output:** `derivatives/PSD_features/`

Step3 computes trial-level Welch PSD estimates (`pwelch`) for each of the 62 EEG channels with **600 Hz sampling**, **256-sample windows**, **128-sample overlap**, and **256-point FFT**. Band summaries are arranged in the following order:

| Band | Nominal range |
| --- | --- |
| Delta | 1–4 Hz |
| Theta | 4–8 Hz |
| Alpha | 8–13 Hz |
| Beta | 13–30 Hz |
| Gamma | 30–50 Hz |
| HighGamma | 50–80 Hz |

The resulting `feature_PSD` variable contains **6 × 62 = 372 features per retained trial**, organized by block. Steps5 and 6 use the **first five** bands. The PSDs are computed from the **1–40 Hz filtered** Step2 data; consequently, estimates above the filtering cutoff do not represent unfiltered high-frequency EEG power.

## 8. Step4 — Event-marker-locked EEG visualization

**Input:** Step2 `sub_preData` files and `eloc62stroop.loc`  
**Output:** `derivatives/ERP/EXP1/` and `derivatives/ERP/EXP2/`

Step4 applies an additional **40 Hz low-pass filter** to retained epochs and extracts **301 samples (0–500 ms)**, using the software-defined event marker as time zero. It averages retained trials within each participant and then averages across participants. The visualizations include Fz (channel 12) and Cz (channel 30) waveforms, participant-level traces, and scalp topographies averaged over **150–250 ms** and **250–300 ms**.

For plotted Fz/Cz waveforms, a further **0–50 ms centering** is applied for visualization; the plotted shaded bands are pointwise approximate **95% confidence intervals** (`1.96 × SEM`). Scalp maps use the time-window means of the grand-average EEG. Output filenames contain the labels `N200` and `P300` for the selected windows; these names do not by themselves establish component identity. Reported latencies are relative to the event marker, not a separately measured loudspeaker onset.

The program exports participant and grand-average `.mat` files as well as `.png`, `.fig` and `.tif` figures.

## 9. Step5 — Participant-level PSD comparison

**Inputs:** Step1 group files, Step3 PSD features for both tasks, and `eloc62stroop.loc`.  
**Output:** `derivatives/stats/PSD_ranksum_FDR/`

Step5 analyzes the **accuracy-weight-0.8 Strong/Weak grouping**. For each participant, PSD is first averaged across retained trials **within each task**, then averaged **equally across Exp1 and Exp2**. The resulting participant-level observations form two groups of **16 participants each**. The script tests each channel using a two-group Wilcoxon rank-sum test and applies **Benjamini–Hochberg FDR correction at q ≤ 0.05**, separately across the **62 channels within each of five bands** (Delta, Theta, Alpha, Beta, and Gamma).

The five scalp maps display an interpolated **0/1 significance mask**, not power difference magnitude. The script also records raw and adjusted p values and Strong/Weak PSD summaries and directions.

Key outputs include:

```text
derivatives/stats/PSD_ranksum_FDR/
├── subjectlevel_PSD_groups_w08.mat
├── PSD_ranksum_FDR_stats.mat
├── PSD_ranksum_FDR_topoplot_pubstyle.png / .fig / .tif / .pdf
├── Delta_subjectlevel_direction.csv
├── Theta_subjectlevel_direction.csv
├── Alpha_subjectlevel_direction.csv
├── Beta_subjectlevel_direction.csv
├── Gamma_subjectlevel_direction.csv
├── PSD_subjectlevel_band_summary.csv
└── Alpha_direction_subjectlevel_report.txt
```

Step5 uses the standard EEGLAB `readlocs` and `topoplot` functions.

## 10. Step6 — Leave-one-subject-out classification

**Script:** `step6_PSD_LOSO_classification.py` (Python/PyTorch).  
**Inputs:** Step1's `top_week_sub_ACC_RT.mat`, `commonSubIDs.mat` and `labelinf_ACC_RT.mat` (when available), plus both tasks' Step3 PSD feature files.  
**Output:** `derivatives/classification/`

Step6 evaluates **linear SVM**, **Random Forest**, a **PyTorch MLP** (fully connected layers, batch normalization, ReLU and 30% dropout), and a **convolutional ResNet1D** (Conv1D stem, residual Conv1D blocks, and global average pooling). These architectures are used in the manuscript's Python classification benchmark. The neural networks use AdamW, class-weighted binary cross-entropy, gradient clipping, a validation-loss learning-rate scheduler and early stopping.

The fourth Step1 grouping solution (accuracy weight **0.8**) supplies the Strong/Weak labels. Step6 converts the MATLAB grouping's **1-based row indices** into participant IDs using `commonSubIDs.mat`, rather than storing fixed subject lists in the Python program. Each LOSO fold holds out one participant. Training examples comprise retained-trial PSD features from the other participants, with Exp1 and Exp2 trials combined. Feature standardization is fitted on the training data and applied to the held-out participant. Trial-level predictions for the test participant are averaged into one participant-level class probability.

The script reports participant-level accuracy, precision, recall, F1, and AUC, plus approximate 95% confidence intervals using **2,000 bootstrap resamples** of participant-level predictions. It sets NumPy and PyTorch random seeds to **42**. It analyzes the first **five** PSD bands and writes per-band prediction CSVs, result JSON files, metric MAT/NPY files, a training log, and per-band and combined TIFF/PDF figures.

Run the following commands with the dataset root specified as needed:

```bash
# Check that Step1 group assignments can be read (no training)
python code/step6_PSD_LOSO_classification.py --dataset-root /path/to/dataset --check-groups

# Run the complete classification benchmark
python code/step6_PSD_LOSO_classification.py --dataset-root /path/to/dataset
```

The script also accepts `MORSEEEG_BIDS_ROOT` as an environment variable and `--output-dir` to override the default results folder. The classification benchmark will generally take substantially longer than the preceding processing scripts. To compare with previously computed results, first verify that Step1 yields the same Strong/Weak participant IDs. Training results may differ across Python/PyTorch versions, CPUs and GPUs.

These are exploratory benchmarks for the behavior-defined Strong/Weak labels, not an externally validated measure of auditory ability.

## 11. Electrode-location and supplementary files

`eloc62stroop.loc` provides the 62-channel electrode coordinates used by Steps4 and 5. Channel order in the `.loc` file should correspond to the channel order of the EEG and PSD features.

`topoplot_addparameter.m`, when included, is an **optional plotting utility**. It is **not called** by the current Steps0–6. Keep the copyright and redistribution information within that source file if distributing it.

## 12. Citation

If using MorseEEG-ATP data or companion scripts, cite the public dataset record and the associated *Scientific Data* Data Descriptor.
