# Neow 统一 gym 协议 v0.2（草案）

> v0.2（2026-10-30）：依据游戏源码勘察（docs/combat-actions.md）补充战斗内选择类动作：hand_select / grid_select / scry / potion_discard。

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
    "type": "combat | map | reward | shop | event | rest | game_over | ...",
    "...": "按 screen 类型展开的专有字段（商品列表、事件选项、奖励卡池等）"
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
{ "type": "play",    "card_index": 2, "target": 0 }        // target = monsters 下标
{ "type": "potion",  "potion_index": 1, "target": null }
{ "type": "relic",   "relic_index": 3, "target": null }
{ "type": "map",     "node_index": 7 }
{ "type": "shop_buy_card | shop_buy_relic | shop_buy_potion | shop_sell | shop_card_removal", "index": 0 }
{ "type": "event",   "option_index": 1 }
{ "type": "rest",    "option": "sleep | smith | dig | ..." }
{ "type": "reward",  "take_indices": [0, 2], "take_potion": false, "skip": false }

// —— 战斗内选择屏（v0.2，源码勘察：HandCardSelectScreen / GridCardSelectScreen / Scry）——
{ "type": "hand_select",  "indices": [0, 3] }        // 手牌选择屏：过量弃牌、赌博、双持等
{ "type": "grid_select",  "pile": "draw|discard|exhaust|hand|deck", "indices": [0, 1] }  // 牌堆网格选择屏：寻求、挖掘、发现等
{ "type": "scry", "discard_indices": [0, 2], "top_indices": [1, 3] }   // 占卜：置顶顺序敏感
{ "type": "potion_discard", "potion_index": 1 }        // 倒掉药水
```

## step 语义

`step(action) -> (observation, legal_actions, reward, done, info)`

- `reward`：引擎只给**终局 ±1**（胜/负）；训练侧的 shaping 在 `neow-ai` 内部做，不污染协议。
- `legal_actions`：当前局面全部**合法正常**动作列表（利用动画时序/路径判定等漏洞的序列不在动作空间内，见 docs/parity-spec.md 偏差登记表）；`step` 收到非法动作直接抛错（训练侧负责 mask）。
- `info.protocol_version`：本协议版本号，三方必须一致才能互通。

## conformance 测试

`neow-sim/tests/conformance/` 内放置"种子 + 动作序列 → 期望观测摘要"的黄金用例；
`neow-bridge` 的对拍 runner（同种子同动作序列跑真游戏）必须通过同一套用例。
