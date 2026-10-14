# 版本与证据边界

| 身份 | 当前记录 |
|---|---|
| 阅读版本 | ovphysx-0.6.3 源码标签；SDK 头文件 5.11.0 |
| 官方源码 | [NVIDIA-Omniverse/PhysX](https://github.com/NVIDIA-Omniverse/PhysX/tree/da950a3537927784951853c66618036f332ca0ce) |
| 固定提交 | `da950a3537927784951853c66618036f332ca0ce` |
| 源文件身份 | [sources.json](sources.json) 中的 Git blob 已与下载文件核对 |
| 已完成检查范围 | 文档相对链接、固定来源与源码文件身份；C++ 片段检查范围见 [E1](validation/e1.md)、[E2](validation/e2.md) |
| DexLab 独立原生斜面批次 | SDK 5.9.0 / CPU / FP32；与本课程源码身份分开，见 [配置解读](solver-reading-guide.md) |
| DexLab 历史宿主/布料批次 | native core/build 与部分 solver 读回仍缺失，保留 #126 审计边界；不以当前默认值回填 |
| 原生运行与实验 | 本阶段未开展 |
| 完整专题 | [课程目录](curriculum.md)记录 E1–E6 源码专题与新增 solver 配置解读；A0 完整安装与 E7 全路线审校继续暂停 |

固定文件身份不能证明整个引擎已审查，也不能证明候选二进制与源码具有相同构建配置。核心、绑定、插件和宿主身份分别记录。当前版本的默认值不用于补填 DexLab 历史配置。

[DexLab](https://github.com/huangkiki/Dexlab) 保留实验版本与协议；本仓不修改或追认其结果。
