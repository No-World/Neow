# Neow 统一 gym 协议 v0.3（草案）

> v0.2（2026-10-30）：依据游戏源码勘察（docs/combat-actions.md）补充战斗内选择类动作：hand_select / grid_select / scry / potion_discard。
> v0.3（2026-10-30）：neow-bridge 落地时补齐游戏实际存在但 v0.2 缺失的决策面：
> 卡牌奖励屏（card_pick / card_bowl / skip）、宝箱（chest_open）、Boss 遗物（relic）、Boss 门（boss）、
> 商店进入（shop_enter）、通用推进/后退（proceed / cancel）；明确选择屏动作为参数化描述符；
> 定义 bridge 传输封装（见文末）。

这是 `neow-sim`、`neow-bridge`、`neow-ai` 三个子项目共享的**唯一接口真相源**。
任何一方的接口改动都必须先改这里，并同步三方的 conformance 测试。

## 设计原则

1. 一切决策点都是 `(observation, legal_actions, reward, done, info)` 五元组。
2. observation 是纯 JSON，**无损包含玩家合法可知的信息**；隐藏信息（抽牌堆顺序等）只暴露集合。
3. 动作是结构化三元组 `(type, entity, target)`，由引擎给出合法动作列表，双方不做合法性猜测。
4. **确定性**：同 seed + 同动作序列 ⇒ 完全相同的轨迹（neow-sim 保证；neow-bridge 通过暴露游戏 RNG 种子尽量保证）。

## observation

```jsonc
{
  "meta": {
    "character": "IRONCLAD",      // IRONCLAD | SILENT | DEFECT | WATCHER
    "act": 1, "floor": 5,
    "ascension": 20,
    "turn": 3,                     // 战斗内回合数，非战斗为 null
    "seed": 12345,                 // 引擎级 RNG 种子（bridge 需尽力暴露）
    "over": false, "won": null
  },
  "player": {
    "hp": 72, "max_hp": 80, "block": 0,
    "energy": 3, "energy_cap": 3, "gold": 156,
    "powers": [ { "id": "Strength", "amount": 2 } ]
  },
  "hand":   [ card, ... ],
  "draw_pile":  { "count": 12, "cards": [ card, ... ] },  // 集合，顺序隐藏
  "discard_pile": [ card, ... ],
  "exhaust_pile": [ card, ... ],
  "master_deck":  [ card, ... ],
  "relics":  [ { "id": "BurningBlood", "counter": -1, "enabled": true }, ... ],
  "potions": [ { "id": "FirePotion" } | null, null, null ],
  "monsters": [
    {
      "id": "JawWorm", "hp": 40, "max_hp": 44, "block": 5,
      "powers": [ ... ],
      "intent": { "type": "ATTACK", "damage": 11, "count": 1 },  // 可见部分
      "move_history": [ ... ]
    }
  ],
  "map": {                         // 仅 screen.type == "map" 时展开
    "nodes": [ { "index": 7, "type": "MONSTER|REST|SHOP|EVENT|ELITE|TREASURE|BOSS", "edges": [8,9] }, ... ],
    "current": 5
  },
  "screen": {
    "type": "combat | map | combat_reward | card_reward | shop | shop_room | event | rest | chest | boss_relic | hand_select | grid_select | game_over | complete | gremlin_match | gremlin_wheel",
    "...": "按 screen 类型展开的专有字段（商品列表、事件选项、奖励项、选择约束等），见 neow-bridge/PROTOCOL.md"
  }
}
```

### card 对象

```jsonc
{
  "id": "Strike_R", "upgraded": 0,        // 0 | 1 | 2
  "cost": 1, "cost_for_turn": 1,
  "type": "ATTACK", "target_type": "enemy | self | none | all",
  "damage": 6, "block": 0, "magic_number": 0,   // 已按力量/虚弱等修正后的可见值
  "exhaust": false, "ethereal": false, "innate": false
}
```

## action

```jsonc
{ "type": "end_turn" }
{ "type": "play",    "card_index": 2, "target": 0 }        // target = monsters 下标；无目标卡省略
{ "type": "potion",  "potion_index": 1, "target": null }  // target 仅投掷类药水需要
{ "type": "potion_discard", "potion_index": 1 }             // 倒掉药水（战斗中/药水满槽领奖时）
{ "type": "relic",   "relic_index": 0 }                     // v0.3：Boss 遗物三选一屏（原版无其余可主动使用遗物）
{ "type": "map",     "node_index": 7 }                      // node_index = map.nodes[].index
{ "type": "boss" }                                            // v0.3：地图顶层的 Boss 门
{ "type": "shop_enter" }                                      // v0.3：进入商店房间后与商人交互打开商店
{ "type": "shop_buy_card | shop_buy_relic | shop_buy_potion", "index": 0 }   // index 按对应列表下标
{ "type": "shop_card_removal" }                               // 打开删卡服务（随后进入 grid_select）
{ "type": "event",   "option_index": 1 }
{ "type": "rest",    "option": "sleep | smith | dig | lift | recall | toke" }
{ "type": "reward",  "take_indices": [0, 2], "skip": false }  // take 逐项结算；遇 CARD 项停在卡牌奖励屏
{ "type": "card_pick", "index": 2 }                          // v0.3：卡牌奖励屏选卡（index 按奖励卡列表）
{ "type": "card_bowl" }                                       // v0.3：唱歌碗（+2 最大生命）替代拿卡
{ "type": "skip" }                                            // v0.3：跳过卡牌奖励
{ "type": "chest_open" }                                      // v0.3：打开宝箱（随后进入奖励屏）
{ "type": "proceed" }                                         // v0.3：通用推进（房间完成继续 / 结算继续 / 游戏结束返回主菜单）
{ "type": "cancel" }                                          // v0.3：通用后退（关闭地图 / 离开商店 / 选择屏取消）

// —— 战斗内选择屏（v0.2，源码勘察：HandCardSelectScreen / GridCardSelectScreen / Scry）——
{ "type": "hand_select",  "indices": [0, 3] }        // 手牌选择屏：过量弃牌、赌博、双持等
{ "type": "grid_select",  "pile": "draw|discard|exhaust|hand|deck|mixed", "indices": [0, 1] }  // 牌堆网格选择屏
{ "type": "scry", "discard_indices": [0, 2], "top_indices": [1, 3] }   // 占卜：弃掉 discard_indices；保留牌维持原序（top_indices 仅校验用）
```

### 选择屏动作的参数化描述符

多选组合是组合空间（C(10,5)=252），逐条枚举不现实。选择屏（hand_select / grid_select / scry）在
`legal_actions` 中只给一个**参数化描述符**（indices 为空数组占位），合法组合的约束放在 `screen` 内：
`num_to_select` / `max_to_select` / `can_pick_zero` / `any_number`。`step` 对具体 indices 做范围与数量校验，
非法即抛错。自回归动作头在训练侧据此构造 mask。

## step 语义

`step(action) -> (observation, legal_actions, reward, done, info)`

- `reward`：引擎只给**终局 ±1**（胜/负）；训练侧的 shaping 在 `neow-ai` 内部做，不污染协议。
- `legal_actions`：当前局面全部**合法正常**动作列表（利用动画时序/路径判定等漏洞的序列不在动作空间内，见 docs/parity-spec.md 偏差登记表）；`step` 收到非法动作直接抛错（训练侧负责 mask）。
- `info.protocol_version`：本协议版本号，三方必须一致才能互通。

## v0.3 勘误与说明

- 原版游戏**没有**可通过主动点击使用的遗物（`AbstractRelic` 无 use 路径），`relic` 动作映射到 Boss 遗物三选一屏；`shop_sell` 在原版不存在，移除。
- 原版占卜（ScryAction）**不允许重排**保留牌的顺序，保留牌维持抽牌堆原序；`top_indices` 语义弱化为校验用。
- `potion` 动作仅战斗内合法（`AbstractPotion.canUse()` 要求 RoomPhase.COMBAT）。
- `reward.take_indices` 逐项结算：若中途某 CARD 项打开卡牌奖励屏，则当次 step 停在 `screen.type == "card_reward"`，
  其余未处理下标作废（一次 step 最多打开一个卡牌奖励屏）。
- 药水满槽时领药水奖励：先 `potion_discard` 腾槽，再 `reward` 领取（游戏 UI 同路径）。

## bridge 传输封装（neow-bridge 专用，gym 语义不变）

传输层与 gym 五元组解耦，规格见 `neow-bridge/PROTOCOL.md`。概要：

- 通道：本地 TCP（默认 127.0.0.1:37771）或 stdio，逐行 JSON（UTF-8，`\n` 分帧）。
- 请求：`{"id": 1, "cmd": "step", "action": {..}}`；命令集：`ping / new_run / step / get_state / quit_run / kill_game / config`。
- 响应：`{"id": 1, "ok": true, "result": {gym五元组}}` 或 `{"id": 1, "ok": false, "error": "..", "error_type": ".."}`。
- `step` 在游戏线程执行动作、等待状态稳定（多帧指纹不变 + 队列清空）后才返回 —— 单请求即单决策点，天然适合请求/回复式 gym 封装。

## conformance 测试

`neow-sim/tests/conformance/` 内放置"种子 + 动作序列 → 期望观测摘要"的黄金用例；
`neow-bridge` 的对拍 runner（同种子同动作序列跑真游戏）必须通过同一套用例。
