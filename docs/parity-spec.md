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

---

## 7. 实测基线（2026-04，seed=PMCWKZEF）

### 7.1 对拍工具链

旧版 `parity_check.py` 只重放动作、断言「不抛异常」，**不比对任何状态**——
「8 条轨迹 1062 步全 PASS」的实际含义仅为「引擎在 1062 个动作下不崩溃」。
已替换为真 diff 工具链：

| 工具 | 作用 |
|---|---|
| `neow-sim/tests/conformance/record_trace.py` | 录制真游戏轨迹，每步附 `get_state` 完整快照（含 13 条 RNG 计数器） |
| `neow-sim/tests/conformance/parity_diff.py` | 逐字段 diff（hp/energy/hand/monsters/piles/relics/rng），差异即 FAIL |
| `GymEnv.state_json()` | 引擎侧结构化状态导出（字段名与 bridge `Obs` 对齐） |
| `MonsterKind::id()` | 怪物 id 与 bridge `monsterId` 逐字节对齐 |

### 7.2 已验证对齐项

基线：`PMCWKZEF`（seed=1585262650280）/ A0 / Ironclad / floor 1 turn 1。
证据来自 bridge `get_state` 与 `.tools/javagen/{MapDump,NeowDump,EncDump}.java`
直接调用真游戏 jar 的产物。

**13 条 RNG 计数器（进入首场战斗时）全部精确一致：**

| 流 | sim | 真游戏 |
|---|---|---|
| monster | 38 | 38 |
| map | 95 | 95 |
| relic | 5 | 5 |
| monster_hp | 6 | 6 |
| ai | 2 | 2 |
| shuffle | 1 | 1 |
| misc | 2 | 2 |
| event / merchant / card / treasure / potion / card_random | 0 | 0 |

**首场战斗：** `FuzzyLouseNormal 15/15 + FuzzyLouseNormal 12/12`（种类与 HP 均一致）

**Neow 4 个选项：** Transform a card / Max HP +10% / Rare relic (Lose 10% max HP) / Boss relic swap

### 7.3 为达成对齐而修正的语义（均已落地）

| 项 | 原始错误 | 正确语义 |
|---|---|---|
| mapRng 播种 | 全流共用 `seed` | 每幕独立：`seed+actNum`(A1) / `*100`(A2) / `*200`(A3) / `*300`(A4) |
| 进房重播种 | 无 | `monsterHp/ai/shuffle/cardRandom/misc = new Random(seed+floorNum)` |
| 遗物池洗牌 | 未实现 | 开局 5 个池各 `Collections.shuffle(..., new Random(relicRng.randomLong()))` |
| `Collections.shuffle(roomList, rng.random)` | 用 `rng.random_range`（推进 counter） | 走 `RandomXS128.nextLong(n)`，**不推进 STS counter**（否则 map counter 154 vs 95） |
| 地图祖先规则 | 「修正」了 Java 的笔误 | Java `getCommonAncestor` 比较的是 `node1.x < node2.`**`y`**——必须照抄这个 bug |
| 遭遇权重 roll | 按定义序累积 | `normalizeWeights` 先 `Collections.sort`（weight 升序）再归一化，roll 遍历排序后列表 |
| 弱怪数量 | 恒 2 | Act1 = 3，Act2/3 = 2 |
| 强怪数量 | `weak+12` | 「再追加 12 项」→ 总数 = `weak + 1(首强) + 12` |
| 首强怪 | 无排除 | `populateFirstStrongEnemy` + `generateExclusions`（按弱怪末项排除） |
| 怪物种类 | 静态写死 | 运行时掷：`2 Louse` 逐只 `miscRng.randomBoolean()`；`Small Slimes` 逐只；`Gang/Shapes` 无放回 |
| Boss 选择 | 未消耗 RNG | `Collections.shuffle(list, new java.util.Random(monsterRng.randomLong()))` 取 [0] |
| setHp 顺序 | 先 bite 后 HP | 先 `setHp` 后 `biteDamage`（顺序决定 `monsterHpRng` 取值） |
| 开局流程 | 构造即 `enter_next_room` | floor 0 = Neow（独立状态）→ 地图屏 → floor 1 = 首战 |
| 地图起点 | `pos=(0,x)` 直接当已起步 | 真游戏起点是虚拟节点 `y=-1`；首次选点是选 row 0 的**列** |
| Neow 抽签 | 固定 4 项表 | `NeowEvent.rng = new Random(Settings.seed)`，4 个 category 依次抽；cat2 先抽 drawback 并按冲突剔除选项 |
| Neow transform | 用 `cardRng` 的 `roll_rarity` | `transformCard(c, false, NeowEvent.rng)`——**用 NeowEvent.rng，cardRng 不变** |

### 7.4 已知未解决

**当前无未解决项**（生成链层面）。

原「Neow Transform 选出的具体卡」已结案：

- 池的正确构造是 `initializeCardPools` → `tmpPool`（= `addRedCards` 的**类名升序**）
  按稀有度分别 append 到 common/uncommon/rare 池，`returnTrulyRandomCardFromAvailable`
  再按 common → uncommon → rare 串接；排除被 transform 的卡（BASIC 不在池内，故通常无影响）
- 红卡池 = COMMON 20 + UNCOMMON 36 + RARE 16 = **72**，排序键 = (稀有度序, 类名)
- `NeowEvent.rng` 前 5 次被 cat0/cat1/drawback/cat2/cat3 消耗，第 6 次 `nextInt(72) = 53`
  → **Spot Weakness**；`Power Through` 位于 index 41，任何单次调用都取不到
- 结论：引擎实现正确；此前 bridge 那局的 `master_deck` 含 `Power Through`，
  应是 `drive_win.py` 多次 `new_run` 重试导致的牌组残留，不是干净的 Neow 结果。
  已用 `neow_transform_uses_neow_rng_not_card_rng` 回归测试锁定该行为
- 另附带修掉：`Searing Blow` 稀有度提取失败（提取器只扫无参构造器，其 `super(...)`
  在 `SearingBlow(int)` 里）→ 池少一张（71 vs 72）

### 7.5 复现方式

```bash
# 1. 编译 pyo3 并放置 .so
cargo build --manifest-path neow-sim/crates/py/Cargo.toml
cp neow-sim/target/debug/libneow_sim.so neow-sim/python/neow_sim.so

# 2. 真游戏需可见窗口启动并监听 37771
cd neow-bridge && python3 scripts/rpc.py ping

# 3. 录制带状态轨迹
python3 neow-sim/tests/conformance/record_trace.py --seed PMCWKZEF --asc 0 --auto \
        --out traces/pmcwkzef.jsonl --max-steps 300

# 4. 逐字段 diff
python3 neow-sim/tests/conformance/parity_diff.py traces/pmcwkzef.jsonl \
        --fields hp,max_hp,gold,player,hand,monsters,piles,relics,rng
```

> 注：`bridge get_state` 需游戏在实际渲染帧（离屏/最小化会让 GL 冻结），
> 且 `new_run` 后必须等地图屏出现才能采样。
