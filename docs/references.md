# 参考文献 & 项目

## 前辈 AI

| 项目 | 链接 | 要点 |
|---|---|---|
| AlphaGo | Silver et al., *Mastering the game of Go with deep neural networks and tree search*, Nature 2016 | SL→RL→MCTS 三段式范本 |
| AlphaZero | Silver et al. 2017 | 纯 self-play，无人类数据 |
| AlphaStar | Vinyals et al., Nature 2019 | 长时序、部分可观测、大规模并发的工程范本 |
| Mortal（日麻） | github.com/Equim/Mortal | 人类对局监督底座 + RL 微调 + Review 复盘工具；本项目方法论最近的亲戚 |
| Pluto（SC1） | github.com/tscmoo/pluto | 进程注入 + 共享内存推理；工程流天花板 |

## 杀戮尖塔生态

| 资源 | 链接 | 用途 |
|---|---|---|
| CommunicationMod | github.com/ForgottenArbiter/CommunicationMod | 官方生态标准桥：游戏与外部进程 JSON 通信 |
| spirecomm | github.com/ForgottenArbiter/spirecomm | CommunicationMod 的 Python 封装库 |
| ModTheSpire / BaseMod | GitHub 搜索同名仓库 | STS 模组加载器与框架，neow-bridge 的技术栈 |
| sts_lightspeed | github 搜索 `sts_lightspeed` | 现有最强 bot（启发式+搜索，A20 高胜率）：数据源、对拍参照、最终基线 |
| SlayTheSpireFightPredictor | github.com/ForgottenArbiter/SlayTheSpireFightPredictor | 路径概率/战斗预测思路参考 |
| gym-sts 等社区封装 | github 搜索 `gym slay the spire` | 社区 gym 封装，起步可参考 |

## 算法

- PPO: Schulman et al. 2017
- IMPALA (分布式 actor-learner): Espeholt et al. 2018
- 观测 masking / 结构化动作空间：V-MPO、AlphaStar 的自回归动作头
- 部分可观测（POMDP）与 belief 学习：Mortal 的做法（网络从牌河信息隐式推断）

### 现代化升级地图（2026 视角，按用途挑选）

**采纳：**

| 方法 | 解决什么 | 落点 |
|---|---|---|
| **Gumbel MuZero**（Danihelka 2022） | 极少模拟次数下的可靠根节点改进（sequential halving），比 PUCT 采样稳健得多 | P4 战斗搜索：单决策 ≤1s 预算下的首选根策略 |
| **EfficientZero 的 Reanalyze** | 旧轨迹用新网络重估，数据效率翻倍级提升 | P3 RL：模拟器数据不免费，回放重分析直接适用 |
| **Player-of-Games / ReBeL 思想** | 不完全信息下 belief 与 value 耦合搜索 | P4 诚实型层：可能世界按 belief 概率加权聚合（怪物招式概率树公开，天然可加权） |
| **离线 RL（IQL/CQL）** | 不与环境交互、直接从固定数据集学策略/价值 | P1→P3 衔接：从 lightspeed 日志冷启动（Mortal 的实际路线） |
| **Decision Transformer（实验向）** | 序列建模替代 RL | 备选实验：宏观构筑决策作为 return-conditioned 序列预测 |

**明确不采纳：**

| 方法 | 为什么不用 |
|---|---|
| MuZero 学习型 dynamics | 我们有精确模拟器，学出来的世界模型严格更差 |
| AlphaStar 式联盟训练 | 单机博弈无对手群 |
| 大规模分布式 AZ 自博弈 | 单人游戏 + 已有强底座（IL/搜索），性价比低 |
| sts-rl-agent | 链接待核实 | 分析报告引用其结论（终局奖励直训前馈网络失败；裸 MCTS 是战斗层够用解）；结论与本项目独立推理一致，引用待补 |

## 社区/讨论

- STS speedrun & A20 社区的路线知识（Jorbs 等主播的对局思路是免费的"人类棋谱"讲解）
- r/slaythespire、STS 中文社区的进阶攻略（构筑价值判断的先验）
