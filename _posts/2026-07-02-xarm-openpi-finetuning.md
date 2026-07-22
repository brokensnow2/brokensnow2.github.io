---
layout: post
title: "用自己的 xArm 数据微调 openpi：从 NPZ、LeRobot 到 π0.5 checkpoint"
date: 2026-07-02
description: "把 Isaac Sim 采集的 xArm episode 转成 LeRobot 数据集，理解 openpi 的输入输出 transform、delta action 和归一化统计，并在 GPU 服务器上完成 π0 / π0.5 微调。"
categories: [robotics, embodied-ai, openpi, fine-tuning]
---

上一篇数据采集完成后，我手里得到的是一批 `episode_0000.npz`。这篇文章继续把后半段链路走完：先检查 raw episode，再转成 LeRobot 数据集，配置 xArm 的 state/action transform，计算归一化统计，最后在 GPU 服务器上微调 openpi。

这篇更关心“训练之前必须对齐哪些契约”：字段、频率、动作语义、归一化和 metadata。模型能开始跑，只能说明管线没有立刻报错，离真的可部署还差这些细节。

TL;DR:

```text
raw xArm episode_*.npz
  -> 检查图像、关节、夹爪和时间对齐
  -> 转换成 LeRobot dataset
  -> 配置 XArmInputs / XArmOutputs
  -> 绝对关节目标转 delta joint action
  -> 计算或复用 normalization stats
  -> 从 pi0 / pi05 / pi0-fast base checkpoint 微调
  -> 得到可部署 checkpoint
```

本文统一使用下面这些占位变量，不包含我个人机器上的绝对路径：

```bash
export OPENPI_ROOT=/path/to/openpi
export RAW_XARM_DIR=/path/to/raw_xarm_data
export XARM_DATASET_ID="<your-hf-username>/xarm_dataset"
```

## 微调到底在学什么

一条机器人示范可以写成：

```text
observation_t = 图像_t + 机器人状态_t + 语言指令
target_t      = 未来一段时间的机器人动作
```

当前 xArm 配置的输入大致是：

```text
wrist RGB: [224, 224, 3]
base RGB: [224, 224, 3]，可选
joints: [6]，单位 rad
gripper: [1]，0=打开，1=闭合
prompt: "pick up the red block"
```

输出不是一个动作，而是一段 action chunk：

```text
actions: [action_horizon, 7]

每一行:
[joint1, joint2, joint3, joint4, joint5, joint6, gripper]
```

当前三套 xArm 配置都把 `action_horizon` 设为 10。数据频率是 10 Hz 时，这个 chunk 表示大约 1 秒的未来动作。

## 第一步：检查 raw NPZ

假设采集目录是：

```text
raw_xarm_data/
  episode_0000.npz
  episode_0001.npz
  episode_0002.npz
```

每个 episode 至少应该包含：

```text
wrist_image: [T, H, W, 3], uint8
joints: [T, 6], float32
gripper: [T, 1], float32
prompt: str
```

人工采集还可能直接包含：

```text
actions: [T, 7], float32
```

自动采集则可以故意不写 `actions`，留到转换阶段用下一帧状态生成。

先运行检查脚本：

```bash
cd "$OPENPI_ROOT"

uv run examples/xarm/inspect_xarm_npz.py \
  "$RAW_XARM_DIR" \
  --preview-dir /tmp/xarm_preview \
  --num-preview-frames 8
```

检查脚本的核心思路很直接：

```python
with np.load(path, allow_pickle=True) as data:
    wrist_image = data["wrist_image"]
    joints = data["joints"]
    gripper = data["gripper"]

    assert wrist_image.ndim == 4
    assert wrist_image.shape[-1] == 3
    assert wrist_image.dtype == np.uint8
    assert joints.shape[1] == 6
    assert gripper.shape[1] == 1

    lengths = {
        len(wrist_image),
        len(joints),
        len(gripper),
    }
    assert len(lengths) == 1
```

除了 shape，我还会检查：

```text
图像是否有黑帧、冻结帧或过曝
腕部相机是否真的跟随末端
prompt 中的颜色和物体是否与画面一致
关节是否包含 NaN / Inf
关节是否落在真实 xArm 限位内
相邻 step 是否有不合理的大跳变
gripper 是否主要落在 [0,1]
timestamp 是否严格递增
episode 是否真的完成任务
```

## 第二步：把 NPZ 转成 LeRobot

openpi 的常规训练流程使用 LeRobot 数据集。raw NPZ 更适合采集和排查，LeRobot 则负责：

```text
episode 边界
frame 索引
时间戳和 fps
图像/视频存储
任务文本映射
按未来时间查询 action chunk
Hugging Face dataset 接口
```

转换脚本位于：

```text
examples/xarm/convert_xarm_data_to_lerobot.py
```

只使用腕部相机时：

```bash
cd "$OPENPI_ROOT"

uv run examples/xarm/convert_xarm_data_to_lerobot.py \
  --data-dir "$RAW_XARM_DIR" \
  --repo-id "$XARM_DATASET_ID" \
  --fps 10 \
  --overwrite
```

如果 NPZ 中还保存了第三人称 `image`：

```bash
uv run examples/xarm/convert_xarm_data_to_lerobot.py \
  --data-dir "$RAW_XARM_DIR" \
  --repo-id "$XARM_DATASET_ID" \
  --fps 10 \
  --use-base-image \
  --overwrite
```

需要上传到 Hugging Face Hub 时再加：

```bash
--push-to-hub
```

`--overwrite` 会删除相同 `repo-id` 的本地 LeRobot 数据集后重新创建，正式数据最好先保留备份或使用带版本号的新 ID。

## 转换脚本做了什么

### 图像统一成 HWC uint8

```python
def as_uint8_image_sequence(name, value):
    value = np.asarray(value)
    if value.ndim != 4 or value.shape[-1] != 3:
        raise ValueError(
            f"{name} must have shape [T,H,W,3], got {value.shape}"
        )

    if np.issubdtype(value.dtype, np.floating):
        value = np.clip(value, 0.0, 1.0) * 255.0

    return value.astype(np.uint8)
```

这一步不是为了把所有图提前 resize 到 224。LeRobot 可以保留原分辨率，openpi 的 model transform 会在训练和推理时统一 resize/pad。

### 统一状态 shape

```python
def as_2d_float(name, value, dim):
    value = np.asarray(value, dtype=np.float32)

    if value.ndim == 1 and dim == 1:
        value = value[:, None]

    if value.ndim != 2 or value.shape[-1] != dim:
        raise ValueError(
            f"{name} must have shape [T,{dim}], got {value.shape}"
        )

    return value
```

这样无论采集端把 gripper 写成 `[T]` 还是 `[T,1]`，最终都会变成 `[T,1]`。

### 自动生成下一步动作

如果原始 episode 没有 `actions`，转换器会执行：

```python
actions = np.concatenate(
    [joints[1:], gripper[1:]],
    axis=-1,
)

wrist_image = wrist_image[:-1]
joints = joints[:-1]
gripper = gripper[:-1]

if base_image is not None:
    base_image = base_image[:-1]
```

这时第 `t` 条样本的语义是：

```text
observation[t] = image[t], joints[t], gripper[t]
action[t]      = joints[t+1], gripper[t+1]
```

最后一帧没有下一步目标，所以会被丢掉。

如果 NPZ 已经包含 `actions`，转换器会原样使用。此时要自己保证：

```text
shape 是 [T,7]
前六维是 rad
第七维是 [0,1]
action 和 state 使用同一种关节零位约定
action 是绝对目标，而不是已经做过 delta 的值
```

### 定义 LeRobot features

```python
features = {
    "wrist_image": {
        "dtype": "image",
        "shape": wrist_shape,
        "names": ["height", "width", "channel"],
    },
    "joints": {
        "dtype": "float32",
        "shape": (6,),
        "names": ["joints"],
    },
    "gripper": {
        "dtype": "float32",
        "shape": (1,),
        "names": ["gripper"],
    },
    "actions": {
        "dtype": "float32",
        "shape": (7,),
        "names": ["actions"],
    },
}
```

逐帧写入时：

```python
for i in range(len(episode["actions"])):
    frame = {
        "wrist_image": episode["wrist_image"][i],
        "joints": episode["joints"][i],
        "gripper": episode["gripper"][i],
        "actions": episode["actions"][i],
        "task": prompt,
    }
    dataset.add_frame(frame)

dataset.save_episode()
```

`task` 会在 openpi data loader 中重新变成模型使用的 `prompt`。

## fps 不是装饰信息

LeRobot data loader 会根据 dataset fps 查询未来动作。当前 xArm 配置是：

```text
fps = 10
action_horizon = 10
```

加载器内部会构造类似下面的未来时间偏移：

```python
delta_timestamps = {
    "actions": [
        t / dataset_fps
        for t in range(action_horizon)
    ]
}
```

如果真实数据是 10 Hz，转换时却错误写成 20 Hz，训练器会误解动作之间的时间间隔。以后客户端再以错误频率执行，即使模型预测值本身没问题，机械臂的速度和动力学也会错。

## 第三步：把 xArm 字段映射到 openpi

相关代码主要在：

```text
src/openpi/policies/xarm_policy.py
src/openpi/training/config.py
src/openpi/transforms.py
```

LeRobot 中的字段先经过 repack：

```python
repack_structure = {
    "wrist_rgb": "wrist_image",
    "joints": "joints",
    "gripper": "gripper",
    "actions": "actions",
    "prompt": "prompt",
}

if use_base_image:
    repack_structure["base_rgb"] = "image"
```

这里的方向是：

```text
openpi 需要的字段 <- LeRobot 中的字段
```

接下来 `XArmInputs` 把 6 个关节和 1 个夹爪拼成统一 state：

```python
joints = np.asarray(data["joints"])
gripper = np.asarray(data["gripper"])

if gripper.ndim == 0:
    gripper = gripper[np.newaxis]

state = np.concatenate([joints, gripper])
```

图像被放进 openpi 统一的多相机结构：

```python
wrist_image = parse_image(data["wrist_rgb"])

if "base_rgb" in data:
    base_image = parse_image(data["base_rgb"])
    has_base_image = True
else:
    base_image = np.zeros_like(wrist_image)
    has_base_image = False

inputs = {
    "state": state,
    "image": {
        "base_0_rgb": base_image,
        "left_wrist_0_rgb": wrist_image,
        "right_wrist_0_rgb": np.zeros_like(base_image),
    },
    "image_mask": {
        "base_0_rgb": has_base_image,
        "left_wrist_0_rgb": True,
        "right_wrist_0_rgb": False,
    },
}
```

模型主体因此不需要知道当前机器人原始字段叫 `wrist_image` 还是 `camera_0`，也不用为每种机器人重写视觉网络。

上面的 `image_mask` 是 π0 / π0.5 路径的简化写法。当前 π0-FAST 实现不会屏蔽补位图，因此 `base_0_rgb` 和 `right_wrist_0_rgb` 的 mask 处理不同。切换模型类型时应该直接复用 `XArmInputs` 中的分支，不要在自己的转换脚本里写死 mask。

## 绝对 action 为什么要转成 delta

LeRobot 中保存的是绝对目标：

```text
actions = [q_target_1 ... q_target_6, gripper_target]
```

xArm 数据配置默认启用：

```python
use_delta_joint_actions = True
```

并构造 mask：

```python
delta_action_mask = make_bool_mask(6, -1)
```

这个 mask 等价于：

```text
[True, True, True, True, True, True, False]
```

输入模型前执行：

```python
joint_delta = joint_target - current_joint
gripper_target = gripper_target
```

关节动作变成以 0 为中心的局部变化，夹爪仍然保持绝对开合语义。

对应的简化代码是：

```python
mask = np.asarray(delta_action_mask)
actions[..., :7] -= np.expand_dims(
    np.where(mask, state[..., :7], 0),
    axis=-2,
)
```

推理输出时再执行相反操作：

```python
absolute_joint_target = predicted_delta + current_joint
```

所以客户端收到的是绝对关节目标，不应该再手工加一次 current state。

如果自己的原始 `actions` 已经是速度或 delta action，必须关闭这一步：

```python
LeRobotXArmDataConfig(
    repo_id="<your-hf-username>/xarm_dataset",
    use_delta_joint_actions=False,
)
```

否则相当于重复减去 current state。

## 7 维 xArm 为什么会进入 32 维模型

π0 和 π0.5 的默认 `action_dim` 是 32，而 xArm 只需要 7 维。送进模型前，openpi 会补零：

```text
dim 0~5: xArm 六关节
dim 6: 夹爪
dim 7~31: padding 0
```

核心逻辑：

```python
def pad_to_dim(x, target_dim, axis=-1, value=0.0):
    current_dim = x.shape[axis]
    if current_dim >= target_dim:
        return x

    pad_width = [(0, 0)] * x.ndim
    pad_width[axis] = (0, target_dim - current_dim)
    return np.pad(
        x,
        pad_width,
        constant_values=value,
    )
```

模型输出后，`XArmOutputs` 只取前 7 维：

```python
outputs = {
    "actions": np.asarray(data["actions"][:, :7])
}
```

π0-FAST 的 xArm 配置则直接设置：

```python
Pi0FASTConfig(
    action_dim=7,
    action_horizon=10,
    max_token_len=180,
)
```

## 第四步：选择训练配置

当前有三套 xArm 配置：

```text
pi0_xarm
pi05_xarm
pi0_fast_xarm
```

可以把它们粗略理解为：

```text
pi0_xarm:
  flow matching VLA，适合先做基础基线

pi05_xarm:
  π0.5 + flow matching action head
  语言和开放场景能力更强，优先尝试

pi0_fast_xarm:
  把动作编码成 token 后自回归生成
  动作维显式为 7
```

以 `pi05_xarm` 为例：

```python
TrainConfig(
    name="pi05_xarm",
    model=Pi0Config(
        pi05=True,
        action_horizon=10,
        discrete_state_input=False,
    ),
    data=LeRobotXArmDataConfig(
        repo_id="<your-hf-username>/xarm_dataset",
        assets=AssetsConfig(
            assets_dir=(
                "gs://openpi-assets/"
                "checkpoints/pi05_base/assets"
            ),
            asset_id="ur5e",
        ),
        base_config=DataConfig(
            prompt_from_task=True,
        ),
    ),
    lr_schedule=CosineDecaySchedule(
        warmup_steps=1_000,
        peak_lr=5e-5,
        decay_steps=100_000,
        decay_lr=5e-5,
    ),
    optimizer=AdamW(
        clip_gradient_norm=1.0,
    ),
    ema_decay=0.999,
    weight_loader=CheckpointWeightLoader(
        "gs://openpi-assets/checkpoints/pi05_base/params"
    ),
    num_train_steps=30_000,
    batch_size=64,
)
```

需要先把示例里的 dataset ID 换成自己的：

```python
repo_id="<your-hf-username>/xarm_dataset"
```

## 腕部单相机和双相机必须两边一起配置

默认 xArm 配置只使用腕部相机：

```python
use_base_image = False
```

如果训练时要加入第三人称图像，需要同时满足：

```text
raw NPZ 中有 image
转换命令使用 --use-base-image
LeRobotXArmDataConfig 使用 use_base_image=True
推理客户端发送 base_rgb
```

训练配置示例：

```python
data=LeRobotXArmDataConfig(
    repo_id="<your-hf-username>/xarm_dataset",
    use_base_image=True,
    base_config=DataConfig(
        prompt_from_task=True,
    ),
)
```

只在转换时加 `--use-base-image`，并不会让模型自动使用该字段。反过来，配置要求 `base_rgb`，数据集却没有 `image`，加载时也会失败。

## 第五步：理解 normalization stats

不同关节的数值范围不同，模型训练前需要归一化 `state` 和 `actions`。

π0 使用均值和标准差：

```text
x_norm = (x - mean) / (std + 1e-6)
```

π0.5 和 π0-FAST 默认使用分位数归一化：

```text
x_norm = 2 * (x - q01) / (q99 - q01 + 1e-6) - 1
```

这里有两条不同路线。

### 路线 A：复用 UR5e 统计

xArm 和 UR5e 都是六轴单臂加夹爪，所以当前示例先复用 base checkpoint 里的 `ur5e` stats：

```python
assets=AssetsConfig(
    assets_dir=(
        "gs://openpi-assets/"
        "checkpoints/pi05_base/assets"
    ),
    asset_id="ur5e",
)
```

这样做的好处是：模型看到的数值尺度更接近预训练阶段。缺点是：xArm 的关节零位、可达范围和运动分布不一定真的与 UR5e 相同。

### 路线 B：计算新的 xArm 统计

如果要用自己的统计，先把显式指向 UR5e 的 assets 配置去掉：

```python
data=LeRobotXArmDataConfig(
    repo_id="<your-hf-username>/xarm_dataset",
    assets=AssetsConfig(),
    base_config=DataConfig(
        prompt_from_task=True,
    ),
)
```

然后运行：

```bash
cd "$OPENPI_ROOT"

uv run scripts/compute_norm_stats.py \
  --config-name pi05_xarm
```

统计会写到当前配置的 assets 目录中，大致类似：

```text
assets/
  pi05_xarm/
    <your-hf-username>/
      xarm_dataset/
        norm_stats.json
```

这里有一个很隐蔽的坑：

```text
如果 config 仍然写着 assets_dir=gs://... 和 asset_id=ur5e
即使运行了 compute_norm_stats.py
训练时仍然会加载 UR5e stats
```

也就是说，“运行过统计脚本”和“训练真的使用新统计”不是一回事。要做 fresh xArm stats 实验，必须同时修改 `AssetsConfig`。

我更建议把两种方案当成两个独立实验：

```text
xarm_ur5e_stats
xarm_fresh_stats
```

不要在同一个实验训练到一半时切换统计。归一化空间变了，模型输入输出的数值语义也会变。

## 第六步：先跑一个短训练

安装 openpi 环境：

```bash
cd "$OPENPI_ROOT"

GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
```

第一次不要直接跑 30000 step。先用 200 step 验证数据、显存和 checkpoint：

```bash
cd "$OPENPI_ROOT"

XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 \
uv run scripts/train.py pi05_xarm \
  --exp-name xarm_smoke \
  --num-train-steps 200 \
  --save-interval 100 \
  --overwrite
```

短训练主要检查：

```text
能否找到正确的 LeRobot dataset
是否加载了预期的 normalization stats
state/action/image shape 是否正确
首个 batch 的相机图像是否正常
base checkpoint 是否能加载
loss 和 grad_norm 是否有限
是否能保存并重新加载 checkpoint
```

训练脚本启动后会打印第一批数据的大致结构。xArm 数据应该能看到类似：

```text
state: [batch, 32]
actions: [batch, 10, 32]
images/base_0_rgb: [batch, 224, 224, 3]
images/left_wrist_0_rgb: [batch, 224, 224, 3]
```

这里的 32 是模型 padding 后的维度，不代表 xArm 突然有 32 个自由度。

## 正式训练

短训练正常后再启动正式实验：

```bash
cd "$OPENPI_ROOT"

XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 \
uv run scripts/train.py pi05_xarm \
  --exp-name xarm_run_001 \
  --overwrite
```

当前默认参数大致是：

```text
num_train_steps: 30000
batch_size: 64
log_interval: 100
save_interval: 1000
keep_period: 5000
ema_decay: 0.999
```

checkpoint 会保存到：

```text
checkpoints/
  pi05_xarm/
    xarm_run_001/
      1000/
      2000/
      3000/
      ...
```

中断后继续：

```bash
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 \
uv run scripts/train.py pi05_xarm \
  --exp-name xarm_run_001 \
  --resume
```

`--resume` 和 `--overwrite` 不能一起使用：

```text
--resume: 恢复已有实验
--overwrite: 删除/覆盖同名实验并从头开始
```

## 显存和多卡

openpi 给出的经验显存需求大致是：

```text
推理: > 8 GB
LoRA 微调: > 22.5 GB
全量微调: > 70 GB
```

当前 xArm 示例配置默认是全量微调，80 GB 级别 GPU 更合适。如果显存不足，可以先降低 batch size，或者使用多卡 FSDP：

```bash
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 \
uv run scripts/train.py pi05_xarm \
  --exp-name xarm_run_001 \
  --batch-size 32 \
  --fsdp-devices 2 \
  --overwrite
```

全局 batch size 必须能被设备数整除。

如果仍然 OOM，我通常按下面的顺序排查：

```text
确认 GPU 上没有其他大进程
降低 batch size
增加 FSDP 设备数
考虑关闭 EMA
建立与当前模型 variant 对应的 LoRA 配置
```

不要为了省显存随手改变 action horizon、相机数量和模型 variant，然后继续把实验和旧结果直接比较。那已经是不同的训练条件。

## 训练循环内部发生了什么

JAX 训练主循环可以简化为：

```python
data_loader = create_data_loader(
    config,
    shuffle=True,
)
data_iter = iter(data_loader)

train_state = init_train_state(
    config,
    init_rng,
    mesh,
)

for step in range(
    start_step,
    config.num_train_steps,
):
    batch = next(data_iter)
    train_state, info = train_step(
        config,
        train_rng,
        train_state,
        batch,
    )

    if step % config.log_interval == 0:
        log(info)

    if step % config.save_interval == 0:
        save_checkpoint(train_state)
```

π0 / π0.5 的 action head 使用 flow matching。可以用一个不严格但直观的理解：

```text
训练时:
真实动作 + 不同程度噪声
  -> 模型学习把噪声动作推回专家动作的方向

推理时:
从噪声动作开始
  -> 多步更新
  -> 得到完整 action chunk
```

π0-FAST 则把连续动作压缩成离散 token，自回归生成后再解码回连续动作。

## loss 下降不等于会抓取

训练 loss 下降，只能说明模型更接近训练目标。它仍然可能学到：

```text
输出数据集平均姿态
只会从固定 home 开始运动
只记住红色方块的位置
接近物体但不会闭爪
正确闭爪但没有学会抬升
背景一变化就不动
```

所以评估最好拆成三层：

```text
训练内基线:
  复现某一条训练 episode 的物体、位置和背景

同分布新组合:
  颜色、位置、背景都来自训练范围，但组合没见过

轻微分布外:
  在训练范围外做小幅变化，观察模型怎样退化
```

训练内基线都做不到时，不要先讨论泛化。应该先检查：

```text
checkpoint 是否正确
normalization stats 是否正确
相机是否一致
关节零位约定是否一致
控制频率是否与 fps 一致
客户端是否把 absolute action 当成了 delta
```

## checkpoint 里不只有模型参数

一个可部署的 checkpoint 还需要包含：

```text
模型参数
normalization stats
训练配置对应的 input/output transforms
机器人和动作空间 metadata
```

xArm 配置可以写入：

```python
policy_metadata = {
    "robot_type": "xarm6",
    "action_dim": 7,
    "control_hz": 10.0,
    "joint_convention": "xarm-standard",
    "training_robot_asset": "agx_gripper.usd",
    "wrist_camera_prim": "/World/Robot/link6/Camera",
}
```

这些字段会在远程推理连接建立时发给客户端。客户端可以先确认模型确实属于当前机器人、当前控制频率和当前相机，再决定是否执行动作。

## 常见问题

### 转换时报长度不一致

说明某个采集通道少帧或多帧。不要直接把所有数组裁到最短长度。图像和状态可能已经发生时间错位，应该回到采集端排查。

### 训练找不到 dataset

检查：

```text
转换命令的 repo-id 和 config 是否完全相同
是否在同一个 HF_LEROBOT_HOME 下运行
Hub dataset 是否存在并有读取权限
本地 dataset 是否被另一次 --overwrite 删除
```

### 报 normalization stats 不存在

先决定要复用 UR5e stats 还是计算 fresh xArm stats。如果使用 fresh stats，确认配置已经移除指向 GCS/UR5e 的 `AssetsConfig`，再运行统计脚本。

### 模型只会保持不动

优先检查：

```text
自动数据是否错误保存了 actions ≈ joints
是否正确用 t+1 状态生成 action
delta transform 是否被重复应用
state 和 action 是否混入两种关节零位约定
episode 是否包含足够明显的运动
```

### loss 变成 NaN

检查 raw 数据和 norm stats 是否有 NaN/Inf；检查某个维度的 `std` 或 `q99-q01` 是否接近 0；再检查学习率和混合数据中的动作单位。

### 加了第三人称图像但模型没有使用

确认四件事：

```text
NPZ 中有 image
转换时有 --use-base-image
训练配置有 use_base_image=True
推理客户端会发送 base_rgb
```

## 小结

这一阶段真正完成的是下面这条链路：

```text
episode_*.npz
  -> 数据字段和时间对齐检查
  -> LeRobot dataset
  -> XArmInputs / XArmOutputs
  -> joint absolute <-> delta transform
  -> normalization
  -> 图像和 prompt 预处理
  -> π0 / π0.5 / π0-FAST 微调
  -> checkpoint + stats + metadata
```

对我来说，这部分最容易误判的地方有两个：

1. 训练命令能跑，不代表 action 的时间语义和关节约定正确。
2. 运行过 `compute_norm_stats.py`，不代表训练真的加载了刚算出的 stats。

把数据、transform、normalization 和部署 metadata 一起当成模型契约，后面的远程推理才有可能稳定工作。
