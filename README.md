<div align="center">

# Agent Trajectory Hub

**Agent 轨迹数据 Pipeline 编排层 - 串联全流程，产出可训练的数据集**
**Orchestrate the full pipeline: Task -> Sandbox -> Recorder -> Reward -> Export**

[![PyPI](https://img.shields.io/pypi/v/knowlyr-hub?color=blue)](https://pypi.org/project/knowlyr-hub/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![MCP](https://img.shields.io/badge/MCP-3_Tools-purple.svg)](#mcp-server)

[Quick Start](#quick-start) | [Pipeline Flow](#pipeline-flow) | [Export Formats](#export-formats) | [MCP Server](#mcp-server) | [ROADMAP](ROADMAP.md)

</div>

---

**GitHub Topics**: `agent-trajectory`, `pipeline`, `orchestrator`, `rl-data`, `sft`, `dpo`, `code-agent`

knowlyr 生态的编排层。调用 agent-sandbox、agent-recorder、agent-reward、data-check、data-label 等原子项目，产出训练就绪的数据集。

## Architecture / 架构

```
                  agent-trajectory-hub (编排层 / Orchestrator)
                            |
       +--------------------+--------------------+
       |                    |                    |
  Task Layer           Exec Layer          Value Layer
  (任务层)              (执行层)             (价值层)
       |                    |                    |
  +----+----+       +------+------+      +------+------+
  |         |       |      |      |      |      |      |
TaskSource Recipe  Sandbox Recorder Reward  SFT    DPO   Publish
(JSONL/    (复用)   (新建)  (新建)   (新建)  Export Export HuggingFace
 SWE-bench)                          |
                          +----------+----------+
                          |          |          |
                       Check      Synth      Label
                      (data-check)(data-synth)(data-label)
```

### 项目调用关系 / Project Dependencies

| 原子项目 | PyPI 包名 | 在 Hub 中的角色 |
|----------|-----------|----------------|
| **agent-sandbox** | `knowlyr-sandbox` | 可复现的代码执行环境 (Docker 沙箱) |
| **agent-recorder** | `knowlyr-recorder` | 标准化轨迹录制 (拦截 Agent <-> Sandbox 交互) |
| **agent-reward** | `knowlyr-reward` | 过程级 Reward 计算 (规则层 + 模型层 + 人工校准) |
| **data-check** | `knowlyr-datacheck` | 轨迹数据质检 (规则验证、重复检测) |
| **data-label** | `knowlyr-datalabel` | 偏好对的人工标注 + IAA 一致性验证 |
| **data-synth** | `knowlyr-datasynth` | Reward 模型层的 LLM-as-Judge |

## Pipeline Flow / 流水线流程

```
1. Load Tasks          从 JSONL / SWE-bench 加载任务列表
       |
2. For each (Task x Agent):
       |
   2a. Create Sandbox  创建 Docker 沙箱环境 (agent-sandbox)
       |
   2b. Run Agent       在沙箱中运行 Agent (OpenHands / SWE-agent)
       |
   2c. Record          录制执行轨迹 (agent-recorder)
       |
   2d. Score           计算过程级 Reward (agent-reward)
       |
3. Build Pairs         构建偏好对 (同任务多轨迹按 reward 排序)
       |
4. Quality Check       运行数据质检 (data-check)
       |
5. Export              导出为 SFT / DPO / Benchmark 格式
```

## Installation / 安装

```bash
pip install knowlyr-hub
```

可选依赖：

```bash
pip install knowlyr-hub[sandbox]    # 沙箱环境
pip install knowlyr-hub[recorder]   # 轨迹录制
pip install knowlyr-hub[reward]     # Reward 计算
pip install knowlyr-hub[check]      # 数据质检
pip install knowlyr-hub[mcp]        # MCP 服务器
pip install knowlyr-hub[all]        # 全部功能
```

## Quick Start / 快速开始

### CLI

```bash
# 运行完整 Pipeline
knowlyr-hub run tasks.jsonl -o ./output -f openhands -m claude-sonnet-4-20250514

# 从 checkpoint 恢复
knowlyr-hub run tasks.jsonl -o ./output --resume ./output/checkpoint.json

# 查看状态
knowlyr-hub status ./output

# 列出任务
knowlyr-hub tasks tasks.jsonl --language python --difficulty medium

# 导出为 SFT 格式
knowlyr-hub export --format sft -t ./output/trajectories.jsonl -o ./export/sft_train.jsonl

# 导出为 DPO 格式
knowlyr-hub export --format dpo -t ./output/trajectories.jsonl -p ./output/preferences.jsonl -o ./export/dpo_train.jsonl

# 发布到 HuggingFace
knowlyr-hub publish -t ./output/trajectories.jsonl --repo-id username/my-dataset --generate-card
```

### Python API

```python
from trajectoryhub import Pipeline, PipelineConfig
from trajectoryhub.config import TaskSource, AgentConfig

config = PipelineConfig(
    task_source=TaskSource(path="tasks.jsonl"),
    agents=[
        AgentConfig(framework="openhands", model="claude-sonnet-4-20250514"),
        AgentConfig(framework="openhands", model="gpt-4o"),
    ],
    output_dir="./output",
    parallel_workers=4,
)

pipeline = Pipeline(config)
result = pipeline.run()

print(f"完成: {result.completed}/{result.total_tasks}")
print(f"轨迹: {result.trajectories_path}")
print(f"偏好对: {result.preferences_path}")
```

### 导出数据集

```python
from trajectoryhub import DatasetExporter

exporter = DatasetExporter(
    trajectories_dir="./output/trajectories.jsonl",
    preferences_dir="./output/preferences.jsonl",
)

# SFT 格式
exporter.export_sft("./export/sft_train.jsonl")

# DPO 格式
exporter.export_dpo("./export/dpo_train.jsonl")

# 评测基准
exporter.export_benchmark("./export/benchmark.jsonl")

# 生成 Dataset Card
card = exporter.generate_datacard()
```

## Export Formats / 导出格式

### SFT Format (监督微调)

```jsonc
// 每行一个 JSON
{
    "instruction": "Fix the bug in parser module",
    "input": "{\"repo\": \"owner/repo\", \"base_commit\": \"abc123\", ...}",
    "response": "Step 1:\nThought: Read the file\nAction: file_read /test.py\n...",
    "task_id": "repo__issue-123",
    "reward": 0.85,
    "metadata": {"agent_framework": "openhands", "agent_model": "claude-sonnet-4-20250514", "total_steps": 5}
}
```

### DPO Format (偏好学习)

```jsonc
// 每行一个 JSON
{
    "prompt": "Solve the following task:\n\nTask ID: repo__issue-123",
    "chosen": "Step 1:\nThought: ...\nAction: ...\n...",
    "rejected": "Step 1:\nThought: ...\nAction: ...\n...",
    "task_id": "repo__issue-123",
    "reward_margin": 0.55,
    "metadata": {
        "chosen_model": "claude-sonnet-4-20250514",
        "rejected_model": "gpt-4o",
        "chosen_reward": 0.85,
        "rejected_reward": 0.30
    }
}
```

### Benchmark Format (评测基准)

```jsonc
{
    "task_id": "repo__issue-123",
    "description": "Fix the bug in parser module",
    "repo": "owner/repo",
    "base_commit": "abc123",
    "test_command": "pytest tests/test_parser.py",
    "reference_trajectories": [...],
    "difficulty": "medium",
    "expected_reward_range": [0.3, 0.85]
}
```

## MCP Server / Claude Integration

在 Claude Desktop / Claude Code 中直接使用。

### 配置 / Config

添加到 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "knowlyr-hub": {
      "command": "uv",
      "args": ["--directory", "/path/to/agent-trajectory-hub", "run", "python", "-m", "trajectoryhub.mcp_server"]
    }
  }
}
```

### 可用工具 / Tools

| 工具 | 功能 |
|------|------|
| `run_pipeline` | 运行完整 Pipeline (Task -> Sandbox -> Recorder -> Reward -> Export) |
| `export_dataset` | 导出为指定格式 (SFT / DPO / Benchmark / HuggingFace) |
| `pipeline_status` | 查看 Pipeline 执行状态和进度 |

---

## 命令参考 / Command Reference

| 命令 | 功能 |
|------|------|
| `knowlyr-hub run <tasks>` | 运行完整 Pipeline |
| `knowlyr-hub export --format <fmt>` | 导出数据集 |
| `knowlyr-hub status <dir>` | 查看 Pipeline 状态 |
| `knowlyr-hub tasks <source>` | 列出/过滤任务 |
| `knowlyr-hub publish` | 发布到 HuggingFace |

### Run 选项

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `-o, --output` | 输出目录 | `./output` |
| `-f, --framework` | Agent 框架 | `openhands` |
| `-m, --model` | LLM 模型 | `claude-sonnet-4-20250514` |
| `--max-steps` | 最大步数 | `30` |
| `-w, --workers` | 并行数 | `1` |
| `--resume` | 从 checkpoint 恢复 | - |

---

## 项目架构 / Project Structure

```
src/trajectoryhub/
├── __init__.py      # 包入口
├── config.py        # Pipeline 配置 (Pydantic models)
├── pipeline.py      # 核心编排器 (Pipeline + PipelineResult)
├── tasks.py         # 任务加载与管理 (Task + TaskLoader)
├── exporter.py      # 数据集导出 (SFT / DPO / Benchmark / HuggingFace)
├── cli.py           # CLI 命令行 (Click)
└── mcp_server.py    # MCP Server (3 tools)
```

---

## knowlyr 生态 / Ecosystem

> 覆盖 Agent 轨迹数据全流程，均支持 CLI + MCP，可独立使用也可通过 Hub 编排。

| 项目 | 功能 | PyPI |
|------|------|------|
| **Agent Trajectory Hub** | Pipeline 编排层 | `knowlyr-hub` (You are here) |
| **Agent Sandbox** | 代码执行沙箱 | `knowlyr-sandbox` |
| **Agent Recorder** | 轨迹录制 | `knowlyr-recorder` |
| **Agent Reward** | 过程级 Reward | `knowlyr-reward` |
| **DataCheck** | 数据质检 | `knowlyr-datacheck` |
| **DataSynth** | 数据合成 | `knowlyr-datasynth` |
| **DataLabel** | 数据标注 | `knowlyr-datalabel` |
| **DataRecipe** | 数据集分析 | `knowlyr-datarecipe` |
| **AI Dataset Radar** | 数据集监控 | `knowlyr-radar` |

```
Hub (编排) -> Sandbox (执行) -> Recorder (录制) -> Reward (打分) -> Export (导出)
                                                      |
                                          Check (质检) + Synth (Judge) + Label (标注)
```

---

## License

[MIT](LICENSE)

---

<div align="center">
<sub>面向 Code Agent 的 RL 环境，产出带过程级 Reward 的执行轨迹数据</sub>
</div>
