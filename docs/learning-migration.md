# Learning 迁移与 solver 导读检查

日期：2026-10-14。范围：当前 PhysX Learning 品牌、导航和 [solver 配置解读](solver-reading-guide.md)。这是本轮静态交付记录；E1–E6 的原验收记录保持不变。A0 完整安装与 E7 全路线审校继续暂停。

## 改动与来源

- 当前 README 中英、源码地图、章节返回入口与工程规则使用 PhysX Learning / Sim Learning 和新的仓库 URL。
- 新导读覆盖 PGS、TGS、surface cloth、patch friction、两个 every iteration 标签，以及 historical readback incomplete (#126) / pending qualification；提供原生字段、最小公式、诊断次序、六道附答案练习和 English summary。
- 先修指向总仓 Physics-Based Simulation 阅读路线；不将书中 IPC 等同为 PhysX 的接触实现。
- 当前课程源码仍是 `ovphysx-0.6.3` / `da950a3537927784951853c66618036f332ca0ce`，SDK 头文件 5.11.0。`docs/sources.json` 保持不变。
- DexLab 对照固定为 `7e4a0774efb4f8b332e866e1495e032b4969c8aa`，使用 taxonomy、原生斜面报告、历史 solver 审计和 cloth 报告。独立 5.9.0 CPU 批次、历史宿主批次与本仓源码身份分别说明。

## 保留旧名称的例外

| 位置 | 保留内容 | 原因 |
|---|---|---|
| `docs/validation/e1.md` 至 `e6.md` | 原阶段 PhysX Atlas / Sim Atlas 名称及原 Issue URL | 不改写当时验收的引用与身份；当前导航已使用新地址 |
| `examples/e3_contact_readback.cpp`、`e4_scene_query.cpp`、`e5_batch_boundaries.cpp`、`e6_constraint_row.cpp` | `physx_atlas` C++ namespace | 既有片段的源码标识不承担网站导航；本轮只改文档，保留已验收片段身份与外部使用兼容性 |

没有改写官方原生 API、上游源码、数据集/模型身份、历史实验结果或本仓原验证记录；没有以当前默认值补齐旧批次未知设置。

## 静态验收

在仓库根目录运行；`PHYSX_SOURCE` 指向核验后的固定上游源码缓存，包含 `physx/include`：

```bash
git diff --check
python3 scripts/check_docs.py
python3 scripts/check_source_links.py --source-root "$PHYSX_SOURCE"
python3 -m compileall -q scripts
```

结果：四项检查均退出码 0；本地 Markdown 链接通过，**184 个固定源码 blob、632 处源码链接**身份/行号通过，维护脚本语法通过。新增内容不含可执行示例；既有 C++ 片段未修改，无需为改名重新构建 SDK。另对新文件检查空白，并确认来源 manifest、所有既有示例和 `docs/validation/e1.md` 至 `e6.md` 与基线完全一致。

行号检查只证明来源文件身份与引用行存在，不能自动证明语义。作者另核对了更新开关的适用分支、TGS 内部时钟、surface API 身份、冲量单位和历史证据边界。总仓跨仓链接在集成交付时统一检查。

没有 native build/import、CUDA 编译/JIT、物理步、渲染、训练或性能基准；来源核对不构成新的物理资格或教程全路线运行验收。外部发布与主审结果由总仓任务记录，本文件不宣称已合并。
