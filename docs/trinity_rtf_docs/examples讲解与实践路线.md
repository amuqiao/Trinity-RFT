# Trinity-RFT `examples` 讲解与实践路线

> 目标：解释 `examples` 在项目中的作用、覆盖能力边界，以及如何按“从易到难”逐步上手。

---

## 1. `examples` 是什么

`examples` 是 Trinity-RFT 的“可运行能力地图”。  
每个子目录通常包含：

- 一份或多份 `yaml` 配置（定义训练模式、算法、数据、模型、同步策略）；
- 可选 `README.md`（场景说明、依赖、运行命令）；
- 可选脚本/工作流代码（用于数据准备、评估、定制 workflow）。

它不是简单的 demo 集合，而是把框架核心能力（算法、数据流水线、多轮 Agent、多模态、异步模式、后端切换）映射成可复现模板。

---

## 2. `examples` 分类总览（按能力）

```mermaid
flowchart LR
    %% 样式定义
    classDef rootStyle fill:#ffd700,stroke:#333,stroke-width:2px
    classDef basicStyle fill:#9ff,stroke:#333,stroke-width:2px
    classDef algoStyle fill:#9f9,stroke:#333,stroke-width:2px
    classDef agentStyle fill:#ff9,stroke:#333,stroke-width:2px
    classDef dataStyle fill:#f99,stroke:#333,stroke-width:2px
    classDef infraStyle fill:#f9f,stroke:#333,stroke-width:2px
    classDef subgraphStyle fill:#f5f5f5,stroke:#666,stroke-width:1px,rounded:10px

    A[examples/]:::rootStyle

    subgraph basic["基础入门"]
        B1[grpo_gsm8k]:::basicStyle
        B2[ppo_countdown]:::basicStyle
        B3[sft_mot / dpo_humanlike]:::basicStyle
    end
    class basic subgraphStyle

    subgraph algo["算法研究"]
        C1[mix_chord / mix_math]:::algoStyle
        C2[rec_gsm8k / topr_gsm8k]:::algoStyle
        C3[asymre / cispo / sppo / opmd]:::algoStyle
    end
    class algo subgraphStyle

    subgraph data["数据流水线"]
        D1[grpo_gsm8k_task_pipeline]:::dataStyle
        D2[grpo_gsm8k_experience_pipeline]:::dataStyle
        D3[dpo_human_in_the_loop]:::dataStyle
    end
    class data subgraphStyle

    subgraph agent["Agent与环境交互"]
        E1[agentscope_react / tool_react]:::agentStyle
        E2[agentscope_websearch]:::agentStyle
        E3[grpo_alfworld / webshop / sciworld]:::agentStyle
    end
    class agent subgraphStyle

    subgraph infra["运行模式与后端"]
        F1[async_gsm8k]:::infraStyle
        F2[tinker]:::infraStyle
        F3[ppo_countdown_megatron]:::infraStyle
        F4[grpo_vlm / mix_vlm]:::infraStyle
    end
    class infra subgraphStyle

    A --> B1
    A --> C1
    A --> D1
    A --> E1
    A --> F1

    linkStyle 0,1,2,3,4 stroke:#333,stroke-width:2px,arrowheadStyle:filled
```

---

## 3. 代表示例与用途

- `grpo_gsm8k`：最小主线示例，适合理解标准 `mode=both` 闭环。
- `async_gsm8k`：将 Explorer 与 Trainer 分离为两个配置（`explore` / `train`），用于异步训练。
- `grpo_gsm8k_task_pipeline`：演示 task pipeline 在训练前做任务筛选/重排。
- `grpo_gsm8k_experience_pipeline`：演示 experience pipeline 在训练中做奖励重塑。
- `mix_chord`：SFT+RL 混合算法实践，偏研究场景。
- `agentscope_websearch`：多轮 ReAct + 外部搜索工具，偏 Agentic workflow。
- `grpo_vlm`：视觉语言模型训练，偏多模态扩展。
- `tinker`：无 GPU 场景的后端替代方案（有功能边界）。
- `learn_to_ask` / `bots`：更完整的论文级实验模板（含数据准备和评估流程）。

---

## 4. 从 `yaml` 到训练闭环（关键流程）

```mermaid
flowchart LR
    %% 样式定义
    classDef inputStyle fill:#f9f,stroke:#333,stroke-width:2px
    classDef parseStyle fill:#ffd700,stroke:#333,stroke-width:2px
    classDef exploreStyle fill:#9ff,stroke:#333,stroke-width:2px
    classDef dataStyle fill:#9f9,stroke:#333,stroke-width:2px
    classDef trainStyle fill:#ff9,stroke:#333,stroke-width:2px
    classDef syncStyle fill:#f99,stroke:#333,stroke-width:2px
    classDef noteStyle fill:#fff8e6,stroke:#ffb74d,stroke-width:1px,rounded:8px
    classDef subgraphStyle fill:#f5f5f5,stroke:#666,stroke-width:1px,rounded:10px

    subgraph layer1["配置与启动"]
        A[examples/*/*.yaml]:::inputStyle
        B[trinity run --config ...]:::inputStyle
        C[Config 解析与校验]:::parseStyle
    end
    class layer1 subgraphStyle

    subgraph layer2["运行闭环"]
        D[Explorer<br/>任务探索]:::exploreStyle
        E[WorkflowRunner<br/>执行workflow]:::exploreStyle
        F[ExperiencePipeline<br/>经验处理]:::dataStyle
        G[ExperienceBuffer]:::dataStyle
        H[Trainer<br/>采样训练]:::trainStyle
        I[Synchronizer<br/>权重同步]:::syncStyle
    end
    class layer2 subgraphStyle

    A -->|指定模式/算法/数据| B
    B -->|加载| C
    C -->|创建Actor| D
    D -->|调度任务| E
    E -->|产出经验| F
    F -->|写入| G
    H -->|读取采样| G
    H -->|更新权重| I
    D -->|拉取版本| I

    linkStyle 0,1,2,3,4,5,6 stroke:#333,stroke-width:2px,arrowheadStyle:filled
    linkStyle 7,8 stroke:#4299e1,stroke-width:2px,arrowheadStyle:filled

    Note[配置是入口：<br/>同一套代码通过 yaml 切换算法、数据流、同步策略和运行模式。]:::noteStyle
    Note -.-> C
```

---

## 5. 两种“数据增强”思路：Task Pipeline vs Experience Pipeline

```mermaid
flowchart LR
    %% 样式定义
    classDef sourceStyle fill:#f9f,stroke:#333,stroke-width:2px
    classDef taskPipeStyle fill:#9ff,stroke:#333,stroke-width:2px
    classDef expPipeStyle fill:#9f9,stroke:#333,stroke-width:2px
    classDef coreStyle fill:#ff9,stroke:#333,stroke-width:2px
    classDef noteStyle fill:#fff8e6,stroke:#ffb74d,stroke-width:1px,rounded:8px
    classDef subgraphStyle fill:#f5f5f5,stroke:#666,stroke-width:1px,rounded:10px

    subgraph taskPath["任务前处理路径（Task Pipeline）"]
        A[原始Taskset]:::sourceStyle
        B[Task Operators<br/>过滤/排序/难度评估]:::taskPipeStyle
        C[Explorer]:::coreStyle
    end
    class taskPath subgraphStyle

    subgraph expPath["经验后处理路径（Experience Pipeline）"]
        D[Explorer输出Experience]:::sourceStyle
        E[Experience Operators<br/>打分/奖励塑形]:::expPipeStyle
        F[Trainer]:::coreStyle
    end
    class expPath subgraphStyle

    A -->|训练前重构任务分布| B --> C
    D -->|训练中重构学习信号| E --> F

    linkStyle 0,1,2,3 stroke:#333,stroke-width:2px,arrowheadStyle:filled

    Note[关键区别：<br/>Task Pipeline 改“输入任务”；Experience Pipeline 改“训练信号”。]:::noteStyle
    Note -.-> B
    Note -.-> E
```

---

## 6. 异步模式示例（`async_gsm8k`）关键流程

```mermaid
flowchart LR
    %% 样式定义
    classDef cfgStyle fill:#ffd700,stroke:#333,stroke-width:2px
    classDef expStyle fill:#9ff,stroke:#333,stroke-width:2px
    classDef trainStyle fill:#ff9,stroke:#333,stroke-width:2px
    classDef storeStyle fill:#9f9,stroke:#333,stroke-width:2px
    classDef syncStyle fill:#f99,stroke:#333,stroke-width:2px
    classDef subgraphStyle fill:#f5f5f5,stroke:#666,stroke-width:1px,rounded:10px

    subgraph explorerSide["Explorer 侧（mode=explore）"]
        A[explorer.yaml]:::cfgStyle
        B[Explorer 进程]:::expStyle
    end
    class explorerSide subgraphStyle

    subgraph trainerSide["Trainer 侧（mode=train）"]
        C[trainer.yaml]:::cfgStyle
        D[Trainer 进程]:::trainStyle
    end
    class trainerSide subgraphStyle

    E[共享ExperienceBuffer]:::storeStyle
    F[Synchronizer<br/>checkpoint sync]:::syncStyle

    A --> B
    C --> D
    B -->|持续写入经验| E
    D -->|持续读取经验| E
    D -->|发布checkpoint版本| F
    B -->|拉取最新权重| F

    linkStyle 0,1,2,3 stroke:#333,stroke-width:2px,arrowheadStyle:filled
    linkStyle 4,5 stroke:#4299e1,stroke-width:2px,arrowheadStyle:filled
```

---

## 7. 推荐实践顺序（新同学）

1. 先跑 `grpo_gsm8k/gsm8k.yaml`，理解最小闭环；
2. 再跑 `grpo_gsm8k_task_pipeline` 与 `grpo_gsm8k_experience_pipeline`，理解数据处理位置差异；
3. 再看 `async_gsm8k`，理解生产/消费解耦；
4. 然后选择方向：
   - 算法方向：`mix_chord`、`rec_gsm8k`、`topr_gsm8k`
   - Agent 方向：`agentscope_websearch`、`grpo_alfworld`
   - 基础设施方向：`tinker`、`grpo_vlm`

---

## 8. 常见误区

- 只改算法参数，不核对 `buffer` 与 `synchronizer`，容易导致吞吐不匹配；
- 把 Task Pipeline 与 Experience Pipeline 混为一谈，导致调参方向错误；
- 异步模式下先停 Explorer，可能导致 Trainer 侧读取不足（建议按示例说明先启动 Trainer）；
- 没有从 `README` 补齐外部依赖（例如 AgentScope/MCP、Tinker API Key）。

