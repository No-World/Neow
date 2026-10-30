# 战斗内操作清单（源码勘察）

> 勘察日期：2026-10-30。来源：本机 `D:\Program Files (x86)\Steam\steamapps\common\SlayTheSpire\desktop-1.0.jar`
> 类名枚举（`unzip -l | grep .class`），共 4872 类。类名可读、混淆轻，逐个语义可用 CFR 反编译补齐。
> 注意：游戏目录里存在 `desktop-1.0.jar改`（历史改版备份），反编译一律以原版 `desktop-1.0.jar` 为准。

## 一、玩家级战斗操作（决策点全集，共 7 类）

战斗中玩家真正能"做"的操作，全部落在以下 7 类上（protocol v0.2 动作集据此定义）：

| # | 操作 | 游戏内载体 | 协议动作 |
|---|---|---|---|
| 1 | 出牌（含目标选择） | 拖牌/点击 | `play(card_index, target)` |
| 2 | 结束回合 | `EndTurnButton` | `end_turn` |
| 3 | 喝药水（部分需目标）/ 倒药水 | `ui/panels/PotionPopUp` | `potion` / `potion_discard` |
| 4 | 使用主动遗物 | `RelicView` | `relic` |
| 5 | 手牌选择屏 | `screens/select/HandCardSelectScreen` + `CardSelectConfirmButton` | `hand_select(indices)` |
| 6 | 牌堆网格选择屏 | `screens/select/GridCardSelectScreen` + `GridSelectConfirmButton` | `grid_select(pile, indices)` |
| 7 | 占卜（置顶 or 弃） | `ScryAction`（utility）→ 网格屏特殊模式 | `scry(discard, top)` |

### 选择屏的典型触发（= 卡牌/规则效果的"二次决策"）

- **hand_select**：回合结束手牌 >10 强制弃牌（`DiscardAtEndOfTurnAction`）、赌博（`GamblingChipAction`）/千算（`CalculatedGambleAction`）、双持（`DualWieldAction`）、Well-Laid Plans 持牌等
- **grid_select**：寻求（Seek，抽牌堆）、挖掘（Exhume，消耗堆）、发现（`DiscoveryAction`）、弃牌堆置顶（`DiscardPileToTopOfDeckAction`）、Codex 等
- **scry**：Watcher 专属；`top_indices` 顺序敏感（置顶次序影响下回合抽牌）
- CommunicationMod 把 5~7 统一抽象成 `CHOOSE`；我们在 `legal_actions` 中展开为具体类型，更适合自回归动作头

## 二、引擎级 action（280 类）

子包分布：

| 子包 | 数量 | 内容 |
|---|---|---|
| unique | 85 | 具体卡牌效果（双持、末日锤、吸血……） |
| common | 61 | 战斗原语 |
| defect | 45 | 充能球系 |
| watcher | 42 | 观息/姿态系 |
| utility | 25 | 出牌/队列/占卜/演出 |
| animations / deprecated / 根级 | 22 | 演出、弃用、队列管理（`GameActionManager`、`ClearCardQueueAction`） |

### common（61，战斗原语，M1-M2 的核心实现范围）

`DamageAction DamageAllEnemiesAction GainBlockAction GainEnergyAction GainEnergyAndEnableControlsAction DrawCardAction FastDrawPileToHandAction DiscardAction DiscardAtEndOfTurnAction DiscardSpecificCardAction ExhaustAction ExhaustSpecificCardAction ShuffleAction EmptyDeckShuffleAction ApplyPowerAction ApplyPowerToRandomEnemyAction RemoveSpecificPowerAction ReducePowerAction HealAction LoseHPAction LosePercentHPAction GainGoldAction InstantKillAction MakeTempCard*（AtBottom/InDiscard/InDrawPile/InHand/InDiscardAndDeck 五件套） PutOnDeckAction PutOnBottomOfDeckAction PlayTopCardAction ModifyBlockAction ModifyDamageAction ReduceCost* Upgrade* RemoveAllBlockAction EscapeAction SpawnMonsterAction ReviveMonsterAction RollMoveAction SetMoveAction MonsterStartTurnAction …`（全表见 jar）

### utility 关键项

`UseCardAction`（出牌入队）· `QueueCardAction / NewQueueCardAction`（打出队列）· `ScryAction` · `ChooseOneColorless` · `DiscardToHandAction / DrawPileToHandAction / ExhaustToHandAction`（回收）

### 语义要点（对模拟器实现至关重要）

1. **动作队列**：`GameActionManager` 是单线程 FIFO 队列，每个 action 多帧 `update()` 完成 →
   模拟器必须按**原子化规则效果**重写（性能目标决定不能复刻帧循环），但语义上要等价于队列逐步结算
2. **RNG 侵入点**：`com/megacrit/cardcrawl/random/Random` + 所有 `*Random*` action（约 10 处）+
   洗牌（`ShuffleAction`）——M4 对拍前必须逆向对齐其种子流与调用顺序
3. **临时牌语义**：`MakeTempCard*` 五件套 = 本场临时牌的完整生命周期
4. **弃牌结算顺序**：`DiscardAtEndOfTurnAction` 在回合结束时才入队 → 与"手牌上限 10"共同构成回合边界语义

## 三、内容规模底数（M3 工程量估算）

| 内容 | 类数 | 备注 |
|---|---|---|
| 卡牌 | 447 | red 75 / green 75 / purple 77 / blue 76 / colorless 39 / curses 14（status 单独包） |
| powers | 162 | 增益/减益效果 |
| 怪物 | 73 | 含Boss与事件怪 |
| 遗物 | 191 | 单类一遗物 |
| 药水 | 44 | |
| 事件 | 56 | 三幕+城市 |
| 地图房间 | 13 + map 8 | |

→ 结论：M2（单场战斗）只涉及 common 原语 + 个位数卡牌；M3 的真实工作量在 447 卡 + 162 power + 191 遗物的**效果实现表**上，必须数据驱动（查表）而非逐类手写行为。

## 四、建议下一步（反编译清单，工具：CFR / Vineflower）

优先级从高到低：

1. `GameActionManager` —— 队列与结算顺序（模拟器状态机的骨架）
2. `AbstractCard` / `AbstractPlayer.useCard` —— 出牌管线、目标校验、费用结算
3. `random/Random` —— 种子算法（M4 对拍的命门）
4. `ScryAction` + `HandCardSelectScreen` + `GridCardSelectScreen` —— 选择屏状态机
5. `DiscardAtEndOfTurnAction` —— 回合边界
6. `AbstractMonster`（`RollMoveAction`/`SetMoveAction`）—— 怪物意图机制
7. `PotionPopUp` / `AbstractRelic` —— 药水与主动遗物入口
