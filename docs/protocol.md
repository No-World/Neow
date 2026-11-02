# Neow 统一 gym 协议 v0.5（草案）

> v0.5（2026-11-01）：协议层对齐审查修订（22 项，见 subagent-artifacts/90bd0327_delegate_output.md）：
> - **可见性规则**（§隐藏信息）：Runic Dome→intent 不可见；Frozen Eye→draw_pile 有序；seed 移入 debug
> - **观测补齐**：player 补 stance/orbs/计数器/pending_kind 等 10 字段；monster 补 alive/escaped/half_dead/stasis_card；card 补 drawn_this_turn/target_type=random
> - **动作对齐**：新增 6 个 ActionType（boss/shop_enter/relic/card_bowl/chest_open/discovery）；多选规范化
> - **vocab 基础设施**：info 补 encoding_version/vocab_hash/unk_policy
> - **screen 子 schema 内联**（移除"见 bridge"外挂）
>
> v0.4（2026-10-31）：`play` 支持按 card_uuid/card_id 定位；事件选项附带结构化 effects。
> v0.3（2026-10-30）：neow-bridge 落地时补齐缺失决策面；选择屏参数化描述符。
> v0.2（2026-10-30）：战斗内选择类动作。
>
> 这是 `neow-sim`、`neow-bridge`、`neow-ai` 三个子项目共享的**唯一接口真相源**。

## 设计原则

1. 一切决策点都是 `(observation, legal_actions, reward, done, info)` 五元组。
2. observation 是纯 JSON，**无损包含玩家合法可知的信息**；可见性由 §隐藏信息可见性 规则决定（遗物/状态驱动）。
3. 动作是结构化三元组 `(type, entity, target)`，由引擎给出合法动作列表，双方不做合法性猜测。
4. **确定性**：同 seed + 同动作序列 ⇒ 完全相同的轨迹。
5. **多选规范化**：`indices` / `take_indices` 一律**升序去重**；`legal_actions` MUST NOT 枚举等价排列。

---

## 隐藏信息可见性

观测字段的输出由**玩家合法可知**决定，而非引擎原样导出：

| 条件 | 规则 |
|---|---|
| **默认** | `draw_pile.order = "hidden"`；`cards` 按 `id+upgraded` 稳定排序输出（顺序无语义，MUST 与引擎内部 `Vec` 顺序无关） |
| **持有 Frozen Eye** | `draw_pile.order = "exact"`；`cards[0]` = 堆顶（下回合第一抽），`cards[last]` = 堆底 |
| **持有 Runic Dome** | 全部 `monsters[].intent = {"type": "UNKNOWN"}`；`damage`/`count` 字段**不得出现**（非 null，键不存在）；`move_history` 一并隐去（原版玩家不可见） |
| **Runic Dome + Frozen Eye 叠加** | 各自独立生效 |
| **cheat 模式**（仅调试） | `draw_pile.order = "exact"`（无视 Frozen Eye）；`intent` 照常输出（无视 Runic Dome） |

sim 与 bridge **共享同一过滤函数**；conformance 测试断言：无 Runic Dome 时 intent 字段存在，有时不存在。

---

## observation

```jsonc
{
  "meta": {
    "character": "IRONCLAD",      // IRONCLAD | SILENT | DEFECT | WATCHER
    "act": 1, "floor": 5,
    "ascension": 20,
    "turn": 3                      // 战斗内回合数，非战斗为 0（非 null——bridge gson 会省略 null 键）
  },
  "player": {
    "hp": 72, "max_hp": 80, "block": 0,
    "energy": 3, "energy_cap": 3, "gold": 156,
    "stance": "NEUTRAL",           // NEUTRAL | WRATH | CALM | DIVINITY（Watcher）
    "powers": [ { "id": "Strength", "amount": 2 } ],
    // ── 回合内计数器（v0.5 新增；引擎 combat.rs 已有，序列化出口在此）──
    "attacks_this_turn": 2,        // 本回合已打攻击牌数（Finisher 依赖）
    "discarded_this_turn": 1,      // 本回合弃牌数（Eviscerate 依赖）
    "last_card_type": "ATTACK",    // 上一张出牌类型（ATTACK|SKILL|POWER|""=无）
    "next_turn_draw": 0,           // 已确定的下回合额外抽牌数
    "next_turn_energy": 0,         // 已确定的下回合同合能量
    "next_turn_copy_count": 0,     // 下回合副本数（Nightmare）
    "hand_cost_zero_this_turn": false,  // 本回合手牌全 0 费
    "hand_cost_max_one": false,          // 本回合手牌费用 ≤1
    // ── 充能球（Defect；v0.5 新增）──
    "orbs": [                       // 按槽位序，[] = 无球
      { "kind": "LIGHTNING", "passive": 3, "evoke": 5, "slot": 0 },
      { "kind": "FROST", "passive": 2, "evoke": 3, "slot": 1 }
    ],
    "max_orb_slots": 3,
    // ── 挂起选择屏（v0.5 新增；screen.type 无法区分 purpose）──
    "pending_kind": null            // null=无挂起 | "hand_discard"|"hand_discard_any"|"scry"|"hand_pick"|"pile_pick"|"discovery"|"stance_choice"|"gamblers_discard"|"hand_exhaust_any"|"discard_pick_free_cost"
  },
  "hand":   [ card, ... ],
  "draw_pile":  { "count": 12, "cards": [card,...], "order": "hidden" },   // order ∈ "hidden"|"exact"；hidden 时 cards 按稳定 key 排序
  "discard_pile": { "count": 6, "cards": [card,...], "order": "chrono" },  // index 0=堆底，末尾=堆顶
  "exhaust_pile": { "count": 1, "cards": [card,...], "order": "chrono" },
  "master_deck": { "count": 12, "cards": [card,...], "order": "sorted" }, // 按 id+upgraded 排序
  "relics":  [
    { "id": "BurningBlood", "counter": -1, "usable": true,
      "period": null,               // 周期类遗物的满值（Shuriken=3, PenNib=10, ...）；null=非周期
      "used_this_turn": false       // 本回合是否已触发（Akabeko/TeaSet 的"已用"状态）
    }, ...
  ],
  "potions": [
    { "id": "FirePotion", "potency": 20, "can_use": true, "can_discard": true } | null,
    null, null
  ],
  "monsters": [
    {
      "id": "JawWorm", "hp": 40, "max_hp": 44, "block": 5,
      "alive": true,                // v0.5 新增——target mask 生成依据
      "escaped": false,             // v0.5 新增——Mugger/Looter 逃离
      "half_dead": false,           // v0.5 新增——Darkling 半死状态
      "stasis_card": null,          // v0.5 新增——Bronze Orb 吸走的卡（null=无 | card id 字符串）
      "powers": [ ... ],
      "intent": { "type": "ATTACK", "damage": 11, "count": 1 },
        // type ∈ ATTACK|DEFEND|ATTACK_DEFEND|ATTACK_BUFF|ATTACK_DEBUFF|BUFF|DEBUFF|STRONG_DEBUFF|DEFEND_BUFF|DEFEND_DEBUFF|SLEEP|UNKNOWN|ESCAPE|NONE|STUN|MAGIC
        // Runic Dome 持有时 → {"type": "UNKNOWN"}，无 damage/count 键
      "move_history": [1, 1, 2]    // 近 N 招（move_id int；含当前招；index 0=最旧）
    }
  ],
  "map": {
    "nodes": [
      { "index": 7, "type": "MONSTER", "row": 3,
        "edges": [8, 9],
        "steps_to_boss": 11,        // v0.5 新增——到 Boss 的最少步数
        "elite_in_3": 1,            // v0.5 新增——3 步内精英数（路径规划）
        "rest_in_3": 0, "shop_in_3": 0
      }, ...
    ],
    "current": 5
  },
  "screen": {
    "type": "combat|map|combat_reward|card_reward|shop|shop_room|event|rest|chest|boss_relic|hand_select|grid_select|scry|discovery|stance_choice|game_over|complete",
    "...": "按类型展开（见下方 §screen 子 schema）"
  }
}
```

---

## screen 子 schema

### combat
```jsonc
{ "type": "combat" }  // 无额外字段——全局+手牌+怪物+orbs 已覆盖
```

### map
```jsonc
{ "type": "map", "at_boss": false }  // at_boss=true 时顶层有 boss 门
```

### combat_reward / card_reward
```jsonc
{ "type": "combat_reward",
  "items": [
    { "index": 0, "kind": "GOLD", "gold": 25 },
    { "index": 1, "kind": "CARD", "cards": [card,...] },   // 3 张候选
    { "index": 2, "kind": "POTION", "potion_id": "FirePotion" },
    { "index": 3, "kind": "RELIC", "relic_id": "Lantern" }
  ]
}
```

### shop / shop_room
```jsonc
{ "type": "shop_room" }  // 进店后与商人交互前的屏
{ "type": "shop",
  "cards":     [ { "card": card, "price": 120, "sold": false }, ... ],   // 5 张
  "colorless": [ { "card": card, "price": 150, "sold": false }, ... ],   // 2 张
  "relics":    [ { "id": "Lantern", "price": 160, "sold": false }, ... ], // 3 件
  "potions":   [ { "id": "FirePotion", "price": 55, "sold": false }, ... ], // 3 瓶
  "purge_cost": 75, "purge_available": true,
  "gold": 156
}
```

### event
```jsonc
{ "type": "event", "event_id": "Big Fish", "screen_num": 0,
  "options": [
    { "index": 0, "tag": "banana", "label": "[香蕉] 回复 26 HP",
      "heal": 26, "disabled": false },   // 数值参数按事件展开
    ...
  ]
}
```

### rest
```jsonc
{ "type": "rest", "options": [
  { "index": 0, "name": "sleep", "usable": true, "heal": 24 },
  { "index": 1, "name": "smith", "usable": true, "cards": [card,...] },
  { "index": 2, "name": "toke",  "usable": true, "cards": [card,...] },
  { "index": 3, "name": "dig",   "usable": true },
  { "index": 4, "name": "lift",  "usable": true }
] }
```

### chest
```jsonc
{ "type": "chest", "boss": false, "opened": false,
  "chest_type": "large",
  "contents": { "gold": true, "gold_base": 75, "relic_tier": "uncommon" } }
// boss: { "contents": { "relics": ["...","..."], "choose": 3 } }
```

### boss_relic
```jsonc
{ "type": "boss_relic", "candidates": ["Ectoplasm","Sozu","Runic Pyramid"] }
```

### hand_select / grid_select
```jsonc
{ "type": "hand_select",
  "purpose": "discard|exhaust|duplicate|dual_wield|...",  // 对应 pending_kind
  "num_to_select": 2, "max_to_select": 2,
  "can_pick_zero": false, "any_number": false,
  "cards": [card,...]   // 候选牌（手牌或牌堆展开）
}
{ "type": "grid_select",
  "pile": "draw|discard|exhaust|hand|deck|mixed",   // 候选来源
  "purpose": "purge|transform|upgrade|...",
  "num_to_select": 1, "max_to_select": 1,
  "can_pick_zero": false,
  "cards": [card,...]   // 候选牌（最多 60 张）
}
```

### scry
```jsonc
{ "type": "scry", "shown": [card,...] }   // 从抽牌堆顶展示的牌（顺序=堆顶序）
```

### discovery
```jsonc
{ "type": "discovery", "options": [card, card, card] }   // 三选一（探素药水等）
```

### stance_choice
```jsonc
{ "type": "stance_choice" }   // 姿态药水：选 Wrath(true) 或 Calm(false)
```

---

## card 对象

```jsonc
{
  "id": "Strike_R", "upgraded": false,     // bool（Searing Blow 多次升级场景用 upgrade_times 补充）
  "upgrade_times": 0,                        // 仅 Searing Blow 等可反复升级的卡 >0
  "cost": 1, "cost_for_turn": 1,
  "type": "ATTACK",
  "target_type": "enemy|self|none|all|random",  // random = Sword Boomerang 等
  "damage": 6, "block": 0, "magic_number": 0,   // 已修正值
  "exhaust": false, "ethereal": false, "innate": false,
  "drawn_this_turn": true                    // v0.5 新增——本回合抽到 vs 上回合保留
}
```

---

## action

```jsonc
// ── 战斗内 ──
{ "type": "play",        "card_index": 2, "target": 0 }       // target=monsters 下标；None/All/Random 目标卡省略 target
{ "type": "play",        "card_id": "Shiv", "target": 0 }     // 按卡牌 ID 定位
{ "type": "end_turn" }
{ "type": "potion",      "potion_index": 1, "target": 0 }    // target 仅投掷类需要
{ "type": "potion_discard", "potion_index": 1 }

// ── 选择屏（参数化：indices 为空占位，约束在 screen 内）──
{ "type": "hand_select",  "indices": [0, 3] }                 // 多选：升序去重
{ "type": "grid_select",  "pile": "discard", "indices": [0] }
{ "type": "scry",         "discard_indices": [0, 2] }         // 弃掉 shown[0,2]；其余保留
{ "type": "discovery",    "index": 1 }                         // 三选一
{ "type": "stance_choice", "wrath": true }

// ── 局外 ──
{ "type": "map",          "node_index": 7 }
{ "type": "boss" }                                            // Boss 门
{ "type": "event",        "option_index": 1 }
{ "type": "rest",         "option": "sleep" }
{ "type": "reward",       "take_indices": [0, 2] }             // 升序去重
{ "type": "reward",       "take_indices": [1], "card_pick": 0 }
{ "type": "reward",       "skip": true }
{ "type": "card_pick",    "index": 2 }
{ "type": "card_bowl" }                                       // 唱歌碗替代拿卡
{ "type": "skip" }
{ "type": "chest_open" }
{ "type": "relic",        "relic_index": 0 }                   // Boss 遗物三选一

// ── 商店 ──
{ "type": "shop_enter" }
{ "type": "shop_buy_card",   "index": 0 }
{ "type": "shop_buy_relic",  "index": 1 }
{ "type": "shop_buy_potion", "index": 2 }
{ "type": "shop_card_removal" }

// ── 通用 ──
{ "type": "proceed" }
{ "type": "cancel" }
```

---

## info

```jsonc
{
  "protocol_version": "0.5",       // 本文档版本
  "encoding_version": "2.1",       // observation-encoding.md 版本
  "vocab_hash": "0x9f3c...",       // CARD/RELIC/MOB/POWER/POTION/MOVE/ORB vocab + RELIC_PERIOD + MAX_T + F 的联合 hash
  "feature_dim": 24,               // token 特征维度 F
  "max_t": 224,                    // 观测序列长度上限
  "unk_policy": "index0",          // 未知 ID 的处理策略（index 0 = UNK）
  "vocab_misses": 0,               // 本次 step 中未知 ID 命中次数
  "escaped": false,                // 烟雾弹逃离标记
  "debug": {                       // 仅 --debug 模式输出；训练侧必须过滤
    "seed": 12345
  }
}
```

---

## step 语义

`step(action) -> (observation, legal_actions, reward, done, info)`

- `reward`：引擎只给**终局 ±1**；shaping 在 neow-ai 内部做。
- `legal_actions`：当前局面全部合法动作列表（JSON 格式如上）；`step` 收到非法动作抛错。
- `info.protocol_version` + `info.vocab_hash`：三方必须一致才能互通。Python 侧启动时校验 vocab_hash，不匹配**硬失败**。

---

## conformance 测试

`neow-sim/tests/conformance/` 内放置「种子 + 动作序列 → 期望观测摘要」的黄金用例。

**新增测试**：
1. `obs_leak_test`：无 Runic Dome 时 intent 字段存在；有时 `{"type":"UNKNOWN"}` 且无 damage/count 键。
2. `draw_order_test`：无 Frozen Eye 时 `draw_pile.cards` 排列与引擎内部顺序无关（打乱内部 → 输出不变）。
3. `budget_test`：Σ 槽位 ≤ MAX_T(224)；grid_select 候选 ≤ 64 token。
4. `action_bijection_test`：`legal_actions` ↔ encoding `flat_actions` 双向一一映射。
5. `vocab_capacity_test`：447 卡 / 170+ power 全覆盖；hash 匹配；UNK 命中 0。
