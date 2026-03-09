# Retail Factor Agent 使用说明书

这份文档是 `retail_factor_agent` 的完整说明。  
它面向“第一次接触项目的人”，尤其是非纯技术背景的读者。你可以把它当作一本小型操作手册：先理解项目在解决什么问题，再按步骤运行，最后学会看结果。

---

## 一、这个 Agent 到底在做什么

一句话概括：**它把散户帖子中的主观表达，自动转换成可量化的因子数据，然后结合历史总表训练模型，最终告诉你“大家在想什么、为什么这么想、这些想法和市场结果是否一致”。**

你可以把它想成一个“研究助理”：

- 它先帮你读大量帖子
- 再帮你提炼统一口径的因子
- 然后用同一时间窗口的历史数据训练基准模型
- 最后给出解释性分析，而不是只给一个黑盒结论

---

## 二、为什么需要它

研究散户认知时，人工方式通常会遇到三个问题：

1. **看不完**：帖子量太大，人工无法覆盖足够样本  
2. **看不齐**：不同人记因子的方式不同，统计口径容易飘  
3. **看不透**：容易被情绪或叙事吸引，忽略真正驱动判断的因素

这个 Agent 的价值，就是把这些主观、分散、噪声高的文本信息，变成结构化、可复盘、可比较的分析结果。

---

## 三、这个 Agent 回答哪些核心问题

它主要服务四类问题：

- 某个时间段，散户最关注哪些因子？
- 这些因子被看成利多还是利空？强度大概如何？
- 这些认知与市场实际表现一致还是背离？
- 哪些因子对判断贡献最大，可能导致了偏差？

---

## 四、整体流程

整个流程由七步组成，推荐按顺序理解：

### 第 1 步：Wind 同步（可选）
对接 Wind 数据流程，处理并合并市场数据与因子表。  
这一步不是每次都必须跑，但当底层数据更新时建议执行。

### 第 2 步：数据获取（帖子）
读取本地帖子源，或者在需要时在线爬取作为兜底来源。  
随后按你指定的开始/结束日期做时间过滤。

### 第 3 步：抽样前清洗（关键）
在抽样之前先过滤低质量文本，默认规则包括：

- 正文最小字数过滤
- 剔除疑似 AI 生成内容
- 剔除公告类文本
- 仅保留有逻辑信号的帖子

这样做的目的是先控质量，再控数量。

### 第 4 步：LLM 因子提取
对清洗后的帖子调用大模型，提取三类结构化信息：

- 因子名称
- 因子数值
- 因子影响方向（利多/利空/中性）

并且有白名单约束，防止模型“自由发挥”生成不在体系内的因子名。

### 第 5 步：总表重训（关键）
先按用户当前时间段过滤总因子表，再训练模型。  
这确保“训练口径”和“用户帖子分析口径”一致，不会出现跨周期错配。

### 第 6 步：用户因子解释分析
把用户帖子因子输入训练好的模型，输出每条帖子在模型中的贡献因子和方向判断。  
你不止能看到“看多/看空”，还能看到“为什么这么判断”。

### 第 7 步：可视化
自动生成图表用于报告或展示，如：

- 模型指标图
- 因子权重图
- 因子相关性图
- 用户预测分布图

---

## 五、目录说明（按职责理解）

- `config.py`：默认参数和路径配置中心  
- `pipeline.py`：总入口，负责任务编排  
- `steps/crawl_posts.py`：数据读取、清洗、抽样  
- `steps/llm_extract.py`：LLM 抽取与白名单过滤  
- `steps/train_model.py`：总表重训 + 用户因子解释  
- `steps/factor_analysis.py`：相关性统计分析  
- `steps/visualize.py`：图表输出  
- `steps/wind_sync.py`：Wind 流程衔接  
- `workspace/`：所有运行产物存放目录

---

## 六、如何运行（推荐顺序）

### 1) 一键全流程

```bash
python -m retail_factor_agent.pipeline --all
```

适合初次体验，直接跑完从数据到图表。

### 2) 分步运行（便于排错）

```bash
python -m retail_factor_agent.pipeline --crawl
python -m retail_factor_agent.pipeline --llm
python -m retail_factor_agent.pipeline --train --analyze --viz
```

适合你想检查每一步结果是否合理时使用。

### 3) 指定时间窗口与清洗强度

```bash
python -m retail_factor_agent.pipeline --start-date 2021-01-01 --end-date 2023-12-31 --sample-size 5000 --min-chars 120 --crawl --llm --train --analyze --viz
```

这是做阶段复盘时最常用的方式。

### 4) 仅执行 Wind 处理

```bash
python -m retail_factor_agent.pipeline --wind
```

若需要重新拉取 Wind 数据，可在支持环境下附加 `--wind-fetch`。

---

## 七、运行前准备

### 依赖安装

```bash
pip install -r retail_factor_agent/requirements.txt
```

### 密钥配置

LLM 步骤至少需要一个环境变量：

- `LLM_API_KEY`
- 或 `OPENAI_API_KEY`

### 数据基础

默认使用：

- 因子总表：`analysis_results/matched_data_2024.csv`
- 帖子源：`run_data/guba_posts_cleaned_min100.csv`

---

## 八、关键参数说明（实操版）

- `--start-date` / `--end-date`：时间窗口
- `--sample-size`：清洗后随机抽样数量
- `--min-chars`：最小正文长度
- `--crawl --llm --train --analyze --viz`：分步开关
- `--wind`：Wind 同步处理
- `--symbols`：在线爬取时的代码列表（逗号分隔）
- `--allow-ai`：允许 AI 文本通过（默认不允许）
- `--allow-announcement`：允许公告通过（默认不允许）
- `--no-logic-filter`：关闭逻辑过滤（默认开启）

建议默认保持严格过滤，先保证质量再扩大样本。

---

## 九、结果文件怎么读

以下文件最关键，建议按顺序看：

1. `workspace/outputs/llm_factor_table.csv`  
   帖子级结构化因子表（可作为后续分析输入）

2. `workspace/outputs/training_master_table_filtered.csv`  
   当前时间窗口下的训练数据快照（保证可复盘）

3. `workspace/outputs/ml_metrics.csv`  
   模型效果指标（用于看稳定性）

4. `workspace/outputs/factor_weights.csv`  
   因子重要性/权重（用于解释模型）

5. `workspace/outputs/factor_correlation.csv`  
   因子与收益/方向相关性（用于有效性分析）

6. `workspace/outputs/user_factor_analysis.csv`  
   每条帖子最关键贡献因子（最适合展示解释能力）

7. `workspace/outputs/viz_*.png`  
   可直接用于报告的图表结果

---

## 十、质量保障机制（为什么结果可控）

为避免“看起来智能但不可控”，项目有三层防线：

1. **白名单约束**：因子名称必须来自总表体系  
2. **后处理过滤**：LLM 越界输出会被二次剔除  
3. **时间段重训**：每次训练都在当前窗口重建数据

这三层保证了结果的稳定性、解释性和可复现性。

