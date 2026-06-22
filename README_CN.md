# ManiSkill HDF5 到 RLDS 转换工具

本仓库提供一个用于将 ManiSkill replay HDF5 轨迹文件转换为 TFDS/RLDS 格式的工具。转换后的数据可以用于下游机器人学习数据管线，包括 VLA 风格的数据处理流程。

## 为什么需要这个工具

ManiSkill 可以生成高质量的仿真机器人操作轨迹，这些轨迹通常在录制或 replay 后保存为 HDF5 文件。然而，许多机器人学习数据管线使用基于 episode 的数据格式，例如 TFDS/RLDS。在这类格式中，一条轨迹通常对应一个 episode，每一个时间步对应一个 step。

本工具用于连接 ManiSkill replay HDF5 文件和 RLDS 数据生态。它会将 ManiSkill 轨迹转换成结构化的 TFDS/RLDS 数据集，包含 observation、action、reward、机器人 proprio、语言指令和 episode metadata。

如果你有以下需求，可以使用本工具：

- 将 ManiSkill replay HDF5 轨迹转换为 TFDS/RLDS 格式。
- 保留 RGB、可选 wrist RGB、depth 和机器人状态观测。
- 将每个 ManiSkill `traj_x` 存储为一个 RLDS episode。
- 在下游学习流程使用数据前，先验证转换结果是否正确。
- 为不同 ManiSkill 任务构建可复用的数据转换流程。

## 概览

转换器以 ManiSkill replay HDF5 文件作为输入，并输出 TFDS/RLDS 数据集。每个 ManiSkill 轨迹组，例如 `traj_0`、`traj_1` 或 `traj_2`，会被转换成一个 RLDS episode。每个 episode 内部的时间步会被保存为 RLDS steps。

整体流程如下：

```text
ManiSkill demo 或自行生成的轨迹
        |
        | replay_trajectory
        v
包含 RGB/RGBD observation 的 ManiSkill HDF5 文件
        |
        | inspect_h5.py
        v
检查 H5 路径、shape 和 dtype
        |
        | tfds build (maniskill_convert_dataset.py)
        v
TFDS/RLDS 数据集
        |
        | check_tfds_dataset.py
        v
验证 episode 和 step 字段
        |
        | check_dlimp_dataset.py
        v
验证 RLDS trajectory 级读取
```

## 安装

创建一个干净的 Python 环境：

```bash
conda create -n maniskill-rlds python=3.10 -y
conda activate maniskill-rlds
```

克隆仓库并进入项目目录：

```bash
git clone <your-repository-url>
cd <your-repository-name>
```

安装依赖：

```bash
pip install -r requirements.txt
```

## 快速开始

本节以 ManiSkill `PickCube-v1` 任务为例。

### 1. 准备 H5 文件

下载 ManiSkill `PickCube-v1` demo 轨迹：

```bash
python -m mani_skill.utils.download_demo "PickCube-v1"
```

下载后的目录大致如下：

```text
demos/
└── PickCube-v1/motionplanning
    ├── trajectory.h5
    └── trajectory.json
```

Replay PickCube 轨迹并生成目标 H5 数据。建议先从少量轨迹开始测试：

```bash
python -m mani_skill.trajectory.replay_trajectory \
  --traj-path ${DEMO_PATH}PickCube-v1/motionplanning/trajectory.h5 \
  --use-first-env-state -c pd_ee_delta_pos -o rgb \
  --save-traj --num-envs 10 -b physx_cpu \
  --count 4
```

创建一个新目录，并将 replay 后的 H5 文件复制进去：

```bash
mkdir -p /path/to/maniskill_h5
cp /path/to/trajectory*.h5 /path/to/maniskill_h5/
```

目录结构应类似：

```text
/path/to/maniskill_h5/
  trajectory_*.h5
```

### 2. 检查 H5 结构

在构建 TFDS/RLDS 数据集之前，先检查 H5 文件结构：

```bash
python maniskill_rlds_tool/inspect_h5.py \
  /path/to/maniskill_h5/trajectory_0.h5
```

示例输出如下：

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

请记录相关 H5 路径和 shape，它们必须和 `maniskill_convert_dataset.py` 中的路径常量以及 TFDS feature schema 保持一致。

默认转换器会使用以下字段：

```text
actions, rgb, qpos, qvel, goal_pos, tcp_pose, is_grasped, terminated, truncated
```

如果 `rewards` 不存在，转换器会自动生成全零 reward。

如果你的 H5 文件使用了不同的相机名称、数据维度或 observation 路径，需要修改：

```text
maniskill_rlds_tool/maniskill_convert_dataset/maniskill_convert_dataset.py
```

常见需要检查的常量包括：

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

### 3. 构建 TFDS/RLDS 数据集

在仓库根目录运行：

```bash
tfds build maniskill_rlds_tool/maniskill_convert_dataset \
  --manual_dir=/path/to/maniskill_h5 \
  --data_dir=/path/to/tfds_output
```

参数说明：

- `--manual_dir`：输入 H5 文件所在目录。
- `--data_dir`：TFDS/RLDS 输出目录。

期望输出目录类似：

```text
/path/to/tfds_output/
  maniskill_convert_dataset/
    1.0.0/
      dataset_info.json
      features.json
      maniskill_convert_dataset-train.tfrecord-00000-of-00001
      maniskill_convert_dataset-val.tfrecord-00000-of-00001
```

### 4. 使用 TFDS 验证数据

使用 TFDS 检查脚本读取一个 RLDS episode，并检查其中一个 step：

```bash
python maniskill_rlds_tool/check_tfds_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=train
```

也可以检查 validation split：

```bash
python maniskill_rlds_tool/check_tfds_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=val
```

期望输出包括：

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

### 5. 使用 dlimp 验证数据

使用 dlimp 将 RLDS episode 展平成 trajectory 级样本：

```bash
python maniskill_rlds_tool/check_dlimp_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=train
```

验证 validation split：

```bash
python maniskill_rlds_tool/check_dlimp_dataset.py \
  --name=maniskill_convert_dataset \
  --data_dir=/path/to/tfds_output \
  --split=val
```

期望输出示例：

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

其中 `T` 是某一条 trajectory 的长度。如果 TFDS 和 dlimp 验证都通过，说明 RLDS 数据集已经成功生成。

## 输入 H5 格式与输出 RLDS 格式

完整的 HDF5 输入结构和 RLDS 输出 schema 请参考：[dataset_schema.md](docs/dataset_schema.md)。

## 可选模态

如果 wrist RGB、depth 或 wrist depth 不存在，转换器会填充零占位数据，并记录对应的有效性标志：

```text
has_wrist_image
has_depth
has_wrist_depth
```

你可以通过这些标志判断某个模态是否真实存在于原始 H5 文件中。

## Train/Val 划分

在 TFDS builder 过程中，数据会被划分为 train 和 val：

```text
train: 90%
val:   10%
```

你可以通过 `check_tfds_dataset.py` 打印的 `builder.info` 检查 split 数量。示例：

```text
splits={
    'train': <SplitInfo num_examples=90>,
    'val': <SplitInfo num_examples=10>,
}
```

如需修改划分比例，可以在 `maniskill_convert_dataset.py` 的 `_split_generators()` 中调整。

## 常见问题

### No H5 files found

错误示例：

```text
No .h5 files found in manual_dir
```

原因：

```text
--manual_dir 指向的目录中没有 H5 文件，或者路径不正确。
```

修复：

```bash
ls /path/to/maniskill_h5
```

确认目录中存在以 `.h5` 结尾的文件。

### KeyError: object does not exist

错误示例：

```text
KeyError: Unable to open object
```

原因：

```text
builder 中配置的 H5 路径在你的 H5 文件中不存在。
```

修复：

```text
1. 运行 inspect_h5.py。
2. 找到 H5 文件中的真实路径。
3. 修改 maniskill_convert_dataset.py 中对应的路径常量。
```

### Shape mismatch

原因：

```text
_info() 中定义的 TFDS schema 和你的 H5 数据实际 shape 不一致。
```

常见例子：

```text
action shape 不是 (4,)
qpos/qvel shape 不是 (9,)
image 分辨率不是 128 x 128
depth 缺少 channel 维度
```

修复：

```text
更新 _info() 中的 feature shape，然后重新 build 数据集。
```

## 致谢

本项目基于 ManiSkill、TensorFlow Datasets、RLDS 和 dlimp 构建。

## 许可证

本项目使用 MIT License 发布。
