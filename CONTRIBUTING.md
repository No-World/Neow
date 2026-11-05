# 贡献规范（CONTRIBUTING）

> 本项目的唯一目标是**与真游戏逐位一致**（见 `docs/parity-spec.md`）。
> 下面每条约束都是踩过坑之后写下来的，不是理论要求。

## 1. 铁律：实现前先贴源码

**任何涉及规则语义的代码，动手前必须先把对应的 Java 源码片段贴到提交说明或代码注释里。**

适用范围（不限于）：

| 类别 | 例子 |
|---|---|
| 遗物获得/失去 | `relic_on_obtain` 的每个分支（`WingBoots()` 构造器设 `counter=3`） |
| 选择/跳过 | `BossRelicSelectScreen.relicSkipLogic()`（只关屏、**无补偿**） |
| 怪物 AI | `getMove(int num)` 的每个分支（`num<33` 可能**不看历史**） |
| RNG 消耗 | `cardRng.randomBoolean(chance)` 是否被 `&&` 短路跳过 |
| 房间流程 | `AbstractChest.open()`（宝箱**从不加卡牌奖励**） |
| 池顺序 | `CardLibrary.addRedCards(ArrayList)` 遍历 `cards.entrySet()`（HashMap 序） |

### 为什么

凭印象写实现产生的错误**无法从症状反推**。实测踩过的例子：

| 我的假设 | 源码事实 | 症状 |
|---|---|---|
| Boss 遗物跳过换 +1 能量 | `relicSkipLogic` 只关屏，无补偿 | 能量对不上，难定位 |
| 飞鞋幕间重置 | 只在 Flight mod 启用时移除 | counter 长期错 |
| `Looter.getMove` 有 `num<40` 分支 | 全文只有一行恒 move 1 | 第 2 回合就出 Lunge |
| `GremlinNob` 两分支结构相同 | `num<33` 无条件 return | 意图 24 vs 12 |
| `AngerPower` = 直接加力量 | 是「玩家打 SKILL → 持有者加力量」 | 力量无限增长 |

**共同点：静态读代码时"按对称性脑补"，跳过中间几行。**

## 2. RNG 消耗必须逐次核对，不能只看总数

同一段逻辑的**总消耗次数相同**但**顺序不同**时，后续所有流都会错位。

已验证的坑：

- `getRewardCards()` 是逐张 `getCard(rarity)`（交错），不是「取完 3 张再统一掷升级」
- `c.rarity != RARE && cardRng.randomBoolean(...)` —— RARE 卡**不掷**
- `slashCount == 0 && aiRng.randomBoolean(0.6f)` —— 非首次**不掷**
- `Collections.shuffle(list, rng.random)` 走 `RandomXS128.nextLong(n)`，**不推进 STS counter**
- 宝箱遗物从**预洗池弹出**，运行时 **0 次 relicRng**

**核对方法**：用 bridge 侧 `RngWatch`（`NEOW_TRACE_RNG=1`）记录逐帧 counter 变化，比只看终值可靠。

## 3. 对拍口径：只在双方同一状态时比对

`tests/conformance/parity_run.py` 的做法：

- 只在两边都处于 `combat` 时逐字段比对
- 屏幕状态机步数不同（如 Neow：bridge 4 步 / 引擎 1 步）时显式 skip，不硬套
- **不要**因为短轨迹通过就宣布 PASS——先确认轨迹覆盖了分歧区域

## 4. 覆盖率数字必须来自运行时

**不要用「枚举变体数」当覆盖率。**

实测教训：`RelicKind` 有 181 个变体，但 `IMPLEMENTED_RELICS` 只列了 18 只 ⇒
「遗物 181/186 97%」是假的，真实可发放只有 **18**（差了 10 倍）。

正确做法：跑一局统计**实际出现**的卡/遗物/事件种类，或读发放路径上的白名单常量。

## 5. 同类 bug 的预防

### 5.1 裸元组 → 具名结构体

`MonsterState::spawn` 原返回裸 7 元组
`(chomp_dmg, thrash_dmg, thrash_block, bellow_str, bellow_block, dark_strike_dmg, ritual_amt)`，
字段含义靠注释记忆。**连踩 4 次**同一类 bug（Louse / SpikeSlime_M / AcidSlime_S / Looter
都把主攻击伤害放进 `.0`，而读取方读 `.5`）。

已改为 `MonsterStats` 具名结构体。**新代码禁止裸元组传多字段。**

### 5.2 Java 的 `&&` 短路会改变 RNG 消耗

写条件判断时，若右侧含 RNG 调用，必须显式确认短路行为是否与源码一致。

## 6. 提交前检查清单

```bash
# 1. 测试全绿
cargo test -p neow-sim-core

# 2. 对拍不退化（compared 数不应下降）
python3 tests/conformance/parity_run.py traces/ai.jsonl traces/long.jsonl \
        --fields hand,player,monsters,piles,rng

# 3. 生成链对拍（改动地图/遭遇/遗物池时必跑）
python3 tools/parity_gen.py --count 40
```

## 7. 文档同步

改动以下内容时同步更新对应文档：

| 改动 | 文档 |
|---|---|
| 观测特征/维度 | `docs/observation-encoding.md` |
| 规则语义/偏差 | `docs/engine-semantics.md`、`docs/parity-spec.md` §7.3 |
| 覆盖率 | `docs/coverage.md`（须注明采集方式） |
