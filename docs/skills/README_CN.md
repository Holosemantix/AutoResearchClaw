# AutoResearchClaw Skill 系统梳理与编写样例

这份 README 总结 AutoResearchClaw 仓库里的 skill 体系：skill 放在哪里、`SKILL.md` 如何写、系统如何加载和匹配、内置 skill 覆盖哪些阶段，以及如何写一个可直接使用的自定义中文样例。

本文基于以下文件阅读整理：

- `README.md` 中的 Skills Library 说明
- `.claude/skills/*/SKILL.md`
- `researchclaw/skills/builtin/**/SKILL.md`
- `researchclaw/skills/schema.py`
- `researchclaw/skills/loader.py`
- `researchclaw/skills/registry.py`
- `researchclaw/skills/matcher.py`
- `researchclaw/pipeline/_helpers.py`
- `researchclaw/metaclaw_bridge/lesson_to_skill.py`
- `tests/test_skills_library.py`

## 核心结论

AutoResearchClaw 的 skill 是面向 23-stage autonomous research pipeline 的轻量知识注入单元。每个 skill 用一个 `SKILL.md` 表达：

- 什么时候触发；
- 属于哪个类别；
- 适用于哪些 pipeline stage；
- 优先级如何；
- agent 应该遵循的具体步骤、模板或反模式。

运行时，系统会根据当前 stage 和 topic/context 自动匹配若干 skill，把内容注入 LLM prompt，不需要用户手动启用。

## Skill 目录来源

AutoResearchClaw 当前有四类主要 skill 来源。

| 来源 | 路径 | 用途 |
| --- | --- | --- |
| 内置 skill | `researchclaw/skills/builtin/` | 包内默认加载，覆盖写作、实验、工具、领域知识 |
| 项目级 skill | `.claude/skills/` | 当前仓库自带，面向 Claude/OpenCode/项目运行上下文 |
| 用户级 skill | `~/.researchclaw/skills/` | 用户通过 `researchclaw skills install` 安装，跨项目复用 |
| MetaClaw learned skill | `~/.metaclaw/skills/` | 从历史失败 lesson 自动生成，跨运行注入 |

此外，`config.arc.yaml` 或 `config.researchclaw.example.yaml` 可以配置：

```yaml
skills:
  enabled: true
  custom_dirs:
    - /path/to/team-shared-skills
  external_dirs:
    - /path/to/external-agent-skills
  max_skills_per_stage: 3
  fallback_matching: true
```

加载顺序在 `researchclaw/pipeline/_helpers.py` 中定义：

1. built-in skills；
2. `~/.researchclaw/skills/`；
3. `.claude/skills/`；
4. `~/.metaclaw/skills/`；
5. config 中的 `custom_dirs` 和 `external_dirs`。

如果多个来源出现同名 skill，后注册的 skill 会覆盖前一个。因此项目级或用户级 skill 可以有意覆盖内置 skill。

## `SKILL.md` 格式

AutoResearchClaw 使用 agentskills.io 风格的 Markdown 文件，前面是 YAML frontmatter，后面是正文。

最小格式：

```markdown
---
name: my-skill
description: One-line description explaining when this skill should be used.
metadata:
  category: experiment
  trigger-keywords: "keyword1,keyword2,keyword3"
  applicable-stages: "9,10,12"
  priority: "3"
  version: "1.0"
  author: your-team
---

## Main Guidance

1. Concrete instruction.
2. Concrete validation step.
3. Common anti-pattern to avoid.
```

字段含义：

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| `name` | 是 | skill ID。建议 lowercase-kebab-case |
| `description` | 是 | 一句话说明能力和触发场景 |
| `metadata.category` | 建议 | 分类：`writing`、`domain`、`experiment`、`tooling` |
| `metadata.trigger-keywords` | 建议 | 逗号分隔关键词，用于 context 匹配 |
| `metadata.applicable-stages` | 建议 | 逗号分隔 stage 编号；空则适用于所有 stage |
| `metadata.priority` | 建议 | 数字越小优先级越高 |
| `metadata.version` | 建议 | 版本号，便于维护 |
| `metadata.code-template` | 可选 | 可注入 prompt 的代码模板 |
| `metadata.references` | 可选 | 参考来源，多个来源可用分号分隔 |
| `enabled: false` | 可选 | 禁用某个 skill |

`loader.py` 会读取 `SKILL.md`，把 `metadata` 展平到 `Skill.metadata`。`schema.py` 再提供兼容属性，例如：

- `skill.category`
- `skill.trigger_keywords`
- `skill.applicable_stages`
- `skill.priority`
- `skill.references`
- `skill.code_template`

## 23 个 Pipeline Stage

`researchclaw/skills/schema.py` 把 stage name 映射到编号：

| 编号 | Stage name | 主要任务 |
| --- | --- | --- |
| 1 | `topic_init` | 初始化研究主题 |
| 2 | `problem_decompose` | 拆解问题 |
| 3 | `search_strategy` | 文献检索策略 |
| 4 | `literature_collect` | 收集文献 |
| 5 | `literature_screen` | 筛选文献 |
| 6 | `knowledge_extract` | 抽取知识 |
| 7 | `synthesis` | 综合已有发现 |
| 8 | `hypothesis_gen` | 生成假设 |
| 9 | `experiment_design` | 实验设计 |
| 10 | `code_generation` | 生成实验代码 |
| 11 | `resource_planning` | 资源规划 |
| 12 | `experiment_run` | 执行实验 |
| 13 | `iterative_refine` | 迭代修复和改进 |
| 14 | `result_analysis` | 结果分析 |
| 15 | `research_decision` | PROCEED / REFINE / PIVOT 决策 |
| 16 | `paper_outline` | 论文大纲 |
| 17 | `paper_draft` | 论文初稿 |
| 18 | `peer_review` | 多 agent peer review |
| 19 | `paper_revision` | 修改论文 |
| 20 | `quality_gate` | 质量门 |
| 21 | `knowledge_archive` | 归档知识 |
| 22 | `export_publish` | 导出发布材料 |
| 23 | `citation_verify` | 引文验证 |

写 skill 时应尽量绑定具体 stage。例如：

- literature 类 skill：stage 3-6；
- hypothesis 类 skill：stage 7-9；
- experiment design 类 skill：stage 9-12；
- code/tooling 类 skill：stage 10、12、13；
- writing 类 skill：stage 16-19、22；
- quality/citation 类 skill：stage 20、23。

## 匹配机制

`researchclaw/skills/matcher.py` 中的匹配逻辑可以概括为：

1. 先按 stage 过滤：如果 `applicable-stages` 非空，当前 stage 必须在列表里。
2. 再按关键词匹配：`trigger-keywords` 与 context token 有交集则得分。
3. 若没有关键词且 `fallback_matching=true`，则用 description token overlap 做 0.5 倍折扣匹配。
4. 加上 priority boost：`priority` 越小，boost 越高。
5. 排序后取 `max_skills_per_stage` 个。

prompt 注入发生在 `researchclaw/pipeline/_helpers.py`：

```text
context = f"{stage_name} {topic}"
matched = registry.match(context, stage_name)
skills_text = registry.export_for_prompt(matched, max_chars=4000)
```

也就是说，skill 的关键词必须能匹配用户 topic 或 stage 上下文。若写得太抽象，就很难触发。

## 当前已加载 Skill 概览

运行：

```bash
python3 -m researchclaw skills list
```

当前仓库可加载 22 个 skill，按类别如下。

### `domain`

- `a-evolve`：从失败和观察中生成 skill、prompt patch 或 knowledge entry。
- `biology-biopython`：生物序列、FASTA、BLAST、phylogenetics。
- `chemistry-rdkit`：SMILES、fingerprint、descriptor、substructure search。
- `cv-classification`：CIFAR/ImageNet 分类任务。
- `cv-detection`：COCO/VOC/object detection。
- `nlp-alignment`：RLHF、DPO、preference、instruction tuning。
- `nlp-pretraining`：language model pretraining/fine-tuning。
- `quantum-qiskit`：Qiskit 2.x、VQC、VQE、feature map、ansatz。
- `researchclaw`：运行 AutoResearchClaw 23-stage pipeline。
- `rl-policy-optimization`：RL policy optimization、PPO、SAC、reward design。

### `experiment`

- `experimental-design`：baseline、ablation、seed、controlled experiment。
- `hypothesis-formulation`：可证伪假设、H0/H1、机制预测。
- `literature-search`：PICO、Boolean search、PRISMA、screening。
- `meta-analysis`：跨研究 effect size 聚合。
- `systematic-review`：系统综述和筛选方法。

### `tooling`

- `data-loading`：DataLoader、pin_memory、persistent_workers、I/O bottleneck。
- `distributed-training`：PyTorch DDP、多 GPU 训练。
- `mixed-precision`：FP16/BF16、`torch.cuda.amp`、GradScaler。
- `pytorch-training`：稳定 PyTorch training loop。

### `writing`

- `scientific-visualization`：publication-ready figure、色盲友好、统计标注。
- `scientific-writing`：IMRAD、引用、段落、reporting guideline。
- `statistical-reporting`：统计检验、effect size、confidence interval、APA-style。

## 代表性 Skill 解读

### `hypothesis-formulation`

路径：

```text
researchclaw/skills/builtin/experiment/hypothesis-formulation/SKILL.md
.claude/skills/hypothesis-formulation/SKILL.md
```

适用 stage：7、8、9。

它要求 agent：

- 从观察或文献缺口出发；
- 区分已知事实和未知问题；
- 写出 H0 和 H1；
- 用 `If ... then ... because ...` 表达机制假设；
- 同时提出 2-3 个 competing hypotheses；
- 为每个假设定义可测预测和反证条件。

这是一个好的 experiment skill 样板，因为它不只是告诉 agent “生成假设”，而是要求假设可证伪、可测量、能映射到实验设计。

### `experimental-design`

路径：

```text
researchclaw/skills/builtin/experiment/experimental-design/SKILL.md
```

适用 stage：9、10、12。

它强调：

- 必须有 meaningful baselines；
- 至少 3 个随机种子，理想 5 个；
- 报告 mean +/- std；
- ablation 一次只改变一个变量；
- 使用标准 train/val/test split；
- 同时报告 wall-clock time 和 memory。

这是 AutoResearchClaw 防止“跑了实验但不可比较”的关键 skill。

### `pytorch-training`

路径：

```text
researchclaw/skills/builtin/tooling/pytorch-training/SKILL.md
```

适用 stage：10、12。

它包含 `metadata.code-template`，会把可复用训练循环注入 prompt。核心要求：

- 设置随机种子；
- DataLoader 配置适合 GPU；
- 使用 scheduler；
- validation metric early stopping；
- 保存 best checkpoint；
- evaluation 使用 `torch.no_grad()`；
- `optimizer.zero_grad(set_to_none=True)`。

这是 tooling skill 的样板：正文给原则，frontmatter 里放简洁代码模板，适合 code generation 阶段直接复用。

### `rl-policy-optimization`

路径：

```text
researchclaw/skills/builtin/domain/rl-policy-optimization/SKILL.md
```

适用 stage：9、10。

它覆盖：

- 离散动作：PPO、DQN、A2C；
- 连续动作：SAC、TD3、PPO；
- multi-agent：MAPPO、QMIX；
- offline RL：CQL、IQL、Decision Transformer；
- PPO/SAC 训练参数；
- vectorized env；
- observation/reward normalization；
- evaluation 使用 deterministic policy，报告 10+ episode mean +/- std。

这是 domain skill 的样板：给出算法选择、训练 recipe、评估规范和常见坑。

### `quantum-qiskit`

路径：

```text
researchclaw/skills/builtin/domain/quantum-qiskit/SKILL.md
```

适用 stage：10、13。

这个 skill 更长，像一个 API 兼容性手册。它明确 Qiskit 2.x 的 import path、feature map、ansatz、VQC、VQE、noise model 和常见错误修复。

借鉴点：当上游库 API 变化频繁时，skill 应该写具体 import 和反例，避免 agent 使用旧 API。

### `researchclaw`

路径：

```text
.claude/skills/researchclaw/SKILL.md
```

这是项目级运行 skill，面向 AI assistant：

- 触发条件是用户要求 “research topic”、写论文、运行 ResearchClaw；
- 检查 config；
- 给出 CLI、Python API 和 iterative pipeline 三种运行方式；
- 列出输出目录结构；
- 说明 `simulated`、`sandbox`、`ssh_remote` 三种实验模式；
- 给出 troubleshooting。

这是“项目操作型 skill”的样板，不是某个学科知识，而是把仓库怎么跑讲清楚。

### `a-evolve`

路径：

```text
.claude/skills/a-evolve/SKILL.md
```

它实现 Solve -> Observe -> Evolve -> Gate -> Reload 循环：

- 收集失败证据；
- 诊断 root cause、frequency、severity；
- 生成 skill、prompt patch 或 knowledge entry；
- 用 specificity、testability、blast radius、consistency 做 gate；
- 把结果写到 `.claude/skills/evolved/`、prompt 文件或 knowledge 文件。

它与 AutoResearchClaw 的 Stage 12、13、15、18 自然结合，也是 MetaClaw lesson-to-skill 的思想来源。

## MetaClaw Lesson-to-Skill

`researchclaw/metaclaw_bridge/lesson_to_skill.py` 会把高严重度 failure lesson 转换成新 skill。

转换要求：

- skill 名称必须是 `arc-<slug>`；
- description 说明何时使用；
- category 必须来自 `writing`、`domain`、`experiment`、`tooling`；
- Markdown 正文要有 numbered steps 和 anti-pattern section；
- 写入 `~/.metaclaw/skills/<skill-name>/SKILL.md`。

这条路径适合处理“同一类失败反复出现”的情况，例如：

- 引用格式反复出错；
- 实验结果 JSON schema 反复缺字段；
- 代码生成反复忘记 seed；
- figure 没有 error bar；
- CUDA OOM 后没有 batch-size fallback。

## 写新 Skill 的建议流程

1. 明确目标：这是 domain、experiment、tooling 还是 writing skill。
2. 绑定 stage：只填真正适用的 stage，不要默认 all。
3. 写触发关键词：包含用户自然语言、库名、任务名、常见缩写。
4. 写可执行步骤：让 agent 知道先检查什么、再做什么、如何验证。
5. 写反模式：明确不要做什么。
6. 如果是 code generation skill，考虑加入 `metadata.code-template`。
7. 用 CLI 验证。

验证命令：

```bash
python3 -m researchclaw skills validate .claude/skills/my-skill
python3 -m researchclaw skills list
```

安装到用户级目录：

```bash
python3 -m researchclaw skills install /path/to/my-skill
```

## 具体样例：CUDA OOM Recovery Skill

下面是一个完整的自定义 skill 示例，用于让 AutoResearchClaw 在 stage 10/12 生成或执行 PyTorch 代码时，遇到 CUDA OOM 能自动采用降级策略，而不是只失败退出。

推荐位置：

```text
.claude/skills/cuda-oom-recovery/SKILL.md
```

创建文件：

```bash
mkdir -p .claude/skills/cuda-oom-recovery
```

`SKILL.md` 内容：

```markdown
---
name: cuda-oom-recovery
description: Recover from CUDA out-of-memory errors in PyTorch experiments by reducing memory pressure, preserving metrics, and retrying safely. Use when code generation or experiment execution involves GPU training, batch size, mixed precision, activation checkpointing, or CUDA OOM.
metadata:
  category: tooling
  trigger-keywords: "cuda oom,out of memory,gpu memory,batch size,mixed precision,activation checkpointing,pytorch training"
  applicable-stages: "10,12,13"
  priority: "2"
  version: "1.0"
  author: team
  references: "PyTorch CUDA semantics; AutoResearchClaw experiment stages 10,12,13"
  code-template: |
    def run_with_oom_recovery(train_fn, base_config):
        import copy
        import torch

        attempts = []
        cfg = copy.deepcopy(base_config)
        for attempt in range(4):
            try:
                return train_fn(cfg)
            except RuntimeError as exc:
                msg = str(exc).lower()
                if "out of memory" not in msg and "cuda" not in msg:
                    raise
                attempts.append({"attempt": attempt, "batch_size": cfg.get("batch_size"), "error": str(exc)[:500]})
                if torch.cuda.is_available():
                    torch.cuda.empty_cache()
                cfg["batch_size"] = max(1, int(cfg.get("batch_size", 32)) // 2)
                cfg["num_workers"] = min(int(cfg.get("num_workers", 4)), 2)
                cfg["use_amp"] = True
                if cfg["batch_size"] == 1 and attempt >= 2:
                    cfg["gradient_accumulation_steps"] = max(2, int(cfg.get("gradient_accumulation_steps", 1)) * 2)
        raise RuntimeError(f"CUDA OOM recovery exhausted attempts: {attempts}")
---

# CUDA OOM Recovery

## When to Apply

Use this skill when an experiment uses PyTorch GPU training and mentions CUDA memory, batch size, image resolution, transformer sequence length, mixed precision, activation checkpointing, or an error containing `CUDA out of memory`.

## Required Behavior

1. Do not silently drop the experiment condition.
2. Preserve the original config and record every retry config.
3. Retry with a smaller memory footprint in this order:
   - halve `batch_size`;
   - enable AMP/BF16 if numerically safe;
   - reduce DataLoader `num_workers`;
   - enable gradient accumulation to preserve effective batch size;
   - enable activation checkpointing for large models;
   - reduce sequence length or image resolution only if the research question allows it.
4. Always log:
   - original batch size;
   - final batch size;
   - number of retries;
   - whether AMP was enabled;
   - whether the final result is comparable to the original plan.
5. If all retries fail, return a structured failure with the retry log and recommended next action.

## Anti-Patterns

- Do not replace a failed GPU experiment with fabricated metrics.
- Do not compare a reduced-resolution fallback against full-resolution baselines without marking the protocol change.
- Do not catch all `RuntimeError` exceptions as OOM; inspect the error message.
- Do not call `torch.cuda.empty_cache()` as the only fix.

## Validation

Before accepting the result, check:

1. The run produced a metrics JSON or a structured failure JSON.
2. Any fallback changed only resource-related parameters unless explicitly justified.
3. The paper or result summary discloses fallback settings.
```

验证：

```bash
python3 -m researchclaw skills validate .claude/skills/cuda-oom-recovery
```

预期输出形态：

```text
OK: cuda-oom-recovery
  description: Recover from CUDA out-of-memory errors in PyTorch experiments...
  category:    tooling
  stages:      [10, 12, 13]
  keywords:    ['cuda oom', 'out of memory', 'gpu memory', 'batch size', 'mixed precision']
```

如果 topic 是：

```text
Train a ViT on high-resolution images and handle CUDA OOM during experiment execution.
```

在 stage 10 `code_generation` 或 stage 12 `experiment_run`，context token 会匹配：

- `cuda oom`
- `gpu memory`
- `batch size`
- `mixed precision`
- `pytorch training`

因此该 skill 会优先于普通 `pytorch-training` 或 `mixed-precision` 被注入 prompt，因为它的 priority 更高，并且适用 stage 更精准。

## 防幻觉 Checklist

写 AutoResearchClaw skill 时，重点不是写很多背景知识，而是降低 agent 在自动研究 pipeline 里编造、漏检或误跑的概率。

建议每个 skill 都包含：

- 明确触发条件：不要让 agent 猜什么时候用。
- 明确 stage：不要把所有 skill 都设成 all。
- 明确验证物：metrics JSON、paper section、figure、citation report、checkpoint、logs。
- 明确失败处理：什么时候 retry、什么时候 stop、什么时候交给 human gate。
- 明确反模式：不能伪造结果、不能隐瞒 protocol change、不能引用不存在论文。
- 明确可复现要求：seed、版本、配置、artifact 路径。
- 明确比较公平性：baseline、ablation、same split、same budget。

尤其是 experiment/code skill，必须说明：

- 不得生成无法复现的随机结果；
- 不得用 synthetic result 替代真实执行，除非 `experiment.mode: simulated`；
- 不得把训练集指标当测试集指标；
- 不得在没有 citation verification 的情况下声称某篇论文支持结论；
- 不得在实验失败时输出“看起来合理”的数值。

## 维护建议

1. 新 skill 先放 `.claude/skills/<name>/SKILL.md` 做项目级验证。
2. 稳定后再移入 `researchclaw/skills/builtin/<category>/<name>/SKILL.md`。
3. 如果来自失败经验，优先用 `arc-` 前缀存入 MetaClaw skill 目录。
4. 每次修改后运行：

```bash
python3 -m researchclaw skills validate <skill-dir>
python3 -m researchclaw skills list
```

5. 如果修改 matcher、loader、schema，运行相关测试：

```bash
python3 -m pytest tests/test_skills_library.py
```

## 推荐新 Skill 模板

```text
.claude/skills/<skill-name>/
└── SKILL.md
```

```markdown
---
name: <skill-name>
description: <what this skill does and exactly when to use it>
metadata:
  category: <writing|domain|experiment|tooling>
  trigger-keywords: "<comma-separated keywords>"
  applicable-stages: "<stage numbers>"
  priority: "3"
  version: "1.0"
  author: <author>
  references: "<official docs or paper>"
---

# <Skill Title>

## When to Apply

...

## Procedure

1. ...
2. ...
3. ...

## Validation

1. ...
2. ...

## Anti-Patterns

- ...
```

AutoResearchClaw 的 skill 最有价值的地方，是把自动研究 pipeline 中容易出错的“研究判断”和“工程细节”变成可复用规则。写 skill 时要始终服务于这个目标：让下一次研究运行更可靠、更可验证、更少幻觉。
