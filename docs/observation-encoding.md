# 观测编码与动作解码设计方案 v2.0

> v1.0 三视角合成 → v2.0 DeepSeek v4.1 Flash 严格审查修订。
> 审查报告：`subagent-artifacts/9fc8dd80_delegate_output.md`

## 核心洞察

杀戮尖塔与围棋/星际的关键差异：
1. **回合制完备信息**（抽牌堆内容可知、顺序不可知）→ 不需要 AlphaStar 的 spatial encoder
2. **实体集合而非棋盘**→ 类似 Mortal 的牌张 token 化，但实体类型更多样
3. **决策密度低**（~20/回合 vs 星际 300APM）→ 序列可以更长、每个 token 可以更丰富
4. **隐藏信息是 memoryless 的**（i.i.d. 均匀洗牌，与历史无关）→ **不需要 RNN/LSTM**。
   Mortal 的 LSTM 必须是因为其隐藏信息（对手手牌）与历史相关；STS 纯 transformer + 正确观测 ≈ Markov。
   前提：补齐全部状态变量（§1.4）使状态真正 Markov。

## 一、观测编码

### 1.1 设计原则

- **无损决策信息**：编码后的观测包含做出最优决策所需的全部信息（含回合内计数器、挂起屏、姿态/球等）
- **固定 shape**：pad 到 max size，方便批处理
- **数值归一化**：所有数值 → [0,1] 或 [-1,1]，避免量纲差异
- **ID → embedding**：categorical 用查表（Python 侧 `nn.Embedding`），Rust 只输出 int 索引

### 1.2 接口拆分（Rust 无法产出 learned embedding）

```rust
// Rust 输出原始索引+数值特征，Python 侧做 embedding lookup
struct Observation {
    /// token 类型标签: (T,) int8 — 0=pad, 1=cls, 2=global, 3=hand, 4=mob, ...
    token_types:  [i8; MAX_T],
    /// 实体 ID: (T, 3) int32 — [card_id, relic_id, mob_id] 供查表（-1=无）
    token_ids:    [[i32; 3]; MAX_T],
    /// 数值特征: (T, F) f32 — 归一化后的连续值
    token_feats:  [[f32; F]; MAX_T],
    /// 序列有效长度
    seq_len: usize,
}
```

Python 侧：`emb = table[token_ids] + proj(token_feats)` → d_model 维。

### 1.3 序列布局（MAX_T = 150）

```
[CLS]                       ← 1 token
[GLOBAL]                    ← 1 token：24 维全局状态（含 stance/orbs/计数器）
[HAND_0..9]                 ← 10 tokens（pad=mask），无 position emb（手牌是集合）
[MOB_0..4]                  ← 5 tokens（含 move_history 近 3 招 + intent 11 变体）
[PWR_PLAYER_0..15]          ← 16 tokens（从 8 扩容——长局 9-12 个 power 常见）
[PWR_MOB0_0..7]             ← 8 tokens × 5 mobs = 40（Boss 5+ power）
[RELIC_0..31]               ← 32 tokens（含 counter/period + ready）
[POTION_0..5]               ← 6 tokens
[ORB_0..9]                  ← 10 tokens（Defect：球类型+数值+槽位序）
[PILE_DRAW]                 ← 3 tokens：count + bag_mean + bag_topk
[PILE_DISCARD]              ← 3 tokens
[PILE_DECK]                 ← 3 tokens
[PILE_EXHAUST]              ← 3 tokens
[SCREEN_SPECIFIC]           ← 0-55 tokens（事件选项/商店商品/奖励项/地图可达节点/选择屏候选牌）
── max_total = 150 tokens ──
```

### 1.4 全局 Token 特征（21 维，修复 v1.0 缺失）

| # | 特征 | 归一化 | 说明 |
|---|---|---|---|
| 1 | screen_type | int8 → emb | 11 类（combat/map/event/shop/rest/reward/chest/boss_relic/select/complete/game_over） |
| 2 | act | (act-1)/2 | 1-3 |
| 3 | floor | floor/51 | 1-51 |
| 4 | turn | min(turn,20)/20 | 战斗回合 |
| 5 | hp | hp/max_hp | |
| 6 | max_hp | log(max_hp)/log(999) | |
| 7 | block | x/(x+30) | log1p 式归一化（Demon Form 力量 100+ 不 clip） |
| 8 | energy | min(energy,10)/10 | |
| 9 | gold | log(1+gold)/log(1+9999) | |
| 10 | ascension | asc/20 | |
| 11 | stance | int8 → emb | 4 类（Neutral/Wrath/Calm/Divinity）——**Watcher 必需** |
| 12 | mantra | min(mantra,10)/10 | 观者真言 |
| 13 | attacks_this_turn | min(x,10)/10 | **Finisher 等直接依赖** |
| 14 | discarded_this_turn | min(x,10)/10 | **Eviscerate 等直接依赖** |
| 15 | last_card_type | int8 → emb | 3 类（ATTACK/SKILL/POWER）——条件卡依赖 |
| 16 | next_turn_draw | min(x,5)/5 | 已确定的未来资源 |
| 17 | next_turn_energy | min(x,5)/5 | |
| 18 | next_turn_copy_count | min(x,3)/3 | Nightmare 等 |
| 19 | pending_kind | int8 → emb | 10 类 Pending 变体——挂起选择屏类型 |
| 20 | hand_cost_zero_this_turn | 0/1 | 本回合手牌全 0 费 |
| 21 | hand_cost_max_one | 0/1 | 手牌费用 ≤1 |

> ~~P_draw_engine / P_draw_energy / expected_damage~~ **已删除**：主观定义（"什么算引擎卡"）
> + 模型可通过 predict_next_draw 辅助头自己学概率查询，且能发现人类未预定义的查询。

### 1.5 牌堆编码：bag-of-embeddings + 多 query token（替代直方图）

v1.0 的单 token 直方图压缩被确认为**最大瓶颈**——64 维 MLP 无法在 752 维稀疏输入上做精确计数查询。

**v2.0 方案**：每个牌堆产出 3 个 token：

```
t_count = MLP(pile_size, draw_n, zero_cost_count, attack_count, skill_count, power_count)
t_bag_mean = Σ_i (id_emb[card_i] + upgrade_flag) × count_i / Σ count_i   // 频次加权均值
t_bag_topk = max-pool over {id_emb[card_i] + feat_i}                      // 学「最关心的那张」
```

- `t_bag_mean` 让下游 query 通过点积获得「该类卡的期望数量」——可学习的计数机制
- `t_bag_topk` 捕捉「牌堆里最关键的一张」（如 lethal 需要的那张 Strike）
- 抽牌堆额外给 **top-2 卡的 id_emb**（覆盖 FrozenEye 遗物的全可见场景）
- Rust 侧预算：bag_mean = 对牌堆做一次 embedding lookup 求均值，在引擎侧完成（非 Python 侧）

### 1.6 怪物 Token（含 move_history + intent 11 变体）

| 特征 | 归一化 |
|---|---|
| mob_id | int32 → emb (80) |
| hp | hp/max_hp |
| max_hp | log(max_hp)/log(999) |
| block | x/(x+30) |
| intent_type | int8 → emb (**11 变体**：Attack/DefendBuff/AttackDefend/AttackBuff/AttackDebuff/Buff/Debuff/StrongDebuff/Defend/Sleep/Unknown/Escape) |
| intent_damage | x/(x+30)（已修正值） |
| intent_count | min(x,5)/5 |
| last_move_1 | int8/8 → scalar（move_history[-1]） |
| last_move_2 | 同上（move_history[-2]） |
| last_move_3 | 同上（move_history[-3]） |
| stasis_card | 0/1（铜球是否吸了卡） |
| half_dead | 0/1（小暗灵半死） |
| escaped | 0/1（已逃离） |

### 1.7 遗物 Token（counter 相对化）

| 特征 | 归一化 | 说明 |
|---|---|---|
| relic_id | int32 → emb (192) | |
| counter/period | f32 | **相对进度**（如手里剑 3/3=1.0） |
| (period-counter)/period | f32 | **还差几回合触发** |
| ready | 0/1 | 是否已就绪（counter≥period） |
| enabled | 0/1 | 本回合是否已触发 |

Rust 侧维护 `RELIC_PERIOD: [u8; 192]` 常量表。

### 1.8 药水 Token

| 特征 | 归一化 |
|---|---|
| potion_id | int32 → emb (48) |
| potency | min(x,20)/20 |
| slot_idx | i/5 |

### 1.9 Orb Token（Defect 专用）

| 特征 | 归一化 |
|---|---|
| orb_kind | int8 → emb (5: Lightning/Frost/Dark/Plasma/Void) |
| passive_value | x/(x+30) |
| evoke_value | x/(x+30) |
| slot_idx | i/max_orbs |

### 1.10 数值归一化汇总

| 特征 | 范围 | 归一化 | 理由 |
|---|---|---|---|
| hp / max_hp | 1-999 | hp/max_hp | 比率 |
| max_hp | 1-999 | log(max_hp)/log(999) | 对数尺度 |
| energy | 0-99 | min(energy,10)/10 | clip 上界 |
| gold | 0-9999 | log(1+gold)/log(1+9999) | 对数 |
| damage / block | 0-999+ | **x/(x+30)** | **不用 clip**——log1p 式饱和 |
| cost | -1..10 | (cost+1)/11 | 线性 |
| 卡牌 damage/block | 已修正值 | x/(x+30) | 给模型看最终值 |
| counter/period | 0-1 | 原生 | 相对化 |
| intent_count | 1-5 | min(x,5)/5 | |

### 1.11 屏幕专属 Token

| 屏幕 | token 内容 | 数量 |
|---|---|---|
| combat | 空（全局+手牌+怪物+orbs 已覆盖） | 0 |
| map | **候选可达节点**（2-4 个）+ 路径摘要（到 boss 距离、k 步内 elite/rest/shop 计数） | 4-8 |
| event | 事件选项（tag + 数值参数 + disabled） | 2-5 |
| shop | 商品（卡 5+无色 2+遗物 3+药水 3）+ 删卡服务 | 14 |
| reward | 奖励项（gold/card/potion/relic + 数值） | 3-5 |
| chest | 箱型 + 内容预览 | 1-2 |
| rest | 休息选项（sleep/smith/dig/lift/toke） | 2-4 |
| boss_relic | 3 个候选遗物 | 3 |
| select | **选择屏候选牌**（PilePick 的全牌堆 30-60 张） | **≤55** |
| game_over | 胜负 | 1 |

**地图改进**（v1.0 的行池化丢失边信息）：
- 不用 15 行×7 列池化
- 只给**候选可达节点**（当前节点 edges 指向的 2-4 个）各一个 token（room_type + 到 boss 行距 + k 步内 elite/rest/shop 计数）
- 补 1 个路径摘要 token（全局路线统计）

### 1.12 手牌位置编码

v1.0 写「位置编码 = 出牌序」有歧义。修正：
- 手牌是**集合**，出牌后 index 位移导致 position emb 不稳定
- **不用 position emb**；改为 `drawn_this_turn` 0/1 标志（本回合抽到的 vs 上回合保留的）
- 如需排序信息，用「抽到的相对次序」slot emb + 上述标志

## 二、动作解码

### 2.1 pointer-over-tokens（替代固定 24 槽）

v1.0 的固定 24 entity 槽装不下 PilePick（全牌堆 30-60 张候选）。改为：

```
step1: type_head   → ~18 类
step2: entity_head → 指向观测序列的 token 槽位（0..149）
step3: target_head → 指向观测序列的 token 槽位（仅 Single 目标卡需要）
```

**entity/target 直接索引观测 token**——选择屏候选牌作为 SCREEN_SPECIFIC token 塞进序列后，pointer 天然覆盖。

### 2.2 动作类型（18 类）

```rust
enum ActionType {
    PlayCard,       // entity→hand_token, target→mob_token (Single) 或 skip (None/All/Random)
    EndTurn,
    UsePotion,      // entity→potion_token, target→mob_token (可选)
    DiscardPotion,  // entity→potion_token
    ChooseMap,      // entity→map_node_token
    ChooseEvent,    // entity→event_option_token
    RestAction,     // entity→rest_option_token
    ShopBuy,        // entity→shop_item_token
    ShopPurge,      // entity→deck_card_token (选择屏)
    TakeReward,     // entity→reward_item_token
    SkipReward,
    PickCard,       // entity→card_reward_token
    Proceed,
    Cancel,
    Confirm,        // 多选屏结束/anyNumber 的 0 张
    ToggleSelect,   // 多选屏逐次 toggle（配合 Confirm）
    Scry,           // 占卜（shown token 的弃/留决策）
    StanceChoice,   // 姿态选择（Wrath/Calm）
}
```

### 2.3 目标模式显式化

```rust
enum TargetMode { None, Single, All, Random }
```

- `Single` → step3 有效（mask 只放开存活怪物 token）
- `None / All / Random` → step3 **跳过**（mask 全 true 或全 false，不产生监督）
- 消除 v1.0 的「All 卡 target=0 伪标签」系统性偏差

### 2.4 Mask 生成（按前缀在线重算）

**关键**：训练时 step2 的 mask 必须以 **ground-truth step1** 为条件，不能用轨迹里存的完整 flat action 回填。

```rust
struct LegalActions {
    /// flat 三元组（完整合法动作列表）
    flat: Vec<(u8, u8, u8)>,  // (type, entity_slot, target_slot)
    /// 分步 mask（按当前观测 + 已选前缀在线计算）
    type_mask:   [bool; 18],
    /// entity 槽位 mask，dim = MAX_T (150)，非 24
    entity_mask: [bool; MAX_T],
    /// target 槽位 mask，dim = MAX_T (150)
    target_mask: [bool; MAX_T],  // 仅 Single 时非全 true
}
```

Python 侧 `(3, MAX_T)` bool tensor → 直接喂 decoder。

### 2.5 空 mask 保护

- `sum(type_mask) > 0` 恒成立（至少有 Proceed/EndTurn）
- 奖励屏末态：取完所有项自动 Proceed（加 `auto_proceed` flag）
- 选择屏 deadlock：anyNumber 屏必有 Confirm（0 张可选）

## 三、pyo3 接口

### 3.1 核心签名（修正版）

```rust
#[pyclass]
struct GymEnv { /* RunState + Combat */ }

#[pymethods]
impl GymEnv {
    #[new]
    fn new(seed: u64, ascension: i32, character: &str) -> Self;
    fn reset(&mut self) -> (Observation, LegalActions);
    fn step(&mut self, action_type: i32, entity: i32, target: i32)
        -> (Observation, LegalActions, f32, bool, Info);

    /// 观测：拆分 ids + feats + types（Python 做 embedding lookup）
    fn obs_token_types(&self) -> Py<PyArray1<i8>>;    // (150,)
    fn obs_token_ids(&self) -> Py<PyArray2<i32>>;     // (150, 3)
    fn obs_token_feats(&self) -> Py<PyArray2<f32>>;   // (150, F)
    fn obs_seq_len(&self) -> usize;

    /// 动作 mask：(3, 150) bool
    fn action_masks(&self) -> Py<PyArray2<bool>>;

    /// flat action list
    fn flat_actions(&self) -> Vec<(i32, i32, i32)>;
}
```

### 3.2 序列化策略

**flat array + 拷贝**（不做零拷贝 view）：
- 零拷贝 view 有别名/生命周期风险（Python 持有 view 到下个 step → Rust 可变借用冲突）
- 150 × (3×4 + 24×4 + 1) ≈ 150 × 112 = 16.8KB → 拷贝 ~2μs
- 端到端：engine step (~2μs) + fill (~1μs) + **copy (~2μs)** + pyo3 (~1μs) ≈ **6μs/step**
- 50K steps/s × 6μs = 0.3s → 不影响 P0 吞吐

### 3.3 Vocab

Rust 侧编译时生成 + hash 校验：

```rust
pub static CARD_VOCAB: &[&str] = &[...];   // 376（基础 ID，升级用 offset flag 而非独立 vocab）
pub static RELIC_VOCAB: &[&str] = &[...];   // 192
pub static MONSTER_VOCAB: &[&str] = &[...];  // 80
pub static POWER_VOCAB: &[&str] = &[...];    // 128
pub static POTION_VOCAB: &[&str] = &[...];   // 48
pub static RELIC_PERIOD: [u8; 192] = [...];  // 遗物周期表

pub fn vocab_hash() -> u64 { /* fnv or xxhash */ }
```

Python 启动时校验 hash，不匹配硬失败。

卡牌 vocab 376 而非 752：升级用 `base_id_emb + upgrade_offset_emb` 共享底 ID，样本效率翻倍。

## 四、辅助头（替代 recurrent）

| 辅助头 | 标签来源 | 预期收益 |
|---|---|---|
| predict next draw | 引擎（privileged） | 学牌堆 belief |
| 怪物下一招分布 | move_history + get_move 推演 | 学招式模式 |
| 下回合承受伤害 | 引擎前滚 1 回合 | 学防御时机 |
| 本回合是否 lethal | 引擎 | 极强 shaping（攻防切换） |
| 战终 HP delta | 引擎 | 长程 credit assignment |

## 五、与 Mortal / AlphaStar 对比

| 维度 | Mortal (麻将) | AlphaStar (星际) | 本方案 v2 (STS) |
|---|---|---|---|
| 观测 | 136 token | spatial + entity | **~150 token** |
| 隐藏信息 | 牌山不可见（历史相关） | 战争迷雾 | 抽牌堆顺序（**memoryless**） |
| 记忆 | LSTM（必须） | LSTM（必须） | **不需要**（前提：状态 Markov） |
| 动作 | ~14 | ~10^6 | 18 × 150 × 150（pointer） |
| 辅助任务 | 无 | 无 | **5 个**（含 lethal） |
| 序列长度 | 136 | 变长 | 150（bucketed pad） |

## 六、设计 trade-off 终版

1. **牌堆：逐张 vs bag-of-embeddings** → 选 bag（3 token/堆），概率用超几何标量直接给
2. **统一 vs 分屏** → 统一+mask（非战斗屏免费享受战斗预训练）
3. **原始值 vs 修正值** → 修正值（模型不用学乘区）
4. **RNN vs 纯 transformer** → 纯 transformer（STS 隐藏信息 memoryless）
5. **固定槽 vs pointer** → pointer-over-tokens（覆盖 PilePick 60+ 张候选）

## 七、v1.0 → v2.0 修订清单

| # | 修订项 | v1.0 | v2.0 |
|---|---|---|---|
| 1 | 缺 stance/orbs/计数器/pending | 无 | GLOBAL 24 维 + 10 个 orb token |
| 2 | PWR_PLAYER 8 槽不足 | 8 | **16**（长局 9-12 power 常见） |
| 3 | PWR_MOB 4 槽不足 | 4/怪 | **8**/怪（Boss 5+ power） |
| 4 | 牌堆单 token 直方图 | MLP 硬压缩 | **bag-of-emb + 3 token/堆** |
| 5 | 动作 24 固定槽 | 装不下 PilePick | **pointer-over-150-tokens** |
| 6 | 缺 Confirm/ToggleSelect | 多选屏 deadlock | 新增 2 个 type |
| 7 | intent 8 变体 | 错 | **11** 变体 |
| 8 | token 64 维 | 数学矛盾 | **拆 ids/feats，Python 查表** |
| 9 | max 80 token | 加总 93 溢出 | **MAX_T = 150** |
| 10 | 手牌 position emb | 出牌序歧义 | **不用**；改 drawn_this_turn 标志 |
| 11 | 地图行池化 | 丢失边信息 | **候选节点 + 路径摘要** |
| 12 | 遗物 counter 绝对值 | /99 压没差异 | **counter/period + ready** |
| 13 | 怪物无 move_history | 缺 | **近 3 招 + intent 11 变体** |
| 14 | 零拷贝 view | 别名风险 | **拷贝 ~2μs**（安全） |
| 15 | 卡牌 vocab 752 | 升级独占 vocab | **376 + upgrade offset**（样本×2） |
| 16 | damage clip /99 | Demon Form 100+ 失真 | **x/(x+30) 饱和** |
| 17 | 缺辅助头 | 仅 predict draw | **5 个**（含 lethal / HP delta） |

## 八、验收测试

1. **obs_roundtrip**：从 Observation 重建「抽牌堆多重集 + 手牌 + 怪物状态」，与引擎逐字段比对
2. **mask_consistency**：随机 rollout 1000 步，flat_actions 与三段 mask 展开一致、永不为空
3. **factorization_ce**：teacher-forcing 概率 = flat policy 概率（§2.4 前缀条件）
4. **vocab_stability**：hash 校验通过
5. **no_silent_overflow**：power/relic/orb 槽位溢出时 debug_assert 失败
