# Neow — 杀戮尖塔（Slay the Spire）AI

> **Neow（涅奥）**：尖塔底部的巨鲸之神。玩家每一次死亡，都是 Neow 重新开启一局并赐下新的祝福——
> 她就是 self-play 循环的守护神。

目标：像 AlphaGo 之于围棋、Mortal 之于日麻、Pluto 之于星际1 那样，
**在杀戮尖塔 A20（进阶20）上达到或超越人类顶尖水平**的 AI。

## 项目布局（monorepo，子目录即未来仓库名）

```
Neow/                    # GitHub 私有仓库：No-World/Neow
├── neow-bridge/         # Java：STS 模组 NeowBridge（fork 自 CommunicationMod）——真游戏观测/控制/对拍裁判
├── neow-sim/            # Rust：高速确定性规则引擎——训练主力，独立可用
├── neow-ai/             # Python：AI 本体——IL / RL / 战斗内搜索 / 评测
├── docs/                # 总体文档 + 统一 gym 协议（protocol.md，三者接口真相源）+ 译名对照表（glossary.md）
└── .github/workflows/   # CI：按子项目路径触发，自动构建各自互不干扰
```

依赖方向：`neow-sim` 零依赖；`neow-bridge` ← fork CommunicationMod；`neow-ai` → `neow-sim`（训练）+ `neow-bridge`（评测/演示）。

管理策略：当前 monorepo 便于联调；子目录已按独立仓库命名并各自带 README/构建系统，
未来拆分时 `git filter-repo`（或 subtree split）即可各自成仓，CI workflow 已按路径隔离、随目录平移。

## 产品原则：游戏内形态（硬约束）

最终在游戏内运行的 Neow 必须是“公平玩家”：

1. **无 S/L**：一次连续对局，绝不通过存档/读档重试、探信息或反悔决策
2. **非穷举**：推理预算受限（单决策 ≤1s 量级）——采样 + 神经网络直觉 + 有界浅搜索，不做全分支枚举，区别于暴力 bot
3. **不窥探**：观测只来自玩家合法可知的信息（protocol.md 的隐藏信息规则），不读内存偷看抽牌堆顺序
4. 一切对外成绩（榜单/直播/Review）以游戏内形态为准；离线研究允许放宽预算，但必须另列

## 前辈与启示

| 前辈 | 游戏 | 核心思路 | 对 Neow 的启示 |
|---|---|---|---|
| **AlphaGo / AlphaZero** | 围棋 | 人类棋谱监督 → RL 自博弈 → MCTS 推理增强 | 三段式主路线 |
| **Mortal** (Equim) | 日本麻将 | 天凤人类对局监督底座 → RL 微调 → 推理期可配搜索 + Review 复盘工具 | 数据底座最重要；复盘工具是出圈点 |
| **Pluto** (tscmoo) | 星际争霸1 | 工程流天花板：DLL 寄生游戏进程、共享内存推理、极限微操 | 工程能力决定上限，环境对接是第一战场 |

## 核心认知：STS 与前辈们的关键差异

1. **单机博弈**：没有对手 → 没有对抗 self-play，只有"对环境的自迭代"；对手是随机性（洗牌、地图、事件）。
2. **部分可观测**：抽牌堆顺序、地图未探索分支、事件选项未知 → 需要 belief（信念状态）。
3. **超长时序**：一局 ≈ 40 场战斗 + 地图/商店/事件决策，上千个决策点，奖励只在终局 → 信用分配难。
4. **组合爆炸的实体状态**：卡组构成随构筑演化 → 必须用实体集合编码，不能用固定 one-hot。
5. **真游戏太慢**：RL 需要百万级对局，官方客户端跑不动 → 必须自建高速模拟器（这正是 sts_lightspeed 的做法）。

## 路线图（详见 [docs/roadmap.md](docs/roadmap.md)）

- [ ] **P0** 高速模拟器（`neow-sim/`）：Rust 复刻战斗与地图规则，M1→M3 整局闭环 + 规则级单测
- [ ] **P1** 模仿学习底座（`neow-ai/`）：sts_lightspeed 蒸馏，policy + value 网络
- [ ] **P2** 环境桥接与对拍（`neow-bridge/`）：fork CommunicationMod，M4 对拍锁定模拟器正确性——大规模 RL 前的最后关卡
- [ ] **P3** 强化学习：PPO 自迭代，超越老师
- [ ] **P4** 战斗内搜索：洗牌 determinization + policy/value 剪枝的浅 MCTS
- [ ] **P5** 评测与出圈：固定种子 A20 心脏击杀率（以游戏内形态计量：无S/L、非穷举）+ Mortal 式 Review 工具

## CI（GitHub Actions）

- `sim.yml`：neow-sim 的 fmt / clippy / test / release 构建
- `bridge.yml`：neow-bridge 的资源校验（完整构建需游戏 jar，见工作流内注释）
- `ai.yml`：neow-ai 的 lint / 语法检查

## 第一步

1. `neow-sim/`：骨架已就绪（`cargo build` 即可编译），直接开工 M2——“Strike/Defend/Bash 打 JawWorm”最小战斗闭环。
2. 克隆 `sts_lightspeed`，读它的战斗模拟代码，对照实现伤害/格挡/力量管线。
3. `neow-bridge/`（模组）后置到 P2 再动工，期间不需要装游戏。
