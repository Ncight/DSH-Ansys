# DSH-Ansys：让 DeepSeek Harness + Flash 用 ANSYS 做仿真，先审后算

这是给 DeepSeek Harness（DSH）用的 ANSYS（Mechanical APDL）工作区。模型写命令流建模，你在预览面板里审过、点了批准，它才能求解；算完它要拿结果和估算对账再汇报。

它和 [DSH-Abaqus](https://github.com/Ncight/DSH-Abaqus) 是一套：工具箱 `ans`、预览面板和插件的代码都在 DSH-Abaqus 仓库里，这里只放模型每次会话要读的东西。在 DSH 里把工作区选为 `E:\DSH-Ansys`，它们就自动生效。

## 里面有什么

| 路径 | 是什么 |
|---|---|
| `AGENTS.md` | 每次会话都会读到的工作区说明：工具清单、流程、硬性规则 |
| `.dsh/skills/` | 按需加载的五份知识：建模命令流（`ansys-apdl-build`）、求解段（`ansys-solve-segment`）、单元（`ansys-elements`）、接触（`ansys-contact`）、后处理和应力线性化（`ansys-postprocess`）。里面引用的每个例子都在本机实跑过、和手算对过 |
| `ans.cmd` | 工具箱入口，调 DSH-Abaqus 里的 `tools\ans\ans.py` |
| `.abq/config.json` | 人工闸门开关（默认开着）；`peer_workspaces` 写和本工作区共用一个排队的工作区（默认 `E:\DSH-Abaqus`）；`mapdl_max_cores` 是 MAPDL 最多用几个核，不写取 4，许可证允许更多（有 HPC）时调大 |
| `cases/` | 已验证的算例清单（`cases/README.md`）；你的题目也放在这里，一题一个文件夹 |

## 一道题怎么走

1. 模型读题、先估算，再写两个文件：`<job>.build.mac`（几何、网格、材料、组件、初始的约束和载荷，不能求解）和 `<job>.solve.dat`（分析类型、载荷步、对组件加载、`SOLVE`，只能用白名单里的命令）。
2. `ans build <job>`：MAPDL 跑建模命令流，用 CDWRITE 导出 `<job>.cdb`，顺手体检。
3. 停下等你审：在预览面板里看 CDB（几何、约束、载荷、材料、接触、每个载荷步），点「批准求解」或「退回修改」，可以对着具体参数写意见。批准针对这一份 CDB 和求解段的内容，哪一份改了都要重审。
4. `ans run <job>`：核对批准记录 → 排队等资源 → 在干净目录里读入 CDB、执行求解段 → 结果摘要和核查结论。核数按本机此刻的空闲核、模型规模和 `mapdl_max_cores` 自动定；和 DSH-Abaqus 工作区的作业在同一个队里，按提交顺序启动；不设墙钟上限，看门狗盯 MAPDL 的 `<job>.mntr`，太久没有新子步就报警，连续两次停止作业。
5. `ans rst <job>` 取结果（`--comp` 组件、`--step` 载荷步、`--lin` 沿评定线做应力线性化），`ans verify <job>` 收工前核查，和估算对比后汇报。

## 安装

在 Windows 10、Ansys 2025 R1、2026 年 11 月的桌面版 DeepSeek Harness 上做和测的。

1. 两个仓库都克隆下来，放在 `E:\DSH-Abaqus` 和 `E:\DSH-Ansys` 最省事：

   ```
   git clone https://github.com/Ncight/DSH-Abaqus.git E:\DSH-Abaqus
   git clone https://github.com/Ncight/DSH-Ansys.git E:\DSH-Ansys
   ```

   放在别处时，把写死的路径换掉：本仓库的 `AGENTS.md`、`ans.cmd`、`cases/README.md`、`.dsh/skills/ansys-apdl-build/SKILL.md`，以及 DSH-Abaqus 的 `dsh-plugin/abq-gate/index.js`、`tools/ans/skill_checks/run_all.py`（在两个仓库里分别用 `git grep -lF "E:\DSH-Abaqus"`、`git grep -lF "E:\DSH-Ansys"` 列出）。
2. MAPDL 默认在 `D:\Program Files\ANSYS Inc\v251\ansys\bin\winx64\ANSYS251.exe`，装在别处就设环境变量 `ANS_MAPDL_EXE`。中文系统上的 MAPDL 2025 R1 缺语言文件 `fx0.msb`，`ans` 只给自己启动的 MAPDL 把 `CADOE_LIBDIR251` 指到 en-us 语言目录（DSH-Abaqus 的 `tools\ans\mapdl_run.py` 里的 `LANG_FIX`），别的版本或装在别处要改这一行。
3. 预览面板：在 DSH-Abaqus 的 `tools\preview\` 下新建 `extra_roots.local.txt`，写一行 `E:\DSH-Ansys`，再双击同一目录下的 `start_preview.cmd`。在 DSH 里按 Ctrl+T 打开右侧栏的浏览器标签，输入 `127.0.0.1:8765`。打开 CDB 时面板会换成 ANSYS 的样子；超过 60 MB 的模型先报出要占多少内存，等你点「仍然解析」。
4. 插件：运行 DSH-Abaqus 的 `dsh-plugin\install.cmd`，重启 DSH。装上以后，在开了闸门的工作区里直接调用 `ANSYS251`、`mapdl`、`runwb2` 的命令会被拒绝，模型只能走 `ans`；模型也改不了 `.abq` 下的闸门开关和批准记录。
5. Python 3。`ans` 只用标准库，`ans mesh` 要 numpy。

## 闸门

- 不想每次都审：在你自己的终端里、在工作区里运行 `ans.cmd gate off`；恢复用 `ans.cmd gate on`。DSH 里的模型运行 `gate off` 会被拒绝。
- 批准记录在 `.abq/review.json`，按 CDB 和求解段内容的 SHA-256 记。CDB 还必须是最近一次 `ans build` 导出的那一份（记录在 `.abq/builds.json`），手改过的不让算。这些记录和 MAPDL 的运行目录 `.ans/`、模型文件 `*.cdb` 都在 `.gitignore` 里。

## 核对

```
py E:\DSH-Abaqus\tools\ans\skill_checks\run_all.py
```

把 skill 里引用的 14 个例子用 `ans build` + `ans run` 全部实跑一遍，和梁理论、Lamé、欧拉公式等手算结果比（清单见 `cases/README.md`）。改了 skill、改了工具或升级 ANSYS 之后重跑。机器上有别的求解占着内存时，先在同一个终端里设 `set ABQ_RESERVE_MEM_GB=2`，免得测试因为预留内存一直排队。

## 许可

本仓库以 GNU 通用公共许可证第 3 版（GPL-3.0）发布，全文见 `LICENSE`。

仓库里没有任何 ANSYS 自带的内容，skill 里的命令流和例子都是自己写、在本机跑过的。ANSYS 和 MAPDL 是 ANSYS, Inc. 的商标；本项目与 ANSYS, Inc.、DeepSeek 均无关联，使用它需要你自己合法取得的 ANSYS 许可。
