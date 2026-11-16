# 已验证的算例

这些算例都在 `.dsh\skills\` 里，用 `ans build` + `ans run` 实跑过，结果和手算对得上
（核对脚本 `E:\DSH-Abaqus\tools\ans\skill_checks\run_all.py`，2026-11-15 全部通过）。新题照最接近的那个改。

| 算例 | 文件 | 实跑结果 / 手算 |
|---|---|---|
| 实体悬臂梁，端部 100 N | `ansys-apdl-build\template_cantilever.*` | 挠度 0.3175 mm / 梁理论 0.32 mm（-0.8%） |
| 壳悬臂板，端部 5 N | `ansys-apdl-build\template_shell.*` | 0.4900 mm / 0.50 mm（-2.0%） |
| 梁悬臂，端部 100 N | `ansys-apdl-build\template_beam.*` | 0.32046 mm / 0.3206 mm（-0.04%） |
| 静力三个载荷步 | `ansys-solve-segment\static_steps.solve.dat` | 结束时间 1、2、3 |
| 模态 | `ansys-solve-segment\modal.solve.dat` | 一阶 40.93 Hz / 40.76 Hz（+0.4%） |
| 谐响应 | `ansys-solve-segment\harmonic.solve.dat` | 峰值在 40 Hz（一阶 40.9 Hz） |
| 瞬态 | `ansys-solve-segment\transient.solve.dat` | 100 个结果集到 0.1 s |
| 特征值屈曲 | `ansys-solve-segment\buckle.solve.dat` | 259156 N / 欧拉 257000 N（+0.8%） |
| 杀死 / 激活单元 | `ansys-solve-segment\ekill.*` | 最后一步反力 4000 N（平衡） |
| 螺栓预紧 | `ansys-solve-segment\bolt.*` | 支座承担 1000 N |
| 轴对称厚壁圆筒内压 | `ansys-elements\axisym_cylinder.*` | 内壁径向位移 0.009847 mm / Lamé 0.009833 mm（+0.14%） |
| 摩擦接触压紧 | `ansys-contact\contact_pressed.*` | 反力 720000 N（平衡） |
| 绑定接触受拉 | `ansys-contact\contact_bonded.*` | 反力 -360000 N（平衡） |
