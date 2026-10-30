# 覆盖率审计（自动生成：tools/coverage.py）

> 日期：2026-10-30。**完整 ≠ 无误**：完整 = 全系统可执行；无误 = L1/L2 对拍全绿（docs/parity-spec.md）。
> 本文件由脚本生成，随引擎推进重新执行更新。

## 漏斗总览（卡牌 375 张）

| 层 | 数量 | 说明 |
|---|---|---|
| 数值表提取 | 375/375 | ✅ 全量 |
| use() 效果 DSL | 264/375 | 217 全解析 + 47 带分支 |
| 系统性未支持 | 116 | 循环放招 0、充能球、姿态、Scry…（需系统级移植） |
| **引擎当前可执行** | **129** | 数值+效果+power 白名单全通过 |

### 引擎拒绝原因分布（Top）

| 数量 | 原因 |
|---|---|
| 25 | 不可打出(-2费) |
| 16 | 表达式未解析 |
| 12 | 未支持动作: ChannelAction |
| 6 | 未支持动作: ChangeStanceAction |
| 5 | power 未实现: FocusPower |
| 3 | ExhaustAction 语义待核 |
| 3 | power 未实现: Strength |
| 3 | 伤害目标异常: Unknown |
| 3 | 未支持动作: NotStanceCheckAction |
| 3 | power 未实现: MantraPower |

### 未实现 power 阻塞清单（实现即解锁，按解锁数排序）

| 卡数 | power |
|---|---|
| 5 | FocusPower |
| 3 | Strength |
| 3 | MantraPower |
| 2 | NoDrawPower |
| 2 | EnergizedPower |
| 2 | IntangiblePlayerPower |
| 1 | BerserkPower |
| 1 | BrutalityPower |
| 1 | CorruptionPower |
| 1 | DarkEmbrace |
| 1 | DemonFormPower |
| 1 | DoubleTapPower |

## 其他实体层

| 内容 | 数据提取 | 行为实现 |
|---|---|---|
| 怪物 | 73/73 ✅ | 2/73（颚虫/邪教徒，手写行为） |
| power | 123 条 ✅ | 11 种（Strength/易伤/虚弱/脆弱/Ritual/LoseStrength/敏捷/毒/金属化/再生） |
| 遗物 | 186 条 ✅ | 0（战斗内主动遗物待做） |
| 药水 | 43 条 ✅ | 0 |
| 角色 | 4/4 ✅ | 1/4（铁甲，无角色专属机制） |

## 系统层

| 系统 | 状态 |
|---|---|
| 回合机/出牌管线/伤害格挡管线 | ✅ |
| 数值管线（calculateCardDamage 等 3 条，f32 逐位） | ✅ |
| RNG 13 流 + 洗牌管线（黄金向量对拍） | ✅ |
| 诚实型搜索 API（Clone/determinize/public_hash） | ✅ |
| 战斗内选择屏（hand/grid select、Scry） | ❌ |
| 充能球（Defect）/姿态（Watcher） | ❌ |
| 药水/主动遗物 | ❌ |
| 地图/事件/商店/休息/奖励（M3） | ❌ |
| pyo3 Python 绑定 | ❌ |
| **L1/L2 对拍（无误的最终裁判）** | ❌ 未开始（依赖 bridge） |
