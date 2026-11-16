---
name: ansys-solve-segment
description: 写 ANSYS（MAPDL）求解段 <作业名>.solve.dat 时加载：白名单和写法规则、静力多载荷步、模态、谐响应、瞬态、特征值屈曲、杀死/激活单元、螺栓预紧七种已验证的写法，以及每条载荷命令在载荷步之间到底怎么起作用。
---

# 求解段 `<job>.solve.dat`

`ans run` 在一个只有 `<job>.cdb` 和 `<job>.solve.dat` 的干净目录里，先 `CDREAD` 读入 CDB，再逐行执行求解段。
面板把两次 `SOLVE` 之间的命令显示成一个载荷步。求解段只能改分析设置和载荷，不能改模型，所以规则很严。

## 写法规则（不满足的行 ans check / ans run 直接拒绝，并指出行号）

- **第一条命令是 `/SOLU`**。
- 每行一条命令，`!` 后面是注释。不能用 `$`。
- 只用白名单里的命令（下表）。没有选择命令（`NSEL`、`CMSEL`…）、没有 `*DO`/`*IF`/`*SET`、没有参数赋值和 `%参数%`。
- 数值只能写数字：`-4`、`1.0E6`。不能写算式（`-100/25`）也不能写参数名。
- 约束和载荷**只能对组件名或 `ALL` 加**，不能写节点号。组件必须在 CDB 里（建模时用 `CM` 建好），类别要对：`D DDELE F FDELE SF SFDELE BF BFDELE` 用节点组件，`EKILL EALIVE` 用单元组件。`SLOAD` 的对象是预紧截面号。

| 用途 | 命令 |
|---|---|
| 进出 | `/SOLU` `FINISH` `ALLSEL` |
| 分析类型和选项 | `ANTYPE` `NLGEOM` `PSTRES` `MODOPT` `MXPAND` `BUCOPT` `HROPT` `HARFRQ` `TRNOPT` `TINTP` `EQSLV` `TIMINT` |
| 时间步和收敛 | `TIME` `NSUBST` `DELTIM` `AUTOTS` `KBC` `NEQIT` `CNVTOL` `LNSRCH` `NROPT` |
| 阻尼 | `ALPHAD` `BETAD` `DMPRAT` `DMPSTR` |
| 输出 | `OUTRES` `OUTPR` `RESCONTROL` |
| 载荷 | `D` `DDELE` `F` `FDELE` `SF` `SFDELE` `BF` `BFDELE` `ACEL` `TUNIF` `TREF` `EKILL` `EALIVE` `SLOAD` |
| 求解 | `SOLVE` |

## 载荷在载荷步之间怎么起作用（MAPDL 2025 R1 实测）

- 载荷一直有效，直到被删掉或改写。CDB 里的约束和载荷就是第一个载荷步的起点。
- `F,组件,FZ,值`：组件里**每个节点**加这个值；对同一组件再写一次是**改写**，不是叠加。`FDELE,组件,FZ` 删掉。
- `SF,节点组件,PRES,值`：作用在节点全属于这个组件的单元面上；会改写这些面上已有的面压（包括 CDB 带来的）。`SFDELE,节点组件,PRES` 把这些面的面压删掉（也包括 CDB 带来的）。
- `D,组件,UX,值`：给定位移；`DDELE,组件,UX` 删掉这个方向的约束（`ALL` 删全部）。
- `BF,节点组件,TEMP,值` / `BFDELE`：节点温度。`ACEL,ax,ay,az`：整体加速度（重力向下就写 `ACEL,0,0,9.81`，惯性力方向与它相反）。
- 没写 `TIME` 的载荷步，结束时间 = 上一步加 1。`KBC,0` 斜坡加载（默认），`KBC,1` 阶跃。
- 换分析类型（比如静力之后做屈曲）要 `FINISH` 再 `/SOLU`，然后写新的 `ANTYPE`。

## 七种已验证的写法

都在本 skill 目录，配合 `ansys-apdl-build/template_cantilever.build.mac`（端部组件 NTIP 有 25 个节点）实跑过，结果和手算对得上：

| 文件 | 内容 | 实跑结果 |
|---|---|---|
| `static_steps.solve.dat` | 静力三个载荷步，改写 F，第三步不写 TIME | 结束时间 1、2、3；200 N 时挠度与梁理论差 2% 以内 |
| `modal.solve.dat` | `ANTYPE,MODAL` + `MODOPT,LANB,6` + `MXPAND,6` | 一阶频率与梁理论 40.76 Hz 差 1% 以内 |
| `harmonic.solve.dat` | `HROPT,FULL`，0～100 Hz 50 个点，`DMPSTR` 结构阻尼 | 响应峰值在一阶频率附近 |
| `transient.solve.dat` | `TRNOPT,FULL`，`DELTIM`，`KBC,1`，加载后 `FDELE` 卸载 | 100 个结果集，到 0.1 s |
| `buckle.solve.dat` | 静力 `PSTRES,ON` → `FINISH` → `/SOLU` → `ANTYPE,BUCKLE` | 载荷因子与欧拉公式 2.570e5 N 差 2% 以内 |
| `ekill.build.mac` + `ekill.solve.dat` | `NROPT,FULL` 写在第一次 SOLVE 之前，`EKILL`/`EALIVE` 单元组件 | 三步都算完，最后反力 4000 N |
| `bolt.build.mac` + `bolt.solve.dat` | `PSMESH` 切预紧截面，`SLOAD,1,PL01,TINY,FORC,1000,1,2` | 预紧力 1000 N 全部由支座承担 |

## 容易错的地方

- **线性静力里用 `EKILL`，`NROPT,FULL` 必须写在第一次 `SOLVE` 之前**，写在后面 MAPDL 报错停算（`ans check` 会指出）。
- 模态、谐响应、瞬态要材料密度（`MP,DENS`），`ans check` 会报缺密度。
- 求解段最后一个 `SOLVE` 之后的命令不起作用（除了 `FINISH`）。
- 结果输出：一般写 `OUTRES,ALL,ALL`，瞬态想少存可以 `OUTRES,ALL,10`。
- 写错组件名、写节点号、用算式，`ans check` 都会指出行号；先 `ans check` 到零错误再请用户审查。
