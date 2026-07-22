---
layout: post
title: "让 xArm 远程调用 openpi：H100 policy server、WebSocket 和安全闭环"
date: 2026-07-22
description: "把微调后的 openpi checkpoint 部署到 GPU 服务器，通过 WebSocket 接收 xArm 图像和状态、返回 action chunk，并在 Isaac Sim 或机器人客户端中完成带安全门控的闭环推理。"
categories: [robotics, embodied-ai, openpi, inference]
---

完成数据采集和 openpi 微调后，下一步是把 checkpoint 真正跑进机器人闭环。

我这里采用的是远程推理结构：模型放在 H100 服务器上，Isaac Sim 或 AGX Orin 只运行轻量客户端。客户端采集图像和机器人状态，通过 WebSocket 发给服务器，再把返回的 action chunk 交给 xArm 控制器。

这篇是这个系列的部署篇：重点不只是“服务端能返回动作”，而是客户端怎样确认动作属于当前机器人，并在异常时主动停下来。

TL;DR:

```text
Isaac Sim / AGX Orin
  -> 读取 wrist RGB、joints、gripper、prompt
  -> MessagePack + WebSocket
  -> H100 openpi policy server
  -> input transforms + normalization + model inference
  -> 返回 [10, 7] action chunk
  -> 客户端检查限位、跳变和跟踪误差
  -> 执行前 5 步
  -> 重新观察并规划
```

为了避免泄露个人机器路径，本文统一使用：

```bash
export OPENPI_ROOT=/path/to/openpi
export ISAAC_SIM_ROOT=/path/to/isaac-sim-standalone-5.1.0
export POLICY_HOST="<server-ip-or-hostname>"
export POLICY_PORT=8000
```

## 为什么要把模型和机器人环境分开

openpi 推理需要 GPU、JAX/PyTorch、checkpoint 和一套比较重的依赖。机器人端则往往已经有：

```text
Isaac Sim 自带 Python
xArm SDK
相机 SDK
ROS 2
底盘和夹爪驱动
```

把所有依赖塞进同一个环境，很容易遇到 Python、CUDA、动态库和包版本冲突。

拆开后，两台机器职责比较清楚：

```text
GPU 服务器:
  加载 checkpoint
  加载 normalization stats
  执行图像和语言 transforms
  运行模型
  返回动作

机器人客户端:
  读取传感器
  构造 observation
  连接服务器
  校验动作
  稳定执行控制循环
  处理急停和 watchdog
```

这种结构还有一个好处：Isaac Sim 客户端和真实 AGX Orin 客户端可以复用同一个 policy server，只要两者遵守完全相同的数据契约。

## 远程推理不是“传一个数组”这么简单

完整链路包含三层问题：

```text
模型层:
  checkpoint、normalization stats、input/output transforms

通信层:
  WebSocket、MessagePack、连接和延迟

控制层:
  频率、关节约定、动作限位、跟踪误差和急停
```

网络能连通，只能说明通信层基本正常。服务器返回了 `[10,7]`，也不代表这 70 个数属于当前机器人，更不代表可以直接发给真实硬件。

## 服务端怎样恢复训练好的策略

服务入口是：

```text
scripts/serve_policy.py
```

对自训练 checkpoint，服务端会做：

```text
读取 --policy.config
  -> 找到训练时的 model 和 data config
读取 --policy.dir
  -> 恢复 JAX params 或 PyTorch model.safetensors
读取 checkpoint/assets
  -> 加载训练时 normalization stats
重建 input/output transforms
  -> 创建可调用 policy
启动 WebSocket server
```

核心逻辑可以简化成：

```python
def create_policy(args):
    train_config = get_config(args.policy.config)

    return create_trained_policy(
        train_config,
        args.policy.dir,
        default_prompt=args.default_prompt,
    )
```

创建 trained policy 时，输入 transform 的顺序大致是：

```python
input_transforms = [
    *repack_transforms.inputs,
    InjectDefaultPrompt(default_prompt),
    *data_config.data_transforms.inputs,
    Normalize(
        norm_stats,
        use_quantiles=data_config.use_quantile_norm,
    ),
    *data_config.model_transforms.inputs,
]
```

输出 transform 反过来：

```python
output_transforms = [
    *data_config.model_transforms.outputs,
    Unnormalize(
        norm_stats,
        use_quantiles=data_config.use_quantile_norm,
    ),
    *data_config.data_transforms.outputs,
    *repack_transforms.outputs,
]
```

这意味着客户端发送的是原始物理量：

```text
joints: rad
gripper: [0,1]
image: uint8 RGB
```

客户端不需要手工归一化。服务器会使用 checkpoint 中与训练一致的 stats。

模型输出的 delta joint action 也会在服务端恢复成绝对关节目标，然后再返回客户端。

## 在 GPU 服务器上启动 policy server

假设使用 `pi05_xarm` 的 30000 step checkpoint：

```bash
cd "$OPENPI_ROOT"

uv run scripts/serve_policy.py \
  --port="$POLICY_PORT" \
  policy:checkpoint \
  --policy.config=pi05_xarm \
  --policy.dir=checkpoints/pi05_xarm/xarm_run_001/30000
```

这里参数顺序值得注意：

```text
--port 是 serve_policy.py 的全局参数
必须写在 policy:checkpoint 子命令之前
```

下面这种写法在当前 Tyro CLI 中会被当成未知参数：

```bash
# 错误示例
uv run scripts/serve_policy.py \
  policy:checkpoint \
  --policy.config=pi05_xarm \
  --policy.dir=checkpoints/pi05_xarm/xarm_run_001/30000 \
  --port=8000
```

第一次启动需要恢复权重和加载 tokenizer；JAX 的第一次实际推理通常还会触发 JIT 编译，所以首个请求会明显慢于后续请求。

服务正常监听后，可以检查健康端点：

```bash
curl "http://127.0.0.1:${POLICY_PORT}/healthz"
```

正常返回：

```text
OK
```

也可以检查监听地址：

```bash
ss -ltnp | grep ":${POLICY_PORT}"
```

健康检查只能说明进程和端口正常，不代表某条 observation 能成功通过模型，也不代表动作能通过客户端安全门控。

## 服务端 metadata 是部署契约

启动 server 时，我给 xArm 训练配置写入了：

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

加载 checkpoint 后，服务端还会补充：

```python
metadata.update(
    {
        "config_name": train_config.name,
        "checkpoint_source": checkpoint_source,
        "action_horizon": int(
            train_config.model.action_horizon
        ),
        "model_action_dim": int(
            train_config.model.action_dim
        ),
        "model_type": str(
            train_config.model.model_type
        ),
        "norm_stats_sha256": norm_stats_fingerprint,
    }
)
```

还可以记录服务端代码版本：

```text
server_git_commit
server_git_dirty
```

这些字段能回答：

```text
这是不是 xArm6 模型
动作是不是 7 维
horizon 是不是 10
训练频率是不是 10 Hz
关节是不是标准/真机约定
训练和推理是不是同一个机器人资产
腕部相机路径是不是一致
normalization stats 是不是同一份
两端代码是不是来自同一个 commit
```

客户端在建立连接后先校验 metadata，再发送 observation。这样比“看到 shape 是 `[10,7]` 就执行”安全得多。

## WebSocket 连接过程

当前服务使用长连接 WebSocket。连接建立后，第一条消息不是 observation，而是 server metadata：

```text
client                         server
  |------ WebSocket connect ---->|
  |<--------- metadata -----------|
  |---------- observation ------->|
  |<------- action response -------|
  |---------- observation ------->|
  |<------- action response -------|
```

服务端 handler 的核心结构：

```python
async def handler(websocket):
    packer = msgpack_numpy.Packer()

    await websocket.send(
        packer.pack(server_metadata)
    )

    while True:
        obs = msgpack_numpy.unpackb(
            await websocket.recv()
        )

        result = policy.infer(obs)

        await websocket.send(
            packer.pack(result)
        )
```

客户端初始化时接收 metadata：

```python
class WebsocketClientPolicy:
    def __init__(self, host, port):
        self.uri = f"ws://{host}:{port}"
        self.packer = msgpack_numpy.Packer()
        self.ws, self.server_metadata = (
            self.wait_for_server()
        )

    def infer(self, obs):
        self.ws.send(self.packer.pack(obs))
        response = self.ws.recv()
        return msgpack_numpy.unpackb(response)
```

## 为什么不用 JSON

224×224×3 的 RGB 图像包含 150528 个 `uint8`。如果展开成 JSON 整数列表，体积和解析开销都会很大。

这里使用 MessagePack，并把 NumPy 数组编码成：

```text
data: 原始 bytes
dtype: NumPy dtype string
shape: 数组 shape
```

简化代码：

```python
def pack_array(obj):
    if isinstance(obj, np.ndarray):
        if obj.dtype.kind in ("V", "O", "c"):
            raise ValueError(
                f"Unsupported dtype: {obj.dtype}"
            )

        return {
            b"__ndarray__": True,
            b"data": obj.tobytes(),
            b"dtype": obj.dtype.str,
            b"shape": obj.shape,
        }

    return obj
```

解码：

```python
def unpack_array(obj):
    if b"__ndarray__" in obj:
        return np.ndarray(
            buffer=obj[b"data"],
            dtype=np.dtype(obj[b"dtype"]),
            shape=obj[b"shape"],
        )

    return obj
```

实现会拒绝 object、void 和 complex dtype，不会退回 pickle。

当前 WebSocket 关闭 compression，并取消默认消息大小限制。所以更应该在客户端先把图像 resize 到 224×224，避免发送不必要的高分辨率画面。

## 服务端返回什么

一次响应大致是：

```python
{
    "actions": np.ndarray,  # [10, 7]
    "policy_timing": {
        "infer_ms": 42.0,
    },
    "server_timing": {
        "infer_ms": 45.0,
        "prev_total_ms": 48.0,
    },
}
```

π0-FAST 还应该返回：

```text
action_decode_ok = true
```

客户端如果不能确认 FAST action token 已经成功解码，就拒绝执行动作。

## 在 Isaac Sim 环境安装轻量客户端

机器人端不需要完整 openpi，只需要 `openpi-client`。

Isaac Sim 必须使用自己的 Python：

```bash
"$ISAAC_SIM_ROOT/python.sh" -m pip install -e \
  "$OPENPI_ROOT/packages/openpi-client"
```

Isaac Sim 推理客户端脚本位于：

```text
third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/run_xarm_policy.py
```

这个脚本会创建与训练接近的机器人、桌面、物体、相机和光照，然后进入远程闭环。

## 先诊断相机，不要一上来就动机械臂

模型最重要的输入之一是腕部图像。如果相机 prim 选错、图像全黑或者相机没有跟随 link6，模型输出再“合理”也没有意义。

可以先运行相机诊断模式：

```bash
cd "$OPENPI_ROOT"

"$ISAAC_SIM_ROOT/python.sh" \
  third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/run_xarm_policy.py \
  --host 127.0.0.1 \
  --diagnose-cameras \
  --diagnose-dir /tmp/xarm_cam_diag
```

诊断模式会保存 Stage 中每个相机的画面后退出，不连接策略，也不运动机械臂。

重点检查：

```text
/World/Robot/link6/Camera 是否存在
画面是否能看到夹爪和目标物体
机械臂运动时相机是否跟着 link6
RGB 是否上下颠倒或过曝
是否存在全黑图
```

当前推理脚本使用较高分辨率渲染，再 resize 到 224×224。原因是某些 RTX/DLSS 配置在过低渲染分辨率下会直接返回黑帧。

图像 reader attach 后还要 warm up，因为前几帧可能为空。验证逻辑大致是：

```python
def require_usable_rgb(
    image,
    expected_shape,
    min_mean=1.0,
    min_nonzero_fraction=0.01,
):
    image = np.asarray(image)

    if image.shape != expected_shape:
        raise RuntimeError("RGB shape mismatch")

    if not np.isfinite(image).all():
        raise RuntimeError("RGB contains NaN/Inf")

    mean = float(image.mean())
    nonzero = float(
        np.count_nonzero(image) / image.size
    )

    if mean < min_mean:
        raise RuntimeError("RGB is too dark")

    if nonzero < min_nonzero_fraction:
        raise RuntimeError("RGB is nearly empty")
```

运行中如果突然出现黑帧，不应该复用上一张正常图。旧图会让机械臂在视觉已经冻结的情况下继续运动。

## 启动 Isaac Sim 闭环客户端

服务端已经启动、相机已经确认后，再运行：

```bash
cd "$OPENPI_ROOT"

"$ISAAC_SIM_ROOT/python.sh" \
  third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/run_xarm_policy.py \
  --host "$POLICY_HOST" \
  --port "$POLICY_PORT" \
  --prompt "pick up the red block" \
  --dump-wrist-image /tmp/xarm_policy_input.png
```

第一次不要加 `--headless`。我会同时观察：

```text
机器人初始姿态是否与训练 home 接近
物体颜色和 prompt 是否一致
物体位置是否落在训练分布中
发给模型的第一帧 wrist image 是否正常
第一个 action 是否通过安全检查
机械臂是否能跟踪每一步目标
```

也可以用某一条训练 episode 的 metadata 构造基线：

```bash
"$ISAAC_SIM_ROOT/python.sh" \
  third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/run_xarm_policy.py \
  --host "$POLICY_HOST" \
  --port "$POLICY_PORT" \
  --baseline-episode /path/to/auto_episode_0000.npz
```

客户端会读取：

```text
object
cube_color
prompt
cube_spawn_sampled
background
```

然后尽量复现训练内场景。原始数据没有保存的 HDRI 强度或旋转仍会使用默认值，因此不一定逐像素相同，但比手工猜一个场景更可靠。

## 客户端发送的 observation

每次重新规划时，客户端读取：

```python
raw_wrist_rgb = read_wrist_rgb()

wrist_rgb = image_tools.convert_to_uint8(
    image_tools.resize_with_pad(
        raw_wrist_rgb,
        224,
        224,
    )
)

observation = {
    "wrist_rgb": wrist_rgb,
    "joints": standard_joints,
    "gripper": np.asarray(
        [gripper_01],
        dtype=np.float32,
    ),
    "prompt": prompt,
}

response = policy.infer(observation)
```

这里几个契约必须与训练一致：

```text
wrist_rgb: HWC uint8 RGB，不是 BGR
joints: [6]，rad，标准/真机零位
gripper: [1]，0=打开，1=闭合
prompt: 普通字符串
```

如果训练使用了第三人称相机：

```python
LeRobotXArmDataConfig(
    use_base_image=True,
)
```

客户端也必须读取同语义相机并发送：

```python
observation["base_rgb"] = base_rgb
```

当前基础推理脚本默认只发送 `wrist_rgb`。双相机训练后需要显式扩展客户端，不能让训练和推理的 image mask 静默变化。

## action chunk 和滚动规划

当前默认控制参数：

```text
physics_dt: 1/60 s
control_dt: 0.1 s
control frequency: 10 Hz
action_horizon: 10
replan_steps: 5
```

一个 0.1 秒动作周期对应：

```text
0.1 / (1/60) = 6 个 physics steps
```

客户端要求 `control_dt` 能被表示成整数个物理步：

```python
def physics_steps_for_control_dt(
    control_dt,
    physics_dt,
):
    steps = round(control_dt / physics_dt)
    effective_dt = steps * physics_dt

    if not np.isclose(
        effective_dt,
        control_dt,
    ):
        raise ValueError(
            "control_dt cannot be represented "
            "by integer physics steps"
        )

    return steps, effective_dt
```

服务器一次返回 10 个动作，但客户端默认只执行前 5 个：

```text
t=0.0 s:
  observation 0 -> [a0, a1, ... a9]

t=0.0 ~ 0.5 s:
  execute a0 ... a4

t=0.5 s:
  observation 1 -> new action chunk
```

这是一种 receding-horizon 控制：

```text
replan_steps=1:
  反馈最及时，但每个动作都要等待网络和模型

replan_steps=5:
  每 0.5 秒重新观察，延迟和反馈之间折中

replan_steps=10:
  推理次数少，但开环执行约 1 秒
```

核心循环：

```python
action_plan = []

while simulation_app.is_running():
    if not action_plan:
        response = policy.infer(
            build_observation()
        )
        safe_chunk = validate_action_chunk(
            response["actions"]
        )
        action_plan = list(
            safe_chunk[:replan_steps]
        )

    action = action_plan.pop(0)
    execute_action(action)
```

## 仿真时间和墙上时间

默认客户端每个 action 固定推进 6 个物理步，不强制墙上时间恰好等待 0.1 秒。

```text
仿真快时: 10 Hz 仿真控制可能快于真实时间
仿真慢时: 10 Hz 仿真控制可能慢于真实时间
但仿真内部每个 action 的间隔仍然是 0.1 秒
```

加上：

```bash
--realtime-pacing
```

才会用墙上时钟把每个动作周期补到 `control_dt`。

真实机器人端不能只靠普通 `time.sleep(0.1)` 就假设自己拥有稳定的 10 Hz 控制。更稳的做法是把网络推理和硬件控制拆成两个线程或进程。

## 最容易出错的关节零位换算

训练数据使用标准/真机 xArm 约定。某些 AGX 仿真资产使用带偏移的关节零位，因此需要：

```text
q_standard = q_sim + delta
```

发送 observation 时：

```python
sim_joints = np.asarray(
    [read_joint(name) for name in joint_names],
    dtype=np.float32,
)

standard_joints = sim_joints + zero_offsets
```

执行服务器返回的绝对动作时：

```python
sim_joint_targets = (
    action[:6] - zero_offsets
)
```

默认推理资产 `agx_gripper.usd` 使用：

```text
--robot-native-convention agx-offset
```

如果换成原生关节值已经是标准约定的资产，则应该使用：

```text
--robot-native-convention standard
```

不能对标准约定再加一次 offset。最典型的现象是 joint2 从训练 home 附近的 `-1.58` 变成 `-3.16`，机械臂一启动就扭到奇怪姿态。

客户端启动时会把当前关节转换成 standard，并与训练 home 比较：

```python
training_home = np.asarray(
    [0.0, -1.58, 0.0, 0.0, -1.57, 3.14],
    dtype=np.float32,
)

max_deviation = np.max(
    np.abs(standard_joints - training_home)
)

if max_deviation > 0.3:
    raise RuntimeError(
        "Start joints do not match training home"
    )
```

这个检查应该在连接模型之前完成。

## 客户端怎样校验 server contract

客户端预期：

```python
required = {
    "robot_type": "xarm6",
    "action_dim": 7,
    "action_horizon": 10,
    "control_hz": 10.0,
    "joint_convention": "xarm-standard",
}
```

简化检查：

```python
def validate_server_contract(metadata):
    for key, expected in required.items():
        if key not in metadata:
            raise RuntimeError(
                f"Missing server metadata: {key}"
            )

        actual = metadata[key]
        if actual != expected:
            raise RuntimeError(
                f"Contract mismatch: "
                f"{key}={actual}, expected={expected}"
            )
```

真实实现还会比较训练机器人资产和腕部相机 prim。

调试旧 server 时虽然可以提供“允许 metadata 缺失”或“允许契约不一致”的开关，但不应该把这些参数写进长期启动命令。如果每次都要跳过检查才能运行，说明部署契约还没对齐。

## action chunk 安全检查

服务端动作在进入仿真或真实机器人前，需要经过 fail-closed 检查。

### Shape 和有限值

```python
chunk = np.asarray(
    actions,
    dtype=np.float32,
)

if chunk.shape != (10, 7):
    raise RuntimeError(
        f"Unexpected action shape: {chunk.shape}"
    )

if not np.isfinite(chunk).all():
    raise RuntimeError(
        "Action contains NaN or Inf"
    )
```

### 关节限位

```python
safe_lower = joint_lower + limit_margin
safe_upper = joint_upper - limit_margin

arm = chunk[:, :6]

if np.any(arm < safe_lower[None, :]):
    raise RuntimeError("Action below joint limit")

if np.any(arm > safe_upper[None, :]):
    raise RuntimeError("Action above joint limit")
```

### 相邻动作跳变

```python
joint_steps = np.abs(
    np.diff(
        np.vstack([
            current_joints[None, :],
            arm,
        ]),
        axis=0,
    )
)

if joint_steps.max() > 0.10:
    raise RuntimeError(
        "Joint target jump is too large"
    )
```

这里的 0.10 rad 是每 0.1 秒目标变化上限，粗略对应 1 rad/s，但它不是完整的速度和加速度规划器。

### 夹爪范围和跳变

```python
gripper = chunk[:, 6]

if np.any(gripper < -0.02):
    raise RuntimeError("Gripper below 0")

if np.any(gripper > 1.02):
    raise RuntimeError("Gripper above 1")

gripper = np.clip(gripper, 0.0, 1.0)
```

还要检查第一个 gripper target 与当前实测状态的差，以及 chunk 内相邻目标的跳变。

### FAST 解码状态

```python
if is_fast_model:
    if not response.get(
        "action_decode_ok",
        False,
    ):
        raise RuntimeError(
            "FAST action decoding failed"
        )
```

## 持续检查关节跟踪误差

动作本身合法，不代表机器人真的跟得上。

每执行一个 action 后，客户端比较：

```python
tracking_error = np.abs(
    measured_joints - last_commanded_joints
)

if tracking_error.max() > 0.25:
    raise RuntimeError(
        "Joint tracking error is too large"
    )
```

跟踪误差过大可能表示：

```text
机械臂撞到物体或自身
drive stiffness / damping 不合理
控制频率错误
关节目标变化太快
仿真动力学不稳定
真实机器人进入保护状态
```

继续消费旧 action chunk 会让目标和实测状态越差越远，所以这里应该直接停止并重新进入安全状态。

## 异常时固定当前位姿

仿真客户端发生异常时，可以把每个 DOF target 写成当前实测位置：

```python
def emergency_hold(
    dc,
    joint_handles,
    gripper_handles,
):
    for handle in joint_handles.values():
        current = dc.get_dof_position(handle)
        dc.set_dof_position_target(
            handle,
            float(current),
        )

    for handle in gripper_handles:
        current = dc.get_dof_position(handle)
        dc.set_dof_position_target(
            handle,
            float(current),
        )
```

这只是仿真中的软件保护，不是真实机器人急停。真机必须有独立硬件急停、驱动器限位和通信 watchdog。

## 这些安全检查仍然不够

现有检查能拦住：

```text
黑图
错误 shape
NaN / Inf
关节越限
单步大跳变
夹爪范围错误
明显跟踪失败
```

但它不能证明一条动作路径在空间中安全。真实机器人至少还需要：

```text
关节速度和加速度限制
末端工作空间限制
机身和桌面禁入区
自碰撞和环境碰撞检查
奇异点监测
力矩、电流和接触监测
通信超时 watchdog
人员进入工作区检测
硬件急停和人工使能
```

所有 action 的终点都在关节限位内，也不代表从当前位置运动到目标的过程中不会扫到桌子、底盘或人。

## 延迟应该怎么看

响应中会带两类时间：

```text
policy_timing.infer_ms:
  模型 sample_actions 本身耗时

server_timing.infer_ms:
  服务端整个 policy.infer 耗时

server_timing.prev_total_ms:
  上一个请求从接收到发送结束的总耗时
```

总延迟可以拆成：

```text
客户端读取和预处理图像
  + MessagePack 编码
  + 上行网络
  + 服务端解码和 transforms
  + 模型推理
  + 输出 transforms 和编码
  + 下行网络
```

如果 `policy_timing` 很小，但客户端等了很久，问题更可能在网络、序列化或排队。如果模型和 server timing 都很大，则要检查 GPU 负载、JAX 首次编译和推理参数。

action chunk 可以摊薄推理延迟，但不能让旧 observation 变新。网络断开时不能一直执行旧动作，客户端必须有明确超时和 watchdog。

## 调试时记录服务端输入输出

服务端可以打开记录模式：

```bash
cd "$OPENPI_ROOT"

uv run scripts/serve_policy.py \
  --port="$POLICY_PORT" \
  --record \
  policy:checkpoint \
  --policy.config=pi05_xarm \
  --policy.dir=checkpoints/pi05_xarm/xarm_run_001/30000
```

注意 `--record` 也是全局参数，需要放在子命令之前。

记录内容包含：

```text
客户端 observation
服务端 action response
policy timing
```

图像很快会占用磁盘，也可能包含真实环境画面。只在调试时开启，并注意磁盘容量和数据隐私。

## 网络安全

当前服务绑定：

```text
0.0.0.0:<port>
```

基础实现没有启用 TLS，也没有在服务端真正校验 API key。客户端类里即使存在 `api_key` 参数，也不能自动把当前服务变成有认证的接口。

所以不应该把 policy 端口直接暴露到公网。至少使用下面一种方式：

```text
受信任实验室局域网
主机防火墙限制客户端 IP
VPN
SSH tunnel
带 TLS 和鉴权的反向代理
```

SSH 本地转发示例：

```bash
ssh -N \
  -L 8000:127.0.0.1:8000 \
  YOUR_USER@SERVER_HOST
```

之后客户端连接：

```text
127.0.0.1:8000
```

SSH 隧道解决链路加密和登录认证，不会替代机器人动作安全检查。

## 从 Isaac Sim 换成真实 xArm

真实机器人端的基本 observation 仍然是：

```python
observation = {
    "wrist_rgb": wrist_rgb_uint8,
    "joints": joints_rad,
    "gripper": gripper_01,
    "prompt": task_prompt,
}
```

但下面这些函数必须接到自己的硬件：

```python
def get_wrist_rgb():
    """从相机 SDK 或 ROS topic 读取 HWC RGB。"""
    raise NotImplementedError


def get_joints_rad():
    """从 xArm SDK 读取六关节弧度。"""
    raise NotImplementedError


def get_gripper_01():
    """把真实夹爪位置映射到 [0,1]。"""
    raise NotImplementedError


def send_action_to_xarm(action):
    """安全检查后发送 6+1 维绝对目标。"""
    raise NotImplementedError
```

真机控制建议拆成：

```text
推理线程:
  低频获取 action chunk

动作缓冲区:
  只保存已通过检查的目标

控制线程:
  稳定 10 Hz 消费动作

安全线程/硬件:
  更高频检查限位、急停、碰撞和 watchdog
```

不要在 WebSocket 回调里直接调用硬件 SDK，再用一个 `sleep(0.1)` 同时承担网络、推理和控制调度。

## 常见问题

### 客户端一直显示 waiting for server

依次检查：

```text
server 是否完成 checkpoint 加载
/healthz 是否能访问
端口是否监听 0.0.0.0
防火墙或容器端口映射是否正确
客户端 host 是否写错
VPN / SSH tunnel 是否建立
```

客户端遇到 `ConnectionRefusedError` 会定期重试。“Still waiting”不代表服务器正在推理。

### server metadata mismatch

不要先加跳过检查的参数。直接比较：

```text
config_name
checkpoint_source
action_horizon
control_hz
joint_convention
training_robot_asset
wrist_camera_prim
norm_stats_sha256
server_git_commit
```

最常见原因是启动了错误 checkpoint、客户端代码落后、训练使用双相机，或者更换 USD 后没有修改关节约定。

### 机械臂一启动就扭到奇怪姿态

立即停止并检查：

```text
q_sim + delta 是否接近训练 home
当前资产应该使用 standard 还是 agx-offset
客户端是否把 absolute action 又加了一次 current state
关节单位是不是 degree 而不是 rad
关节顺序是不是 joint1 ~ joint6
```

### 相机 warmup 超时

先运行相机诊断，再检查：

```text
Camera prim
渲染分辨率
RTX / DLSS 状态
灯光
相机朝向
是否真的 attach 到 render product
```

不要把 `min_mean` 调成 0 来绕过黑图。

### action 被单步跳变检查拒绝

先看被拒绝的 row、joint、当前实测值和目标值。可能原因：

```text
checkpoint 和 stats 错配
起始姿态不在训练分布
控制频率错误
模型还没学好
关节零位错误
输入图像严重 OOD
```

不要第一反应就把 0.10 rad 阈值调大。

### 动作看起来平滑，但跟踪误差一直增大

检查：

```text
drive stiffness / damping / max effort
机械臂是否被碰撞卡住
physics_dt 和 control_dt
真实控制器模式
网络线程是否阻塞控制线程
```

如果目标平滑但实测跟不上，这是控制或动力学问题；如果目标本身跳变，才优先回到模型和数据检查。

### 能接近物体，但抓不起来

把失败拆成阶段：

```text
不朝物体运动:
  图像、prompt、checkpoint 或训练问题

到附近但对不准:
  相机外参、训练覆盖或闭环频率问题

对准但不闭爪:
  gripper 数据、normalization 或动作时序问题

闭爪但掉落:
  夹爪映射、摩擦、接触或执行速度问题
```

## 推荐的上线顺序

不要同时引入“新 checkpoint + 新场景 + 真机”。更稳的顺序是：

```text
1. server 本机健康检查
2. 离线 observation 通信测试
3. Isaac Sim 相机诊断
4. Isaac Sim 训练 episode 基线
5. Isaac Sim 同分布新组合
6. Isaac Sim 轻微分布外测试
7. 真机只读 observation，不下发动作
8. 真机架空或隔离工作区低速测试
9. 真机固定基线任务
10. 再扩大任务和场景范围
```

每一级都应该有明确的停止条件和回退方法。

## 小结

远程推理最终打通的是：

```text
xArm / Isaac Sim observation
  -> MessagePack
  -> WebSocket
  -> H100 policy server
  -> checkpoint transforms + normalization
  -> model action chunk
  -> unnormalize + absolute joint target
  -> client contract validation
  -> action safety checks
  -> 10 Hz execution
  -> new observation and replanning
```

真正贯穿整个采集、微调和推理过程的，是一组始终一致的契约：

```text
图像来自哪台相机
RGB 的 shape 和 dtype
关节使用什么零位约定
动作是 absolute 还是 delta
夹爪 0 和 1 分别代表什么
数据和控制频率是多少
action horizon 是多少
normalization stats 属于哪份数据
```

这些契约只要有一项在部署时静默变化，程序仍然可能“正常运行”，但机械臂会做错事。

所以远程推理最重要的部分不是 WebSocket 本身，而是让 checkpoint、metadata、客户端安全检查和真实机器人控制共同形成一个 fail-closed 的闭环。
