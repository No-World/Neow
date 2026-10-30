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
5. M2 最小集：useCard 管线 + Damage/GainBlock/Draw/Shuffle/DiscardAtEndOfTurn + 回合边界 + 颚虫（完整规格见 §9，已勘察完毕）
6. 下一步反编译：DrawCardAction（含手牌上限触发链）、DiscardAction、EnergyPanel、（可选）Strike/Defend/Bash、DamageInfo

## 9. 怪物侧（AbstractMonster / MonsterGroup / JawWorm）

### 意图机生命周期

```
战斗开始: init() → rollMove() = getMove(aiRng.random(99)) → setMove（记入 moveHistory）
显示: createIntent()（从 move 刷新，意图伤害走 calculateDamage 管线）
行动: takeTurn() 按招式 enqueue 效果，末尾必带 RollMoveAction
         → 下回合意图在“本回合行动结束时”掷出
决策依据: moveHistory 的 lastMove / lastTwoMoves / lastMoveBefore
```

**随机源确认：意图选择全部走 `aiRng`**（主掷 `random(99)` + 招内 `randomBoolean(p)=nextFloat()<p`），完全可复刻。

### 数值管线（意图伤害 calculateDamage）

```
base → (无尽blight) → 怪powers atDamageGive → 玩家powers atDamageReceive
     → stance atDamageReceive → BackAttack×1.5(Surrounded背刺) 
     → 怪powers atDamageFinalGive → 玩家powers atDamageFinalReceive → floor
```

- HP 掷值：`setHp(min,max) = monsterHpRng.random(min,max)`
- `applyPowers()`：每条 damage 过管线 + BackAttack 修正

### 怪物受伤钩子顺序（damage()）

```
IntangiblePlayer→1 → decrementBlock → 玩家遗物 onAttackToChangeDamage
→ 攻方powers onAttackToChangeDamage → 自身powers onAttackedToChangeDamage
→ 玩家遗物 onAttack → 自身powers wasHPLost → 攻方powers onAttack
→ 自身powers onAttacked → 扣血
死亡: 自身powers onDeath → 玩家遗物 onMonsterDeath → 基本全灭时 cleanCardQueue+清limbo
```

### 怪物回合边界（MonsterGroup）

- `applyPreTurnLogic`：怪掉格挡（无壁垒）+ startOfTurnPowers —— 发生在**怪物行动前**
- `queueMonsters`：所有非死/逃怪入 monsterQueue 逐个 takeTurn
- `applyEndOfTurnPowers`：怪 applyEndOfTurnTriggers + 双方 powers atEndOfRound
- 死亡状态机：isDying(动画中)→isDead；`areMonstersBasicallyDead`(isDying||isEscaping) vs `areMonstersDead`(isDead||escaped)
- `getRandomMonster(exception, aliveOnly, rng)`：3参版走传入 RNG；2参版用 MathUtils.random（全局非种子）——调用方决定确定性

### JawWorm 完整规格（M2 对手）

| 招式 | byte | 意图 | 效果 | A2+ | A17+ |
|---|---|---|---|---|---|
| 咬噬 Chomp | 1 | ATTACK | 11 伤 | 12 | 12 |
| 低吼 Bellow | 2 | DEFEND_BUFF | +3力 +6格挡 | +4力 | +5力 +9格挡 |
| 猛击 Thrash | 3 | ATTACK_DEFEND | 7伤 +5格挡 | 同 | 同 |

HP：40-44（A7+：42-46，monsterHpRng）；hardMode 变体（M2 用不到）：firstMove=false + 战前 +力+格挡

决策树（getMove，num = aiRng.random(99)）：

```
首招恒为 Chomp（firstMove）
num<25 (25%): 上一招Chomp? → randomBoolean(0.5625)? Bellow : Thrash；否则 Chomp
25≤num<55 (30%): 连续两招Thrash? → randomBoolean(0.357)? Chomp : Bellow；否则 Thrash
num≥55 (45%): 上一招Bellow? → randomBoolean(0.416)? Chomp : Thrash；否则 Bellow
```

### M2 状态小结

一场 JawWorm 战斗 = 出牌管线 + 三招怪物 + 四条 RNG 流（aiRng / monsterHpRng / shuffleRng / cardRng）+ 回合边界。**语义已齐，可以开写 Rust。**
