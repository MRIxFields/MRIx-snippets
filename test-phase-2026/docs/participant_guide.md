# MRIxFields2026 Test Phase Docker I/O Guide

**Status**: Current test-phase Docker file-tree interface, updated 2026-07-13.

This document defines the participant Docker input and output interface. Participants submit one Docker image containing their inference code and model weights. Segmentation is not part of the participant interface.

## Path Convention

Repository paths in participant-facing material are relative to the repository root and begin with:

```text
MRIxFields2026/Submission/testing-2026
```

This document is located at:

```text
MRIxFields2026/Submission/testing-2026/doc/participant_guide.md
```

No platform host filesystem path is exposed to participants. `/input` and `/output` below are paths inside the participant container, not host paths.

## Execution Model

~~The platform runs the same image once for each task: Task 1, Task 2, and Task 3. Each run receives one read-only input tree and one writable output tree.~~ 每个任务有一个docker image

Dataset construction, organizer-side segmentation, host storage, and transfer are platform-internal operations. They do not change the participant interface and are not visible inside the container.

Container mounts:

| Container path | Mode | Meaning |
|---|---|---|
| `/input` | Read-only | One task's input files and `manifest.json` |
| `/output` | Writable | Predictions for that task |

~~Environment variables:~~ 应该用不上吧

| Variable | Value |
|---|---|
| `INPUT_DIR` | `/input` |
| `OUTPUT_DIR` | `/output` |
| `MANIFEST` | `/input/manifest.json` |
| `TASK_ID` | `task1`, `task2`, or `task3` |

The container must read `/input/manifest.json` and produce every prediction listed there.

## Input Tree

```text
/input/
  manifest.json
  {MOD}/{SRC}_to_{TGT}/src/P_{MOD}_{SRC}_{ID}.nii.gz
```

Examples:

```text
/input/T1W/0.1T_to_7T/src/P_T1W_0.1T_0021.nii.gz
/input/T2W/0.1T_to_3T/src/P_T2W_0.1T_0028.nii.gz
/input/T2FLAIR/7T_to_1.5T/src/P_T2FLAIR_7T_0039.nii.gz
```

Tokens:

| Token | Values |
|---|---|
| `MOD` | `T1W`, `T2W`, `T2FLAIR` |
| `SRC` | Source field strength |
| `TGT` | Requested target field strength |
| `ID` | Four-digit case ID |

Only the current task's input tree is visible. Ground truth, GT segmentation, private assignments, and other task directories are not mounted.

## Output Tree

Write each prediction to the exact relative path in `manifest.samples[*].output`:

```text
/output/
  {MOD}/{SRC}_to_{TGT}/pred/P_{MOD}_{TGT}_{ID}.nii.gz
```

Examples:

```text
/output/T1W/0.1T_to_7T/pred/P_T1W_7T_0021.nii.gz
/output/T2W/0.1T_to_3T/pred/P_T2W_3T_0028.nii.gz
/output/T2FLAIR/7T_to_1.5T/pred/P_T2FLAIR_1.5T_0039.nii.gz
```

Do not replace `SRC` with `TGT` in directory names. The mapping directory remains `{SRC}_to_{TGT}`; only the prediction filename uses the target field strength.

## Manifest

~~`manifest.json` is the source of truth for the current run.~~ Do not derive additional cases or mappings outside the `/input/manifest.json` file.

Example:

```json
{
  "challenge": "MRIxFields2026",
  "phase": "test",
  "task": "task1",
  "input_root": "/input",
  "output_root": "/output",
  "volume_shape": [364, 436, 364],
  "samples": [
    {
      "task": "task1",
      "case_id": "0021",
      "modality": "T1W",
      "source_field": "0.1T",
      "target_field": "7T",
      "pair": "0.1T_to_7T",
      "input": "T1W/0.1T_to_7T/src/P_T1W_0.1T_0021.nii.gz",
      "output": "T1W/0.1T_to_7T/pred/P_T1W_7T_0021.nii.gz"
    }
  ]
}
```

Both `input` and `output` are relative paths under `/input` and `/output` respectively.

## Workload

| Task | Mapping design | Required predictions |
|---|---|---:|
| Task 1 | 4 mappings x 5 cases x 3 modalities | 60 |
| Task 2 | 4 mappings x 5 cases x 3 modalities | 60 |
| Task 3 | 20 mappings x 2 cases x 3 modalities | 120 |
| **Total** |  | **240** |

The manifest for each run contains only that task's samples.

## Required Prediction Properties

| Item | Requirement |
|---|---|
| Format | Compressed NIfTI, `.nii.gz` |
| Shape | `(364, 436, 364)` |
| Coverage | Full 3D volume; no slice clipping |
| Data type | Float-compatible; `float32` recommended |
| Intensity | `[0, 1]` |
| Invalid values | No NaN or infinity |
| Filename | Exactly as listed in the manifest |

Every slice must be inferred and written. Do not submit cropped slabs or validation-style z-clipped volumes.

The platform may reject a run with missing or extra prediction files, unreadable NIfTI files, incorrect paths or filenames, incorrect shapes, NaN/infinity values, or values outside the required range.

## Segmentation

Do not create segmentation files.

For Task 1 and Task 2, the organizers generate segmentation from submitted T1W predictions after inference. Task 3 has no segmentation step. Participant containers write only prediction files under `/output`; there is no `/output_seg` container interface.

## Completion Checklist

Before the container exits successfully:

~~- Read the manifest belonging to the current `TASK_ID`.~~ 固定的 /input/manifest.json 吧
- Produce exactly one output for every manifest sample.
- Use each sample's exact `output` relative path.
- Preserve the full `(364, 436, 364)` volume.
- Ensure every output is a readable `.nii.gz` file with finite values.
- Do not write segmentation files.
- Do not add extra prediction `.nii.gz` files.
