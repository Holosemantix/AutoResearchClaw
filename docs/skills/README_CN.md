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

## 23 个 Stage 的作用、输入输出与限制

AutoResearchClaw 的每个 stage 都有一个 I/O contract，定义在 `researchclaw/pipeline/contracts.py`。这些 contract 用来约束 stage 读什么、必须产出什么、什么时候算完成，以及失败后最多重试几次。

表里的“输入/输出”是 artifact 文件或目录名，通常位于一次运行的 `artifacts/<run-id>/stage-<N>/` 附近，后续 stage 会读取前序 stage 的产物。

| Stage | 作用 | 输入 | 输出 | 主要限制 |
| --- | --- | --- | --- | --- |
| 1 `TOPIC_INIT` | 把用户自然语言课题变成可执行的 SMART research goal，并记录硬件环境 | 用户 topic、config、硬件探测 | `goal.md`, `hardware_profile.json` | 不做文献和实验；只定义范围、目标、约束。`max_retries=0`，目标不清会直接失败 |
| 2 `PROBLEM_DECOMPOSE` | 把目标拆成子问题、变量、风险和优先级 | `goal.md` | `problem_tree.md` | 至少要有 3 个优先级明确的 sub-questions；不能直接跳到实验 |
| 3 `SEARCH_STRATEGY` | 设计检索策略、数据源和 query 组合 | `problem_tree.md` | `search_plan.yaml`, `sources.json`, `queries.json` | 至少 2 种 search strategy；数据源必须可验证，不能只写泛泛的“查论文” |
| 4 `LITERATURE_COLLECT` | 按检索计划收集候选论文或资料 | `search_plan.yaml` | `candidates.jsonl` | 候选集不能为空；最多重试 2 次；受 API、网络、检索源覆盖范围限制 |
| 5 `LITERATURE_SCREEN` | 对候选文献做 relevance 和 quality 双重筛选 | `candidates.jsonl` | `shortlist.jsonl` | Gate stage，默认需要 human approval；拒绝后回滚到 stage 4；不能保留不相关或低质量来源 |
| 6 `KNOWLEDGE_EXTRACT` | 从 shortlisted papers 中抽取结构化知识卡片 | `shortlist.jsonl` | `cards/` | 每篇入选文献应有 card；需要区分方法、数据、结果、限制和可复用证据 |
| 7 `SYNTHESIS` | 聚类文献发现，归纳研究 gap | `cards/` | `synthesis.md` | 至少识别 2 个 research gaps；不能只总结文献，必须说明未解决问题 |
| 8 `HYPOTHESIS_GEN` | 生成可证伪 hypothesis 和可观测预测 | `synthesis.md` | `hypotheses.md` | 至少 2 个 falsifiable hypotheses；需要能映射到实验条件 |
| 9 `EXPERIMENT_DESIGN` | 设计实验计划、baseline、ablation、metrics | `hypotheses.md` | `exp_plan.yaml` | Gate stage，默认需要 approval；拒绝后回滚到 stage 8；必须包含 baselines、ablations、metrics |
| 10 `CODE_GENERATION` | 根据实验计划生成可执行实验项目 | `exp_plan.yaml` | `experiment/`, `experiment_spec.md` | 最多重试 2 次；代码应可运行、可复现；`hep_ph` profile 下额外产生 `collider_plan.md` 且可成为 gate |
| 11 `RESOURCE_PLANNING` | 规划 GPU/CPU/时间/并发等资源 | `exp_plan.yaml` | `schedule.json` | 需要估计资源和运行顺序；不能安排超出硬件 profile 的任务 |
| 12 `EXPERIMENT_RUN` | 按 schedule 执行实验，收集原始结果 | `schedule.json`, `experiment/` | `runs/` | 最多重试 2 次；不能伪造结果；sandbox/SSH/Docker 等执行模式受环境依赖限制 |
| 13 `ITERATIVE_REFINE` | 对失败或弱结果做 edit-run-eval 迭代 | `runs/` | `refinement_log.json`, `experiment_final/` | 最多重试 2 次；必须记录每次修改和原因；不能改变研究问题后仍声称同一实验 |
| 14 `RESULT_ANALYSIS` | 统计分析实验结果，生成结论和表格 | `runs/` | `analysis.md` | 需要统计检验和结论；不能把训练集指标当测试集指标；不能忽略失败 run |
| 15 `RESEARCH_DECISION` | 根据证据决定 PROCEED、REFINE 或 PIVOT | `analysis.md` | `decision.md` | PIVOT 回滚到 stage 8，REFINE 回滚到 stage 13；最多 2 次 pivot，防止无限循环 |
| 16 `PAPER_OUTLINE` | 生成论文结构和 section-level plan | `analysis.md`, `decision.md` | `outline.md` | 必须覆盖摘要、引言、方法、实验、结果、限制；不能写成空泛目录 |
| 17 `PAPER_DRAFT` | 写完整论文初稿 | `outline.md` | `paper_draft.md` | 必须与已有证据一致；引用应来自 collected literature；不能编造结果或引用 |
| 18 `PEER_REVIEW` | 多视角模拟 peer review，指出问题 | `paper_draft.md` | `reviews.md` | 至少 2 个 review perspectives；反馈必须 actionable，不只是笼统评价 |
| 19 `PAPER_REVISION` | 根据 review 修改论文 | `paper_draft.md`, `reviews.md` | `paper_revised.md` | 需要回应 review comments；不能无解释删除负面结果或限制 |
| 20 `QUALITY_GATE` | 最终质量门，检查质量分数和批准状态 | `paper_revised.md` | `quality_report.json` | Gate stage，拒绝后回滚到 stage 16；这是 critical stage，不能跳过低质量论文 |
| 21 `KNOWLEDGE_ARCHIVE` | 归档回顾、实验 bundle 和复现索引 | 通常读取整次 run 的上下文 | `archive.md`, `bundle_index.json` | 非关键 stage，失败不应影响 paper output；但会影响长期复用和复现 |
| 22 `EXPORT_PUBLISH` | 导出最终论文、代码和提交包 | `paper_revised.md` | `paper_final.md`, `code/` | 必须使用目标格式；不能在 citation verification 之前宣称引用全部可靠 |
| 23 `CITATION_VERIFY` | 用真实 API/规则验证引用，标记幻觉引用 | `paper_final.md`，可选 `references.bib` | `verification_report.json`, `references_verified.bib` | 伪造引用必须 block export；`references.bib` 可选，但 final paper 中引用必须可核验 |

### Gate、Rollback 与 Retry 限制

默认 gate stage：

- Stage 5 `LITERATURE_SCREEN`：拒绝后回滚到 stage 4，重新收集或筛选文献。
- Stage 9 `EXPERIMENT_DESIGN`：拒绝后回滚到 stage 8，重新生成或修正 hypotheses。
- Stage 20 `QUALITY_GATE`：拒绝后回滚到 stage 16，重写 outline/draft/revision。

特殊情况：

- `hep_ph` profile 下，stage 10 `CODE_GENERATION` 会成为强制 gate，因为昂贵的 ColliderAgent 物理流水线运行前需要检查 `collider_plan.md`。
- Stage 15 的 `PIVOT` 回滚到 stage 8，`REFINE` 回滚到 stage 13。
- `MAX_DECISION_PIVOTS=2`，避免 pipeline 无限换题。
- Stage 21 `KNOWLEDGE_ARCHIVE` 是 noncritical；其他质量、引用和实验相关 stage 不应随意跳过。

## User Case：一个研究课题如何流经 23 个 Stage

假设用户输入：

```text
研究课题：在 CIFAR-100 小样本设置下，token pruning 是否能降低 Vision Transformer 推理成本，同时保持分类准确率？
约束：只能使用单张 24GB GPU；总实验时间控制在 6 小时内；必须比较 ResNet-18、ViT-base、ViT+token pruning；至少 3 个随机种子；输出 NeurIPS 风格论文草稿。
```

最终期望输出：

```text
artifacts/rc-YYYYMMDD-HHMMSS-<hash>/
├── paper_final.md
├── references_verified.bib
├── verification_report.json
├── code/
├── stage-14/analysis.md
├── stage-17/paper_draft.md
├── stage-19/paper_revised.md
└── stage-22/...
```

这个课题的 stage-by-stage 输入输出可以这样理解：

| Stage | 本案例中的输入 | 本案例中的输出 | 本案例中的限制/检查 |
| --- | --- | --- | --- |
| 1 | 用户课题、GPU/时间/seed 约束 | `goal.md` 写明 CIFAR-100、token pruning、准确率/延迟/FLOPs；`hardware_profile.json` 记录 24GB GPU | 如果课题没有数据集、指标或资源约束，stage 1 应要求补全或失败 |
| 2 | `goal.md` | `problem_tree.md` 拆成准确率、推理成本、小样本鲁棒性、baseline 公平性 | 子问题不能少于 3 个；需要标出优先级 |
| 3 | `problem_tree.md` | `search_plan.yaml` 包含 ViT pruning、dynamic token sparsification、CIFAR small-data、efficient inference 查询 | 检索源至少含 arXiv/Semantic Scholar/OpenAlex 等可核验来源 |
| 4 | `search_plan.yaml` | `candidates.jsonl` 收集候选论文，如 DeiT、DynamicViT、ToMe、ViT pruning、小样本学习文献 | 候选为空要重试；不能把博客当主要证据 |
| 5 | `candidates.jsonl` | `shortlist.jsonl` 保留与 token pruning、ViT efficiency、小样本分类直接相关文献 | Gate；人类可拒绝“只收集到 CNN pruning，缺少 ViT pruning”的 shortlist |
| 6 | `shortlist.jsonl` | `cards/` 每篇论文抽取方法、数据集、指标、结论、限制 | card 需区分 paper claim 和可复现实验事实 |
| 7 | `cards/` | `synthesis.md` 总结 gap：小样本 ViT 下 pruning 是否损害泛化、是否只提升 FLOPs 而非 wall-clock | 至少 2 个 gap，不能只是摘要拼接 |
| 8 | `synthesis.md` | `hypotheses.md`：H1 token pruning 降低 FLOPs 且 accuracy drop <1%；H2 小样本下 pruning 需正则化才能稳定 | hypothesis 必须可被实验反驳 |
| 9 | `hypotheses.md` | `exp_plan.yaml`：3 seeds、CIFAR-100 subset、ResNet-18/ViT/ViT-pruned、accuracy/FLOPs/latency、ablation pruning ratio | Gate；如果缺 baseline、seed 或 ablation，应拒绝 |
| 10 | `exp_plan.yaml` | `experiment/` 生成训练、评估、FLOPs/latency 测量代码；`experiment_spec.md` 说明运行方式 | 代码必须记录 config、seed、metrics；不能只输出随机模拟结果 |
| 11 | `exp_plan.yaml` | `schedule.json` 安排 seed、baseline、batch size、预计时长 | 6 小时和 24GB GPU 是硬限制；超出需缩小模型或减少非核心 ablation |
| 12 | `schedule.json`, `experiment/` | `runs/` 保存每个 baseline/seed 的 metrics、logs、checkpoints | CUDA OOM 要 retry 或结构化失败；不能删除失败 run |
| 13 | `runs/` | `refinement_log.json` 记录修复，如 batch size 降低、AMP 开启；`experiment_final/` 保存最终代码 | 修改必须只影响资源或 bug，不能偷偷改变 hypothesis |
| 14 | `runs/` | `analysis.md` 比较 accuracy、FLOPs、latency、方差、显著性和失败情况 | 需要 mean +/- std；不能只报告最好 seed |
| 15 | `analysis.md` | `decision.md` 决定 PROCEED/REFINE/PIVOT。例如 accuracy drop 过大则 REFINE pruning ratio | REFINE 回 stage 13；若 hypothesis 不成立，可 PIVOT 到新的 hypothesis |
| 16 | `analysis.md`, `decision.md` | `outline.md` 规划 NeurIPS 风格结构：Intro、Related Work、Method、Experiments、Limitations | 大纲必须和实验结论一致 |
| 17 | `outline.md` | `paper_draft.md` 初稿，包含方法、表格、图和局限性 | 不得声称未测数据集；不得伪造引用 |
| 18 | `paper_draft.md` | `reviews.md` 多视角 review：ML 方法、公平比较、统计可靠性、写作质量 | 至少 2 个具体 review perspectives |
| 19 | `paper_draft.md`, `reviews.md` | `paper_revised.md` 回应 reviewer，补充限制和实验设置 | 不得为了更好看而删掉负面发现 |
| 20 | `paper_revised.md` | `quality_report.json` 检查质量、证据一致性、实验充分性 | Gate；如果结果不足支撑 claim，应回滚 stage 16 |
| 21 | 整个 run 上下文 | `archive.md`, `bundle_index.json` 归档 config、代码、结果和经验 | 失败不应阻断论文，但会降低复现性 |
| 22 | `paper_revised.md` | `paper_final.md`, `code/` 导出最终稿和代码包 | 导出不等于引用已验证；仍需 stage 23 |
| 23 | `paper_final.md` 和可选 `references.bib` | `verification_report.json`, `references_verified.bib` | 如果 DynamicViT/ToMe 等引用不存在或不匹配，必须标记并修复 |

这个 user case 展示了 skill 应该如何帮助 pipeline：

- stage 3-6 会匹配 `literature-search` / `systematic-review`；
- stage 7-9 会匹配 `hypothesis-formulation` / `experimental-design`；
- stage 10-12 会匹配 `pytorch-training`、`data-loading`、`mixed-precision`，如果 topic 包含 RL 则会匹配 `rl-policy-optimization`；
- stage 14 会匹配 `statistical-reporting`；
- stage 16-19 会匹配 `scientific-writing`；
- stage 22 会匹配 `scientific-visualization`；
- 如果用户加入自定义 `cuda-oom-recovery`，stage 10/12/13 会在 CUDA OOM 相关 context 下自动注入。

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
