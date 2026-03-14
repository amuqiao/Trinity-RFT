有，而且这仓库里相关内容是比较完整的。你可以按下面这条线看，能把 **SFT、RFT、Agent-RFT** 一次串起来。

## 先回答你的3个核心问题

- **项目里有 SFT 吗？** 有。标准离线 SFT 示例是 `examples/sft_mot/sft.yaml`，文档在 `docs/sphinx_doc/source/tutorial/example_dpo.md`（同页含 SFT 配置说明）。
- **项目里有 RFT 吗？** 有，且是主线能力。入门示例 `examples/grpo_gsm8k/gsm8k.yaml`，文档 `docs/sphinx_doc/source/tutorial/example_reasoning_basic.md`。
- **有“基于 Agent 做 RFT”吗？** 有，且是重点能力。示例 `examples/agentscope_react/gsm8k.yaml`、`examples/agentscope_websearch/agentscopev1_websearch_agent.yaml`，文档 `docs/sphinx_doc/source/tutorial/example_react.md`、`docs/sphinx_doc/source/tutorial/example_multi_turn.md`、`docs/sphinx_doc/source/tutorial/example_step_wise.md`。

---

## SFT vs RFT 在 Trinity 里的流程差异（你要抓住的主干）

- **SFT（离线监督）**
  - 典型配置：`mode: train` + `algorithm_type: sft`
  - 数据入口：`buffer.trainer_input.experience_buffer`（直接喂训练集）
  - 无 Explorer 采样闭环，主要是 Trainer 读取数据训练
  - 代表：`examples/sft_mot/sft.yaml`

- **RFT（强化微调）**
  - 典型配置：`mode: both` + `algorithm_type: grpo`（或其他 RL 算法）
  - Explorer 先跑 workflow 生成 experience -> Buffer -> Trainer 更新参数 -> Synchronizer 同步权重
  - 代表：`examples/grpo_gsm8k/gsm8k.yaml`

- **Agent-RFT（多轮交互强化）**
  - workflow 里驱动 agent 与环境多轮交互，自动收集 history，转成 experience 再训练
  - 常见算法：`multi_step_grpo`
  - 关键配置：`enable_openai_api`、`enable_history`、`default_workflow_type: agentscope_react_workflow`
  - 代表：`examples/agentscope_react/gsm8k.yaml`

---

## “在 agent 基础上做 SFT + RFT”怎么理解

当前仓库里**现成模板主要是 Agent-RFT**；**Agent-SFT 没有单独的现成样例**，但有两条可行路径：

- **路径A（推荐）先 SFT 再 Agent-RFT**
  - 先用 `examples/sft_mot/sft.yaml` 做 SFT 基座；
  - 再用 Agent RFT 示例继续强化（如 `examples/agentscope_react/gsm8k.yaml`）。
  - 仓库也给了阶段化思路：在 RFT 配置里加入 `stages` 先跑 `sft_warmup`，再跑 `rft`（`examples/grpo_gsm8k/gsm8k.yaml` 里有注释模板）。

- **路径B 用 agent 轨迹做离线 SFT**
  - 把 agent 对话轨迹整理成 `messages` 格式数据；
  - 用 `mode: train` + `algorithm_type: sft` 训练。
  - 这条路线需要你自己准备数据，但框架支持。

---

## 你应该看哪些内容（按顺序）

1. **总览与对照**
   - `docs/sphinx_doc/source/tutorial/example_dataset_perspective.md`
   - 先看这个能快速知道“哪个数据集/任务对应哪个 example”。

2. **SFT 主线**
   - `docs/sphinx_doc/source/tutorial/example_dpo.md`（里面有 SFT 配置段）
   - `examples/sft_mot/sft.yaml`

3. **RFT 主线**
   - `docs/sphinx_doc/source/tutorial/example_reasoning_basic.md`
   - `examples/grpo_gsm8k/gsm8k.yaml`

4. **Agent-RFT 主线**
   - `docs/sphinx_doc/source/tutorial/example_react.md`
   - `examples/agentscope_react/gsm8k.yaml`
   - `trinity/common/workflows/agentscope/react/react_workflow.py`

5. **多轮/环境交互进阶**
   - `docs/sphinx_doc/source/tutorial/example_multi_turn.md`
   - `docs/sphinx_doc/source/tutorial/example_step_wise.md`
   - `examples/grpo_alfworld/alfworld.yaml`
   - `examples/grpo_alfworld_general_multi_step/alfworld.yaml`

6. **自定义 agent workflow（你后续自己改造时看）**
   - `docs/sphinx_doc/source/tutorial/develop_workflow.md`
   - `docs/sphinx_doc/source/tutorial/develop_overview.md`
   - `trinity/common/workflows/__init__.py`（workflow 注册点）

---

## 给你的最短实践路线（建议）

- 第1步：跑通 `examples/sft_mot/sft.yaml`（确认 SFT 训练链路）
- 第2步：跑通 `examples/grpo_gsm8k/gsm8k.yaml`（确认 RFT 闭环）
- 第3步：跑通 `examples/agentscope_react/gsm8k.yaml`（确认 Agent-RFT）
- 第4步：把你的 agent 接到自定义 workflow，按 `develop_workflow` 改 `default_workflow_type`

如果你愿意，我可以下一步直接给你一份“**你的场景专用** SFT->Agent-RFT 配置迁移清单”（按字段逐项对照，改哪几个键就能跑）。