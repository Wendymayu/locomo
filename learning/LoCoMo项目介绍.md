# LoCoMo 项目介绍

> 本文档是对本仓库的自学习整理，帮助快速理解项目的目标、数据、代码结构与使用方式。
> 如需权威说明，请以根目录 `README.MD` 与论文原文为准。

---

## 一、项目是什么

**LoCoMo**（**Lo**ng-**Co**nversation **Mo**nitoring / Memory）是一个用于评估 **LLM 智能体超长期对话记忆能力**（very long-term conversational memory）的基准数据集与评测框架。

- **来源**：Snap Research，ACL 2024 论文 *《Evaluating Very Long-Term Conversational Memory of LLM Agents》*
- **作者**：Adyasha Maharana、Dong-Ho Lee、Sergey Tulyakov、Mohit Bansal、Francesco Barbieri、Yuwei Fang
- **论文**：arXiv:2402.17753（仓库内 `static/paper/locomo.pdf`）
- **核心动机**：既有的对话基准通常只有几轮到十几轮对话，无法检验模型在跨越数月、上千轮对话后是否仍能"记住"事实、时间与因果关系。LoCoMo 用一个生成式框架造出*很长*的对话，并据此构造三类任务来定量衡量记忆能力。

通俗说：**它既是"考卷"（带标注的评测数据），也是"出题机"（生成超长对话的代码框架）。**

---

## 二、核心概念速览

| 术语 | 英文 | 一句话释义 |
|------|------|-----------|
| 超长对话 | very long-term conversation | 跨越数十个 session、时间跨度可达数百天的多人对话 |
| 会话 | session | 对话中的一个时间片段，对应一个日期时间戳 |
| 轮次 | turn | session 内的一条发言 |
| 人设 | persona | 预先指派给 LLM 智能体的人物设定（名字 + 简介） |
| 事件图 | event graph | 为每个角色生成、带因果与时序连接的事件知识图谱 |
| 观察 | observation | 由 `gpt-3.5` 从对话中抽取的客观事实，用作 RAG 数据库 |
| 会话摘要 | session summary | 对单个 session 的归纳，用作 RAG 数据库 |
| 事件摘要 | event summary | 跨 session、按角色组织的带因果/时序连接的事件归纳（标注任务） |
| 检索增强 | RAG | 检索相关上下文再喂给模型，以缓解上下文窗口限制 |
| 检索器 | retriever | 把文本转向量做相似度检索的模块，默认 contriever |

---

## 三、数据集构成

数据文件：`data/locomo10.json`，共 **10 段对话**。

### 规模（实测统计）

| 指标 | 数值 |
|------|------|
| 对话数 | 10 |
| session 总数 | 272（每段 19–32 个不等） |
| QA 问题总数 | 1986 |

### conversation / session / turn 的层级关系（对照 ChatGPT）

⚠️ **易混点**：LoCoMo 里的 `session` 与 ChatGPT 里俗称的"会话"不是一回事，整整差了一级。

- **conversation（对话）**：两个角色之间跨越数月的完整故事线，内含多个 session。
- **session（会话）**：某一天两人"见一次面"聊的那十几轮，各带一个日期时间戳；session 之间隔几天到几周。
- **turn（轮次）**：session 内的一条发言（A 说一句 = 1 turn，B 回一句 = 1 turn）。

样本 0（Caroline ↔ Melanie，共 19 个 session）的真实时间线：

```
conversation（Caroline ↔ Melanie，跨 5月 → 7月+ 的整条友谊线）
├─ session_1   5月8日  1:56 pm   18 turn   ← 第一次见面
├─ session_2   5月25日 1:14 pm   17 turn   ← 隔 17 天，开头 "since we last chatted..."
├─ session_3   6月9日  7:55 pm   23 turn   ← 又隔 15 天
├─ session_5   7月3日            16 turn
├─ session_10  7月20日 8:56 pm   24 turn
└─ ...（全数据集共 272 个 session）
```

对照 ChatGPT 使用经验：

| 你的 ChatGPT 经验 | LoCoMo 对应 |
|---|---|
| 不刷新页面一直聊的"会话" | **conversation**（整条故事线） |
| 你一句 + GPT 回一句 = 一次交互 | **turn**（session 内一条发言） |
| ——（ChatGPT 无此层级） | **session**：某天两人见一次面聊的那十几轮 |

**为什么多出 session 这一层**：LoCoMo 测的是**长期记忆**而非长上下文窗口。若把 272 个 session 摊平成连续上千 turn 塞给模型，只是在测窗口能塞多长；用"隔几天又见面"的真实节奏，才能考模型能否在 7 月想起 5 月聊过的事。代码里 `generate_conversations.get_session_date()` 会在事件时间线上取两人都有事可聊的新日期、再加 1~2 天偏移，主动制造这种时间跳跃。

### 单条样本结构

每条样本对应一段对话，顶层字段如下：

```
sample_id            # 样本 ID
conversation         # 对话本体
  ├─ speaker_a / speaker_b      # 两位说话人姓名
  ├─ session_<n>                 # 第 n 个 session 的轮次列表
  ├─ session_<n>_date_time       # 该 session 的时间戳
  └─ turn: speaker / dia_id / text / (img_url / blip_caption / query)
observation          # （生成）每个 session 的客观观察，用于 RAG
session_summary      # （生成）每个 session 的摘要，用于 RAG
event_summary        # （标注）每个 session 内各角色的重要事件，事件摘要任务的 ground truth
qa                   # （标注）问答对，含 question / answer / category / evidence
```

> 说明：本项目**不发布图片本体**，但保留了图片的 web URL、BLIP 生成的 caption 与检索 query。

### QA 任务的五大题型

`qa` 中每条问题带 `category` 标签（1–5），类别既决定题目难度，也决定打分方式。全数据集 1986 题的分布：

| category | 题型 | 题量 | 评测方式 |
|----------|------|------|---------|
| 1 | 多跳推理（multi-hop） | 282 | 答案按 `,` 拆子答案，对每个真值子答案取预测里的最高 F1，再求均值 |
| 2 | 时间型（temporal） | 321 | 问题追加"用对话日期给近似日期"，单答案 F1 |
| 3 | 开放域（open-domain） | 96 | 单答案 F1，真值先按 `;` 取第一段 |
| 4 | 单跳（single-hop） | 841 | 标准单答案 F1 |
| 5 | 对抗型（adversarial） | 446 | 二选一，选"对话中未提及"才得分 |

逐类说明（含真实样例）：

**1 · 多跳推理（multi-hop，282 题）** —— 需跨多个 session 或多轮综合才能答。
- 例：Q "What is Caroline's relationship status?" → A "Single"，`evidence = ['D3:13','D2:14']`（要跨 session 2、3 两处才定论）。
- 评测：`f1()` 把预测与真值都按 `,` 切成子答案，对每个真值子答案在预测子答案里取最高 F1 再求均值，奖励"答对一部分"。

**2 · 时间型（temporal，321 题）** —— 考时间点记忆。
- 例：Q "When did Caroline go to the LGBTQ support group?" → A "7 May 2023"，`evidence = ['D1:3']`。
- 评测：问题会自动追加一句 "Use DATE of CONVERSATION to answer with an approximate date"，单答案 F1。

**3 · 开放域（open-domain，96 题，最少）** —— 答案非对话字面，需基于对话内容推断/概括。
- 例：Q "What fields would Caroline be likely to pursue in her education?" → A "Psychology, counseling certification"，`evidence = ['D1:9','D1:11']`。
- 评测：单答案 F1，真值先按 `;` 取第一段（应对多答案标注）。

**4 · 单跳（single-hop，841 题，最多）** —— 从对话某处直接可答。
- 例：Q "What did the charity race raise awareness for?" → A "mental health"，`evidence = ['D2:2']`。
- 评测：标准单答案 F1。

**5 · 对抗型（adversarial，446 题）** —— 抑制"基于主题相关就硬编答案"的幻觉，全表最巧妙的设计。
- 字段特殊：没有 `answer`，而是 `adversarial_answer`（干扰项）。例：Q "What did Caroline realize after her charity race?" → `adversarial_answer = "self-care is important"`，`evidence = ['D2:3']`。
- 注意 `evidence` 非空：对话里**确实聊过 charity race**，但"self-care is important"这个具体结论**对话里没明说**——干扰项因此听起来很合理，专门诱导模型硬编。
- 评测时把 `adversarial_answer` 与 "Not mentioned in the conversation" 随机排成 (a)/(b) 二选一，**正确答案是后者**。模型输出含 "not mentioned / no information available" 才得 1 分，否则 0 分。
- ⚠️ **实战提示**：发布数据中对抗题字段为 `adversarial_answer`，而 `task_eval/gpt_utils.py` 构造选项时仍引用 `qa['answer']`（见 `get_gpt_answers` 内 `category == 5` 分支），字段名不一致，跑对抗题评测前需对齐字段；其余 4 类不受影响。

---

## 四、三大任务

LoCoMo 围绕对话构造了三类任务：

1. **问答（Question Answering）**——已完整实现。给模型（截断后的）对话作为上下文，回答 1986 道题。
2. **事件摘要（Event Summarization）**——按角色跨 session 总结带因果/时序连接的事件。README 标注 *Coming soon*。
3. **多模态对话生成（Multimodal Dialog Generation）**——在 `MiniGPT-5` 上训练与评测。README 标注 *Coming soon*。

本文重点关注已完整开放代码的 **QA 任务**及其 RAG 变体。

---

## 五、代码结构

```
locomo/
├── data/                          # 数据集
│   ├── locomo10.json              # 主数据（10 段对话 + 标注 + 生成结果）
│   ├── msc_personas_all.json      # MSC 数据集的人设池（用于生成对话时采样人设）
│   └── multimodal_dialog/example/ # 自定义人设示例 agent_a.json / agent_b.json
│
├── generative_agents/             # 对话生成框架（"出题机"）
│   ├── generate_conversations.py  # 主入口：生成 persona → 事件图 → 多 session 对话
│   ├── conversation_utils.py      # 对话拼装、prompt 模板、检索式记忆
│   ├── event_utils.py             # 事件图/子事件/因果连接生成
│   ├── memory_utils.py           # 反思、事实抽取、上下文检索
│   └── html_utils.py             # 把对话渲染成 HTML
│
├── task_eval/                     # 评测框架（"阅卷机"）
│   ├── evaluate_qa.py            # QA 评测主入口，分发到不同模型后端
│   ├── evaluation.py             # 指标实现：F1 / EM / ROUGE-L / BERTScore / Recall
│   ├── evaluation_stats.py       # 聚合统计（按类别、整体准确率）
│   ├── gpt_utils.py              # OpenAI 模型 + 截断策略 + RAG 上下文构造
│   ├── claude_utils.py           # Anthropic Claude
│   ├── gemini_utils.py           # Google Gemini
│   ├── hf_llm_utils.py           # HuggingFace 开源模型（gemma/llama/mistral）
│   ├── rag_utils.py             # 检索器与 embedding（contriever 等）
│   ├── get_facts.py              # 抽取事实
│   └── get_session_summaries.py  # 生成 session 摘要
│
├── scripts/                       # 一键脚本（先 source env.sh 再跑）
│   ├── env.sh                    # 全局配置：路径、API key（需自行填）
│   ├── generate_conversations.sh # 生成对话
│   ├── generate_observations.sh  # 生成 observation
│   ├── generate_session_summaries.sh
│   ├── evaluate_gpts.sh          # 评测 OpenAI
│   ├── evaluate_claude.sh        # 评测 Anthropic
│   ├── evaluate_gemini.sh        # 评测 Gemini
│   ├── evaluate_hf_llm.sh        # 评测 HuggingFace 开源模型
│   └── evaluate_rag_gpts.sh      # 评测 RAG 版 gpt-3.5
│
├── prompt_examples/              # 各类生成任务的 in-context 示例
│   ├── persona_generation_examples.json
│   ├── event_generation_examples.json
│   ├── graph_generation_examples.json
│   ├── causal_event_kg_example_1.json / output_1.json
│   ├── visual_graph_generation_examples.json
│   ├── image_sharing_examples.json
│   ├── fact_generation_examples*.json
│   ├── sub_event_generation_examples.json
│   ├── remove_context_examples.json
│   └── chatgpt_instructions.json
│
├── global_methods.py              # 公共方法：run_chatgpt / 各家 API key 设置
├── requirements.txt               # conda 环境快照（python 3.9, torch 2.0 + cu117）
├── static/                        # 项目主页资源（图片、论文 pdf）
├── index.html                     # 项目主页
└── LICENSE.txt
```

---

## 六、对话是怎么"造"出来的

`generative_agents/generate_conversations.py` 用三个开关 `--persona / --events / --session` 串起一条生成流水线：

1. **造人设（persona）**：从 MSC 人设池采样并扩写成详细人设，或直接读自定义 `agent_a.json / agent_b.json`。
2. **造事件图（events）**：为每个角色生成 N 个事件，事件之间带**因果**（`caused_by`）与**时序**（`date`）连接，形成知识图谱。
3. **造对话（session）**：按时间线把事件分配到各 session，让两个 LLM 智能体轮流发言：
   - 每个 session 取一个随机日期时间；
   - 把该时段相关事件喂进 prompt 作为"该聊什么"的骨架；
   - 后续 session 还会带上**上一 session 摘要**与**检索式记忆**（embedding 相似度）来保证连贯；
   - 检测到"分享图片"意图时，用 `icrawler` 抓图、用 **BLIP** 生成 caption；
   - 写出 `[END]` 即结束本 session。

可调参数举例：`--num-sessions`（session 数）、`--num-days`（时间跨度，默认 240 天）、`--max-turns-per-session`、`--num-events`、`--reflection`（是否开启反思模块）等。

---

## 七、QA 评测是怎么跑的

### 7.1 总体流程（`task_eval/evaluate_qa.py`）

1. 解析参数：`--model`（如 `gpt-4-turbo`、`claude-...`、`gemini-pro-1.0`、`llama...`）、`--data-file`、`--out-file`、是否 `--use-rag` 等。
2. 按模型名分发到对应 `*_utils.py`：`get_gpt_answers / get_claude_answers / get_gemini_answers / get_hf_answers`。
3. 逐样本生成预测答案 → 调 `evaluation.eval_question_answering` 打分 → 写回输出文件。
4. 调 `evaluation_stats.analyze_aggr_acc` 汇总整体与分类别准确率。

### 7.2 上下文截断策略（`gpt_utils.get_input_context`）

对话通常远超模型上下文窗口。代码从**最新 session 向前倒推**，逐 turn 累加 token，直到逼近该模型的 `MAX_LENGTH` 预算（减去问题与答案的预留 token），从而把"最近的、最可能相关"的对话塞进上下文。各模型的窗口长度在 `gpt_utils.MAX_LENGTH` 中列出。

### 7.3 RAG 变体

`--use-rag` 时，不再塞整段对话，而是从三种**数据库**中检索 top-k 片段：

| `--rag-mode` | 数据库内容 | 说明 |
|--------------|-----------|------|
| `dialog` | 原始轮次 | 按 turn 粒度检索 |
| `observation` | 每个 session 的客观观察 | 抽取式事实 |
| `session_summary` | 每个 session 的摘要 | 概括式事实 |

检索默认用 **contriever**，通过点积相似度取 top-k（`--top-k`），再把命中文本拼进 prompt。

### 7.4 评测指标（`task_eval/evaluation.py`）

- **F1（主指标，`f1_score` / `f1`）**：衡量预测答案与标准答案在**词层面重合度**的指标，是 Precision 与 Recall 的调和平均；答对一部分也给部分分（区别于非 0 即 1 的 EM）。算法：
    1. **归一化** `normalize_answer`：小写 → 去标点 → 去冠词（a/an/the/and）→ 去逗号 → 合并多余空格；
    2. **分词 + 词干还原**（PorterStemmer）：`running→run`、`cats→cat`，让不同词形算同一个词；
    3. **Precision** = 交集 token 数 / 预测 token 数；**Recall** = 交集 token 数 / 真值 token 数；
    4. **F1 = 2·P·R / (P+R)**。

    例：真值 `7 May 2023`、预测 `May 2023` → 交集 {may, 2023}=2，P=2/2=1.0、R=2/3≈0.667，F1≈0.8。
  - **单答案 F1（`f1_score`）**：一段预测对一段真值算一次——题型表里的"标准单答案 F1"指它（用于时间型、开放域、单跳）。
  - **多答案 F1（`f1`）**：预测与真值都按 `,` 切成子答案，对每个真值子答案在预测子答案里取最高 F1 再求均值——多跳题（category 1）用这个，奖励"答对其中一个"。
- **Exact Match**：归一后词集合相等。
- **ROUGE-L / BERTScore**：在 `rougel_score / bert_score` 中实现（用于对话式任务）。
- **Recall**：RAG 模式下，统计检索到的上下文是否覆盖了标注的 `evidence`（dialog id）。

### 7.5 支持的模型

OpenAI（gpt-3.5-turbo 各窗口变体、gpt-4、gpt-4-turbo、gpt-4-32k）、Anthropic Claude、Google Gemini、HuggingFace 开源模型（gemma / llama / mistral，支持 4bit 量化 `--use-4bit`）。

---

## 八、快速上手

### 8.1 环境准备

```bash
# 用 requirements.txt 创建 conda 环境（python 3.9 + torch 2.0 + cu117）
conda create --name locomo --file requirements.txt
conda activate locomo
```

### 8.2 填写 API Key

编辑 `scripts/env.sh`，填入：

```bash
export OPENAI_API_KEY=...
export GOOGLE_API_KEY=...
export ANTHROPIC_API_KEY=...
export HF_TOKEN=...
```

### 8.3 跑通 QA 评测（以 GPT 为例）

```bash
source scripts/env.sh
bash scripts/evaluate_gpts.sh
```

脚本内部会调用：

```bash
python task_eval/evaluate_qa.py \
  --data-file ./data/locomo10.json \
  --out-file ./outputs/locomo10_qa.json \
  --model gpt-4-turbo
```

### 8.4 跑 RAG 评测

```bash
bash scripts/evaluate_rag_gpts.sh
# 关键参数：--use-rag --rag-mode {dialog|observation|summary} --top-k 5 --retriever contriever
```

### 8.5 重新生成 observation / session 摘要

仓库已附带生成好的 observation 与 summary。若要自建：

```bash
bash scripts/generate_observations.sh        # 生成 observation
bash scripts/generate_session_summaries.sh  # 生成 session 摘要
```

### 8.6 造一段自己的超长对话

```bash
# 方式 A：自定义人设（放 agent_a.json / agent_b.json 到 --out-dir）
bash scripts/generate_conversations.sh

# 方式 B：从 MSC 人设池随机采样（--out-dir 指向空目录即可）
```

---

## 九、学习要点小结

1. **"生成式造数"思路**：用 LLM + 事件知识图谱来造可控、有因果的长对话，比纯人工或纯自由生成更结构化。
2. **记忆的三种表征**：原始 dialog / observation / session_summary，正好对应 RAG 的三种粒度，便于对比检索粒度的影响。
3. **截断 vs 检索的对照**：同一份对话，既测"硬塞进窗口"的上限，又测"检索增强"的天花板，是一组很干净的对照实验。
4. **分类别、分难度评测**：QA 拆成单跳/多跳/时间/开放域/对抗五类，对抗题用"未提及即正确"来抑制幻觉，这种设计值得借鉴。
5. **多模型统一接口**：`global_methods.run_chatgpt` + 各 `*_utils.py` 把 OpenAI/Claude/Gemini/HF 统一进同一评测流程，便于横向对比。

---

## 十、参考

- 论文 PDF：`static/paper/locomo.pdf`
- 项目主页：`index.html`
- 上游仓库：[snap-research/locomo](https://github.com/snap-research/locomo)
- arXiv：[2402.17753](https://arxiv.org/abs/2402.17753)

```bibtex
@article{maharana2024evaluating,
  title={Evaluating very long-term conversational memory of llm agents},
  author={Maharana, Adyasha and Lee, Dong-Ho and Tulyakov, Sergey and Bansal, Mohit and Barbieri, Francesco and Fang, Yuwei},
  journal={arXiv preprint arXiv:2402.17753},
  year={2024}
}
```
