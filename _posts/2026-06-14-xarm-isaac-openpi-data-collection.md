---
layout: post
title: "用 Isaac Sim 为 openpi 采集 xArm6 数据：从 DOF 调试到 marker teleop"
date: 2026-06-14
description: "在 Isaac Sim 5.1 中加载移动底盘 + xArm6 机器人，调通夹爪 DOF、marker teleop、episode 记录，并为后续 openpi / LeRobot 数据转换做准备。"
categories: [robotics, embodied-ai, data-collection]
---

这篇文章记录的是一次从仿真机器人加载到 episode 数据落盘的完整过程。目标不是“把 Isaac Sim 打开”，而是搭出一条后续能服务于 openpi / LeRobot 微调的数据采集链路。

TL;DR:

```text
Isaac Sim 5.1
  -> 加载移动底盘 + xArm6 + 夹爪 + 相机
  -> 用 marker teleop 控制末端
  -> 记录图像、关节、夹爪、动作和语言指令
  -> 保存为 episode_0000.npz
  -> 后续转换为 LeRobot / openpi 可用的数据
```

本文里的路径来自我的本地环境，读者需要替换成自己的 openpi、Isaac Sim 和资产仓库路径。

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

这些数据后面可以转换成 LeRobot 风格的数据集，再拿去微调 openpi 里的 pi0 / pi0-fast 类模型。更长远一点，这条链路也可以扩展到自动生成 `press_button`、`pick_cup`、`place_cup`、`pick_colored_object` 这类近距离操作任务。

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

目前保存的字段包括：

```text
wrist_image: [T, H, W, 3], uint8
joints: [T, 6]
gripper: [T, 1]
actions: [T, 7]
timestamp: [T]
prompt: str
target_marker_position: [T, 3]
base_wheel_velocity: [T]
fps: scalar / metadata
```

其中 `actions` 采用 7 维：

```text
6 个 xArm 关节目标 + 1 个夹爪目标
```

也就是说，即使用 marker、IK 或 scripted expert 来控制机械臂，最终记录进数据集里的动作仍然保持为关节空间目标。这对后续和 openpi 的 xArm policy 对接会更直接。

![episode_0000.npz 内部保存的字段](/assets/posts/xarm-isaac-openpi/episode-fields.png)

## 子模块提交顺序

Isaac Sim 资产仓库是 openpi 主仓库里的 git submodule：

```text
third_party/isaac_sim_AGX
```

这意味着 openpi 主仓库不会直接记录子模块里的每个文件，只会记录子模块指向哪个 commit。

如果主仓库里看到类似这样的 diff：

```diff
diff --git a/third_party/isaac_sim_AGX b/third_party/isaac_sim_AGX
index 19c0e5e..dee99b7 160000
--- a/third_party/isaac_sim_AGX
+++ b/third_party/isaac_sim_AGX
@@ -1 +1 @@
-Subproject commit 19c0e5ee997c609717367e57e0c99b903077b669
+Subproject commit dee99b764ace0aeb69dc80cbea043a24cc3e944f-dirty
```

意思是：主仓库原来记录的子模块 commit 是 `19c0e5e`，现在本地切到了 `dee99b7`，并且子模块内部还有未提交修改。

正确顺序是先提交子模块，再提交主仓库里的子模块指针：

```bash
cd third_party/isaac_sim_AGX
git add ros2_car_model_ws/src/scripts/collect_xarm_episode.py
git commit -m "Update Isaac xArm episode collection controls"
git push origin master

cd /Data/xiaodx/openpi
git add third_party/isaac_sim_AGX
git commit -m "Update Isaac Sim AGX submodule"
```

## 启动脚本

不要用 openpi 的 Python 环境直接跑 Isaac Sim 脚本。Isaac Sim 有自己的 Python、扩展系统和动态库加载方式，应该使用它自带的 `python.sh`：

```bash
/Data/xiaodx/isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
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

## 三个 Isaac / PhysX 基础概念

调机器人之前，先把三个概念分清会省很多时间。

DOF，即 Degree of Freedom。机器人里的一个可动关节通常对应一个 DOF。例如：

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

Articulation 可以理解为一整棵由刚体和关节连起来的机器人动力学结构。当前机器人里，底盘、轮子、云台、xArm 和夹爪都在同一个或相关 articulation 结构里。因此一个夹爪 mimic joint、root joint 或 drive 参数设置不合理，都可能影响整个机器人稳定性。

Kinematic 刚体通常由外部直接设置位姿，不由物理力自然推动。排查夹爪抖动时可以临时试一下，但如果夹爪仍属于 articulation，强行改成 kinematic 可能破坏关节关系，所以不适合作为正常采集方案。

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

所以真正应该控制的是：

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
    set_base_steering_straight(dc, base_steering_handles)
    set_base_wheel_velocity(dc, base_wheel_handles, state.base_wheel_velocity)
    state.base_command_dirty = False
```

## 从关节控制到 marker teleop

直接控制 6 个关节虽然简单，但不适合人工采集：

```text
不直观
容易自碰
容易接近奇异点
很难表达“末端往前一点、往上抬一点”
```

更自然的方式是在场景里放一个 target marker，通过键盘移动 marker，再让机械臂末端用 IK 跟随 marker。

键盘控制如下：

```text
W/S: marker 前后
A/D: marker 左右
Q/E: marker 上下
Z/X: 夹爪开合
R: 开始/暂停记录
N: 保存 episode
C: 清空缓存
Esc: 退出
```

第一版 marker IK 使用 Jacobian 差分 IK，只控制末端位置，不控制姿态：

```python
lhs = linear_jacobian @ linear_jacobian.T + np.eye(3, dtype=np.float32) * float(damping**2)
delta_q = linear_jacobian.T @ np.linalg.solve(lhs, error * float(gain))
delta_q = np.clip(delta_q, -abs(max_joint_step), abs(max_joint_step))
```

这不是最完整的机械臂控制器，但作为第一版采集入口已经够用：它能把“移动末端”的操作从六个关节目标里解放出来。

![marker teleop 记录过程中的终端日志](/assets/posts/xarm-isaac-openpi/teleop-recording-log.png)

图里能看到脚本打印的键盘说明、记录帧数、marker target，以及最终保存到 `/tmp/xarm_raw_data/episode_0000.npz`。

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
3. 检测 IK 是否卡住。

我常用的调试参数是：

```bash
--marker-step 0.01
--marker-max-radius 0.25
--ik-gain 0.35
--ik-max-joint-step 0.015
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

如果启动时看到：

```text
已绑定机器人调试材质: xArm silver=21, gripper=15, camera/pan_tilt=19, base=70
```

说明材质绑定成功。

如果想保留 USD 原始外观，可以加：

```bash
--no-visual-materials
```

材质创建的核心逻辑如下：

```python
def create_preview_material(stage, path, color, *, metallic=0.0, roughness=0.35):
    material = UsdShade.Material.Define(stage, path)
    shader = UsdShade.Shader.Define(stage, f"{path}/PreviewSurface")
    shader.CreateIdAttr("UsdPreviewSurface")
    shader.CreateInput("diffuseColor", Sdf.ValueTypeNames.Color3f).Set(Gf.Vec3f(*color))
    shader.CreateInput("metallic", Sdf.ValueTypeNames.Float).Set(float(metallic))
    shader.CreateInput("roughness", Sdf.ValueTypeNames.Float).Set(float(roughness))
    material.CreateSurfaceOutput().ConnectToSource(shader.ConnectableAPI(), "surface")
    return material
```

绑定时不要只匹配 `UsdGeom.Gprim`。有些 USD 里可见对象挂在 `Xform/visuals/mesh_*` 这类层级上，匹配 `UsdGeom.Imageable` 会更稳。

## 推荐采集命令

稳定采集可以先用：

```bash
/Data/xiaodx/isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
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
记录 image + joints + action + prompt + timestamp
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

计划中的近距离 VLA 任务包括：

```text
press_button
pick_cup
place_cup
pick_colored_object
put_object_on_target
```

以 `press_button` 为例：

```text
随机生成电梯面板和按钮
给按钮加 semantic label
随机 prompt: "按下 3 楼按钮"
oracle 找到 floor_3 button
生成 approach pose
生成 press pose
IK 跟踪 approach -> press -> retreat
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
保存 episode
```

## 小结

到这里，基础链路已经跑通：

```text
加载 1557_all.usd
确认 xArm、底盘、云台、夹爪 DOF
识别夹爪主动关节 Joint_1
避开 Isaac Sim 快捷键冲突
解决持续下发 target 导致的机器人抖动
实现 marker teleop
加入 marker 工作空间限制和 IK 卡住检测
运行时绑定机器人材质
保存 openpi / LeRobot 后续可用的 episode 数据
```

对我来说，最关键的收获是：机器人数据采集不是从训练模型开始的，而是先把控制、记录、保存、检查和后续格式转换这一整条链路打通。

接下来主要有两条线：

1. 继续增强 teleop：加入姿态控制、拖拽交互、轨迹平滑，以及更稳定的 IK / RMPFlow。
2. 做近距离 VLA 自动采集：实现 semantic scene generator 和 scripted expert，自动生成 `press_button`、`pick_cup`、`place_cup`、`pick_colored_object`、`put_object_on_target` episode。
