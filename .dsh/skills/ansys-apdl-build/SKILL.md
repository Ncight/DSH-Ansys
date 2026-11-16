---
name: ansys-apdl-build
description: 写 ANSYS（MAPDL）建模命令流 <作业名>.build.mac 时加载：文件结构、实体/壳/梁三个已验证的模板、组件怎么建、哪些东西放建模里、ans build 的输出怎么看、常见错误。
---

# 建模命令流 `<job>.build.mac`

## 它是什么

`ans build <job>` 在一个干净目录里用 MAPDL 无界面跑这个文件，然后由工具在末尾追加导出命令，写出 `<job>.cdb`。
用户在预览面板里审查的就是这个 CDB，求解时也只读这个 CDB。所以：

- 建模命令流里**不能有求解**（`SOLVE`，以及用宏、`/INPUT`、`*USE` 间接调用的求解）。工具在跑之前扫一遍，跑完再查累计载荷步数，求解过就作废、不产出 CDB。
- **CDB 不能手改**。改模型就改 `.build.mac`，重新 `ans build`（之后要重新审查）。
- 分析类型、时间步、载荷步的变化都写在求解段 `<job>.solve.dat` 里（skill `ansys-solve-segment`）。建模里可以写**初始**的约束和载荷，它们会写进 CDB，成为第一个载荷步的起点。

## 文件结构（照这个顺序）

```
/PREP7
单元类型 ET、KEYOPT
材料 MP（EX、PRXY、DENS…）、截面 SECTYPE/SECDATA、实常数 R
几何 BLOCK / RECTNG / K、L …
网格 ESIZE + VMESH / AMESH / LMESH
组件 NSEL/ESEL … + CM
初始约束和载荷 D / F / SF（可选）
ALLSEL,ALL
FINISH
```

## 组件：要加载荷和取结果的地方都先建好

求解段只能**对组件名**加约束和载荷，`ans rst --comp 组件名` 也按组件取结果。所以建模时把它们都建成组件：

```
NSEL,S,LOC,X,0        ! 选 x = 0 的节点
CM,NFIX,NODE          ! 存成节点组件 NFIX
ESEL,S,CENT,X,2,4     ! 选形心 x 在 2～4 的单元
CM,ERIGHT,ELEM        ! 存成单元组件（EKILL / EALIVE 用）
ALLSEL,ALL            ! 选完一定恢复全选，后面的命令才作用到整个模型
```

- 名字用英文字母开头，字母、数字、下划线，不要和参数（`*SET`、`=` 赋的值）同名。
- 约束、力、面载荷、温度用**节点**组件；杀死/激活单元用**单元**组件。
- 想取某一点的位移，就在那里建一个只含这个节点的组件（例如端部中点）。

## 三个已验证的模板

都在本 skill 目录里，`ans build` + `ans run` 实跑过，结果和手算对得上（核对脚本 `E:\DSH-Abaqus\tools\ans\skill_checks\run_all.py`）：

| 模板 | 内容 | 实跑结果 |
|---|---|---|
| `template_cantilever.build.mac` + `.solve.dat` | 实体悬臂梁 1 m × 0.05 m × 0.05 m，SOLID185，端部 100 N | 端部挠度和梁理论 3.2e-4 m 差 1% 以内 |
| `template_shell.build.mac` + `.solve.dat` | 壳悬臂板 1 m × 0.2 m × 0.01 m，SHELL181 + `SECTYPE,1,SHELL` | 和梁理论 5.0e-4 m 差 5% 以内 |
| `template_beam.build.mac` + `.solve.dat` | 梁悬臂，BEAM188 + `SECTYPE,1,BEAM,RECT`，方向关键点 | 和梁理论（含剪切）3.206e-4 m 差 1% 以内 |

新题照最接近的那个改：换尺寸、换材料、换网格密度、换组件。

## F 是每个节点的值

`F,组件,FZ,值` 给组件里**每个节点**加这个值，不是总力。总力 = 值 × 节点数。
`ans check` 的模型摘要里列出每个组件的节点数，也会把 F 写成「每节点 × 节点数 = 合力」。要总力 P，就写 P / 节点数。
均匀分布的面力用面压更自然：`SF,节点组件,PRES,压强`（作用在节点全属于这个组件的单元面上，正值压向表面）。

## ans build 之后看什么

- `✓ 建模完成` 之后是模型摘要和体检。体检的错误要先改掉；警告逐条看。
- 常见体检结果：材料缺密度（模态、瞬态、谐响应、重力都要）、`√(E/ρ)` 量级不对（单位制混用）、有一块网格在某个方向没有约束（静力会刚体位移）、两块实体网格贴着却没连上（要绑定接触或共节点，skill `ansys-contact`）。
- 建模出错时，工具列出 MAPDL 的 ERROR 原文，全文在作业文件夹下 `.ans\<job>\build\output.out`。

## 常见错误

- 选择集没恢复：`NSEL`、`ESEL` 之后忘了 `ALLSEL,ALL`，后面的网格划分或约束只作用在部分实体上。
- 几何体互相独立：两个 `BLOCK` 挨着放并不共节点。要连在一起，用 `VGLUE,ALL`（划网格前）或建接触对。
- 单位制混用：长度用 m 时模量用 Pa、密度用 kg/m³；用 mm 时模量 MPa、密度 t/mm³（`ans units`）。
- 作业名只能用英文字母、数字、下划线，字母开头。
