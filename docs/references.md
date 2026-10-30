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

## 社区/讨论

- STS speedrun & A20 社区的路线知识（Jorbs 等主播的对局思路是免费的"人类棋谱"讲解）
- r/slaythespire、STS 中文社区的进阶攻略（构筑价值判断的先验）
