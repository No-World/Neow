# neow-ai — Neow AI 本体（Python）

深度学习子项目：模仿学习底座 → 强化学习 → 战斗内搜索 → 评测。

## 目录

- `env/`：薄适配层——统一 gym 接口，后端二选一：
  - `neow-sim` 的 Python wheel（训练主力，海量并发）
  - `neow-bridge` + 真游戏（最终评测 / 对拍 / 演示）
- `data/`：对局数据管道（lightspeed 蒸馏数据、RL buffer 落盘）
- `models/`：网络结构与 checkpoint
- `training/`：IL 预训练、PPO 自迭代
- `inference/`：战斗内搜索（determinization + 浅 MCTS）
- `evaluation/`：固定种子集评测、榜单
- `scripts/`：杂项

协议见 `../docs/protocol.md`；路线图见 `../docs/roadmap.md`（P2~P5 在本目录）。
