# 覆盖率审计（自动生成：tools/coverage.py）

> 日期：2026-10-31。**完整 ≠ 无误**：完整 = 全系统可执行；无误 = L1/L2 对拍全绿（docs/parity-spec.md）。
> 本文件由脚本生成，随引擎推进重新执行更新。

## 漏斗总览（卡牌 375 张）

| 层 | 数量 | 说明 |
|---|---|---|
| 数值表提取 | 375/375 | ✅ 全量 |
| use() 效果 DSL | 368/375 | 308 全解析 + 60 带分支 |
| 系统性未支持 | 4 | 循环放招 0、充能球、姿态、Scry…（需系统级移植） |
| **引擎当前可执行** | **220** | 数值+效果+power 白名单全通过 |

### 引擎拒绝原因分布（Top）

| 数量 | 原因 |
|---|---|
| 25 | 表达式未解析 |
| 25 | 不可打出(-2费) |
| 5 | power 未实现: FocusPower |
| 4 | 伤害目标异常: Unknown |
| 3 | power 未实现: Strength |
| 3 | power 未实现: MantraPower |
| 2 | power 未实现: NoDrawPower |
| 2 | 循环体内 Damage 不支持 |
| 2 | power 未实现: EnergizedPower |
| 2 | power 未实现: IntangiblePlayerPower |

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
| 怪物 | 73/73 ✅ | 47/73（MonsterKind 逐只手写，Act 1+2+3 普通全覆盖） |
| power | 123 条 ✅ | 38 种（PowerKind 枚举，含钩子行为） |
| 遗物 | 186 条 ✅ | 0（战斗内主动遗物待做） |
| 药水 | 43 条 ✅ | 42/43（可执行药水全量；第 43 条为 PotionSlot 占位符——游戏用它表示空栏，引擎用 `Option::None` 等价建模，非待办） |
| 角色 | 4/4 ✅ | 1/4（铁甲，无角色专属机制） |

## 系统层

| 系统 | 状态 |
|---|---|
| 回合机/出牌管线/伤害格挡管线 | ✅ |
| 数值管线（calculateCardDamage 等 3 条，f32 逐位） | ✅ |
| RNG 13 流 + 洗牌管线（黄金向量对拍） | ✅ |
| 诚实型搜索 API（Clone/determinize/public_hash） | ✅ |
| 战斗内选择屏（hand/grid select、Scry、探素、姿态选择） | ✅ |
| 充能球（Defect）/姿态（Watcher） | ✅ |
| 药水系统（使用/丢弃/免死/掉落采样） | ✅ |
| 主动遗物 | ❌ |
| 地图/事件/商店/休息/奖励（M3） | ❌ |
| pyo3 Python 绑定 | ❌ |
| **L1/L2 对拍（无误的最终裁判）** | ❌ 未开始（依赖 bridge） |
