# DMG Tracker

Containerized nnU-Net v2 inference for diffuse midline glioma (DMG) tumor segmentation.

The image bundles the trained `Dataset113_DMG_BRATSPEDS_BCHPrePost` model
(`nnUNetTrainer__nnUNetResEncUNetLPlans__3d_fullres`, 5 folds) and runs prediction as its
entrypoint, so a container run only needs an input and an output volume.

## Model

| Property | Value |
| --- | --- |
| Configuration | `3d_fullres` |
| Trainer / plans | `nnUNetTrainer` / `nnUNetResEncUNetLPlans` |
| Folds | `0 1 2 3 4` (ensembled) |
| File ending | `.nii` |

Input channels, in order:

| Channel | Modality | BraTS suffix |
| --- | --- | --- |
| `0000` | T1 pre-contrast | `t1n` |
| `0001` | T1 post-contrast | `t1c` |
| `0002` | T2 | `t2w` |
| `0003` | FLAIR | `t2f` |

Output labels: `1` = enhancing tumor, `2` = whole tumor (non-enhancing remainder).

## Running

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
    -v /path/to/input:/input:ro \
    -v /path/to/output:/output \
    ghcr.io/aim-kannlab/dmg-tracker-inference:latest
```

`--shm-size` must be raised above Docker's 64 MB default, otherwise nnU-Net's background
export workers are killed and the run fails with "Background workers died".

`/input` is read-only-safe: BraTS-style inputs are staged into a temporary directory
inside the container, never into the mount.

### Input layouts

The entrypoint auto-detects one of two conventions.

**nnU-Net style** — flat directory, channel indices in the filename:

```
/input/CASE_0000.nii
/input/CASE_0001.nii
/input/CASE_0002.nii
/input/CASE_0003.nii
```

**BraTS style** — per-case subdirectories (searched recursively), modality suffixes:

```
/input/BraTS-PED-00001-000/BraTS-PED-00001-000-t1n.nii.gz
/input/BraTS-PED-00001-000/BraTS-PED-00001-000-t1c.nii.gz
/input/BraTS-PED-00001-000/BraTS-PED-00001-000-t2w.nii.gz
/input/BraTS-PED-00001-000/BraTS-PED-00001-000-t2f.nii.gz
```

BraTS cases are symlinked into a staging directory under the nnU-Net naming scheme.
nnU-Net only picks up files matching the `file_ending` in `dataset.json` (`.nii`), so when
the inputs are `.nii.gz` the model folder is shadowed with a symlink tree whose
`dataset.json` declares `.nii.gz`. Nothing is decompressed, and predictions are written
with the same extension as the inputs. Extra files in a case directory (for example
`-seg.nii.gz`) are ignored.

Predictions are written to `/output` as `<case_id>.nii[.gz]`, one per case, alongside the
`dataset.json`/`plans.json` copies nnU-Net emits.

### Configuration

All defaults can be overridden with environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `INPUT_DIR` | `/input` | Input directory |
| `OUTPUT_DIR` | `/output` | Output directory |
| `INPUT_NAMING` | `auto` | Force a layout: `auto`, `nnunet`, or `brats` |
| `FOLDS` | `0 1 2 3 4` | Folds to ensemble |

Extra arguments are forwarded to `nnUNetv2_predict`, e.g. to use a single fold and
disable test-time augmentation for a quick run:

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
    -e FOLDS=0 \
    -v /path/to/input:/input:ro \
    -v /path/to/output:/output \
    ghcr.io/aim-kannlab/dmg-tracker-inference:latest --disable_tta
```

## Requirements

- NVIDIA GPU with a driver supporting CUDA 13.0
- NVIDIA Container Toolkit (`--gpus` support)
- Case directories must be real directories in the mount; symlinked case folders inside
  `/input` are not followed
