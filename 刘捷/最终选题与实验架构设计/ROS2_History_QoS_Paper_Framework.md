# ROS 2 History QoS 过载、内存滞留与恢复行为研究

## 选题与实验框架 V2.1（审核修订版）

- **版本：** V2.1
- **整理日期：** 2026-11-08
- **状态：** 研究方向与题目冻结；实验方案作为下一阶段执行基线，具体参数须经 G1 P0/P1 实测后冻结。
- **研究类型：** Measurement and Characterization（测量与特征化研究）
- **目标：** 以第一篇 SCI 四区论文为投稿目标。目标不是录用保证；JCR Q4 与中科院四区是不同分区体系，投稿时须按当年目标期刊和分区核验。

> 本文区分三种信息：
> - **已知/源码依据：** 有论文、官方文档或对应版本源码支持。
> - **设计选择：** 当前研究方案所采用的实验设计。
> - **待验证：** 必须通过 G1 实际运行时审计或实验确认，不能提前写成结果。

---

# 1. 最终选题

## 1.1 中文题目

**基于 ROS 2 rclcpp/CycloneDDS 的传感器管线在不同 History QoS 策略下的过载积压、内存滞留与恢复行为实证研究**

## 1.2 英文题目

*An Empirical Study of Overload Backlog, Memory Retention, and Recovery under Different History QoS Policies in ROS 2 rclcpp/CycloneDDS Sensor Pipelines*

## 1.3 题目状态

题目暂时冻结。P2 试点后可以评估标题中的 `Memory Retention` 是否与结果匹配，但不得因为某一轮实验未观察到明显滞留，就提前更换研究方向。

## 1.4 研究边界

研究对象限定为：

- ROS 2 Humble；
- `rclcpp`；
- `rmw_cyclonedds_cpp`；
- 实际审计确认的 CycloneDDS 版本及配置；
- Unitree G1-EDU 搭载的 Jetson Orin NX 16GB；
- 同机、独立进程 Publisher/Subscriber 传感器数据管线。

结论首先适用于实际测试的软件版本、配置、硬件和通信路径，不直接泛化为所有 ROS 2 版本、DDS 实现或机器人平台。

---

# 2. 研究背景与核心问题

机器人传感器数据通常以持续速率输入下游处理节点。当 Publisher 的实际输入速率高于 Subscriber 的实际处理能力时，系统进入过载状态：

$$
\lambda > \mu
$$

其中：

- $\lambda$：Publisher 的实际输入速率，必须测量，不使用配置目标值代替；
- $\mu$：Subscriber callback 的实际完成速率，必须测量；
- $\lambda_r$：Recovery 阶段 Publisher 的实际输入速率。

ROS 2 History QoS 影响中间件历史样本的保留策略。不同策略可能影响：

1. 过载期间消息的保留、覆盖或拒绝；
2. callback 收到的数据年龄；
3. Writer 侧背压和 `publish()` 调用行为；
4. DDS、allocator、进程和内核网络层的内存占用；
5. 过载结束后的队列/数据年龄恢复与内存恢复；
6. 同一进程多轮过载后是否出现超出正常运行基线的内存漂移。

本研究不预先断言 KEEP_ALL 一定造成内存泄漏，也不预先断言 KEEP_LAST 一定形成稳定内存平台。所有结论必须由实测支持。

## 2.1 内存分层概念图

```text
                 ROS 2 数据管线
                       |
          +------------+------------+
          |                         |
     Publisher                  Subscriber
          |                         |
       rclcpp                     rclcpp
          |                         |
       RMW / CycloneDDS
          |
   +------+------------------------------+
   |                                      |
Writer History Cache (WHC)      Reader History Cache (RHC)
   |                                      |
   +-------------- DDS 内部路径 ----------+
                       |
                用户态分配器
                 glibc malloc
                       |
              Linux 进程虚拟内存
                       |
                RSS / PSS / maps

             Linux 内核网络层
                       |
               UDP Socket Buffer
                       |
           cgroup memory.stat: sock
```

这是概念上的观测层次，不表示每个字节都能被精确地一一归属。特别是进程 RSS、glibc 统计和 cgroup 统计口径不同，不能简单相减得到某个组件的精确占用。

---

# 3. Research Questions（RQ）

## RQ1：不同 History QoS 对持续过载期间的消息积压和内存行为有什么影响？

观察不同 Subscriber History 策略下：

- callback 完成速率；
- callback 序列号及跳号模式；
- Publish-to-Callback Age 分布；
- Publisher `publish()` 调用耗时、阻塞行为和实际调用速率；
- Publisher / Subscriber 的 RSS、PSS；
- cgroup 内存分类；
- 内存增长轨迹、峰值和增长斜率。

## RQ2：过载结束后，消息新鲜度恢复与进程内存恢复是否存在时间尺度差异？

区分以下量：

- $T_{drain}$：基于可观测序列号/回调事件定义的队列排空代理时间；
- $T_{age}$：Publish-to-Callback Age 恢复到预先定义基线新鲜度范围的时间；
- $T_{mem50}$：进程 RSS 释放峰值新增内存 50% 所需时间；
- $T_{mem90}$：进程 RSS 释放峰值新增内存 90% 所需时间。

研究重点是实测恢复曲线与近似排队模型的差异，以及消息新鲜度恢复和内存恢复之间是否存在可重复的偏离。不预先假定 $T_{mem90} \gg T_{age}$，也不预先假定二者接近。

## RQ3：重复过载是否导致超出无过载运行基线的内存漂移？

在同一进程生命周期内重复执行 Overload/Recovery，观察每轮恢复后的 RSS/PSS、allocator 状态和 cgroup 变化；同时设置同样时长、同样轮数、始终保持低负载的 Soak Control，区分过载额外影响与单纯运行时间/监控开销。

统计单位是**独立进程运行**。同一个进程内部的多轮数据是重复测量，不得把每一轮当成独立样本。

---

# 4. 预期贡献（待数据验证，不是既定结论）

## C1：History QoS 过载行为的实证特征化

在实际 ARM64 机器人计算平台上，测量不同 Subscriber History 策略在持续过载期间的消息行为、发布端背压和多层内存变化。

## C2：消息恢复与内存恢复的时间尺度分析

区分 $T_{drain}$、$T_{age}$、$T_{mem50}$、$T_{mem90}$，比较实测恢复行为与简化流体排队模型的预测偏差。

## C3：重复过载相对于无过载运行的内存基线变化

通过独立进程重复运行和匹配的低负载 Soak Control，分析是否存在额外基线漂移、线性增长或首轮后平台化等现象。

## C4：多层次内存归因方法

结合 `/proc`、cgroup v2、glibc allocator 统计、Socket 统计和独立干预实验，判断观察到的内存变化可能来自哪些层次。不得把某一种 allocator 指标直接等同于 DDS 活跃对象内存。

---

# 5. 实验平台与软件环境

## 5.1 G1-EDU 目标平台

| 项目 | 当前已知配置 |
|---|---|
| 机器人 | Unitree G1-EDU |
| SoC | NVIDIA Jetson Orin NX |
| CPU | 8× Cortex-A78AE |
| RAM | 16GB LPDDR5 统一内存系统 |
| GPU | NVIDIA Ampere 架构 |
| 架构 | ARM64 / aarch64 |
| Ubuntu | 22.04.5 LTS |
| ROS 2 | Humble |
| JetPack | 6.2.3 |
| Linux Kernel | 5.15.185-tegra |
| CUDA | 12.6 |
| TensorRT | 10.3 |
| RMW | 目标为 `rmw_cyclonedds_cpp`，实际加载情况需 P0 确认 |
| CycloneDDS | 实际版本需 P0 确认 |
| glibc | 实际版本需 P0 确认 |

以上是当前平台基线。凡是会影响实验的版本、动态库、环境变量、XML 配置、时钟和传输路径，都必须重新审计并写入实验报告。

## 5.2 RMW 启动约束

实验进程显式设置：

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

但不能仅凭环境变量断言实际加载了 CycloneDDS。必须通过 `/proc/<PID>/maps`、`ldd`/动态链接信息、包版本和进程环境等核实实际加载库及路径。

ROS 2 Humble 的常规二进制默认 RMW 通常为 Fast DDS；本研究选择 CycloneDDS 的理由是 G1 的实际部署路径与研究范围，而不是声称 CycloneDDS 是 Humble 的默认实现。

## 5.3 开发与下载约束

- 主要机器人节点使用 C++17 / rclcpp。
- Python 用于离线分析、数据处理和辅助脚本。
- 依赖和工具包优先在 Ubuntu 开发笔记本获取，再通过 SSH/SCP 传到 G1；避免实验平台临时联网下载。
- 不默认启用 `jetson_clocks` 或修改 `nvpmodel`。先记录出厂/当前运行状态、CPU 频率、温度和负载；是否固定时钟应作为独立且有记录的实验条件。

---

# 6. 核心实验架构

## 6.1 进程布局

```text
                       G1 Orin NX
                            |
                     ROS 2 Humble
                            |
                  rmw_cyclonedds_cpp
                            |
                       CycloneDDS
                            |
             +--------------+--------------+
             |                             |
      Publisher Process             Subscriber Process
             |                             |
         DataWriter                    DataReader
             |                             |
      publish() loop              SingleThreadedExecutor
                                           |
                                      Subscription
                                           |
                                       Callback

             +-----------------------------+
             |
        External Monitor / Watchdog
             |
      +------+------+----------+
      |             |          |
    /proc        cgroup v2  tegrastats / ss
      |             |          |
      +-------------+----------+
                    |
               CSV / JSONL
                    |
              Python analysis
```

## 6.2 架构约束

1. Publisher 与 Subscriber 必须是两个独立 C++17 进程。
2. Subscriber 使用 `SingleThreadedExecutor`，主实验排除 intra-process communication。
3. 主实验确保走 rclcpp/RMW/DDS 实际路径；具体同机传输是 loopback、实际网卡、共享内存还是混合路径，必须 P0 抓包/运行时审计确认。
4. Publisher 与 Subscriber 使用统一消息类型和固定处理逻辑，实验组之间只改变预定自变量。
5. 独立 `ROS_DOMAIN_ID` 只隔离 ROS 2 发现域和话题，不隔离 CPU、RAM、网络、温度等物理资源，也不保证不会与 Unitree 服务争用资源。
6. 不运行额外 CUDA/TensorRT 压力负载。
7. 外部 Watchdog 与数据保存程序必须位于实验 cgroup 外部。

## 6.3 Benchmark Message

候选消息结构：

```text
uint64 sequence_id
int64  publish_monotonic_ns
uint8[] payload
```

记录 Publisher 调用开始时间、调用返回时间、Subscriber callback 入口时间和完成时间。实际序列化后消息大小必须记录，不能只按 payload 长度推断 DDS 实际传输字节数或 malloc 请求大小。

---

# 7. 实验变量与主实验矩阵

## 7.1 主实验：Subscriber History QoS

Publisher QoS 固定：

```text
Reliability = RELIABLE
History     = KEEP_LAST
Depth       = 10
Durability  = VOLATILE
```

Subscriber 侧主矩阵：

| 组别 | Subscriber History | Depth | Reliability |
|---|---|---:|---|
| A | KEEP_LAST | 1 | RELIABLE |
| B | KEEP_LAST | 10 | RELIABLE |
| C | KEEP_LAST | 100 | RELIABLE |
| D | KEEP_ALL | 不适用 | RELIABLE |

主实验只改变 Subscriber History。Publisher 的 QoS 和其他变量保持不变。

## 7.2 KEEP_ALL 解释边界

- `DDS_HISTORY_KEEP_ALL` 与 `DDS_LENGTH_UNLIMITED` 只说明 History policy 的配置语义，不等于整个系统物理内存无限。
- 必须审计实际 Reader `ResourceLimits`、CycloneDDS 内部队列/分片限制、Writer WHC 和 publish 背压。
- CycloneDDS 0.10.2 源码的默认 Reader ResourceLimits 为三个 `UNLIMITED` 项；这是版本源码参考，不能直接当作 G1 生效值。
- 如果标准 rclcpp 路径无法设置或暴露某个 ResourceLimits 变体，不绕过 rclcpp 改用原生 DDS 以免改变研究对象；将其作为局限或仅在受支持配置下增加补充对照。

## 7.3 消息尺寸

初始候选：4 KiB、16 KiB、64 KiB。P2 试点后确定一个主实验尺寸，避免三个尺寸与全部 QoS 组合形成过大的交叉矩阵。

敏感性实验另外增加一个大于 128 KiB 的尺寸，例如 256 KiB 或 1 MiB，仅测试代表性 QoS 条件。

注意：glibc mmap 阈值是动态机制，默认初始阈值并非所有请求路径的硬分界；DDS 消息大小也不等于单次 malloc 请求大小。大消息尺寸实验用于检验分配路径/缓冲机制敏感性，不得预先断言必然走 mmap。

## 7.4 负载阶段

| 阶段 | 输入速率 | 目的 |
|---|---|---|
| Baseline | 目标约为实际 μ/2 | 建立 Age、内存与吞吐基线 |
| Overload | 实测 λ > 实测 μ | 制造可验证过载 |
| Recovery | 降至目标约 μ/2，实际速率重新测量 | 观察持续输入下的恢复 |
| Silent Recovery | 队列排空代理事件后停止 Publisher 输入 | 观察无新消息复用时的内存恢复 |
| Repeated Overload | 同一进程内重复上述循环 | 研究基线变化 |

`sleep_for(40ms)` 只是设定服务等待时间，不代表实际 μ=25Hz。μ 必须根据 callback 完成数除以实测时间计算。

---

# 8. 速率、背压与工况有效性闸门

## 8.1 必须记录的 Publisher 指标

每条消息至少记录：

- `t_publish_start`：调用 `publish()` 前的 `CLOCK_MONOTONIC` 时间戳；
- `t_publish_return`：`publish()` 返回后的本地记录时间戳；
- `publish_duration = t_publish_return - t_publish_start`；
- `sequence_id`；
- 发布循环计划截止时间、实际开始时间及 missed-deadline 计数。

发布循环优先使用绝对时间调度，例如 `clock_nanosleep(..., TIMER_ABSTIME, ...)`，避免因 publish 阻塞而把相对 `sleep_for` 的累积漂移误当作配置速率。

## 8.2 速率定义

- 计划发布速率：调度目标，仅作配置记录。
- $\lambda_{call}$：实际 `publish()` 调用数 / 时间。
- `publish()` 耗时分布：用于识别写端阻塞和背压。
- callback rate：Subscriber 实际 callback 完成数 / 时间。
- $\mu$：callback 完成速率。
- $\lambda_{rx}$：DDS Reader 实际接收速率，如无可靠的 middleware 观测手段则标记为不可直接测量，不得用 callback rate 代替。

## 8.3 P1 工况有效性闸门

P1 未通过之前不得进入正式 P2：

1. 证明指定负载下实际 $\lambda_{call}>\mu$。
2. 记录 `publish()` 耗时分布、阻塞时间比例和 missed deadlines。
3. 检查系统 UDP 接收/发送错误、drop 计数和 `ss -m` 数据。
4. 对 Publisher 序列号、callback 序列号、覆盖/丢失、未处理/残留样本进行尽可能完整的对账；不能观测到的类别必须明确列为未知，不做伪精确守恒式。
5. 检查 Subscriber 确实出现目标过载表现，而不是 Publisher 被 WHC/网络限制到低于 μ。
6. 空 callback 与固定负载 callback 分别测量 μ₀、μ。
7. 测量监控与记录开销。

若实际 λ_call 因背压下降到不再满足过载条件，该轮不能标记为有效过载实验；应调整负载设计并重新测量，而不是继续按计划速率解释。

---

# 9. 消息年龄与恢复模型

## 9.1 时间戳与 Age 口径

Publisher 与 Subscriber 同机，使用同一系统的：

```cpp
clock_gettime(CLOCK_MONOTONIC, ...)
```

记录：

- `t_publish_start`：调用 publish 之前；
- `t_publish_return`：publish 返回之后，在 Publisher 本地记录；
- `t_callback_entry`：Subscriber callback 入口立即读取；
- `t_callback_end`：callback 完成时读取。

主指标定义：

$$
Age_{call\rightarrow callback}=t_{callback\_entry}-t_{publish\_start}
$$

该指标包含 Publisher 调用阶段可能发生的等待/背压和后续中间件交付时间，因此应同时报告 `publish_duration`。不得简单把 `t_publish_return` 当作所有样本的 Age 起点，因为 Subscriber callback 可能在 Publisher 的 `publish()` 尚未返回时已经开始执行。

另外可以记录 callback 相对于 `publish()` 返回事件的先后关系；仅对确实晚于返回的样本报告 `t_callback_entry - t_publish_return`，对 callback 先于返回的情况单独标注，不构造负值后再当作常规 Age。

该指标是 **Publish-call-start-to-Callback Age**，不是传感器采集到机器人输出的完整端到端 Data Age。

## 9.2 T_age 判据

P1 用稳定低负载数据建立 Age 基线分布和接受范围。不得沿用固定 10 秒窗口 × 3 窗口的粗判据，因为可能无法分辨较短的恢复时间。

建议以短窗口/连续样本规则为基础：

1. 通过 P1 确定基线接受边界、连续合格样本数 K 和窗口长度 W；
2. K/W 应保证在实际 callback 频率下有可说明的时间分辨率和足够样本量；
3. 正式实验前冻结阈值；
4. 保留逐消息 Age 原始时间序列，T_age 只是摘要指标；
5. 报告判据的时间分辨率和样本数，避免将离散估计包装成高精度连续时间。

## 9.3 队列排空代理指标

rclcpp Wait Set 不直接暴露完整 DDS Reader History 长度。序列号跳号和最终处理序列号只能作为排空代理指标，不能直接当作内部队列长度。

RELIABLE 主实验可在停止 Publisher 后，根据最终有效发布序列号和 Subscriber 实际处理到的序列号，定义排空代理事件。该事件必须考虑在途消息、Publisher 写端阻塞和 callback 时序。

BEST_EFFORT 补充实验中，最后一个数据序列号可能丢失，需要使用单独控制标记或明确说明无法确认最终样本是否到达。

## 9.4 流体排队模型

建立一阶近似模型：

$$
\hat T_{drain}=\frac{Q_0}{\mu-\lambda_r},\quad \mu>\lambda_r
$$

其中 $Q_0$ 是 Recovery 开始时的有效积压量。若 $\lambda_r\approx\mu/2$，则：

$$
\hat T_{drain}\approx\frac{2Q_0}{\mu}
$$

重要限制：

- $Q_0$ 不能简单等于 History Depth；
- KEEP_LAST 可能覆盖旧样本，实际积压/样本年龄结构不等同于 FIFO 无损队列；
- Writer WHC、ResourceLimits、分片/重组、网络路径和 callback 调度均可能偏离理想流体模型；
- 若无法直接测量 $Q_0$，必须把估计方式和不确定性写清楚。

模型用于给出基线预测，研究实测 T_drain/T_age 与预测之间的偏差，而不是把理想公式当作实际结果。

---

# 10. 内存指标与 T_mem 定义

## 10.1 进程级指标

Publisher 和 Subscriber 分别采集：

| 指标 | 来源 |
|---|---|
| VmSize / VSZ | `/proc/PID/status` |
| VmRSS / RSS | `/proc/PID/status` |
| RssAnon、RssFile、RssShmem | `/proc/PID/status`（以目标内核实际字段为准） |
| PSS、Private/Shared | `/proc/PID/smaps_rollup`、`smaps` |
| VmHWM | `/proc/PID/status` |
| major/minor faults | `/proc/PID/stat` 或对应 proc 统计 |
| 映射区域 | `/proc/PID/maps`、阶段边界 `/proc/PID/smaps` |

建议采样：

- `/proc/PID/status`：初始 100ms；
- cgroup `memory.current`：初始 100ms；
- `smaps_rollup`：初始 1s；
- `tegrastats`：初始 1s。

最终采样频率必须通过监控开销对照确认。`malloc_info()` 不高频调用。

## 10.2 cgroup v2

记录：

```text
memory.current
memory.peak
memory.stat
memory.events
memory.max
memory.swap.current
memory.swap.max
memory.oom.group
```

重点分析 `anon`、`file`、`sock` 和可用的 kernel 相关字段。不同内核版本提供的统计键可能不同，以 G1 实际输出为准。

不能使用 `memory.current - RSS` 作为 Socket Memory。cgroup 与进程 RSS 口径不同，差值包含多种类别。

## 10.3 Jetson 整机指标

使用 `tegrastats` 记录 RAM、SWAP、CPU/GPU 负载与频率、温度。整机 RAM 不是单个实验进程的 RSS。

## 10.4 T_mem 定义

以 Subscriber RSS 为主要 T_mem 计算对象；PSS、Publisher RSS 和 cgroup 指标独立分析。

过载前稳定阶段：

$$
M_0=\operatorname{median}(RSS_{baseline})
$$

峰值：

$$
M_{peak}=\max(RSS_{overload})
$$

新增内存：

$$
\Delta M=M_{peak}-M_0
$$

释放比例：

$$
F(t)=1-\frac{M(t)-M_0}{\Delta M}
$$

- `T_mem50`：F(t) 首次达到 0.5 且连续维持预先冻结的稳定时间；
- `T_mem90`：F(t) 首次达到 0.9 且连续维持预先冻结的稳定时间；
- P1 根据基线 RSS 波动与监控噪声确定 `ΔM_min`；
- 若 $\Delta M<\Delta M_{min}$，该轮 T_mem 标记为不可判定；
- 超过预设观测窗口仍未恢复，记录右删失，例如 `T_mem90 > 600 s`（600 秒仅为初始候选，须 P1/P2 后冻结）；
- RSS 低于 M0 时 F(t) 可以大于 1，不截断；
- 持续 μ/2 Recovery 与 Silent Recovery 分别报告，不能混为同一自然恢复条件。

---

# 11. Memory Attribution：正式内存归因方法

## 11.1 归因目标与限制

目标是区分观察到的内存变化更可能来自：

1. 应用/ROS 2/DDS 仍持有的用户态对象；
2. glibc allocator 中可识别的空闲块或 arena 保留；
3. 直接 mmap、线程栈、共享库和其他非普通 heap 映射；
4. 内核 Socket/网络缓冲区；
5. 其他 cgroup/内核内存类别。

本研究采用多证据联合分析，不将任何单一计数器直接解释为 DDS 对象的精确字节数。

## 11.2 glibc 观测工具

### `mallinfo2()`

低频采集 glibc 统计字段，例如 arena、uordblks、fordblks、hblkhd 等。必须记录 G1 实际 glibc 版本，并核对对应版本源码和 man-pages。

**限制：** `uordblks` 不能严格等同于仍被应用/DDS 对象引用的内存；`fordblks` 也不包含所有可能已释放但仍保留在每线程 tcache 等结构中的块。不得用 `uordblks + fordblks` 或其与 RSS 的差值精确拆分 DDS 活跃内存和 allocator 保留内存。

### `malloc_info()`

在阶段边界输出 XML，观察多个 arena、空闲 bins 和 allocator 的系统内存规模。调用会获取 arena 锁，并且输出路径本身可能引入监控开销；不得高频调用。应使用监控开关对照量化开销。

### glibc tcache 重要限制

`free()` 后，小块内存可能先进入线程 tcache，不一定立即进入 arena 的普通空闲 bins。因此 mallinfo2/malloc_info 的字段不是完整的对象引用/释放账本。allocator 统计只能作为归因证据之一。

## 11.3 内核与映射辅助证据

- 阶段边界读取完整 `/proc/PID/smaps`，查看 `[heap]`、匿名映射、线程栈和库映射；
- 记录 `RssAnon`、`RssFile`、`VmHWM`、major/minor faults；
- cgroup `memory.stat` 记录 `anon`、`file`、`sock` 等；
- `ss -m` 观察 Socket skmem、队列及 drops；
- 若有必要，增加低开销分配追踪，但先评估其扰动，不能直接把高开销 profiler 用于主实验。

## 11.4 独立 allocator 干预

主实验自然恢复和静默恢复结束后，执行独立诊断：

| 条件 | 目的 | 优先级 |
|---|---|---|
| 原始 glibc | 自然运行基准 | 必须 |
| `malloc_trim(0)` | 尝试向系统释放符合条件的空闲堆页 | 必须 |
| `MALLOC_ARENA_MAX` | 改变 arena 数量，检查竞争和保留行为变化 | 必须 |
| `MALLOC_MMAP_THRESHOLD_` | 在对应 glibc 支持下改变 mmap/动态阈值行为，作为定向诊断 | 针对性实验 |
| jemalloc / mimalloc | 替换 allocator 进行对照 | 后续扩展 |

约束：

- `malloc_trim(0)` 是主动干预，不能混入自然恢复阶段；优先在独立运行或最后一个实验周期后执行。
- `MALLOC_ARENA_MAX`、`MALLOC_MMAP_THRESHOLD_` 必须在进程启动前设置。
- arena 限制可能改变锁竞争和实际 μ；每一种 allocator/环境条件均须重测 μ₀、μ、λ_call、publish 耗时和过载有效性。
- 替换 allocator 必须确认动态库实际加载与分配路径，不能只凭设置了 `LD_PRELOAD` 就宣称生效。
- 即使 trim 后 RSS 下降，也只能说明一部分驻留页面可由 allocator 干预释放，不能证明剩余 RSS 全由 DDS 对象持有。

## 11.5 监控扰动控制

Subscriber 使用预分配定长环形缓冲区记录必要事件；callback 运行期间不逐条打印日志、不动态扩展容器、不频繁写盘。阶段边界或运行结束后再落盘。

设置监控/记录开关对照，量化：

- 记录缓冲区开销；
- mallinfo2 采样线程开销；
- malloc_info 快照开销；
- 文件 I/O 和日志开销。

---

# 12. CycloneDDS、WHC、RHC、内部队列与网络审计

## 12.1 RMW 与实际运行版本

P0 必须确认：

- `RMW_IMPLEMENTATION`；
- 实际加载的 `librmw_cyclonedds_cpp` 和 `libddsc` 路径；
- 包版本、ELF/动态库版本和构建来源；
- `CYCLONEDDS_URI`；
- 实际 XML/配置文件；
- 实际网络接口和传输路径。

`/proc/PID/maps` 只能提供映射路径，不能单独证明库版本；版本还需结合 dpkg、包元数据、ELF/库信息等确认。

## 12.2 History 与 ResourceLimits

必须区分：

- DataReader History；
- DataWriter History；
- Reader ResourceLimits；
- CycloneDDS 内部 defrag/reorder/delivery queue 限制；
- Writer WHC。

CycloneDDS 0.10.2 源码参考：默认 Reader ResourceLimits 的 `max_samples`、`max_instances`、`max_samples_per_instance` 为 `DDS_LENGTH_UNLIMITED`。该值不能替代 G1 实际实体 QoS 和配置审计。

## 12.3 Writer WHC

审计参数：

```text
WhcAdaptive
WhcHigh
WhcHighInit
WhcLow
```

WHC 属于写端 Writer History Cache。达到高水位可能暂停发送，待 WHC 降到低水位后恢复。它不是 Subscriber Reader History。

CycloneDDS 0.10.2 配置 Schema 中的参考默认值：

| 参数 | 0.10.2 Schema 参考值 |
|---|---:|
| WhcAdaptive | true |
| WhcHigh | 500 kB |
| WhcHighInit | 30 kB |
| WhcLow | 1 kB |

这些是特定版本源码/Schema 的参考，不代表 G1 生效水位。自适应机制还可能改变运行时高水位。P0 应记录配置级日志（如 Tracing Verbosity=config）及实际 XML 配置，并在正式结果中引用实际版本。

## 12.4 CycloneDDS 内部队列/分片配置

审计但不扫描调参：

```text
DefragReliableMaxSamples
DefragUnreliableMaxSamples
DeliveryQueueMaxSamples
PrimaryReorderMaxSamples
SecondaryReorderMaxSamples
MaxSampleSize
FragmentSize（若该版本暴露）
SocketReceiveBufferSize
```

这些参数各自属于不同内部路径，不能直接当作 DataReader History Depth。内部参数原则上只做版本/生效配置审计，不做参数扫描；除非 P2/P3 发现明确证据，否则不扩展变量矩阵。

## 12.5 Linux 网络与 IP 分片

P0 记录：

```bash
cat /proc/sys/net/core/rmem_default
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/ipv4/ipfrag_time
cat /proc/sys/net/ipv4/ipfrag_high_thresh
```

同时记录实际 UDP Socket 的 `SO_RCVBUF`，并用 `ss -m` 获取 Socket 统计。

区分：

- DDSI 层分片/重组；
- IP 层分片/重组。

`ipfrag_*` 只在实际链路存在 IP 分片时作为重点解释变量。如果本机传输经 loopback 且报文未触发 IP 分片，则该参数对本轮结果可能无关；若流量经 MTU 1500 的网卡且出现 IP 分片，则需要分析。

Socket 内存可能计入 cgroup `memory.stat` 的 `sock`，但不一定体现在普通用户态进程 RSS 中。

---

# 13. 实验阶段与交付物（V2.1）

## T0：P0 G1 运行时环境审计

按优先级限时执行，先确认版本、实际传输路径和生效配置，再做次要细项。

审计项目：

1. ROS 2、RMW、CycloneDDS、glibc 实际版本；
2. 实际加载动态库路径；
3. `RMW_IMPLEMENTATION`、`CYCLONEDDS_URI`、XML 配置；
4. Reader/Writer QoS 与 ResourceLimits（能观测的实际实体属性）；
5. WHC、Cyclone 内部队列/分片配置；
6. 实际传输路径、网卡/loopback/共享内存及 Socket；
7. `rmem_*`、`ipfrag_*`、实际 Socket receive buffer；
8. cgroup v2 控制接口能力；
9. MemAvailable、SWAP/zram、系统温度、CPU/GPU 频率；
10. 实验安全预算与 Watchdog 可行性。

交付：`P0_G1_RUNTIME_ENVIRONMENT_AUDIT.md`。

## T0.5：P1 基线与工况有效性闸门

任务：

- 空 callback μ₀；
- 固定服务负载的实际 μ；
- λ_call、publish 耗时、missed deadlines；
- KEEP_LAST(1) 低负载基线；
- Age 基线与 T_age 判据；
- RSS/PSS/cgroup 稳定性；
- mallinfo2 基线；
- `ΔM_min` 冻结；
- 监控与记录扰动对照；
- P1 工况有效性闸门。

交付：`P1_BASELINE_AND_VALIDITY_GATE_REPORT.md`。

未通过过载有效性闸门，不进入 P2。

## T1：A/B/C/D History 主实验

- 单一主消息尺寸（由 P2 试点确定）；
- A/B/C/D 四组；
- 随机化组别顺序；
- 每组至少 5 次独立进程运行作为初始方案；
- 最终重复次数根据 P2 方差/效应量决定；
- 每次运行保存原始 CSV/JSONL、配置、版本和异常记录。

交付：`P2_BOUNDED_HISTORY_PILOT_REPORT.md`、正式原始数据集和实验报告。

## T2：恢复与重复过载

主恢复流程：

```text
Baseline
   |
Overload
   |
Recovery（Publisher 持续以实测低于 μ 的速率输入）
   |
记录 Age、排空代理事件与 RSS
   |
队列排空代理事件
   |
Silent Recovery（停止 Publisher 输入）
   |
记录 RSS/PSS、allocator 与 cgroup
```

对 A/D 边界条件优先开展重复过载运行；每次独立进程运行含多个周期（初始候选至少 10 轮，最终由试点确认），第一轮可同时为单次过载主实验提供数据。若 B/C 出现异常非单调趋势，再扩大其重复过载测试。

交付：`P5_RECOVERY_AND_REPEATED_OVERLOAD_REPORT.md`。

## T3：无过载 Soak Control

匹配条件：

- 与重复过载组相同的进程生命周期和总运行时长；
- 相同轮数/阶段时间安排；
- 相同低负载输入；
- 相同 QoS；
- 相同监控、记录缓冲区和采样策略。

比较两组恢复后/阶段边界基线的变化。以独立进程运行作为统计单位；分析轮次到基线的斜率与置信区间，同时区分线性漂移和首轮后平台化。

交付：`T3_SOAK_CONTROL_REPORT.md`。

## T4：Memory Attribution 诊断

代表性条件优先选 A 与 D、单一主尺寸：

1. 自然持续恢复和静默恢复；
2. 阶段边界 malloc_info 快照；
3. 低频 mallinfo2；
4. 完整 smaps 与 cgroup/ss 辅助观测；
5. 最后一轮后独立 malloc_trim(0)；
6. MALLOC_ARENA_MAX 对照；
7. 必要时 MALLOC_MMAP_THRESHOLD_ 定向对照；
8. 每个 allocator 条件都重新测量 μ 和 λ，确认仍处于有效过载工况。

交付：`T4_MEMORY_ATTRIBUTION_REPORT.md`。

## T5：敏感性实验

仅在代表性 QoS/消息尺寸下做小规模验证：

- 256 KiB 或 1 MiB 大消息；
- CPU 忙等型 callback 变体；
- 匹配 Publisher/Subscriber Reliability 的补充实验（KEEP_LAST(10)、KEEP_ALL）。

P6 Reliability 矩阵：

| 组别 | Publisher Reliability | Subscriber Reliability | Subscriber History |
|---|---|---|---|
| R1 | BEST_EFFORT | BEST_EFFORT | KEEP_LAST(10) |
| R2 | RELIABLE | RELIABLE | KEEP_LAST(10) |
| R3 | BEST_EFFORT | BEST_EFFORT | KEEP_ALL |
| R4 | RELIABLE | RELIABLE | KEEP_ALL |

Publisher History 仍为 KEEP_LAST(10)。这是系统级端到端 Reliability 对照，不将结果描述为纯读端效应。

交付：`T5_SENSITIVITY_AND_RELIABILITY_REPORT.md`。

## T6：延伸实验（可砍）

- jemalloc/mimalloc：仅代表性 A/D 组和单一尺寸；
- Fast DDS：小规模对照，不进入 CycloneDDS 主矩阵；
- P7 PointCloud2 rosbag2 真实数据回放：真实传感器外部有效性验证。

这些不是当前论文核心实验完成的硬性前置条件。PointCloud2 不是论文启动的阻塞项。

---

# 14. 统计设计与有效实验单位

1. **独立样本单位：** 独立进程运行，而不是同一进程内的 Overload 周期。
2. **重复测量：** 同一进程的多个周期属于相关数据，分析轮次趋势时使用重复测量/混合效应或其他适合的纵向分析方法，不作为独立 n。
3. **随机化：** A/B/C/D 的实验运行顺序随机化，减少温度、后台服务和运行时间顺序效应。
4. **初始重复数：** 每个主条件至少 5 次独立运行作为初值；依据 P2 方差与效应量调整。
5. **描述统计：** 中位数、IQR、均值、标准差、Age 分位数、时间序列和增长斜率。
6. **推断统计：** 根据分布、重复结构和样本量选择检验；报告效应量与置信区间，不为追求显著性而事后挑选方法。
7. **右删失：** 恢复超过观察窗口的 T_mem 数据按右删失报告，不伪造为窗口上限值或随意删除。
8. **无效轮次：** 预先定义工况有效性和异常终止条件，完整保留无效轮次记录并说明排除理由。

---

# 15. 安全机制

## 15.1 cgroup 布局

```text
g1_qos_experiment
    |
    +---- publisher
    |
    +---- subscriber
```

父 cgroup 统一约束实验整体资源；Publisher/Subscriber 分别位于子 cgroup。外部 Watchdog 和结果保存程序在实验 cgroup 外部。

## 15.2 保护参数

在 G1 实际 cgroup v2 支持的前提下，评估使用：

```text
memory.max
memory.swap.max = 0
memory.oom.group = 1
```

- `memory.max` 根据 P0 实测可用内存、Unitree 服务保留量和安全余量确定；不能用固定的想象值。
- `memory.swap.max=0` 用于避免实验任务依赖 swap/zram 抖动；若接口不可用，记录并重新设计安全预算。
- `memory.oom.group=1` 可使实验组作为整体被 OOM Killer 处理；必须先在独立测试 cgroup 中验证。
- 实验进程可按风险评估设置较高 `oom_score_adj`，但该设置不是绝对保障，也不能取代 `memory.max` 和 Watchdog。
- 不修改 Unitree 生产系统服务的 cgroup、OOM 优先级或系统级内存配置。

## 15.3 Watchdog 软停止

监控：

- experiment cgroup `memory.current`、`memory.events`；
- `/proc/meminfo` `MemAvailable`；
- tegrastats RAM/SWAP、温度、CPU 频率；
- 必要时 Socket drops。

软停止触发：

1. 停止 Publisher；
2. 保存阶段与实验状态；
3. 停止 Subscriber；
4. 保存最终记录。

`memory.max` 是最后一道硬保护，不作为优雅停止机制。主实验不主动设置 `memory.high`，以免引入额外回收/节流变量。

---

# 16. 论文结构

## 1. Introduction

- 机器人传感器管线与持续数据输入；
- ROS 2 History QoS 的作用；
- 过载、背压与内存恢复问题；
- 相关研究不足；
- RQ1–RQ3；
- C1–C4 预期贡献；
- 明确研究平台和适用范围。

## 2. Background and Technical Preliminaries

### 2.1 ROS 2 QoS

History、Depth、Reliability、Durability。

### 2.2 rclcpp Executor

Wait Set、SingleThreadedExecutor、消息 take 与 callback 执行路径。

### 2.3 CycloneDDS History and Resource Management

DataWriter、DataReader、WHC、RHC、ResourceLimits、背压和内部队列。

### 2.4 Process Memory and Allocator

VSZ/RSS/PSS、glibc malloc、arena、tcache、mmap、trim、内核 Socket Memory。

### 2.5 Data Age and Recovery Metrics

Publish-to-Callback Age、T_drain、T_age、T_mem50、T_mem90、流体排队模型。

## 3. Related Work

建议五类：

1. Executor 与订阅缓冲区的时序分析；
2. ROS 2/DDS 性能测量（含内存）；
3. 数据年龄与追踪方法；
4. DDS 背压、HistoryCache 与替代传输机制；
5. Allocator 与内存保留行为。

明确区分同行评审论文、预印本、官方文档、GitHub Issue 和厂商报告。灰色文献不能当作同行评审证据。

## 4. Experimental Methodology

### 4.1 Experimental Platform and Runtime Environment

### 4.2 Benchmark Publisher and Subscriber

### 4.3 QoS Configurations and Workload

### 4.4 Overload Validity Gate

### 4.5 Message Age, Queue Drain and Recovery Metrics

### 4.6 Memory Attribution Method

### 4.7 Safety and Monitoring Overhead

### 4.8 Metric Definitions and Pre-registered Thresholds

## 5. Experimental Results

### 5.1 Runtime Environment and Workload Validation

### 5.2 Overload Behavior under Different History QoS Policies

### 5.3 Memory Footprint and Growth during Overload

### 5.4 Message Age, Queue Drain and Memory Recovery

### 5.5 Repeated Overload versus No-overload Soak Control

### 5.6 Memory Attribution and Allocator Diagnostics

### 5.7 Reliability and Message-size/CPU Sensitivity

## 6. Discussion

- History QoS 的资源和新鲜度取舍；
- Reader/Writer 两端行为；
- 模型预测与实际恢复偏差；
- 内存各层可能贡献；
- G1/Orin NX 实际部署建议。

所有工程建议严格限定在实测平台、版本和配置。

## 7. Threats to Validity

至少讨论：

1. 单一硬件平台；
2. 单一 RMW/DDS 实现和版本；
3. 同机通信路径；
4. 合成消息与真实传感器数据差异；
5. 监控开销；
6. glibc 归因局限（含 tcache）；
7. Jetson DVFS、温度与后台负载；
8. cgroup 与进程 RSS 的口径差异；
9. 排空代理指标不等于直接读取 Reader 队列长度。

## 8. Data and Artifact Availability

计划提供：实验源代码、配置文件、P0 环境审计、原始 CSV/JSONL、分析脚本、版本信息和数据处理说明。公开前检查第三方许可证、Unitree 代码/IP 和任何敏感设备信息。

## 9. Conclusion

总结实测结果和限制，不把单平台结果泛化为全部 ROS 2/DDS 系统。

---

# 17. 当前相关工作与引用状态

| 文献/材料 | 当前状态 | 在本文中的用途与限制 |
|---|---|---|
| ROS 2 QoS 官方设计文档 | 官方资料，引用前逐项核对原文 | 定义 History/Depth/Reliability 等语义 |
| Teper et al., RTCSA 2023, DOI `10.1109/RTCSA58653.2023.00032` | 出版信息已核实 | Executor、订阅缓冲和时序分析；是邻近同行评审工作之一 |
| Tang et al., JSA 2024, DOI `10.1016/j.sysarc.2024.103259` | 出版信息已核实 | Data Refreshing、处理链时序和 buffer 配置理论 |
| Choi et al., IEEE ESL 2026, DOI `10.1109/LES.2026.3727658` | 庆北国立大学官方成果页与 DOI 记录可核实，Accepted/In press | ROS 2 Data Age Tracing；不得把 Data Age 测量本身当作创新 |
| He & Shi, IEEE IV 2026, DOI `10.1109/IV66570.2026.11624004` | 正式会议论文已核实 | SIM 共享内存与有界双缓冲；与本研究对象不同 |
| Maruyama et al., EMSOFT 2016, DOI `10.1145/2968478.2968502` | 出版信息及全文已核对 | 不同 Depth 的时延测试；第 3.6 节测共享库文件 footprint，不是运行时 RSS；不含 CycloneDDS |
| Bédard et al., `ros2_tracing`, IEEE RA-L 2022, DOI `10.1109/LRA.2022.3174346` | 正式论文 | 低开销 ROS 2 tracing 方法基础 |
| ros2_rust Issue #725 / PR #727 | GitHub 工程材料；PR closed 但未合并 | 特定 Rust 客户端路径的无界队列问题，不可当作 rclcpp 行为证据 |
| rosbag2 Issue #2366 | GitHub 工程材料，非同行评审 | 不同 DDS/SHM/allocator 下的内存现象线索，不得当作根因证明 |
| arXiv 2508.11366、2608.15380、2607.01304、2509.03381、2508.10413 等 | 预印本，需逐篇核对版本/全文 | 仅作前沿线索，必须标注非同行评审状态 |
| OSRF TSC RMW reports | 厂商/项目评估材料 | 了解 RMW 对比；注意不同报告立场与方法差异 |
| glibc Bugzilla / libc-help | 工程讨论 | allocator 保留行为背景，不是本文创新证据 |

## 17.1 当前查新结论

建议表述：

> 在现有检索范围内，尚未发现完整覆盖 History QoS、持续过载、消息排空后的进程内存恢复，以及重复过载基线漂移这一组合问题的同行评审研究。但当前查新尚不足以证明该方向存在严格意义上的研究空白。

正式查新仍需完成 IEEE Xplore、Scopus、Web of Science/Google Scholar 等学术数据库检索、关键词组合和施引文献追踪，特别检查 Teper 2023、Maruyama 2016 等相邻研究的后续工作。

---

# 18. V2.1 相对 V2.0 的正式修改记录

1. **保留原题目、研究方向、RQ1–RQ3 和 A/B/C/D 主 History 矩阵。**
2. 增加持续 μ/2 Recovery 后的 **Silent Recovery**，区分内存保留与新消息复用。
3. 增加同样时长/轮数/监控的 **No-overload Soak Control**，排除进程运行时间和监控本身造成的漂移。
4. 将 T_age 判据从固定 10 秒 × 3 窗口改为 P1 基线驱动的短窗口/连续样本判据，并报告时间分辨率。
5. 增加流体排队一阶模型，重点比较实测与预测偏差；不把 History Depth 直接当成真实 Q0。
6. 增加 P1 工况有效性闸门：真实 λ、μ、publish 阻塞、missed deadlines、网络 drops 和序列号对账。
7. 同时记录 publish 开始/返回时间、调用耗时和 callback 入口；避免把 publish 返回时间误用为所有样本 Age 起点。
8. 明确独立进程运行是统计单位；同一进程内轮次属于重复测量。
9. 增加大于 128 KiB 的消息尺寸作为敏感性实验，不进入全矩阵。
10. 增加 CPU 忙等 callback 代表性敏感性实验，不进入全矩阵。
11. 将内存归因升级为正式方法学，但纠正 `mallinfo2`/`malloc_info` 对 tcache 的观测盲区；不使用 uordblks/fordblks 简单相减做精确归因。
12. 增加 smaps、RssAnon/RssFile/VmHWM、缺页统计、cgroup `sock` 和 `ss -m` 作为联合证据。
13. 增加监控/记录开销对照和预分配环形缓冲区。
14. 增加 `memory.swap.max`、`memory.oom.group` 和实验 cgroup 安全约束；具体能力必须在 G1 验证。
15. P4/P5 部分合并采集流程：A/D 优先开展多轮重复过载，首轮数据可复用；B/C 根据主实验结果决定是否扩展。
16. Reliability 补充实验采用 Publisher/Subscriber 匹配设计，定位为系统级效应。
17. jemalloc/mimalloc、Fast DDS、PointCloud2 延后，不作为论文核心完成的硬性前置条件。
18. 修订文献记录：Maruyama 2016 不含 CycloneDDS，其 Depth 实验测时延，静态库 footprint 与运行时 RSS 分开描述；Choi ESL 2026 和 SIM IEEE IV 2026 纳入已核实相关工作。

---

# 19. 当前执行优先级

下一步不再反复改题或无限扩张实验矩阵，按照以下顺序推进：

1. **T0/P0：** 审计 G1 实际运行时版本、配置、传输路径和安全资源预算。
2. **T0.5/P1：** 建立基线、完成工况有效性闸门、冻结 Age 判据和 `ΔM_min`，评估监控开销。
3. **T1/P2：** 先做 KEEP_LAST(1/10/100) 试点，确认消息与内存曲线及实验安全性。
4. **T1/P3：** 对 KEEP_ALL 独立安全校准，实测 Writer 背压和 Reader/内部资源限制。
5. **T1/T2：** 冻结正式参数，执行 A/B/C/D 主实验、持续恢复和静默恢复。
6. **T3：** 运行匹配无过载 Soak Control。
7. **T4：** 仅对代表性条件做内存归因和 allocator 诊断。
8. **T5/T6：** 根据核心结果决定是否投入敏感性和延伸实验。

**方向冻结原则：** 除非正式查新发现实质性重叠、P1/P2 证明核心问题无法有效测量，或 G1 平台不可安全运行，否则不因相邻研究或个别预期未出现而反复更换方向。

---

# 20. 关键官方/原始来源链接

以下链接用于继续核查；论文引用时应以正式出版信息、对应版本文档和最终实际使用的源码版本为准。

- ROS 2 QoS 官方设计：<https://design.ros2.org/articles/qos>
- ROS 2 Humble DDS tuning：<https://github.com/ros2/ros2_documentation/blob/humble/source/How-To-Guides/DDS-tuning.rst>
- ROS 2 Humble rclcpp Executor：<https://github.com/ros2/rclcpp/blob/humble/rclcpp/src/rclcpp/executor.cpp>
- ROS 2 Humble rmw_cyclonedds QoS 映射：<https://github.com/ros2/rmw_cyclonedds/blob/humble/rmw_cyclonedds_cpp/src/rmw_node.cpp>
- CycloneDDS 0.10.2 Reader 默认 QoS：<https://github.com/eclipse-cyclonedds/cyclonedds/blob/0.10.2/src/core/ddsi/src/ddsi_plist.c>
- CycloneDDS 0.10.2 Reader 创建路径：<https://github.com/eclipse-cyclonedds/cyclonedds/blob/0.10.2/src/core/ddsc/src/dds_reader.c>
- CycloneDDS 0.10.2 配置 Schema：<https://github.com/eclipse-cyclonedds/cyclonedds/blob/0.10.2/etc/cyclonedds.rnc>
- CycloneDDS 0.10.2 WHC specifics：<https://cyclonedds.io/docs/cyclonedds/0.10.2/config/cyclonedds_specifics.html>
- glibc 2.35 malloc implementation：<https://github.com/bminor/glibc/blob/glibc-2.35/malloc/malloc.c>
- Linux cgroup v2 documentation：<https://docs.kernel.org/admin-guide/cgroup-v2.html>
- Maruyama et al. EMSOFT 2016：<https://web.ics.purdue.edu/~rvoyles/Classes/ROS_MFET642/Maruyama.ExploringROS2.2016.pdf>
- Teper et al. RTCSA 2023：<https://doi.org/10.1109/RTCSA58653.2023.00032>
- Tang et al. JSA 2024：<https://doi.org/10.1016/j.sysarc.2024.103259>
- Choi et al. IEEE ESL 2026（官方研究成果记录）：<https://research.knu.ac.kr/en/publications/a-non-intrusive-data-age-tracer-for-real-time-ros-2-applications/>
- Choi et al. IEEE Xplore 记录：<https://ieeexplore.ieee.org/document/11667355/>
- He & Shi, IEEE IV 2026 / SIM：<https://doi.org/10.1109/IV66570.2026.11624004>
- Bédard et al. ros2_tracing, IEEE RA-L 2022：<https://doi.org/10.1109/LRA.2022.3174346>
- ros2_rust Issue #725：<https://github.com/ros2-rust/ros2_rust/issues/725>
- ros2_rust PR #727：<https://github.com/ros2-rust/ros2_rust/pull/727>
- ros2/rosbag2 Issue #2366：<https://github.com/ros2/rosbag2/issues/2366>

---

**文档结束。此 V2.1 是当前选题与实验设计基线；实际运行时配置、判据阈值和具体负载参数须经 G1 P0/P1 验证后冻结。**
