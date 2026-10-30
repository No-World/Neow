# 对拍规范（Parity Spec）v1

> 核心承诺：**同种子 + 合法动作序列 ⇒ 与真游戏"正常对局"逐点一致**。
> 一致的对象是规则状态，不是表现层；已知漏洞不在此列（见偏差登记表）。

## 1. 承诺定义

| 术语 | 定义 |
|---|---|
| 轨迹 | 每个决策点的 `(动作, 观测哈希, RNG计数器快照)` 序列 |
| 观测哈希 | 对规则状态的全部规则性字段做稳定哈希（见 §4） |
| 合法动作 | 协议 `legal_actions` 输出的动作；利用性序列**不在其中**（见 §6） |
| 正常对局 | 不利用动画时序 / 路径判定 / 存档操作漏洞的对局 |

## 2. 分级

- **L1 战斗级**：单场战斗内重放动作序列比对（M2 完成后即可开跑，先拿 JawWorm×100 种子）
- **L2 整局级**：从 Neow 祝福开始：地图生成、事件、商店、战斗、奖励、遗物流转（M3 后）

## 3. RNG 政策（依据 `tools/audit_determinism.py` 全量审计）

**必须逐位复刻的（已实现 `rng.rs`，黄金向量测试）：**
- 13 条种子流（monster/map/event/merchant/card/treasure/relic/potion/monsterHp/ai/shuffle/cardRandom/misc）
- "种子化洗牌"管线：`new java.util.Random(某流.randomLong()) → Collections.shuffle`（CardGroup、AbstractDungeon、Exordium 等处同模式）

**真游戏自身非确定（豁免登记，对拍跳过）：**
真游戏下列调用走 libgdx 全局无种子随机，**换台机器重开一局结果都不同**，无法也不需复刻；sim 统一改走 `miscRng`（保证 sim 内部可复现）：

| 豁免点 | 位置 | 说明 |
|---|---|---|
| `CardGroup.getRandomCard()` 系 | cards/CardGroup.java:489/506/537 | 无参随机取卡（少数卡牌效果用） |
| `MonsterGroup.getRandomMonster` 两参版 | monsters/MonsterGroup.java:205-228 | 随机敌人（3 参版走传入流，正常复刻） |
| 随机遗物索引 | characters/AbstractPlayer.java:1989 | 个别效果 |
| 其余 gameplay 包内 MathUtils.random | 审计 255 处 | 逐条核过：绝大多数是动画/音效/粒子/对话框随机，不影响规则状态 |

**已知唯一状态级泄漏：**
- `DiscardAtEndOfTurnAction`：`Collections.shuffle(hand副本)`（无种子）仅影响 `triggerOnEndOfPlayerTurn` **触发顺序**。sim 固定为手牌顺序；对拍用例若涉及顺序敏感卡（如 The Bomb 类），单独标注。

## 4. 观测哈希

每个决策点对以下字段做稳定序列化后哈希（版本化，`obs_hash_v1`）：
玩家 hp/max/block/energy/powers；手牌（有序，含升级与临时修改）；抽牌堆（**集合+计数**，顺序不算——它本就隐藏）、弃牌堆、消耗堆；卡组；遗物列表与计数器；药水；怪物 hp/block/powers/意图/moveHistory/nextMove；`turn` 与 GameActionManager 各静态计数器；RNG 13 流的 counter 快照。

## 5. 对拍流程（harness）

```
真游戏(bridge) --固定seed+动作序列--> 黄金轨迹 (action, obs_hash, counters)[]
    sim 重放同序列 ----> 实际轨迹
    逐点比对；首个 mismatch → dump 双方状态全量 diff + counter 对照（定位到具体流）
```
- 黄金轨迹存放 `neow-sim/tests/conformance/`，M2 起每版本回归
- bridge 必须支持：固定 seed 启动、每步导出 counters（诊断模式）

## 6. 偏差登记表（exploit 不复刻）

原则：**模拟器实现"设计意图"，不复刻代码缺陷**。利用性序列不进入合法动作空间；
评测协议同样禁用（与"游戏内形态：无S/L、非穷举、不窥探"同源——AI 也不许用）。

| 编号 | Bug | 现象 | sim 处置 |
|---|---|---|---|
| D-01 | 烟雾弹逃心脏 | 心脏/Boss 战用 Smoke Bomb，`isEscaping` 误判为战斗胜利 | Boss/心脏房 `cannotEscape=true`，药水合法性检查含房间标记 |
| D-02 | 岔路连走两房 | 地图转场动画期间可点击第二个节点 | 地图动作原子化：一步一决策，转场不可插入 |
| D-03 | （待补）社区已知 glitch 清单 | 随 M3 地图/事件实现逐项登记 | 每项：现象 → 判定 → sim 行为 → 对拍豁免说明 |

> 发现新偏差时的流程：先记录（现象+触发序列）→ 判定"bug 还是机制" → 登记或复刻。

## 7. 验收（对应 roadmap P2 / M4）

- [ ] L1：100 种子 × JawWorm 战斗，观测哈希 100% 一致
- [ ] L2：整局黄金轨迹 100% 一致（豁免点不计）
- [ ] 偏差登记表 D-01/D-02 在 sim 中有对应防御测试
