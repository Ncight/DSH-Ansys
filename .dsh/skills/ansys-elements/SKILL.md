---
name: ansys-elements
description: 选 ANSYS（MAPDL）单元类型、截面、KEYOPT 和网格时加载：实体/壳/梁/平面与轴对称/质量/弹簧/接触单元各用在哪、截面怎么定义、轴对称怎么建（含已验证的厚壁圆筒算例）、网格命令和退化形状。
---

# 单元与网格

## 先定理想化，再选单元

用够用的最简单的那种：细长构件用梁，薄板用壳，轴对称问题用轴对称平面单元，只有块体才用实体。任务书指定了单元就照任务书。

| 单元 | 用在哪 | 要点 |
|---|---|---|
| SOLID185 | 8 节点实体，规则几何划六面体 | 默认 KEYOPT(2)=0（B-bar）：位移准，但弯曲时外表面一排节点的应力偏低（厚度方向 8 层时约低 5%）。以弯曲为主又要表面应力时用 `KEYOPT,1,2,2`（增强应变）。本工作区模板 `template_cantilever` 已验证 |
| SOLID186 | 20 节点实体（二次） | 应力梯度大、网格粗时用 |
| SOLID187 | 10 节点四面体（二次） | 复杂几何自动划四面体：`MSHAPE,1,3D` + `MSHKEY,0` + `VMESH` |
| SHELL181 | 4 节点壳 | 厚度用 `SECTYPE,n,SHELL` + `SECDATA,厚度,材料号`。模板 `template_shell` 已验证 |
| BEAM188 | 2 节点梁（考虑剪切） | 截面 `SECTYPE,n,BEAM,RECT/CSOLID/CTUBE/I…` + `SECDATA`；`LATT` 给方向关键点。模板 `template_beam` 已验证 |
| LINK180 | 只受拉压的杆 | 截面积用 `SECTYPE,n,LINK` + `SECDATA,面积` |
| PLANE182 | 4 节点平面单元 | KEYOPT(3)：0 平面应力、1 **轴对称**、2 平面应变。下面的厚壁圆筒已验证 |
| MASS21 | 集中质量 | 质量放实常数 R |
| COMBIN14 | 弹簧-阻尼 | 刚度放实常数 R |
| CONTA174 / TARGE170 | 三维面-面接触 | 见 skill `ansys-contact` |
| PRETS179 | 螺栓预紧 | 由 `PSMESH` 生成，不手建；见 skill `ansys-solve-segment` 的 `bolt` 例子 |

## 轴对称（已验证）

`axisym_cylinder.build.mac` + `.solve.dat`：厚壁圆筒内半径 0.1 m、外半径 0.2 m，内压 10 MPa。
- `KEYOPT,1,3,1` 设轴对称；模型画在 X >= 0 一侧，**X 是半径、Y 是轴向**。
- 只约束轴向一个点或一条边（`D,NBOT,UY,0`），半径方向不用约束（环向刚度会挡住）。
- 内压用 `SF,NINNER,PRES,1.0E7`（作用在两个节点都属于 NINNER 的边上）。
- 实跑：内表面径向位移与 Lamé 解 9.833e-6 m 差 1% 以内。

## 网格

- `ESIZE,边长` 定整体尺寸；`LESIZE,线,,,份数` 定某条线分几份。
- 规则块体：`VMESH,ALL`（默认能划六面体就划六面体）；复杂体：`MSHAPE,1,3D` + `MSHKEY,0` 划四面体。
- 面：`AMESH`；线：`LMESH`（梁）。
- 划完用 `ans mesh <job>` 看质量：翻转单元（雅可比 <= 0）必须修；长宽比、角度超出常用上限的先看在哪。
- 预览里 SOLID185 退化成棱柱、四面体、金字塔（重复节点）是正常的，面板会认出来。

## 截面和实常数

- 壳：`SECTYPE,1,SHELL` + `SECDATA,厚度,材料号`；梁：`SECTYPE,1,BEAM,RECT` + `SECDATA,宽,高`。
- 单元用哪个截面：划网格前写 `SECNUM,n`（或 `LATT`/`AATT`/`VATT` 里给）。
- 实常数：`R,编号,值…`，划网格前写 `REAL,编号`。
- `ans check` 会报「单元没有截面」「缺实常数」。
