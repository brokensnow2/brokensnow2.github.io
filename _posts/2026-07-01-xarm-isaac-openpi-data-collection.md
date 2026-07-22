---
layout: post
title: "用 Isaac Sim 为 openpi 采集 xArm6 数据：从 DOF 调试到 marker teleop"
date: 2026-07-01
description: "在 Isaac Sim 5.1 中加载移动底盘 + xArm6 机器人，调通夹爪 DOF、marker teleop、episode 记录，并为后续 openpi / LeRobot 数据转换做准备。"
categories: [robotics, embodied-ai, data-collection]
---

这篇文章记录的是一次从仿真机器人加载到 episode 数据落盘的完整过程。目标不是“把 Isaac Sim 打开”，而是搭出一条后续能服务于 openpi / LeRobot 微调的数据采集链路。

这也是 xArm + openpi 实验系列的第一篇：先把数据怎么来、怎么保存、怎么确认可靠讲清楚，再谈后面的微调和远程推理。

TL;DR:

```text
Isaac Sim 5.1
  -> 加载移动底盘 + xArm6 + 夹爪 + 相机
  -> 用 marker teleop 控制末端
  -> 记录图像、关节、夹爪、动作和语言指令
  -> 保存为 episode_0000.npz
  -> 后续转换为 LeRobot / openpi 可用的数据
```

为了避免把个人机器目录写进博客，本文统一使用两个环境变量：

```bash
export OPENPI_ROOT=/path/to/openpi
export ISAAC_SIM_ROOT=/path/to/isaac-sim-standalone-5.1.0
```

读者只需要把它们替换成自己的 openpi 和 Isaac Sim 安装目录。后文出现的 `third_party/...` 都是相对于 openpi 仓库根目录的路径。

## 背景

我想先打通一条近距离 VLA 数据采集流程：给机器人一条语言指令，例如：

```text
pick up the red block
```

然后在 Isaac Sim 里采集：

```text
图像: 末端相机或其他机器人相机看到的画面
状态: xArm 当前关节位置和夹爪状态
动作: 下一步关节目标和夹爪目标
指令: 当前 episode 对应的 prompt
```

这些数据后面可以转换成 LeRobot 风格的数据集，再拿去微调 openpi 里的 π0、π0.5 或 π0-FAST。更长远一点，这条链路也可以扩展到自动生成 `press_button`、`pick_cup`、`place_cup`、`pick_colored_object` 这类近距离操作任务。

这里先强调一点：采集机器人数据和录屏不是一回事。视频看起来正常，只能证明渲染画面正常；真正可用于训练的数据还需要保证图像、状态、动作和时间戳在同一个 step 上正确对齐。

## 实验环境

当前用到的主要组件：

```text
Isaac Sim: 5.1.0 standalone
Robot: 移动底盘 + 升降柱 + xArm6 + 夹爪 + 云台相机 + 末端相机
Script: collect_xarm_episode.py
Data: episode_0000.npz
Action space: 6 个 xArm 关节目标 + 1 个夹爪目标
```

数据采集脚本位于：

```text
third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/collect_xarm_episode.py
```

默认加载的机器人 USD 是：

```text
third_party/isaac_sim_AGX/AGX_description/src/1557/urdf/1557_all/1557_all.usd
```

这个 USD 里已经包含移动底盘、xArm、夹爪、云台相机和末端相机。

![Isaac Sim 中加载的移动底盘和 xArm6 机器人](/assets/posts/xarm-isaac-openpi/isaac-robot-loaded.png)

图中可以看到，默认材质下机器人整体偏白，和背景有些混在一起。后面我也在脚本里加了运行时材质绑定，让 xArm、夹爪、云台和底盘更容易区分。

## 数据格式

当前每个 `.npz` 文件表示一个 episode：

```text
一个 .npz = 一个 episode / trajectory
每个 index t = 一个 step
```

人工采集目前保存的主要字段包括：

```text
wrist_image: [T, H, W, 3], uint8
joints: [T, 6], float32
gripper: [T, 1], float32
actions: [T, 7], float32
timestamp: [T], float64
prompt: str
target_marker_position: [T, 3]
base_wheel_velocity: [T]
base_turn_velocity: [T]
fps: scalar / metadata
```

这里还有一个当前人工脚本的实现细节：`gripper` 记录的是归一化软件目标 `gripper_target`，不一定是 PhysX 中手指真正到达的位置。如果夹爪被物体挡住，命令值和实测值会不同。后续自动采集脚本已经改成从主夹爪 DOF 反算实测 `[0,1]` 状态；如果人工数据也要用于正式训练，最好做同样修改。

其中 `actions` 采用 7 维：

```text
6 个 xArm 关节目标 + 1 个夹爪目标
```

也就是说，即使用 marker、IK 或 scripted expert 来控制机械臂，最终记录进数据集里的动作仍然保持为关节空间目标。这对后续和 openpi 的 xArm policy 对接会更直接。

![episode_0000.npz 内部保存的字段](/assets/posts/xarm-isaac-openpi/episode-fields.png)

## 状态和动作为什么不能随便记

机器人模仿学习通常希望一条样本表达：

```text
给定 t 时刻看到的图像和机器人状态
预测 t 之后应该执行的动作
```

因此下面几个概念要分开：

```text
实测关节位置: PhysX 当前真正模拟出来的 joints
关节目标: 控制器希望机械臂到达的 joint_targets
实测夹爪状态: 手指当前真正到达的位置
夹爪命令: 软件希望夹爪打开或闭合到什么程度
```

例如夹爪命令已经是 `1.0`，不代表手指真的完全闭合。如果中间夹住了物体，实测夹爪可能停在 `0.6`。模型的 state 应该使用实测值，而监督 action 才使用目标值。

人工 teleop 脚本里，`actions` 记录的是最近一次下发给控制器的目标。后续做自动采集时，我更倾向于只保存实测状态，再在转换阶段构造：

```text
actions[t] = concat(joints[t + 1], gripper[t + 1])
```

这样能避免把 RMPFlow 在 60 Hz 物理循环里给出的微小单步目标直接当成 10 Hz 训练动作。否则很容易得到：

```text
action[t] ≈ state[t]
```

模型最后学到的就是“保持不动”。

人工脚本当前按墙上时钟 `time.time()` 判断采样间隔，而自动采集会按 physics step 计算仿真时间。GUI 卡顿时，墙上时钟采样可能让相邻记录对应的物理步数不一致。因此人工模式更适合调试和少量示范；需要批量、可复现的数据时，最好使用仿真时钟。

## 子模块提交顺序

Isaac Sim 资产仓库是 openpi 主仓库里的 git submodule：

```text
third_party/isaac_sim_AGX
```

这意味着 openpi 主仓库不会直接记录子模块里的每个文件，只会记录子模块指向哪个 commit。

如果主仓库里看到类似这样的 diff：

```diff
diff --git a/third_party/isaac_sim_AGX b/third_party/isaac_sim_AGX
index <old-commit>..<new-commit> 160000
--- a/third_party/isaac_sim_AGX
+++ b/third_party/isaac_sim_AGX
@@ -1 +1 @@
-Subproject commit <old-commit>
+Subproject commit <new-commit>-dirty
```

意思是：主仓库记录的子模块 commit 已经变化，并且 `-dirty` 表示子模块内部还有未提交修改。

正确顺序是先提交子模块，再提交主仓库里的子模块指针：

```bash
cd "$OPENPI_ROOT/third_party/isaac_sim_AGX"
git add ros2_car_model_ws/src/scripts/collect_xarm_episode.py
git commit -m "Update Isaac xArm episode collection controls"
git push origin YOUR_BRANCH

cd "$OPENPI_ROOT"
git add third_party/isaac_sim_AGX
git commit -m "Update Isaac Sim AGX submodule"
```

这里不应该照抄某个固定分支名。用自己的开发分支和远程仓库即可。

## 启动脚本

不要用 openpi 的 Python 环境直接跑 Isaac Sim 脚本。Isaac Sim 有自己的 Python、扩展系统和动态库加载方式，应该使用它自带的 `python.sh`：

```bash
cd "$OPENPI_ROOT"

"$ISAAC_SIM_ROOT/python.sh" \
  third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/collect_xarm_episode.py \
  --mode marker \
  --output-dir /tmp/xarm_raw_data \
  --prompt "pick up the red block" \
  --gripper-dof Joint_1 \
  --gripper-command-mode position \
  --gripper-open-position 0.0 \
  --gripper-closed-position 0.5
```

![使用 Isaac Sim 自带 python.sh 启动数据采集脚本](/assets/posts/xarm-isaac-openpi/launch-command.png)

## 四个 Isaac / PhysX 基础概念

调机器人之前，先把几个概念分清会省很多时间。

### DOF

DOF 即 Degree of Freedom。机器人里的一个可动关节通常对应一个 DOF。例如：

```text
joint1 ~ joint6      xArm6 的六个关节
fl_wheel             左前轮旋转
pan_tilt_yaw_joint   云台 yaw
Joint_1              夹爪主动关节
```

控制一个 DOF，本质上就是给它下发位置或速度目标：

```python
dc.set_dof_position_target(handle, target_position)
dc.set_dof_velocity_target(handle, target_velocity)
```

### Articulation

Articulation 可以理解为一整棵由刚体和关节连起来的机器人动力学结构。当前机器人里，底盘、轮子、云台、xArm 和夹爪都在同一个或相关 articulation 结构里。

因此一个夹爪 mimic joint、root joint 或 drive 参数设置不合理，都可能影响整个机器人稳定性。

### Kinematic

Kinematic 刚体通常由外部直接设置位姿，不由物理力自然推动。排查夹爪抖动时可以临时试一下，但如果夹爪仍属于 articulation，强行改成 kinematic 可能破坏关节关系，所以不适合作为正常采集方案。

### USD 和 URDF

USD 是 Isaac Sim 真正用来渲染和做 PhysX 仿真的场景。机器人材质、刚体、碰撞、质量、关节 drive、相机和灯光都在 USD Stage 里。

URDF 更像运动学和机器人结构描述。RMPFlow 底层的 Lula 会读取 URDF，在自己的模型里计算 FK、Jacobian、关节限位和碰撞球。

如果 USD 和 URDF 的关节零位、root link、固定挂载链或末端 frame 不一致，就会出现两个“数学世界”：

```text
PhysX 认为末端在位置 A
Lula 认为末端在位置 B
```

这时 RMPFlow 即使内部已经收敛，画面里的机械臂也可能离 marker 很远。调试时可以计算：

```python
model_gap = np.linalg.norm(lula_ee_position - usd_ee_position)
```

这个误差应该接近 0，而不是靠放大容差掩盖。

## 坑 1：确认夹爪真正的 DOF

USD 里的 prim 名称和 articulation 里的 DOF 名称不一定一样。比如模型里有 `/World/Robot/gripper`，但真正能控制的 DOF 不叫 `gripper`。

打印 articulation DOF 后，可以看到：

```text
0: fl_steering_joint
1: fr_steering_joint
2: rl_steering_joint
3: rr_steering_joint
4: fl_wheel
5: fr_wheel
6: rl_wheel
7: rr_wheel
8: pan_tilt_yaw_joint
9: joint1
10: pan_tilt_pitch_joint
11: joint2
12: joint3
13: joint4
14: joint5
15: joint6
16: Joint_1
17: Joint_2
```

夹爪结构里：

```text
/World/Robot/gripper/joints/Joint_1 [PhysicsRevoluteJoint] AngularDrive
/World/Robot/gripper/joints/Joint_2 [PhysicsRevoluteJoint]
```

所以人工采集时真正应该控制的是：

```bash
--gripper-dof Joint_1
```

我最后采用的夹爪参数是：

```bash
--gripper-dof Joint_1 \
--gripper-command-mode position \
--gripper-open-position 0.0 \
--gripper-closed-position 0.5
```

对应关系是：

```text
gripper_target=0.0 -> Joint_1 targetPosition=0.0
gripper_target=1.0 -> Joint_1 targetPosition=0.5
```

如果换了另一个夹爪 USD，这两个位置不能照抄。应该先打印 DOF limits，再肉眼确认 open/closed 的方向和范围。

## 坑 2：避免键盘快捷键冲突

Isaac Sim GUI 自己占用了一些快捷键。早期我用 `O/P` 控制夹爪时，`P` 会触发 Isaac 的 parent prim 相关快捷键，然后报：

```text
Cannot parent prim as two or more prims are not selected
```

所以脚本里把夹爪控制改成：

```text
Z: 打开夹爪
X: 闭合夹爪
```

同时，键盘回调里如果脚本已经处理了某个键，就返回 `False`，阻止事件继续传给 Isaac 默认快捷键。

核心逻辑大概是：

```python
elif is_keyboard_input(key, "Z"):
    if args.gripper_command_mode == "velocity":
        gripper_velocity_target = 0.0 if is_release else -abs(args.gripper_velocity)
    elif not is_release:
        new_target = max(0.0, gripper_target - args.gripper_step)
        if new_target != gripper_target:
            gripper_target = new_target
            gripper_command_dirty = True
            print(f"夹爪目标: {gripper_target:.2f}")
    return False
```

## 坑 3：每帧下发 target 会让机器人抖

一个很容易踩的坑是：脚本每一帧都给很多 DOF 重复下发 target。

早期逻辑类似这样：

```python
for name, target in zip(XARM_JOINT_NAMES, joint_targets):
    set_dof_target(dc, joint_handles[name], target)

set_base_steering_straight(dc, base_steering_handles)
set_base_wheel_velocity(dc, base_wheel_handles, state.base_wheel_velocity)
```

这会让脚本和 PhysX drive 一直互相拉扯：

```text
PhysX 根据关节 drive、碰撞和约束推进一步
脚本立刻重新写 target
PhysX 下一步继续修正
脚本又写 target
```

表现就是机器人周期性抖动。

排查时可以先禁用控制：

```bash
--gripper-command-mode none \
--disable-arm-control \
--disable-base-control
```

如果这样不抖，就说明问题不是 USD 静态模型自己产生的，而是控制逻辑引起的。

解决方式是改成事件触发：只有按键真正改变目标时，才下发新的 target；平时不再每帧写 joint target 或 wheel target。

```python
if not args.disable_arm_control and state.arm_command_dirty:
    for name, target in zip(XARM_JOINT_NAMES, joint_targets, strict=True):
        set_dof_target(dc, joint_handles[name], float(target))
    state.arm_command_dirty = False

if has_mobile_base and state.base_command_dirty:
    apply_base_motion(
        dc,
        base_wheel_handles,
        base_steering_handles,
        state.base_wheel_velocity,
        state.base_turn_velocity,
    )
    state.base_command_dirty = False
```

## 坑 4：RMPFlow 不能只拿 position target

PhysX 的关节 drive 可以近似理解为一个 PD 控制器：

```text
torque = Kp * (position_target - position)
       + Kd * (velocity_target - velocity)
```

RMPFlow 输出的 `ArticulationAction` 同时包含 joint position 和 joint velocity。正确路径是：

```python
action = articulation_policy.get_next_articulation_action(physics_dt)
robot.apply_action(action)
```

如果中间层只取 position，再逐关节调用：

```python
dc.set_dof_position_target(handle, target)
```

velocity feedforward 就丢了。控制器会一边追移动的位置目标，一边把当前速度往 0 拉，表现通常是跟随很慢、误差降不下去或执行超时。

另一个问题是 USD 导入后的 drive 参数可能严重欠阻尼。例如 stiffness 很大、damping 却接近 0，机械臂每次追新目标都会过冲和回弹。当前脚本会提供运行时覆盖参数，但这些值只适用于当前资产，换机器人后仍需要重新验证。

## 从关节控制到 marker teleop

直接控制 6 个关节虽然简单，但不适合人工采集：

```text
不直观
容易自碰
容易接近奇异点
很难表达“末端往前一点、往上抬一点”
```

更自然的方式是在场景里放一个 target marker，通过键盘或 Isaac Move gizmo 移动 marker，再让机械臂末端用 IK 跟随 marker。

当前脚本的 marker 控制如下：

```text
↑/↓: marker 沿世界 X 方向移动
←/→: marker 沿世界 Y 方向移动
PageUp/PageDown: marker 沿世界 Z 方向移动
鼠标 Move gizmo: 直接拖动 marker
G/Enter: 手动触发跟随
H: 停止当前 marker 执行
Z/X: 夹爪开合
R: 开始/暂停记录
N: 保存 episode
C: 清空缓存
Esc: 退出
```

旧版本曾使用 `W/A/S/D/Q/E` 移动 marker。如果自己的脚本来自较早 commit，应以启动时终端打印的键盘说明为准。

第一版 marker IK 使用 Jacobian 差分 IK：

```python
lhs = (
    linear_jacobian @ linear_jacobian.T
    + np.eye(3, dtype=np.float32) * float(damping**2)
)
delta_q = linear_jacobian.T @ np.linalg.solve(
    lhs,
    error * float(gain),
)
delta_q = np.clip(
    delta_q,
    -abs(max_joint_step),
    abs(max_joint_step),
)
```

它表达的是：先计算末端位置误差，再通过 Jacobian 的阻尼伪逆得到一小步关节增量。

这不是完整的全局运动规划器。它不知道一条路径是否会碰撞，也可能在奇异点附近退化，但作为第一版采集入口已经够用：它能把“移动末端”的操作从六个关节目标里解放出来。

![marker teleop 记录过程中的终端日志](/assets/posts/xarm-isaac-openpi/teleop-recording-log.png)

## 给 marker 加工作空间限制

如果 marker 可以无限移动，机械臂会一直追一个不可达目标。

常见表现是：

```text
marker 跑得很远
机械臂一直动来动去
某些位置 marker 和末端不重合，但机械臂保持静止
```

原因可能是目标超出工作空间、Jacobian 局部退化、机械臂接近奇异点，或者关节增量太小。

所以 marker 模式需要三个保护：

1. 限制 marker 相对初始位置的最大半径。
2. 限制 marker 的 z 高度。
3. 检测 IK 是否卡住或长时间没有进展。

我常用的调试参数是：

```bash
--marker-step 0.01 \
--marker-max-radius 0.25 \
--ik-gain 0.35 \
--ik-max-joint-step 0.015 \
--ik-debug
```

卡住检测逻辑：

```python
if marker_delta_norm < args.ik_stuck_joint_step:
    state.marker_stuck_steps += 1
else:
    state.marker_stuck_steps = 0

if state.marker_stuck_steps >= args.ik_max_stuck_steps:
    print(
        f"marker IK 可能卡住或目标不可达，已停止跟随: "
        f"error={marker_error:.4f}m, dq={marker_delta_norm:.6f}"
    )
    state.marker_follow_active = False
    state.marker_stuck_steps = 0
```

工作空间点云只能回答“这个位置附近是否出现过可达样本”，不能证明目标姿态、整条路径和碰撞也可行。所以它应该作为便宜的预检查，而不是运动规划成功证明。

## 运行时绑定材质

仿真模型默认可能是全白的。云台相机或末端相机看机械臂时，白色模型容易和背景混在一起，也不像现实中的银色金属 xArm。

比较安全的做法是：不要直接修改二进制 USD 本体，而是在脚本启动时运行时绑定材质。

我当前的材质设置是：

```text
xArm: 银色金属
夹爪: 深灰石墨色
云台/相机: 黑色
底盘/其他结构: 深灰色
```

如果想保留 USD 原始外观，可以加：

```bash
--no-visual-materials
```

材质创建的核心逻辑如下：

```python
def create_preview_material(
    stage,
    path,
    color,
    *,
    metallic=0.0,
    roughness=0.35,
):
    material = UsdShade.Material.Define(stage, path)
    shader = UsdShade.Shader.Define(
        stage,
        f"{path}/PreviewSurface",
    )
    shader.CreateIdAttr("UsdPreviewSurface")
    shader.CreateInput(
        "diffuseColor",
        Sdf.ValueTypeNames.Color3f,
    ).Set(Gf.Vec3f(*color))
    shader.CreateInput(
        "metallic",
        Sdf.ValueTypeNames.Float,
    ).Set(float(metallic))
    shader.CreateInput(
        "roughness",
        Sdf.ValueTypeNames.Float,
    ).Set(float(roughness))
    material.CreateSurfaceOutput().ConnectToSource(
        shader.ConnectableAPI(),
        "surface",
    )
    return material
```

绑定时不要只匹配 `UsdGeom.Gprim`。有些 USD 里可见对象挂在 `Xform/visuals/mesh_*` 这类层级上，匹配 `UsdGeom.Imageable` 会更稳。

## 推荐采集命令

稳定采集可以先用：

```bash
cd "$OPENPI_ROOT"

"$ISAAC_SIM_ROOT/python.sh" \
  third_party/isaac_sim_AGX/ros2_car_model_ws/src/scripts/collect_xarm_episode.py \
  --mode marker \
  --output-dir /tmp/xarm_raw_data \
  --prompt "pick up the red block" \
  --gripper-dof Joint_1 \
  --gripper-command-mode position \
  --gripper-open-position 0.0 \
  --gripper-closed-position 0.5 \
  --marker-step 0.01 \
  --marker-max-radius 0.25 \
  --ik-gain 0.35 \
  --ik-max-joint-step 0.015
```

如果要调试 IK：

```bash
--ik-debug
```

如果要打印可显示 prim，用于调试材质匹配：

```bash
--print-visual-prims
```

保存后，输出目录里会出现：

```text
/tmp/xarm_raw_data/episode_0000.npz
```

![采集后生成的 episode_0000.npz](/assets/posts/xarm-isaac-openpi/episode-file.png)

## 保存后先检查，不要直接训练

可以先运行仓库里的检查脚本：

```bash
cd "$OPENPI_ROOT"

uv run examples/xarm/inspect_xarm_npz.py \
  /tmp/xarm_raw_data \
  --preview-dir /tmp/xarm_preview \
  --num-preview-frames 6
```

至少应该检查：

```text
wrist_image 是否是 [T,H,W,3] uint8
joints 是否是 [T,6]
gripper 是否是 [T,1]
actions 是否是 [T,7]
所有主通道的 T 是否一致
timestamp 是否严格递增
关节和夹爪是否存在 NaN / Inf
随机抽帧是否能看到正确的相机画面
prompt 是否与当前任务一致
```

如果某个通道少了一帧，不建议简单裁剪到最短长度。图像和动作可能已经错位，应该回到采集循环检查是哪一个 reader 或分支漏记了数据。

## 后续：自动生成 VLA episode

手动录制能产生质量比较高的数据，但不适合录 1000 条 episode。近距离 VLA 更适合用自动化生成器。

这里的自动化不是让 Isaac Sim 自己“理解任务并抓取”，而是写一个 scripted expert，也可以叫 oracle：

```text
reset scene
随机生成物体、颜色、位置、光照、材质
给物体加 semantic label
根据 prompt 找到目标 prim
读取目标真实 pose / bbox / normal
生成末端目标位姿
用 IK / RMPFlow / 轨迹模板执行
记录 image + joints + gripper + prompt + timestamp
验证抓取是否真正成功
保存 episode
```

semantic label 只负责告诉脚本：

```text
哪个 prim 是 cup
哪个 prim 是 red object
哪个 prim 是 table
哪个 prim 是 elevator_button_3
```

真正的动作仍然需要我们写专家策略。

以 `press_button` 为例：

```text
随机生成电梯面板和按钮
给按钮加 semantic label
随机 prompt: "按下 3 楼按钮"
oracle 找到 floor_3 button
生成 approach pose
生成 press pose
IK 跟踪 approach -> press -> retreat
验证按钮是否被按下
保存 episode
```

以 `pick_cup` 为例：

```text
随机水杯颜色、位置、朝向
prompt: "拿起红色水杯"
oracle 找到 red cup
生成预抓取 pose
接近
闭合夹爪
抬起
等待物体稳定
验证物体抬升高度和末端距离
保存 episode
```

正式数据里不应该默认使用把物体直接吸附到夹爪的 teleport/latch。它可以帮助调试轨迹，但会绕过真实的接触、摩擦和夹持力，让“抓取成功”失去意义。

## 小结

到这里，基础链路已经跑通：

```text
加载机器人 USD
确认 xArm、底盘、云台、夹爪 DOF
识别夹爪主动关节 Joint_1
避开 Isaac Sim 快捷键冲突
解决持续下发 target 导致的机器人抖动
理解 RMPFlow position + velocity 的下发方式
实现 marker teleop
加入 marker 工作空间限制和 IK 卡住检测
运行时绑定机器人材质
保存并检查 episode_0000.npz
```

对我来说，最关键的收获是：机器人数据采集不是从训练模型开始的，而是先把控制、记录、保存、检查和后续格式转换这一整条链路打通。

只有当图像、状态、动作、时间和机器人关节约定都一致时，保存出来的 episode 才真的有训练价值。
