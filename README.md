[English](README.md) | [中文](README_CN.md)
# ManiSkill HDF5 to RLDS Conversion Toolkit
This repository provides a toolkit for converting ManiSkill replay HDF5 trajectory files into the RLDS format, usable by downstream robot learning pipelines, including VLA-style pipelines.

## Why This Toolkit

ManiSkill provides high-quality simulated robot manipulation trajectories, and these trajectories are commonly stored as HDF5 files after recording or replaying demonstrations. However, many robot learning data pipelines use episode-based dataset formats such as TFDS/RLDS, where each trajectory is represented as an episode and each timestep is represented as a step.

This toolkit bridges the gap between ManiSkill replay HDF5 files and the RLDS ecosystem. It converts ManiSkill trajectories into a structured TFDS/RLDS dataset with observations, actions, rewards, robot proprio, language instructions, and episode metadata.

This toolkit is useful if you want to:

- Convert ManiSkill replay HDF5 trajectories into TFDS/RLDS format.
- Preserve RGB, optional wrist RGB, depth, and robot state observations.
- Store each ManiSkill `traj_x` as one RLDS episode.
- Validate converted datasets before using them in downstream learning pipelines.
- Build a reusable dataset conversion workflow for different ManiSkill tasks.

## Overview
The converter takes ManiSkill replay HDF5 files as input and produces a TFDS/RLDS dataset as output. Each ManiSkill trajectory group, such as `traj_0`, `traj_1`, or `traj_2`, is converted into one RLDS episode. Inside each episode, the trajectory timesteps are stored as RLDS steps.

The overall workflow is:

```text
ManiSkill demo or generated trajectory
        |
        | replay_trajectory
        v
ManiSkill HDF5 file with RGB/RGBD observations
        |
        | inspect_h5.py
        v
Check H5 paths, shapes, and dtypes
        |
        | tfds build (maniskill_convert_dataset.py)
        v
TFDS/RLDS dataset
        |
        | check_tfds_dataset.py
        v
Validate episode-level and step-level fields
        |
        | check_dlimp_dataset.py
        v
Validate trajectory-level RLDS reading
```

## Installation
Create a clean Python environment.
```bash
conda create -n maniskill-rlds python=3.10 -y
conda activate maniskill-rlds
```
Clone this repository and enter the project directory:

```bash
git clone https://github.com/exileJordan/Maniskill_RLDS_Converter.git
cd Maniskill_RLDS_Converter
```

Install dependencies from `requirements.txt`:

```bash
pip install -r requirements.txt
```

## Quick Start
In the quick start section, use "PickCube-v1" as example.
### 1. Prepare H5 Files
(1) Download Maniskill "PickCube-v1" Trajectory and Replay
```bash
python -m mani_skill.utils.download_demo "PickCube-v1"
```
The downloaded demo directory should look like:
```bash
# result
demos/
└── PickCube-v1/motionplanning
    ├── trajectory.h5
    └── trajectory.json
```
(2) Replay PickCube trajectory and convert to target dataset. We recommend starting with a small number of trajectories.
```bash
python -m mani_skill.trajectory.replay_trajectory \
  --traj-path ${DEMO_PATH}PickCube-v1/motionplanning/trajectory.h5 \
  --use-first-env-state -c pd_ee_delta_pos -o rgb \
  --save-traj --num-envs 10 -b physx_cpu \
  --count 4
```
(3) Create a new directory and copy your replayed H5 files into that directory:
```bash
mkdir -p /path/to/maniskill_h5
cp /path/to/trajectory*.h5 /path/to/maniskill_h5/
```
The directory should look like:
```text
/path/to/maniskill_h5/
  trajectory_*.h5
```
### 2. Inspect H5 Structure
Before building the dataset, inspect your H5 file structure:
```bash
python maniskill_rlds_tool/inspect_h5.py \
  /path/to/maniskill_h5/trajectory_0.h5
```
Expected output for example:
```text
traj_98/
    traj_98/actions: shape=(81, 4), dtype=float32
    traj_98/env_states/
    traj_98/env_states/actors/
    traj_98/env_states/actors/cube: shape=(82, 13), dtype=float32
    traj_98/env_states/actors/goal_site: shape=(82, 13), dtype=float32
    traj_98/env_states/actors/table-workspace: shape=(82, 13), dtype=float32
    traj_98/env_states/articulations/
    traj_98/env_states/articulations/panda: shape=(82, 31), dtype=float32
    traj_98/obs/
    traj_98/obs/agent/
    traj_98/obs/agent/qpos: shape=(82, 9), dtype=float32
    traj_98/obs/agent/qvel: shape=(82, 9), dtype=float32
    traj_98/obs/extra/
    traj_98/obs/extra/goal_pos: shape=(82, 3), dtype=float32
    traj_98/obs/extra/is_grasped: shape=(82,), dtype=bool
    traj_98/obs/extra/tcp_pose: shape=(82, 7), dtype=float32
    traj_98/obs/sensor_data/
    traj_98/obs/sensor_data/base_camera/
    traj_98/obs/sensor_data/base_camera/rgb: shape=(82, 128, 128, 3), dtype=uint8
    traj_98/obs/sensor_param/
    traj_98/obs/sensor_param/base_camera/
    traj_98/obs/sensor_param/base_camera/cam2world_gl: shape=(82, 4, 4), dtype=float32
    traj_98/obs/sensor_param/base_camera/extrinsic_cv: shape=(82, 3, 4), dtype=float32
    traj_98/obs/sensor_param/base_camera/intrinsic_cv: shape=(82, 3, 3), dtype=float32
    traj_98/success: shape=(81,), dtype=bool
    traj_98/terminated: shape=(81,), dtype=bool
    traj_98/truncated: shape=(81,), dtype=bool
```
Record the relevant H5 paths and shapes. They must match the constants and feature schema in `maniskill_convert_dataset.py`.
The converter uses actions, rgb, qpos, qvel, goal_pos, tcp_pose, is_grasped, terminated, and truncated. If rewards are missing, zero rewards are generated.
If your H5 file uses different camera names, dimensions, or observation paths, update the constants in:

```text
maniskill_rlds_tool/maniskill_convert_dataset/maniskill_convert_dataset.py
```

Common constants to check:

```python
RGB_PATH
WRIST_RGB_PATHS
DEPTH_PATHS
WRIST_DEPTH_PATHS
QPOS_PATH
QVEL_PATH
TCP_POSE_PATH
IS_GRASPED_PATH
GOAL_POS_PATH
```

### 3. Build TFDS/RLDS Dataset
Run `tfds build` from the repository root:
```bash
tfds build maniskill_rlds_tool/maniskill_convert_dataset \
  --manual_dir=/path/to/maniskill_h5 \
  --data_dir=/path/to/tfds_output
```
- `--manual_dir`: directory containing your input H5 files.
- `--data_dir`: directory where TFDS/RLDS output will be written.

Expected output directory:

```text
/path/to/tfds_output/
  maniskill_convert_dataset/
    1.0.0/
      dataset_info.json
      features.json
      maniskill_convert_dataset-train.tfrecord-00000-of-00001
      maniskill_convert_dataset-val.tfrecord-00000-of-00001
```

### 4. Validate with TFDS
Use the TFDS checker to read one RLDS episode and inspect one step:

```bash
python maniskill_rlds_tool/check_tfds_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=train
```

Validate the validation split as well:

```bash
python maniskill_rlds_tool/check_tfds_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=val
```

Expected output includes:

```text
Episode keys: episode_metadata, steps
Observation keys: image, wrist_image, has_wrist_image, depth, has_depth, wrist_depth, has_wrist_depth, qpos, qvel, tcp_pose, is_grasped, goal_pos
Image shape: (128, 128, 3)
Qpos shape: (9,)
Qvel shape: (9,)
TCP pose shape: (7,)
Goal pos shape: (3,)
Action shape: (4,)
```
### 5. Validate with dlimp
Use dlimp to flatten RLDS episodes into trajectory-level samples:

```bash
python maniskill_rlds_tool/check_dlimp_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=train
```

Validate the validation split:

```bash
python maniskill_rlds_tool/check_dlimp_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=val
```

Expected output examples:

```text
Trajectory keys: ...
Observation keys: ...
Image shape: (T,)
Image dtype: string
Qpos shape: (T, 9)
Qvel shape: (T, 9)
TCP pose shape: (T, 7)
Goal pos shape: (T, 3)
Action shape: (T, 4)
Language shape: (T,)
```
`T` is the length of one trajectory.
If both TFDS and dlimp validation pass, the RLDS dataset was generated successfully.

## Input H5 Format and Output RLDS dataset format
For the complete input HDF5 and output RLDS schema, see [dataset_schema.md](docs/dataset_schema.md).

## Optional Modalities
This is supported.

If wrist RGB, depth, or wrist depth is missing, the converter fills zero placeholders and records validity flags:

```text
has_wrist_image
has_depth
has_wrist_depth
```

Use these flags to determine whether the modality was actually present in the original H5 file.

## Train/Val Split
During the TFDS builder process, the dataset is split into train and val.  
```text
train: 90%
val:   10%
```

To check the split sizes, inspect the `builder.info` output printed by `check_tfds_dataset.py`.

You should see something like:

```text
splits={
    'train': <SplitInfo num_examples=90>,
    'val': <SplitInfo num_examples=10>,
}
```
You can change the proportion in `_split_generators()` in maniskill_convert_dataset.py. 

## Troubleshooting
### No H5 files found

Error example:

```text
No .h5 files found in manual_dir
```

Cause:

```text
--manual_dir does not contain H5 files, or the path is incorrect.
```

Fix:

```bash
ls /path/to/maniskill_h5
```

Make sure the directory contains files ending in `.h5`.

### KeyError: object does not exist

Error example:

```text
KeyError: Unable to open object
```

Cause:

```text
A path in the builder does not exist in your H5 file.
```

Fix:

```text
1. Run inspect_h5.py.
2. Find the real path in your H5 file.
3. Update the corresponding path constant in maniskill_convert_dataset.py.
```

### Shape mismatch

Cause:

```text
The schema in `_info()` does not match the actual shape of your H5 data.
```

Common examples:

```text
action shape is not (4,)
qpos/qvel shape is not (9,)
image resolution is not 128 x 128
depth shape is missing the channel dimension
```

Fix:

```text
Update the feature shapes in `_info()` and rebuild the dataset.
```


## Citation/Acknowledgement
This project builds on ManiSkill, TensorFlow Datasets, RLDS, and dlimp.

## License

This project is released under the MIT License.
