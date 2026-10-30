# 译名对照表（Glossary）

> 权威来源：游戏官方简体中文本地化（`neow-sim/data/zhs/*.json`，与游戏 jar 内 ZHS 资源一致）。
> 规范：
> 1. **文档与交流一律用官方译名**，首次出现时括注英文/ID；代码、协议字段、JSON 键保持英文不动。
> 2. 译名有争议时以 `data/zhs/` 为准，新增条目先查文件再入表。
> 3. 本表只收**机制与系统级**术语；卡牌/遗物/药水/怪物等实体名走数据文件，不入表。

## 一、系统与流程

| 英文/ID | 官方译名 | 备注 |
|---|---|---|
| Ascension (A) | **进阶**（进阶等级/进阶模式） | ❌ 不是"升天/登塔等级"；A20 = 进阶 20 |
| Act | **幕**（官方 UI 作"第 X 阶段"） | 社区通称"幕"，文档统一用「第X幕」 |
| Exordium | 第一幕「入口」 | |
| The City | 第二幕「城市」 | |
| The Beyond | 第三幕「深处」 | |
| The Ending | 第四幕「终局」 | 需三钥匙（红宝石/绿宝石/蓝宝石）进入 |
| Floor | 楼层 | 协议字段 `floor` |
| Seed | 种子 | 协议字段 `seed`；种子代码 `seed_code` |
| Daily Run | 每日挑战 | |
| Endless | 无尽模式 | |
| Save Slot | 存档位 | 存档按角色分文件（`IRONCLAD.autosave` 等），非按位编号 |
| Run | 对局 / 一局 | 项目内常说"跑局"（自动跑完整局） |
| Win / Victory | 通关 / 胜利 | 击杀史莱姆首领（Act3 Boss） |
| Heart Kill | 击杀心脏 | 终局的觉醒者心脏；对外成绩的核心指标 |

## 二、角色

| 英文/ID | 官方译名 | 协议 `character` 值 |
|---|---|---|
| Ironclad | **铁甲战士** | `IRONCLAD` |
| The Silent | **静默猎手** | `THE_SILENT` |
| Defect | **故障机器人** | `DEFECT` |
| Watcher | **观者** | `WATCHER` |
| Neow | 涅奥 | 开局赐福的事件神明，本项目守护神 :) |

## 三、卡牌

| 英文/ID | 官方译名 | 备注 |
|---|---|---|
| Card Type: ATTACK / SKILL / POWER | 攻击牌 / 技能牌 / 能力牌 | 协议 `type` 保持英文枚举 |
| STATUS / CURSE | 状态牌 / 诅咒牌 | |
| Rarity: COMMON/UNCOMMON/RARE | 普通/罕见/稀有 | BASIC 基础、SPECIAL 特殊 |
| Upgrade | **升级** | ❌ 不叫"强化"；协议 `upgraded` = 升级次数 |
| Transform | 变化 | |
| Exhaust | **消耗** | 进消耗堆，战斗内移除 |
| Ethereal | **虚无** | 回合结束在手则消耗 |
| Innate | **固有** | 开局在手 |
| Retain | **保留** | 回合结束不弃 |
| Unplayable | 不能被打出 | |
| Cost / X-Cost | 耗能 / X 耗能 | 协议 `cost`；-1 = X 耗能，-2 = 不能被打出 |
| Magic Number | 魔法数字 | 协议 `magic_number`，卡牌效果数值 |
| Target: ENEMY/SELF/ALL/... | 目标：敌方/自身/全体… | 协议 `target_type` 保持英文小写 |

## 四、牌堆与区域

| 英文/ID | 官方译名 | 协议字段 |
|---|---|---|
| Master Deck | **牌组** | `master_deck` |
| Hand | 手牌 | `hand` |
| Draw Pile | **抽牌堆** | `draw_pile`（观测仅暴露集合，顺序隐藏） |
| Discard Pile | **弃牌堆** | `discard_pile` |
| Exhaust Pile | **消耗堆** | `exhaust_pile` |
| Limbo | 缓冲区（limbo） | 出牌结算中的临时区 |

## 五、战斗机制

| 英文/ID | 官方译名 | 备注 |
|---|---|---|
| Energy | 能量 | `energy` / `energy_cap` |
| Block | **格挡** | |
| Strength / Dexterity | 力量 / 敏捷 | |
| Vulnerable / Weak / Frail | **易伤 / 虚弱 / 脆弱** | |
| Poison | 中毒 | |
| Intent | **意图** | 怪物下一回合动作；Runic Dome 下隐藏 |
| Power | 能力（战斗中的增益/减益实体） | 与卡牌"能力牌"区分：这里是 buffs/debuffs |
| Artifact | 人工制品（buff） | 注意与 Relic 区分 |
| Thorns | 荆棘 | |
| Shiv | 小刀 | |
| Play / End Turn | 出牌 / 结束回合 | |

## 六、角色专属机制

| 英文/ID | 官方译名 | 角色 |
|---|---|---|
| Orb | **充能球** | 故障机器人 |
| Channel / Evoke | **生成 / 激发** | 充能球；❌ Evoke 不叫"引发/释放" |
| Focus | 集中 | |
| Lightning/Frost/Dark/Plasma | 闪电/冰霜/黑暗/等离子 | |
| Stance | **姿态** | 观者 |
| Calm / Wrath / Divinity | 平静 / 愤怒 / 神格 | |
| Mantra | 真言 | |
| Scry | **预见** | ❌ 不是"占卜/占卜术" |
| Vigor | 活力 | |

## 七、地图与决策面

| 英文/ID | 官方译名 | 协议 `screen.type` / 动作 |
|---|---|---|
| Map / Node | 地图 / 节点 | `map`；节点类型 MONSTER/ELITE/REST/SHOP/TREASURE/EVENT/BOSS |
| Elite | **精英** | |
| Boss | 首领（Boss） | |
| Rest Site / Sleep | **休息处 / 睡眠**（回复） | `rest`，`sleep` |
| Smith | **锻造**（升级一张牌） | `smith` |
| Dig | 挖掘（铲子遗物） | `dig` |
| Lift | 举重（Girya 遗物） | `lift` |
| Recall | 回忆（钝化钥匙） | `recall` |
| Toke | 焚牌（永久删除） | `toke` |
| Shop / Merchant | 商店 / 商人 | `shop` / `shop_enter` |
| Card Removal | **删卡服务** | `shop_card_removal` |
| Event | 事件 | `event` |
| Chest / Treasure | 宝箱 / 宝藏房间 | `chest_open` |
| Reward | 奖励 | `reward` / `card_pick` / `card_bowl` / `skip` |
| Singing Bowl | 唱歌碗（+2 最大生命替代拿卡） | |
| Boss Relic | 首领遗物三选一 | `relic`（Boss 战后） |
| Hand Select Screen | 手牌选择屏 | `hand_select` |
| Grid Select Screen | 牌堆网格选择屏 | `grid_select` |
| Game Over / Death / Victory | 游戏结束 / 死亡 / 胜利结算 | `game_over` |

## 八、遗物 / 药水 / 荒疫

| 英文/ID | 官方译名 | 备注 |
|---|---|---|
| Relic | **遗物** | 协议 `relics`；Tier: 普通/罕见/稀有/首领/商店/特殊 |
| Counter | 计数器 | 遗物上的计数 |
| Potion | 药水 | 3 个药水栏位（Potion Slot） |
| Throw Potion | 投掷药水（需选目标） | `target_required` |
| Blight | **荒疫** | 无尽模式的负面遗物 |

## 九、项目术语（自定，非游戏官方）

| 术语 | 英文 | 含义 |
|---|---|---|
| 统一 gym 协议 | unified protocol | `docs/protocol.md`，三子项目唯一接口真相源 |
| 观测 | observation | 决策点可见的全部合法信息（JSON） |
| 合法动作 | legal actions | 引擎枚举的当前决策点全部正常操作 |
| 决策点 | decision point | 一次 `(obs, legal, r, done, info)` 五元组 |
| 桥接 / 模组层 | bridge | `neow-bridge`：真游戏 ↔ 外部协议 |
| 对拍 | parity check / diff testing | 同种子同动作序列，模拟器 vs 真游戏逐状态比对 |
| 黄金用例 | golden case / conformance | 对拍用的"种子+动作序列→期望观测"固化用例 |
| 信念（状态） | belief | 隐藏信息（抽牌堆顺序、怪物 RNG）的概率表示 |
| 决定性化 | determinization | PIMC：采样一个可能世界后确定性搜索 |
| 跑局 | rollout | 自动完整进行一局 |
| 游戏内形态 | in-game form | 公平玩家约束：无 S/L、非穷举、不窥探 |
| 模仿学习 / IL | imitation learning | 从 sts_lightspeed 对局蒸馏 |
| 去动画提速 | de-animation | FAST_MODE 等加速执行 |
| 隔离运行时 | isolated runtime | `neow-bridge/runtime/`：自动化专用 cwd，存档/偏好与玩家完全隔离 |

## 十、勘误记录（曾经用错 → 更正）

| ❌ 曾用 | ✅ 正确 | 出现处 |
|---|---|---|
| 占卜 | **预见**（Scry） | protocol.md v0.2/v0.3 → 已更正 |
| 升天等级 | **进阶**（等级） | 交流用语 |
| 引发 | **激发**（Evoke） | |
| 强化 | **升级**（Upgrade） | |
| 抽牌库/卡组 | **抽牌堆 / 牌组** | |
