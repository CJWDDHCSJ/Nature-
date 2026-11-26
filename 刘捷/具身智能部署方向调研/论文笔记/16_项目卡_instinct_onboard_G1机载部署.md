# 项目卡 16 | instinct_onboard：Project Instinct 的 G1 机载部署仓库

> 这不是论文，而是官方部署工程。它是目前与 G1-EDU + Ubuntu 22.04 + ROS 2 Humble 最接近的公开入口，但“仓库无权重、无 LICENSE、Python 运行时”是硬缺口。

---

## 1. 基本信息

- **仓库**：https://github.com/project-instinct/instinct_onboard
- **组织**：Project Instinct
- **性质**：机载推理/部署代码
- **机器人**：Unitree G1 29DoF（`G1_29Dof_TorsoBase`）
- **系统**：官方声明 Ubuntu 22.04 + ROS 2 Humble
- **推理**：Python + ONNX Runtime
- **控制**：50 Hz 主循环（`scripts/g1_track.py`）
- **模型**：`<logdir>/exported/actor.onnx` + `policy_normalizer.npz`
- **训练生态**：InstinctLab / instinct_rl

---

## 2. 仓库事实（已核验）

| 项目 | 状态 |
|---|---|
| 代码 | 有 |
| G1 脚本 | `g1_track.py`、`g1_parkour.py`、`g1_perceptive_track.py` |
| ROS 2 节点 | `ros_nodes/unitree.py`、`realsense.py` 等 |
| ONNX Runtime | 有 |
| 模型权重 | 文件树未发现 `.onnx`/`.pt`/`.safetensors` |
| LICENSE | 未发现 |
| Python 版本约束 | 未声明 |
| 依赖锁定 | 仅 `empy==3.3.2`，其余未锁 |
| 维护 | 有提交记录；仓库创建 2026-01-10 |
| 官方描述 | 无 |

---

## 3. 运行链路

```mermaid
flowchart TB
    JS[Unitree 手柄] --> NODE[G1TrackingNode]
    LS[/lowstate] --> NODE
    IMU[/secondary_imu] --> NODE
    NODE --> AG[TrackerAgent]
    AG --> ORT[ONNX Runtime actor]
    ORT --> TJS[TargetJointState]
    TJS --> SAFE[pack_policy_action / 限幅]
    SAFE --> CMD[/lowcmd]
    CMD --> G1[G1]
```

### 关键代码事实

- `G1TrackingNode(UnitreeNode)`，`robot_class_name="G1_29Dof_TorsoBase"`；
- 默认 `dryrun=True`，需 `--nodryrun` 才进真机；
- 主循环 20 ms（50 Hz）；
- 冷启动 Agent 逐步到初始姿态，按 L1 切换跟踪；
- 跟踪结束调用 `_turn_off_motors()`；
- `UnitreeNode` 订阅 `/lowstate`、`/secondary_imu`，发布 `/lowcmd`；
- 关节越限时打印错误、关电机、`SystemExit`；
- `crc_module.get_crc` 计算 CRC；
- 依赖 `unitree_hg.msg`（IMUState/LowCmd/LowState）。

---

## 4. 模型接口

| 项目 | 内容 |
|---|---|
| 模型 | `<logdir>/exported/actor.onnx` |
| 归一化 | `<logdir>/exported/policy_normalizer.npz` |
| 输入 | 归一化观测向量（维度由配置决定） |
| 输出 | 动作向量，reshape 后转目标关节状态 |
| 观测 | 关节位置/速度、IMU、参考运动、历史等 |
| 频率 | 50 Hz |
| Provider | `ort.get_available_providers()`（自动） |

### 缺口

- 仓库不提供 actor.onnx；
- 需要 InstinctLab/instinct_rl 训练并导出；
- 需要与 G1 关节顺序一致的配置；
- 需要确认 ONNX Runtime GPU 在 ARM64 可用；
- Python 实现的抖动/延迟需实测。

---

## 5. 安全机制（可借鉴）

1. 默认 dry-run，发布到随机后缀话题，不进电机；
2. 关节位置保护高低限；
3. NaN 检查；
4. CRC；
5. 冷启动渐变；
6. 结束时关电机；
7. 手柄切换 Agent。

### 不足

- 无独立硬件急停；
- Ctrl+C 路径未显式关电机（需底层/外部保护）；
- Python 主循环可能抖动；
- 无推理超时/状态超时看门狗（需核对 base）；
- 无 LICENSE，商用/再分发需谨慎。

---

## 6. 与现有 G1 环境的匹配度

| 维度 | 匹配度 | 说明 |
|---|---|---|
| Ubuntu 22.04 | 高 | 官方声明 |
| ROS 2 Humble | 高 | 官方声明 |
| G1 29DoF | 高 | 代码明确 |
| CycloneDDS | 中 | 需确认 RMW |
| `unitree_hg` | 中 | 需现有工作空间有对应包 |
| CRC 模块 | 中 | 需 `crc_module` |
| ONNX Runtime | 中 | 版本/GPU provider 需匹配 |
| 权重 | 低 | 仓库无 |
| License | 低 | 无 |
| C++17 | 低 | 官方是 Python |

**结论：需适配后复现，优先级最高，但先解决权重/依赖/License。**

---

## 7. 建议的拆解顺序

1. 只读状态：跑 `UnitreeNode` 的 dry-run，确认 `/lowstate`、IMU、CRC；
2. dummy ONNX：用自己的 MLP 替代 actor，验证 50 Hz 链路；
3. 关节映射：对比 `robot_cfgs` 与真机；
4. 安全层：加看门狗/超时/限幅；
5. 权重：用 InstinctLab 训练/导出，或找公开 G1 策略；
6. C++ 重写核心节点（保留 Python 仅作对照）；
7. 仿真/回放；
8. 真机低风险。

---

## 8. 结论

**需适配后复现。**  
它是学习 G1 机载部署的最佳公开入口，但不是完整可运行产品；不要把“仓库存在”当成“策略可复现”。
