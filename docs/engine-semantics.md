# 引擎语义笔记（反编译勘察）

> 日期：2026-10-30。方法：CFR 0.152 反编译本机 `desktop-1.0.jar` 目标类。
> 反编译产物在 `.tools/decompiled/`（**已 gitignore，勿入库——版权**），仅供本地对照。
> 覆盖类：GameActionManager / AbstractPlayer(部分) / UseCardAction / ScryAction /
> DiscardAtEndOfTurnAction / DamageAction / HandCardSelectScreen / GridCardSelectScreen(签名) /
> CardGroup.shuffle(部分) / Random / AbstractDungeon(RNG 字段)。

## 1. 动作队列 —— 模拟器状态机骨架（GameActionManager）

**四层队列，严格优先级**（getNextAction 依次检查）：

```
actions (FIFO, addToTop/addToBottom)
  → preTurnActions (回合开始前, addToTurnStart 插头)
    → cardQueue (CardQueueItem: card+target+energyOnUse+autoplay)
      → queueMonsters()（每回合一次）
        → monsterQueue (逐怪 takeTurn)
          → turnHasEnded? → 下回合开始序列
```

- `addToTop/addToBottom` 只在 `RoomPhase.COMBAT` 时生效
- phase 状态机：`WAITING_ON_USER ↔ EXECUTING_ACTIONS`（当前 action 多帧 update 直到 isDone）
- 战斗胜利时 `clearPostCombatActions()`：只保留 Heal/GainBlock/UseCard/DAMAGE 类
- 关键静态计数器：`totalDiscardedThisTurn`（遗物 onManualDiscard 判据）、`damageReceivedThisTurn/Combat`、`turn`、`energyGainedThisCombat`
- `incrementDiscard(endOfTurn)`：**手动弃牌 vs 回合结束弃牌走不同钩子**

## 2. 出牌管线（getNextAction → AbstractPlayer.useCard）

```
1. canUse(player, target) 校验（autoplay 卡例外）
2. onPlayCard 钩子风暴：玩家powers → 每只怪powers → 遗物 → 姿态 → blights → 手/弃/抽牌堆每张卡
3. calculateCardDamage(target)（重算目标相关数值）
4. X费卡：能量不足时 energyOnUse = 当前全部能量
5. c.use(player, target)   ← 卡牌效果自己 enqueue（先入队）
6. addToBottom(UseCardAction) ← 钩子+去向结算（后入队）
7. 能量结算：costForTurn<=0 / freeToPlay / autoplay / Corruption(技能) 豁免
8. 手牌无法再出任何牌 → 结束回合按钮发光提示
```

- 目标已死/逃跑时，ENEMY 目标卡**直接消耗（exhaust 特效）且不触发效果**
- `queueExtraCard`（双打/连击类）：复制实例、`purgeOnUse=true`、插入队列第 2 位

## 3. UseCardAction 结算顺序

- **构造时**（入队即触发，非 update 时）：`onUseCard`（玩家 powers、遗物）→ `triggerOnCardPlayed`（手/弃/抽全部卡）→ 怪物 powers `onUseCard`
- update：`onAfterUseCard` → `purgeOnUse` 直接消失 → POWER 类型 `empower`（作为能力打出）→ Strange Spoon 判定（`cardRandomRng.randomBoolean()`）→ 去向：exhaust / rebound回牌库顶 / shuffleBack / returnToHand / 弃牌堆 → `HandCheckAction`

## 4. 回合边界

**回合结束**（`endTurn()` 置 `turnHasEnded`；cardQueue 里的 null 项触发）：

```
callEndOfTurnActions: EOT遗物 → EOT预卡powers → orbs → 手牌 triggerOnEndOfTurnForPlayingCard → stance
DiscardAtEndOfTurnAction:
  - retain/selfRetain 卡 → limbo → RestoreRetainedCardsAction（下回合开始时回手）
  - 无 Runic Pyramid / Equilibrium → 整手牌弃掉（逐张 DiscardAction）
  - ⚠ Collections.shuffle(hand副本) 后逐卡 triggerOnEndOfPlayerTurn
    —— 用的是无种子全局 java.util.Random，仅影响触发顺序（非确定性泄漏点，模拟器需固化并记录）
```

**下回合开始**（怪物全死则跳过）：

```
怪 applyEndOfTurnPowers → 玩家 startOfTurn 遗物/预抽卡/卡/powers/orbs → turn++
→ totalDiscardedThisTurn=0, damageReceivedThisTurn=0, cardsPlayedThisTurn.clear
→ 掉格挡（Barricade/Blur 豁免；Calipers 保留15）→ DrawCardAction(gameHandSize) → EnableEndTurnButton
```

## 5. RNG —— 确定性的核心（AbstractDungeon + Random）

**13 条独立流，全部 `new Random(Settings.seed)` 同种子**，各自带调用计数器（存档系统会保存 counter）：

```
monsterRng / mapRng / eventRng / merchantRng / cardRng / treasureRng / relicRng /
potionRng / monsterHpRng / aiRng / shuffleRng / cardRandomRng / miscRng
```

- 底层：`com.megacrit.cardcrawl.random.Random` 包装 libgdx `RandomXS128`（双 64bit 状态，`getState(0/1)` 可导出快照）+ counter
- **洗牌的真实语义**（CardGroup.shuffle）：每次洗牌从 `shuffleRng.randomLong()` 取一个 long → 种子化新的 `java.util.Random` → `Collections.shuffle`（Fisher-Yates，从尾向头）。→ Rust 复刻 = RandomXS128 + java.util.Random(LCG) + 标准洗牌，完全可行
- `aiRng` 供怪物 AI 随机（对拍时注意调用顺序）
- 已知非确定性泄漏：§4 的无种子 Collections.shuffle（只影响钩子顺序）

## 6. 选择屏契约（协议合法动作的判定依据）

**HandCardSelectScreen.open(msg, amount, anyNumber, canPickZero, forTransform, forUpgrade, upTo)**

- `numCardsToSelect=amount`；确认按钮可用性规则（即合法动作规则）：
  选满 amount 或 `anyNumber && (选≥1 或 canPickZero)` 或 `upTo`
- 卡牌效果从 `selectedCards` 取走结果（`wereCardsRetrieved`）

**GridCardSelectScreen.open** 变体：

```
open(group, numCards, anyNumber, msg)                         // scry 用这个
open(group, numCards, tipMsg, forUpgrade, forTransform, canCancel, forPurge)
open(group, numCards, tipMsg, forUpgrade, forRitual)
openConfirmationGrid(group, tipMsg)
```

**ScryAction**：取抽牌堆**顶** N 张（注意 `group` 列表**尾部 = 堆顶**）→ 未选中者**保持原序**留在堆顶，选中者进弃牌堆；GoldenEye +2；先触发玩家 powers 的 `onScry()`

## 7. 伤害结算钩子顺序（AbstractPlayer.damage 为例）

```
负伤害归零 → Intangible 限 1 → decrementBlock（先扣格挡）
→ onAttackToChangeDamage（自伤时自己的遗物）
→ 攻击方 powers onAttackToChangeDamage
→ 自己的遗物 onAttackedToChangeDamage → 自己的 powers onAttackedToChangeDamage
→ onAttack（遗物/powers）→ onAttacked（powers/遗物，含反伤/荆棘）
→ 最终扣血
```

→ 模拟器实现策略：不逐类复刻钩子，把效果登记为**两层数值管线**（修改器 modifiers + 事件触发 triggers），按上表固定顺序执行。

## 8. 对 neow-sim 的落地清单

1. 状态机 = 四队列 + phase，动作按"原子效果"重写（无帧循环）
2. Rust 实现 `RandomXS128` + `java.util.Random(LCG)` + Fisher-Yates（13 流 + counter）
3. 抽牌堆内部表示：列表尾 = 堆顶
4. 伤害/格挡/能量走"修改器管线"抽象
5. M2 最小集：useCard 管线 + Damage/GainBlock/Draw/Shuffle/DiscardAtEndOfTurn + 回合边界 + 颚虫（含 `rollMove`/`takeTurn`，需再反编译 JawWorm 与 AbstractMonster）
6. 下一步反编译：JawWorm、AbstractMonster（意图机）、DrawCardAction（含游戏手牌上限触发链）、DiscardAction、EnergyPanel
