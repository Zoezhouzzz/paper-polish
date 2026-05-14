# Paper Auto-Review · 论文自动评审与修改

一套基于 Claude Code skill 的论文自动评审 + 修改流水线。丢进一份 `.tex` / `.pdf` / `.zip`，输出三件事：reviewer 评分、修改后的论文、agent 改了哪些地方。

[English version](./README.md)

---

## 做什么

给一篇 CS/AI 论文，它会：

1. 并行启动三个独立 reviewer（Motivation / Experiments / Writing）+ 一个 Area Chair 汇总打分。
2. 根据评审意见修改论文：文字、图表、缺失实验各由专门 skill 处理。
3. 评审 → 修改 → 再评审，直到分数过线或达到最大轮数；如果某轮分数下降会**自动回滚**到上一轮。
4. 最终只在工作目录留**三个**干净的产物文件，所有中间产物收进隐藏的 `.auto-review/`。

## 工作流程

```
                    paper.tex / .pdf
                           │
                           ▼
            ┌─────────────────────────────┐
            │       /review-loop          │ ◄─────────────┐
            │       orchestrator          │               │
            └──────────────┬──────────────┘               │
                           │                              │
   ① REVIEW phase ─────────┴────── (parallel agents)      │
                                                          │
   ┌────────────┐    ┌────────────┐    ┌────────────┐     │
   │ Motivation │    │ Experiment │    │  Writing   │     │
   │  Reviewer  │    │  Reviewer  │    │  Reviewer  │     │
   └─────┬──────┘    └─────┬──────┘    └─────┬──────┘     │
         └─────────────────┼─────────────────┘            │
                           ▼                              │
                   ┌───────────────┐                      │
                   │  Area Chair   │  ← 汇总打分           │
                   │  (sequential) │                      │
                   └───────┬───────┘                      │
                           │                              │
   ② REVISE phase ─────────┴────── (parallel skills)      │
                                                          │
   ┌────────────┐    ┌────────────┐    ┌──────────────┐   │
   │  writing-  │    │  figure-   │    │ experiment-  │   │
   │  reviser   │    │  advisor   │    │  proposer    │   │
   └─────┬──────┘    └─────┬──────┘    └──────┬───────┘   │
    text edits        redraw figs       expt designs      │
         └─────────────────┼─────────────────┘            │
                           ▼                              │
              ┌────────────────────────────┐              │
              │ 分数 ≥ 7  或  轮次 = 3 ?    │── 否 ─────────┘
              │ (分数下降则回滚)            │   下一轮
              └─────────────┬──────────────┘
                            │ 是
                            ▼
              ┌────────────────────────────┐
              │  REVIEW.md                 │
              │  CHANGES.md                │
              │  paper/main_revised.tex    │
              └────────────────────────────┘
```

两个阶段都是**并行调度**：三个 reviewer 同时跑，三个修改 skill 也同时跑。Area Chair 在三个 reviewer 都返回之后才启动（需要汇总），整体流程被 `/review-loop` 编排成多轮循环。

## Skills 一览

| Skill | 角色 | 调用方式 |
|---|---|---|
| `paper-reviewer` | 三 reviewer + AC，给分 + 修改优先级 | `/paper-reviewer paper.pdf` |
| `writing-reviser` | 文字修改；不可自动修的标 `NEEDS_AUTHOR` | （由 `review-loop` 调用） |
| `figure-advisor` | 修图、重画 matplotlib、调度架构图 | （由 `review-loop` 调用） |
| `experiment-proposer` | 设计缺失的消融 / baseline / 鲁棒性实验 | （由 `review-loop` 调用） |
| `review-loop` | 总编排，控制轮次、回滚、最终汇总 | `/review-loop paper.tex` |
| `paper-illustration` | AI 生成架构 / pipeline 图 | （由 `figure-advisor` 调用） |
| `research-lit` | 检索真实文献用于 related work 补充 | （由 `writing-reviser` 调用） |

## 输入

把论文放在项目根目录的 `paper/` 文件夹下，或者直接把路径作为 slash command 的参数传进去：

- **`.pdf`** — 投稿用的 PDF
- **`.tex`** — LaTeX 源文件（**推荐**，只有 `.tex` 才能被自动修改）
- **`.zip`** — 完整 LaTeX 包（图、bib、style 文件）

如果只给 PDF，reviewer 依然能打分给意见，但 `review-loop` 没法做文字修改（没有 `.tex` 可编辑）。

## 怎么用

### 1. 只要评审意见（只读）

```
/paper-reviewer paper/main.pdf
```

生成 `review_output.md`，里面有三个 reviewer 的分数、AC 的最终决定、按优先级排序的修改建议。

### 2. 完整评审 + 修改循环

```
/review-loop paper/main.tex
```

会自动跑评审 → 修改 → 再评审，直到任一条件满足：

- 综合评分 ≥ `SCORE_THRESHOLD`（默认 **7**），或
- 达到 `MAX_ROUNDS`（默认 **3** 轮），或
- 出现两次连续回滚。

### 3. 使用外部 reviewer 意见

把你拿到的外部评审意见放到项目根目录的 `external_reviews.txt`，然后：

```
/review-loop paper/main.tex
```

循环第一轮会跳过自己的 reviewer，直接用你的意见；从第二轮开始仍会调用 `paper-reviewer` 跟踪分数变化。

## 输出

`review-loop` 跑完后，工作目录正好留三个文件：

| 文件 | 内容 |
|---|---|
| `REVIEW.md` | 综合评分、最终决定、各 reviewer 要点、多轮分数轨迹、剩余的修改优先级、待作者补的实验/数据、实验提案摘要 |
| `CHANGES.md` | agent 做的每一处文字修改（before → after，含位置）和每一处图的改动 |
| `paper/main_revised.tex` | 修改后的论文，最新被接受的版本 |

其余所有产物都在 `.auto-review/` 下面，必要时再翻：

```
.auto-review/
├── review_output.md           # 最后一轮 reviewer 原始输出
├── writing_report.md          # 文字修改日志（已并入 CHANGES.md）
├── figure_report.md           # 图修改日志（已并入 CHANGES.md）
├── figure_todo.md             # writing-reviser → figure-advisor 的管道
├── author_todo.md             # 循环无法自动修复的项
├── experiment_proposals.md    # 完整实验设计（REVIEW.md 里有摘要）
├── loop_state.json            # 最后一轮的分数 / 决定 / 回滚状态
└── snapshots/                 # 每轮 main.tex + review.md，含 _FAILED 版本
```

## 评审是怎么得出来的

`paper-reviewer` 会启动四个 agent：

1. **Motivation Reviewer** — 创新性（1–10）+ 逻辑完整性（1–10）
2. **Experiment Reviewer** — 实验完备性（1–10）+ 实验对论点的支持度（1–10）
3. **Writing Reviewer** — 写作质量（1–10）+ 图表设计（1–10）
4. **Area Chair** — 汇总三份评审 → 综合分 + Accept / Weak Accept / Borderline / Reject + 修改优先级

前三个 reviewer **并行**跑，AC 等它们都返回后再起。每个 reviewer 都被强制要求给：数值分、2–3 句话理由、具体弱点 + 位置（章节 / 图 / 表）。

## 修改是怎么做的

每轮循环里，编排器并行调用三个修改 skill：

- **writing-reviser** 把每条修改建议分成 **SAFE**（自动改）、**MARK**（在 .tex 里插 `% [NEEDS_AUTHOR]` 注释提醒作者）、**SKIP**（转给 figure-advisor / experiment-proposer）。SAFE 项直接编辑 `main.tex`，每处用 `% [REV-NNN]` 标记。
- **figure-advisor** 读 `figure_todo.md`，先把 PDF 光栅化看图，然后对数据图重写 matplotlib 代码（**只用论文里已有的数字**），架构图转给 `paper-illustration`。
- **experiment-proposer** 读 `author_todo.md`，写结构化的实验提案（类型、数据集、指标、baseline、预期表格格式）— **只出设计，不出结果**。

## 回滚机制

如果第 N 轮的分数低于第 N-1 轮：

1. 失败的修改版重命名为 `.auto-review/snapshots/main_round_N_FAILED.tex`。
2. 用上一轮的快照覆盖回 `paper/main.tex`。
3. reviser 拿着"上次失败的 diff + 低分评审"重试，被告知更保守地修。
4. 连续两次回滚 → 直接停掉循环，需要人工介入。

这个机制是为了防止"修改退化"：好意的编辑反而破坏行文连贯性，引入比修掉的更多问题。

## 反幻觉硬约束

这些约束直接写在每个 skill 的 system prompt 里：

- **不许编实验结果。** 修改需要新数字时，标 `NEEDS_AUTHOR`，转到 `author_todo.md`。
- **不许编引用。** 新引用要么从 `research-lit`（真实论文检索）里来，要么留 `\cite{PLACEHOLDER_REV_NNN}` 让作者自己核对。
- **不许编图表数据。** `figure-advisor` 只能用论文里已经出现过的数字重画。
- **`experiment-proposer` 不写代码。** 只出实验设计，不出实现。

## 配置项

常量在 `.claude/skills/review-loop/SKILL.md` 顶部：

```
MAX_ROUNDS = 3
SCORE_THRESHOLD = 7
```

直接改文件即可。没有单独的配置文件。

## 目录结构

```
paper-auto-review/
├── .claude/skills/             # 7 个 skill
│   ├── paper-reviewer/
│   ├── writing-reviser/
│   ├── figure-advisor/
│   ├── experiment-proposer/
│   ├── review-loop/
│   ├── paper-illustration/
│   └── research-lit/
├── paper/                      # 把你的 .tex / .pdf 放这里
│   └── main.tex
└── （跑完 review-loop 之后）
    ├── REVIEW.md
    ├── CHANGES.md
    ├── paper/main_revised.tex
    └── .auto-review/
```

## 依赖

- Claude Code CLI
- `python3` + `PyMuPDF`（`fitz`）— 用于 PDF 文本提取和页面光栅化
- 如果要编译修改后的 `.tex`，需要本地 LaTeX 环境
