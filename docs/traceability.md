# 链路核对表（Traceability Ledger）

> **工作规则**（响应"一切从源码中来"）：
> 1. 任何行为实现前，先读反编译源码（`.tools/decompiled-full/`，行号指 CFR 输出）；读完立即在本表登记
> 2. 状态：**✅ 已核对**（源码↔实现一致且有测试锁定）/ **⚠️ 部分**（注明缺口）/ **❌ 未实现**
> 3. 每轮"想当然"被抓（本会话已有 4 次：A6 半血、addToRandomSpot、Acrobatics 抽数、虚弱挂方）都是没走第 1 条
> 4. 最终裁判是 L1/L2 对拍（parity-spec）；本表是过程台账

## 1. RNG 链路

| 环节 | 源码位置 | 实现 | 状态 |
|---|---|---|---|
| 13 条流同种子初始化 | AbstractDungeon#481-492 | `rng.rs RngStreams::new` | ✅ |
| RandomXS128 算法（murmur 种子/xorshift128+/拒绝采样/float 字面量陷阱） | com/badlogic/gdx/math/RandomXS128 | `rng.rs Xs128` | ✅ 黄金向量逐位 |
| STS 包装层 counter/save 快照 | random/Random.java | `rng.rs StsRandom`（counter=88 断言） | ✅ |
| java.util.Random + Collections.shuffle | CardGroup#560-568 | `rng.rs JavaRandom/java_shuffle` | ✅ 黄金向量 |
| addToRandomSpot：`cardRandomRng.random(size-1)`，**索引永不含堆顶位** | CardGroup#446-451 | `Q::MakeInDraw` | ✅（曾误标"全局随机"，源码纠正） |
| getRandomMonster(exception, aliveOnly, rng) | MonsterGroup#197+ | `Q::DamageRandom` | ✅ |
| getRandomCard() 无参版（MathUtils 全局） | CardGroup#489/506/537 | — | ❌ 真游戏自身非确定（parity-spec 豁免） |

## 2. 出牌链路（AbstractPlayer.useCard#1426 + GameActionManager.getNextAction）

| 环节 | 源码 | 实现 | 状态 |
|---|---|---|---|
| calculateCardDamage 先于 use() | useCard#1428 | play_card eval 序 | ✅ |
| use() 效果先入队、UseCardAction 后入队 | useCard#1435-1436 | apply_effect→Q 顺序 | ✅ |
| onPlayCard 钩子风暴（powers/relics/全牌堆卡） | GameActionManager | — | ⚠️ 无触发型卡，暂空实现 |
| 能量结算 + 豁免（costForTurn≤0/freeToPlay/Corruption） | useCard#1449-1451 | `cost_pay` | ⚠️ 豁免分支未实现 |
| exhaust 旗标去向 | UseCardAction.update | exhaust 分支 | ✅ |
| 目标已死：ENEMY 卡消耗不发效果 | GameActionManager | alive 检查 | ✅ |
| canUse 完整条件（能耗/不可打状态） | AbstractCard.canUse | `can_play` | ⚠️ 仅能量+白名单 |

## 3. 数值管线（f32 + MathUtils.floor 逐位）

| 环节 | 源码 | 实现 | 状态 |
|---|---|---|---|
| 玩家卡伤害（单体）：Give→姿态→Receive→FinalGive→FinalReceive→clamp→floor | AbstractCard#2298+ | `damage.rs calc_player_card_damage` | ✅ |
| multiDamage（AoE 各目标独立 receive） | AbstractCard#2330+ | `apply_effect DamageAll` | ✅ |
| 格挡：modifyBlock→modifyBlockLast→clamp→floor | AbstractCard#2259+ | `calc_player_card_block` | ✅ |
| 怪物→玩家：Give→Receive→姿态→FinalGive→FinalReceive→floor→clamp | DamageInfo#31+（!isPlayer 分支） | `calc_monster_damage` | ✅ |
| IntangiblePlayer：FinalReceive 处 dmg>1→1（无类型过滤） | IntangiblePlayerPower | `at_damage_final_receive` | ✅ |
| 受伤钩子全序（onAttackToChangeDamage→…→onAttacked） | AbstractPlayer.damage / AbstractMonster.damage | — | ⚠️ M2 无改伤类 power，管线位保留 |

## 4. Power 链路（逐个源码核对）

| Power | 源码要点 | 状态 |
|---|---|---|
| Strength | atDamageGive +flat（NORMAL 限定）、±999、归零移除、可负 | ✅ |
| Vulnerable | atDamageReceive ×1.5；justApplied=(turnHasEnded&&怪物源)；轮末 -1 | ✅ |
| Weakened | atDamageGive ×0.75，**挂在攻击方**；怪物源 skipFirst | ✅（曾想当然挂错，源码纠正） |
| Frail | modifyBlock ×0.75；怪物源 skipFirst | ✅ |
| Dexterity | modifyBlock +flat、±999、可负 | ✅ |
| Poison | 怪 atEndOfTurn：失 HP=层数（绕格挡）后 -1；封顶 9999 | ✅ tick 先于玩家轮末衰减（MonsterGroup 序） |
| Metallicize | atEndOfTurnPreEndTurnCards→GainBlock；调用点 AbstractRoom#539 | ✅（顺序本轮补证） |
| Regeneration | atEndOfTurn 治疗 amount 后 -1 层 | ✅ |
| LoseStrength | atEndOfTurn(isPlayer)：力量 -amount 后自移除 | ✅ |
| Demon Form / Brutality | atStartOfTurnPostDraw：+STR / 抽n失n血 | ✅ |
| Berserk | atStartOfTurn：+amount 能量 | ✅ |
| Barricade | GameActionManager loseBlock 条件豁免 | ✅ |
| Thorns | onAttacked→DamageAction(THORNS)，不触发对方荆棘 | ✅ |
| Combust | atEndOfTurn：LoseHP(**hpLoss=1**，Combust.java: `new CombustPower(p,1,magic)`) + 全场 THORNS 直伤 | ✅（hpLoss 本轮补证） |
| Rage | onUseCard(ATTACK)→+amount 格挡；EOT 移除 | ✅ |
| IntangiblePlayer | 见 §3；轮末衰减 | ✅ |
| Curl Up | onAttacked：**仅非致死** NORMAL 伤害→+格挡后移除 | ✅ |
| Spore Cloud | onDeath→对玩家易伤（过 Artifact 判定） | ✅ |
| **Artifact 拦截** | **ApplyPowerAction#125：按 powerToApply.type==DEBUFF 判定** | ✅（曾硬编码四种，源码纠正为 type 判定） |
| ApplyPowerAction 堆叠语义 | stackPower 逐类 | ⚠️ 已实现种类按源码，全类堆叠未逐个核 |

## 5. 回合边界链路

| 环节 | 源码 | 状态 |
|---|---|---|
| endTurn→turnHasEnded | GameActionManager#198 | ✅ |
| EOT 玩家 powers 序：Metallicize(PreCards)→LoseStrength/Regen(atEndOfTurn) | AbstractRoom#539 + callEndOfTurnActions | ✅（顺序本轮核） |
| DiscardAtEndOfTurn：retain 暂存、无 Pyramid/Equilibrium 全弃 | 源码全文已读 | ✅（retain 类卡未实现，无来源） |
| EOT 无种子 shuffle（触发顺序不定） | DiscardAtEndOfTurnAction | ⚠️ 固定为手牌序（parity-spec 泄漏登记） |
| **怪物掉格挡 = 怪物回合开始统一一次** | MonsterStartTurnAction ← AbstractRoom#406 单次入队 | ✅（曾逐怪实现，源码纠正） |
| applyEndOfTurnPowers 三段（怪 atEndOfTurn→玩家衰减→怪衰减+Ritual） | MonsterGroup | ✅ |
| 下回合：loseBlock→DrawCardAction(5)→EnableEndTurn | GameActionManager | ✅ |
| 能量重置调用点（EnergyPanel） | 未定位 | ⚠️ L1 时用 counter 对齐 |

## 6. 抽牌链路

| 环节 | 源码 | 状态 |
|---|---|---|
| **hand==10 → 全停；超额截断（多余消失不弃牌）** | DrawCardAction#22-40 | ✅（曾实现为"溢出弃牌"，源码纠正） |
| 牌堆不足先洗回（shuffleRng，消耗序等价） | DrawCardAction + EmptyDeckShuffleAction | ✅ |
| No Draw power 阻断全部抽牌 | DrawCardAction#16 | ❌ NoDraw 未实现（2 卡受阻塞，已登记） |

## 7. 弃牌链路（DiscardAction 全分支）✅

手牌≤amount 自动全弃 / isRandom 无屏（getRandomCard(cardRandomRng)）/ amount<0 任意张 / 恰好 N 张选择屏（选择序入弃堆）。

## 8. 占卜链路（ScryAction）✅

顶 N（`group` 尾=顶）/ 选中入弃 / 其余原序置顶 / 空堆无事发生。GoldenEye +2 未实现（遗物层）。

## 9. 临时牌链路

| 动作 | 源码 | 状态 |
|---|---|---|
| MakeTempCardInHand（手满溢出弃堆） | update#120+ | ✅ |
| MakeTempCardInDiscard | update（numCards<6 逐张） | ✅ |
| MakeTempCardInDrawPile 随机位 | addToRandomSpot | ✅ |
| MakeTempCardInDrawPile 顶/底/autoPosition 变体 | 构造器 5 参 | ⚠️ 仅"顶"，底/自动未细分 |

## 10. 怪物链路

| 环节 | 状态 |
|---|---|
| JawWorm（数值/决策树/HP掷值/takeTurn） | ✅ 逐行 |
| Cultist（含 Ritual skipFirst、A17+1） | ✅ |
| rollMove 恒消耗 aiRng.random(99) | ✅ |
| 死亡两档判定（isDying/isDead） | ⚠️ 单帧简化（无死亡动画窗口），L1 校验 |
| 其余 71 怪 | ❌ 数据全量，行为待逐怪接入 |
| Exordium Wildlife/Thugs「候选全构造」（bottomWildlife/bottomHumanoid：5 只候选全部 spawn 占 monsterHpRng，miscRng 选 2/1） | ✅（实测 PMCWKZEF floor11 monsterHp=7=6 构造+1 CurlUp） |
| 大史莱姆分裂（case3 无 RollMoveAction；4 参构造器 newHp=currentHealth 不掷 HP；SpawnMonsterAction→init→rollMove×2；按 x 位序 [M左,L尸,M右] 包夹插入） | ✅（floor14：monsterHp=1、ai=4、target 下标逐位对上） |
| FungiBeast/SlaverBlue/SlaverRed/Sentry spawn 字段与 take_turn 取用槽位对齐 | ✅（曾错存 chomp/thrash，bite 打 0） |
| Guardian setHp(240)（1 参 setHp 仍耗 monsterHpRng.nextPick 1 次）；战斗继承全局流 | ✅ |
| Guardian 30 伤害阈值模式切换：确定性招式链（Charge→Fierce→Vent→Whirl 循环，takeTurn **无 RollMoveAction**，setMove 链式指定下招）；damage() 累计实际掉血（含 THORNS，如 Mercury）≥ 阈值 → ChangeState(DEFENSIVE) 排底（+20 格挡、阈值 +10、Close Up=Sharp Hide 3）；Twin Slam 双击后 Offensive 回归（清格挡/重挂计数/去刺） | ✅（PMCWKZEF floor16 Boss 战全程零分歧）
| Sharp Hide（`SharpHidePower.onUseCard`）：**每张攻击卡**触发 1 次 THORNS 伤；技能/能力牌不触发；**不受击触发** | ✅ 独立 `PowerKind::SharpHide`（曾误用 Thorns＝每次受击反弹，多段攻击卡会多掉血、Mercury 之类的伤害也会误触发） |

## 10.9 第二条真实轨迹（A20 战士 TRACE02A，traces/a20_trace02a.jsonl）

| 环节 | 状态 |
|---|---|
| 录制：NeowBridge 实机 113 步（floor 7 阵亡）；record_trace.py 修 legal_actions 传参 + cheat_draw_order | ✅ |
| 回放工具 parity_replay.py 支持涅奥多动作编排（event→grid_select→map） | ✅ |
| 起始牌组 A10+ 的 AscendersBane 在**最前**（穷举排列反解洗牌结果唯一匹配） | ✅（曾 push 到末尾） |
| 涅奥「升级一张牌」走 `can_upgrade`（诅咒/状态不可升级） | ✅（曾升级 AscendersBane） |
| **虚无（ethereal）**：回合末先消耗进消耗堆，再弃其余手牌 | ✅（曾完全未实现 → A20 每回合多留 1 张） |
| 商店遗物 `returnEndRandomRelicKey`：从预洗池**末尾**取，0 次 relicRng | ✅（曾每次掷 1 次，3 个槽多 3 次） |
| Havoc（PlayTopCardAction）：先 getRandomMonster(null,true,cardRandomRng) 再打顶牌 | ✅ 掷点已补；⚠️ 顶牌的完整出牌管线（目标/多段）待接 |
| AcidSlime_M 意图（move1=ATTACK_DEBUFF）与伤害槽（chomp/thrash_block） | ✅ |

## 10.8 Boss 特殊机制（Act1-4 逐行核对）

| Boss | 机制 | 状态 |
|---|---|---|
| Hexaghost | Activate→Divider（玩家 HP/12+1 ×6）→ 六球循环（Sear/Tackle/…）→ Inferno（升级全部灼伤 + 3 张升级灼伤入弃堆） | ✅ 核对无改动 |
| SlimeBoss | HP **A9** 150/140；Slam 与 tackle **A4**；Goop A19+ 5 张否则 3；半血定招分裂 → SpikeSlime_L(左)+AcidSlime_L(右) 继承当前 HP；无 RollMoveAction | ✅（曾 4 处错值 + Slam 0 伤 + 缺分裂） |
| BronzeAutomaton | numTurns==4 出 Hyper Beam 并清零；A19+ 光束后接强化而非眩晕；case 4 召唤 2 铜球（各 rollMove） | ✅ 核对无改动 |
| TheCollector | turnsTaken 显式计数（≥3 → MEGA 一次）；isMinionDead 只看火把头槽；复活占原位 | ✅（曾用 history.len 早一回合、minion 判定过宽） |
| Champ | numTurns/forgeTimes/thresholdReached；爆怒清减益+力量×3；处决 ×2；A4/A9/A19 数值 | ✅ 核对无改动 |
| AwakenedOne | 半死转阶段（cannotLose 恒真）→ REBIRTH 满血 320/300、清 DEBUFF、首招 Dark Echo；Regenerate **atEndOfTurn 且半死不回血** | ✅（曾回血时序错+半死也回血+不清减益） |
| Donu / Deca | 交替 攻击/增益；Deca.CIRCLE 全体 +16 格挡（A19+ 加 3 护甲）；Donu.CIRCLE 全体 +3 力量 | ✅（曾漏 16 格挡） |
| TimeEater | 半血一次 Haste；getMove 递归重掷；**TimeWarp 12 张 → 结束回合 + 全体 +2 力量** | ✅（曾漏 +2 力量） |
| CorruptHeart | Debilitate 五连状态；增益轮转 神器2→亡律+1→苦痛之刺→力量+10→力量+50；buffCount 执行计数 | ✅（曾差一） |
| SpireShield / SpireSpear | moveCount%3 轮转 + randomBoolean 分支 | ✅ 核对无改动 |

| 系统性 | 状态 |
|---|---|
| **所有被召唤物**：`SpawnMonsterAction → init() → rollMove()` ⇒ 每个召唤物消耗 1 次 aiRng（Automaton 铜球 / Collector 火把头 / Gremlin Leader 小鬼 / Reptomancer 匕首） | ✅（曾硬编码首招不掷 → 召唤后整条 aiRng 流错位） |
| ⚠️ 召唤物「按 x 位序插入」尚未建模（引擎用槽位/追加）—— 只影响多体战目标下标，不影响 RNG 流 | ⚠️ 待 Act2/3 真实轨迹 |

## 10.6 怪物构造掷点审计（全 64 类，tools/audit_monster_rolls.py）

| 规律 | 状态 |
|---|---|
| `setHp(a,b)` → monsterHpRng 1 次 | ✅ 全量核对 |
| **`setHp(int)`（单参）= setHp(hp,hp) = nextInt(1) → 仍耗 1 次** | ✅（17 类曾漏掷，已补；Guardian 首例） |
| `super(..., monsterHpRng.random(a,b), ...)` 也耗 1 次（且在 setHp 之前） | ✅（Taskmaster/BronzeOrb/OrbWalker/TorchHead 曾少掷 1 次） |
| super 常量（Maw/Transient/SphericGuardian）→ 0 次 | ✅ |
| 进阶分支（if/else）只执行其一 → setHp 计 1 次 | ✅ |

## 10.7 遭遇构造链审计（75 个遭遇，tools/audit_encounter_runtime.py + examples/enc_dump.rs）

| 环节 | 状态 |
|---|---|
| misc 消耗次数 + monsterHp 消耗次数 + 怪物序列三项对拍 | ✅ 30 个主流遭遇运行期零差异 |
| 「候选全构造」：Exordium Wildlife/Thugs 未选中候选也掷 HP | ✅（Wildlife 6 掷 / Thugs 7 掷） |
| 「无放回抽取」：Gremlin Gang(4)/Lots of Slimes(5)/Shapes(3、4) | ✅ |
| Gremlin Leader = 2× spawnGremlin（各 1 次 misc，独立全池）+ 本体 | ✅（曾缺分支 → 退化为单只 JawWorm） |
| Chosen and Byrds = **[Byrd, Chosen]** 两只 | ✅（曾误写 3 只） |
| 11 个缺失遭遇分支（4 Byrds/Sentry and Sphere/Snecko and Mystics/3 Cultists/2 Thieves/Slavers/Masked Bandits/Shelled Parasite and Fungi/Jaw Worm Horde/Colosseum Nobs/Slavers/Flame Bruiser 1-2 Orb） | ✅（其中 6 个在 Act2/3 池中可达，此前全退化单只 JawWorm） |
| JawWorm hardMode（Jaw Worm Horde）：firstMove=false + 战前叠 Strength/Block | ✅ |

## 10.5 遗物/药水/奖励链路（PMCWKZEF 全程回放新增）

| 环节 | 状态 |
|---|---|
| 遗物预洗池 = `Collections.shuffle(pool, Random(relicRng.randomLong()))` ×5，池序 = **Java HashMap 迭代序**（sharedRelics+redRelics） | ✅（javagen/HashMapOrder 实测 = Python 模拟 138/138、11/11；RELIC_POOL_ITER_ORDER 生成入源） |
| 精英 tier 阈值：relicRng.random(0,99) <50 COMMON / >82 RARE（**精英房 82，普通房 85**） | ✅（曾 85，floor7 分叉） |
| pop_pooled_relic：从预洗池 remove(0)，运行时 0 次 relicRng | ✅ |
| 奖励药水 = addPotionToRewards：40%+pity 判定 → returnRandomPotion(false)：tier(65/25/10) → **稀有度不符重抽**（Fruit Juice 跳过）；实测 floor10 消耗 1+1+4=6 | ✅ |
| Exordium Wildlife/Thugs 遭遇（见 §10） | ✅ |
| 战斗内 monsterHp 消耗：怪物 ctor（setHp/bite）+ prebattle CurlUp（虱）+「候选全构造」未选中者 | ✅ |
| per-floor 重播种 5 流（monsterHp/ai/shuffle/cardRandom/misc）；**monster/map/relic/event 等为局级全局流** | ✅ |

## 11. 进阶与角色

| 环节 | 状态 |
|---|---|
| A6 起始 90% HP（round(max×0.9)） | ✅（曾记忆 50%，源码纠正） |
| A10 进阶之灾入组 / A11 药水-1 / A13 Boss 金×0.75 / A5 战后治疗 75% | ✅登记；后三者 M3 层实现 |
| 起手卡组（getStartingDeck）/ HP（CharSelectInfo） | ✅ |
| 全解锁（无过滤） | ✅ 默认 |

## 12. 数据表

| 内容 | 状态 |
|---|---|
| 375 卡数值（cards.json） | ⚠️ 全量提取+抽查通过，批量校验待 M4 |
| 效果 DSL | ⚠️ 同上（洁净检查后 129 卡可执行） |
| 73 怪数值 / 123 power / 186 遗物 / 44 药水 | ⚠️ 提取全量，行为按需接入 |

---

**统计**：✅ 100 项 / ⚠️ 16 项 / ❌ 6 项（2026-11-06 A20 第二条轨迹后更新）
