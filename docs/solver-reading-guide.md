# 读懂 PhysX 的 solver 与配置标签

[Sim Learning](https://github.com/huangkiki/sim-learning) · [共同求解概念](https://github.com/huangkiki/sim-learning/tree/main/docs/solver-guide) · [课程目录](curriculum.md) · [版本边界](versions.md) · [English summary](#english-summary)

面向熟悉 VLA、WAM 或 RL 的工程师与硕博读者。先沿总仓的 [Physics-Based Simulation 先修路线](https://github.com/huangkiki/sim-learning/blob/main/docs/physics-based-simulation.md)建立状态、能量、时间离散和数值求解的共同语言，再用本篇定位 PhysX 的原生字段。书中的 IPC 与本篇的 PhysX 接触实现不能直接画等号。

读完应能回答两个问题：一条实验配置到底改变了哪一层；机器人滑动、接触力异常或布料穿越时，下一份应查的证据是什么。本文不重复 [E3 接触与求解](contact-solvers.md)及 [E6 非刚体与扩展](extensions-boundaries.md)的完整推导，也没有运行引擎。

## 1. 先区分三份版本身份

| 证据范围 | 已登记身份 | 本文怎样使用 |
|---|---|---|
| 本仓源码课程 | `ovphysx-0.6.3`，提交 `da950a3537927784951853c66618036f332ca0ce`，SDK **头文件 5.11.0** | 解释下面的 C++ API 与实现；未证明本地安装或加载此二进制 |
| DexLab 原生斜面批次 | SDK **5.9.0 / CPU / FP32**，PGS/TGS 等四个配置 | 以原批次报告、固定源码与读回为准，不套用本仓头文件版本 |
| DexLab 历史 Isaac/UniSim 与 surface cloth 批次 | 原记录中的宿主、适配器和 surface API；逐批次 native core/build 身份未恢复 | 保留 #126 审计结论；不能统一标成 5.9.0、5.11.0 或 TGS |

DexLab 对照冻结在提交 [`7e4a0774efb4f8b332e866e1495e032b4969c8aa`](https://github.com/huangkiki/Dexlab/tree/7e4a0774efb4f8b332e866e1495e032b4969c8aa)。[原生斜面报告](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/physx-incline-results.md)和[历史 solver 审计](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/physx-solver-audit.md)是不同证据集合。新批次成功读回的设置不能补成旧批次当时的观测。

## 2. 同一行里混合了哪些角色

| DexLab 标签 | 角色 | 在 PhysX 中读哪里 | 不能由标签推出什么 |
|---|---|---|---|
| **PGS** | 刚体约束求解器 | `PxSceneDesc::solverType = PxSolverType::ePGS` | 不是 CPU/GPU、接触几何或浮点精度的名称 |
| **TGS** | 刚体约束求解器 | `PxSolverType::eTGS`；内部位置迭代与时间推进 | 不是一个保证所有任务更准的等级，也不等于多次应用层 `simulate` |
| **surface cloth** | 柔性物体表示与实现路径 | 历史批次为 `OmniPhysicsSurfaceDeformableSimAPI`；本仓当前原生入口为 `PxDeformableSurface` | 不是第三种刚体 solver；不能据此给布料写上 PGS/TGS 身份 |
| **patch friction** | 接触 patch 层的摩擦模型/处理方式 | patch、friction anchors、静/动摩擦系数及切向约束 | 不是 Young's modulus，也不等于每个接触点各有一份完整摩擦容量 |
| **external forces every iteration** | TGS 内部外力施加时机 | `eENABLE_EXTERNAL_FORCES_EVERY_ITERATION_TGS` | 不是提高重力大小；不是每轮都推进控制器或产生新的传感器帧 |
| **friction every iteration** | 刚体摩擦约束的更新频率 | `eENABLE_FRICTION_EVERY_ITERATION` 对 PGS 有效；本版本 TGS 固有地每轮处理摩擦 | 不是另一种摩擦系数，也不是 TGS 可独立切换出的新模型 |
| **historical readback incomplete (#126)** | 历史证据边界 | 对应原记录和 [#126](https://github.com/huangkiki/Dexlab/issues/126) 审计 | 不是算法；未知 solver 不等于默认 PGS/TGS，也不等于运行失败 |
| **pending qualification** | 特定任务/路径的资格状态 | 找到该条矩阵对应的协议、配置和后续任务 | 不是引擎完全不支持物理，更不能抹去其他独立批次已有的结果 |

角色清单来自 [DexLab taxonomy](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/solver-taxonomy.json)。原生枚举和 patch 说明见固定 [`PxSceneDesc.h`](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/include/PxSceneDesc.h#L25-L69)，各开关见[同文件场景标志](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/include/PxSceneDesc.h#L245-L277)。

## 3. PGS 与 TGS：先理解一次约束更新

把一个无摩擦、无柔顺的法向接触行简化为：当前法向相对速度为 $u$，目标速度（含已选定的纠正项）为 $b$，该行的有效逆质量为 $w>0$，累计冲量为 $p\ge0$。一次理想化投影更新是：

$$
p^+=\max\!\left(0,\ p+\frac{b-u}{w}\right),\qquad
\Delta p=p^+-p.
$$

$u,b$ 的单位为 m/s，$w$ 为 kg⁻¹，所以 $p$ 为 N·s。用 **增量** $\Delta p$ 更新速度后，下一行会看到更新后的结果；再次把整个 $p^+$ 加到速度上会重复施加已有冲量。这个纸上模型不包含 articulation 树响应、摩擦、柔顺、最大冲量限制等实际分支。原生 PGS 路径见 [`solveDynamicContacts`](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/source/lowleveldynamics/src/DySolverConstraintsShared.h#L26-L78)；完整响应、材料和报告语义见 [E3](contact-solvers.md)。

PGS 的直觉是“逐行修正，并把本行修正立刻交给后面的行”。一次 sweep 之后，前面的约束又可能受后面的更新影响，所以长链、质量比和排列都会影响有限预算的结果。位置迭代中存在消除穿透的位置 bias；这不表示每轮都直接重设刚体位姿。

TGS 的关键差别之一是位置迭代带内部时间推进。固定 CPU 路径按岛汇聚位置迭代预算 $N$，取内部步长：

$$
\delta t=h/N,
$$

其中 $h$ 是一次公开 `simulate(h)` 的步长。内部更新依据累计位移修正约束，最后的 velocity iterations 不再推进位姿；这不是每小步重新执行完整应用回调、碰撞生成和 `fetchResults`。相关入口是 [`SetStepperTask`](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/source/lowleveldynamics/src/DyTGSDynamics.cpp#L1624-L1630)和[位置/速度循环](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/source/lowleveldynamics/src/DyTGSDynamics.cpp#L2386-L2460)。

因此，PGS 的 8 次、TGS 的 8 次、其他引擎的 8 次 Newton 外迭代不是同一预算单位。`setSolverIterationCounts(position, velocity)` 是求解预算；不要为它虚构一个等价的残差容差。配置请求、实际迭代计数、约束残差和任务是否成功也要分别记录。

## 4. 摩擦模型与两个 every iteration 开关

理想各向同性 Coulomb 摩擦可写成 $\|\mathbf p_t\|\le\mu p_n$，用于解释法向支撑与切向容量的关系。PhysX 的 patch friction 将接触组织成 patch，再在 anchors 上处理切向约束。固定源码的 PGS 按切向轴处理，TGS 对一个 anchor 的两个切向分量联合限幅；法向点报告、friction anchor 报告并非一一对应。深入实现与读回缺项见 [E3 第 4、7 节](contact-solvers.md)。

`friction every iteration` 改变的是**何时求解摩擦行**。固定 PGS 默认在最后三次 position iterations 及全部 velocity iterations 处理摩擦；开关扩展到全部位置迭代。TGS 已经每轮处理摩擦，这个 PGS 开关不能让 TGS 再多出一套新摩擦物理。当前源码只有 patch friction；不要从旧版本文章复原当前已不存在的 one-/two-direction 枚举。

`external forces every iteration` 改变的是**如何把整步外力贡献分配到 TGS 内部位置小步**。它不是每轮重复施加完整的 $Fh$ 冲量。对零初速度、恒加速度 $a$、无接触无阻尼的纸上半隐式更新，若 $N$ 小步各先更新速度再更新位置：

$$
\Delta x=\frac{N+1}{2N}ah^2.
$$

先把整步速度增量 $ah$ 加完再积分则是 $ah^2$。当 $N=4$，两者系数为 $5/8$ 与 $1$；这能解释外力时机会影响离散位移，不能充当真实 PhysX 运行结果。实际内部外力/积分路径及推导条件见 [E3 第 6 节](contact-solvers.md)。这个 flag 只适用于 TGS，不能把无效的 PGS 组合登记成一次有效算法对比。

## 5. surface cloth 为什么另起一条路径

刚体具有整体位姿和惯量，surface cloth 则有三角表面顶点状态、材料、厚度、膜/弯曲响应与自接触。DexLab 历史布料使用宿主 surface API；本仓的固定源码可沿 [`PxDeformableSurface`](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/include/PxDeformableSurface.h#L135-L223)、[`PxDeformableSurfaceMaterial`](https://github.com/NVIDIA-Omniverse/PhysX/blob/da950a3537927784951853c66618036f332ca0ce/physx/include/PxDeformableSurfaceMaterial.h#L28-L77)及 [E6 FEM surface 路径](extensions-boundaries.md)阅读。它们提供概念和当前 API 对照，不能证明历史宿主加载了这份源码。

材料 $E$ 是压力单位（Pa），厚度是长度，接触 offset 也是长度却承担另一职责；同维度并不代表可互换。渲染网格、碰撞表示、实际求解自由度、顶点质量和 attachment 都要核对，不能只凭一张布的画面判断实现。

当布料任务需要逐节点外力，而指定版本只暴露状态或 attachment 接口时，不能通过每步强行改位置来冒充同一力控制实验。[DexLab 历史 cloth 报告](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/demos/physx-contact/cloth.md)保留了相应 unsupported 与表面穿越失败；本篇不重新执行或改写这些结果。

## 6. 用证据定位问题，再选配置

| 现象/目标 | 先查什么 | 查明以后才讨论什么 |
|---|---|---|
| 相同策略在迁移后滑落 | 动作单位/限幅、驱动模式、质量/惯量、碰撞表示、材料组合、时间步和控制频率 | PGS/TGS 及摩擦更新预算；学习回报不能定位单一物理原因 |
| 增加迭代后自由落体位移变了 | TGS、外力时机 flag、岛内 iteration 请求、记录区间 | 是否符合所用离散路径；不能先断言“更多迭代反而更错” |
| 接触力与动量差不一致 | 报告是 impulse 还是 force、法向/切向/纯力矩覆盖、采样阶段、CPU/GPU/宿主路径 | 浮点运算和 solver 误差；不先删失败值或放宽冻结阈值 |
| 布料穿过自身或障碍 | surface 拓扑、offset/厚度、时间采样、连续与离散碰撞、独立几何检查 | 该模型/路径的配置和能力边界；不能只切 rigid PGS/TGS 归因 |
| 要复用历史结果训练/评测 | 原批次 native/host/adapter 身份、实际配置、有效观测和失败记录 | 与新数据的可比范围；不能以当前默认值补历史空值 |

配置卡至少记录：**批次及代码身份 → native/host/adapter → CPU/GPU 与精度证据 → 对象/碰撞表示 → 材料和单位 → $h$ 与控制频率 → solver → position/velocity 预算 → 两个更新开关 → 观测来源和资格状态**。没有读取到的项写“未知”，不要写 `false` 或默认值。这个表是阅读/复现清单，不是已经执行的配置。

力单位的最小自查：若一整步 $h=0.002$ s 的接触冲量为 $0.004$ N·s，平均力是 $p/h=2$ N。只有报告确实覆盖该时间区间与所需分量时才可这样换算；不能把最新一个内部子步或缺失切向/力矩的报告直接叫完整接触力。

## 7. 阅读练习与答案

1. **记录写“surface cloth, 16 position iterations”，能否归入 TGS=16 这一列？**

   不能。先确认柔性实现/API、版本、自由度和对应迭代语义。刚体 solver 与柔性 position iterations 不是由同名计数建立等价关系。

2. **同一场景使用 PGS，开启 friction every iteration 能否证明静摩擦系数变大？**

   不能。改变的是求解摩擦行的频率；材料参数可能未变。有限预算下表现可能变化，是否改善仍需同协议下的独立验证。

3. **TGS 公开步长 2 ms、岛位置迭代 8 次，控制器是否自然变成 4 kHz？**

   不是。内部步长为 0.25 ms，但应用控制器的调度没有因此改变。若每次 `simulate/fetchResults` 才提交控制，它仍是 500 Hz；另有 action decimation 时还要继续换算。

4. **当前源码默认 solver 为 PGS，历史报告 solver=null，是否应该补 PGS？**

   不应该。`null` 只表示缺少这份观测；宿主可覆盖默认值，当前源码也未被证明是旧批次依赖。保留 #126 的历史缺口。

5. **新 5.9.0 CPU 斜面批次有结果，是否能删除全表的 pending qualification？**

   不能整体删除。资格属于任务、配置、路径与协议；CPU 新批次并不补齐 GPU、历史宿主、布料或其他任务的证据。

6. **单行模型中 $p=0.1$ N·s、$u=-0.2$ m/s、$b=0$、$w=2$ kg⁻¹，更新多少？**

   $p^+=0.2$ N·s，实际新增 $\Delta p=0.1$ N·s。该法向速度改变量是 $w\Delta p=0.2$ m/s。这里采用法向接近为负的符号约定，且仅是本节的简化单行模型。

## English summary

This reading guide separates rigid-body constraint solvers (PGS/TGS), deformable representation (surface cloth), the friction model (patch friction), update flags, and evidence status. It bridges shared numerical concepts to native fields, diagnostics and answered exercises for readers experienced in VLA, WAM or RL.

The course source is pinned to `ovphysx-0.6.3`, SDK headers 5.11.0. DexLab's standalone CPU incline cohort records SDK 5.9.0; historical host/surface cohorts retain their missing native-core and solver identities. Current defaults do not repair historical readbacks. This chapter is source/static work only. E7 installation and the full course audit remain paused; no native build, import, JIT, simulation or training is included.
