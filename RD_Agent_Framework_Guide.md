# RD-Agent 框架模块说明文档

> 自顶向下、自核心到外围的功能模块与接口说明
> 适用场景：迁移参考、量化研究工作流对齐

---

## 目录

1. [框架总体架构](#1-框架总体架构)
2. [核心抽象层（Core）](#2-核心抽象层core)
   - 2.1 [Scenario — 场景定义](#21-scenario--场景定义)
   - 2.2 [Task & Experiment — 任务与实验](#22-task--experiment--任务与实验)
   - 2.3 [Hypothesis & Trace — 假设与历史轨迹](#23-hypothesis--trace--假设与历史轨迹)
   - 2.4 [Proposal 接口族 — 提案生成](#24-proposal-接口族--提案生成)
   - 2.5 [Developer — 开发者接口](#25-developer--开发者接口)
   - 2.6 [Evaluation & Feedback — 评估与反馈](#26-evaluation--feedback--评估与反馈)
   - 2.7 [KnowledgeBase — 知识库](#27-knowledgebase--知识库)
   - 2.8 [EvolvingFramework — 演化框架](#28-evolvingframework--演化框架)
3. [工作流引擎（Workflow）](#3-工作流引擎workflow)
   - 3.1 [LoopBase & LoopMeta — 循环调度器](#31-loopbase--loopmeta--循环调度器)
   - 3.2 [RDLoop — R&D 主循环](#32-rdloop--rd-主循环)
4. [组件层（Components）](#4-组件层components)
   - 4.1 [CoSTEER — 代码演化编码器](#41-costeer--代码演化编码器)
   - 4.2 [FactorCoder — 因子编码器](#42-factorcoder--因子编码器)
   - 4.3 [ModelCoder — 模型编码器](#43-modelcoder--模型编码器)
   - 4.4 [DataScience Coder — 数据科学编码器族](#44-datascience-coder--数据科学编码器族)
   - 4.5 [KnowledgeManagement — 向量知识库](#45-knowledgemanagement--向量知识库)
   - 4.6 [DocumentReader — 文档读取器](#46-documentreader--文档读取器)
   - 4.7 [Loader — 任务/实验加载器](#47-loader--任务实验加载器)
   - 4.8 [Agent — 智能体基类](#48-agent--智能体基类)
5. [场景层（Scenarios）](#5-场景层scenarios)
   - 5.1 [Qlib Factor — 量化因子挖掘](#51-qlib-factor--量化因子挖掘)
   - 5.2 [Qlib Model — 量化模型构建](#52-qlib-model--量化模型构建)
   - 5.3 [Qlib Quant — 因子+模型联合优化](#53-qlib-quant--因子模型联合优化)
   - 5.4 [Factor from Report — 从研报提取因子](#54-factor-from-report--从研报提取因子)
   - 5.5 [DataScience — Kaggle/通用数据科学](#55-datascience--kaggle通用数据科学)
   - 5.6 [General Model — 通用模型实现](#56-general-model--通用模型实现)
   - 5.7 [Finetune — 模型微调](#57-finetune--模型微调)
6. [LLM 接入层（OAI）](#6-llm-接入层oai)
7. [运行环境层（Env）](#7-运行环境层env)
8. [日志与监控（Log）](#8-日志与监控log)
9. [配置系统（Config）](#9-配置系统config)
10. [Web UI 层](#10-web-ui-层)
11. [CLI 入口](#11-cli-入口)
12. [数据接口与依赖汇总](#12-数据接口与依赖汇总)
13. [迁移参考：核心/可选/可删减模块](#13-迁移参考核心可选可删减模块)

---

## 1. 框架总体架构

```
┌─────────────────────────────────────────────────────────┐
│                    CLI / App 入口层                       │
│   rdagent/app/cli.py   rdagent/app/qlib_rd_loop/        │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│               工作流引擎 (Workflow Engine)                 │
│   rdagent/utils/workflow/loop.py  (LoopBase)             │
│   rdagent/components/workflow/rd_loop.py  (RDLoop)       │
└──────────────────────────┬──────────────────────────────┘
                           │ 调用
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
    ▼                      ▼                      ▼
┌────────┐         ┌───────────┐         ┌──────────────┐
│Proposal│         │ Developer │         │  Evaluator   │
│(假设生成)│         │(代码开发)  │         │ (反馈生成)    │
└────────┘         └───────────┘         └──────────────┘
    │                      │                      │
    └──────────────────────┼──────────────────────┘
                           │ 依托
┌──────────────────────────▼──────────────────────────────┐
│                    核心抽象层 (Core)                       │
│  Scenario / Task / Experiment / Hypothesis / Trace       │
│  KnowledgeBase / EvolvingFramework                       │
└──────────────────────────┬──────────────────────────────┘
                           │ 基础服务
┌──────────────────────────▼──────────────────────────────┐
│             基础服务层 (Infrastructure)                    │
│   OAI (LLM调用)  |  Env (Docker执行)  |  Log (日志)      │
└─────────────────────────────────────────────────────────┘
```

**核心设计哲学**：
- **R&D 循环**：Propose（生成假设）→ Code（实现代码）→ Run（执行实验）→ Feedback（生成反馈）→ Record（记录历史）→ 循环
- **高度可扩展**：每个阶段均为可插拔的抽象类，通过字符串类路径配置注入
- **场景解耦**：通用框架 + 场景具体化实现，场景间完全独立

---

## 2. 核心抽象层（Core）

> **代码路径**: `rdagent/core/`

### 2.1 Scenario — 场景定义

**文件**: `rdagent/core/scenario.py`

**意义**: 场景是整个框架的"世界观"，定义了任务背景、数据描述、运行时环境等场景级信息，是 Proposal/Developer/Evaluator 等组件的统一上下文来源。

**核心接口**:
```python
class Scenario(ABC):
    @property
    @abstractmethod
    def background(self) -> str:
        """场景背景描述，注入到 LLM 提示词中"""

    def get_source_data_desc(self, task: Task | None = None) -> str:
        """数据源描述，LLM 需要了解有哪些可用数据"""

    @property
    @abstractmethod
    def rich_style_description(self) -> str:
        """Rich 格式的场景展示描述"""

    @abstractmethod
    def get_scenario_all_desc(self, task=None, filtered_tag=None, simple_background=None) -> str:
        """汇总所有描述，传给 LLM 作完整上下文"""

    @abstractmethod
    def get_runtime_environment(self) -> str:
        """运行时环境信息（Python 版本、可用库等）"""

    @property
    def experiment_setting(self) -> str | None:
        """实验设置（时间窗口、回测区间等）"""
```

**用法**: 每个具体场景子类实现以上接口，在 RDLoop 初始化时由字符串类路径实例化（`PROP_SETTING.scen`）。

**迁移关键点**:
- 必须实现：`background`、`get_scenario_all_desc`、`get_runtime_environment`
- 量化场景需要在 `get_source_data_desc` 中描述可用字段（OHLCV、财务指标等）
- `experiment_setting` 填写回测时间区间

---

### 2.2 Task & Experiment — 任务与实验

**文件**: `rdagent/core/experiment.py`

**意义**:
- `Task` 是最小执行单元，包含任务名称和描述（如"计算动量因子"）
- `Experiment` 是一批 Task 的集合，以及对应的代码工作区，是一次完整的 R&D 尝试

**核心类**:

```python
class Task(AbsTask):
    name: str
    description: str           # 任务描述，LLM 理解并实现的目标
    user_instructions: str     # 用户补充指令（最高优先级）

class Workspace(ABC):
    """代码工作区，存储实现代码"""
    def execute(self, env, entry) -> str: ...  # 执行代码
    def copy() -> Workspace: ...               # 快照复制

class FBWorkspace(Workspace):
    """基于文件的工作区（最常用）"""
    file_dict: dict[str, str]  # 文件名 -> 代码内容
    workspace_path: Path        # 实际文件目录

    def inject_files(**files): ...     # 注入/更新代码文件
    def create_ws_ckp(): ...           # 创建工作区快照
    def recover_ws_ckp(): ...          # 恢复工作区快照

class Experiment(ABC):
    sub_tasks: list[Task]               # 子任务列表
    sub_workspace_list: list[Workspace] # 每个子任务的代码工作区
    experiment_workspace: Workspace     # 聚合实验工作区
    based_experiments: list[Workspace]  # 继承的已有实验
    hypothesis: Hypothesis              # 产生本次实验的假设
    result: object                      # 执行结果（回测指标等）
    plan: ExperimentPlan                # 本次实验的规划信息
```

**迁移关键点**:
- 量化因子场景：一个 Experiment 包含多个 FactorTask（每个因子一个 Task）
- 每个 Task 对应一段因子计算代码（Python 文件）
- `based_experiments` 存储历史 SOTA 实验，用于增量构建

---

### 2.3 Hypothesis & Trace — 假设与历史轨迹

**文件**: `rdagent/core/proposal.py`

**意义**:
- `Hypothesis` 是 Agent 在每轮循环中产生的研究假设（某个因子/模型思路）
- `Trace` 是完整的实验历史 DAG，记录每次实验及其反馈，用于指导下一轮的假设生成

**核心类**:

```python
class Hypothesis:
    hypothesis: str           # 假设内容（"动量效应在A股短期内显著"）
    reason: str               # 详细推理
    concise_reason: str       # 简洁推理
    concise_observation: str  # 观察到的现象
    concise_justification: str # 逻辑支撑
    concise_knowledge: str    # 引用的知识

class Trace:
    scen: Scenario
    hist: list[tuple[Experiment, ExperimentFeedback]]  # 历史实验+反馈序列
    dag_parent: list[tuple[int, ...]]                  # 实验间的 DAG 父子关系
    knowledge_base: KnowledgeBase

    def get_sota_hypothesis_and_experiment() -> (Hypothesis, Experiment):
        """获取当前 SOTA 实验"""
    def get_parent_exps(selection) -> list:
        """获取特定节点的祖先链"""
    def sync_dag_parent_and_hist(exp_and_fb, loop_id):
        """追加新实验到历史"""
```

**迁移关键点**:
- Trace 是跨轮次的记忆核心，是 LLM 生成假设的主要输入
- DAG 结构支持多路径探索（并行实验）
- 最简实现可以退化为线性 list，仅保留 `hist`

---

### 2.4 Proposal 接口族 — 提案生成

**文件**: `rdagent/core/proposal.py`

**意义**: 定义了从"观察历史"到"产生假设"再到"设计实验"的完整提案链条。

**核心接口**:

```python
class HypothesisGen(ABC):
    """基于历史轨迹，生成下一个研究假设"""
    def gen(self, trace: Trace, plan: ExperimentPlan | None) -> Hypothesis: ...

class Hypothesis2Experiment(ABC):
    """将假设转化为可执行实验（含 Task 列表）"""
    def convert(self, hypothesis: Hypothesis, trace: Trace) -> Experiment: ...

class ExpGen(ABC):
    """上述两者的组合，直接从 Trace 生成 Experiment"""
    def gen(self, trace: Trace) -> Experiment: ...
    async def async_gen(self, trace: Trace, loop: LoopBase) -> Experiment: ...

class Experiment2Feedback(ABC):
    """从执行后的 Experiment 生成反馈"""
    def generate_feedback(self, exp: Experiment, trace: Trace) -> ExperimentFeedback: ...

class ExperimentFeedback(Feedback):
    decision: bool       # 是否接受此次实验（True=改进了，False=没有改进）
    reason: str          # 反馈原因
    code_change_summary: str  # 代码变化摘要
    exception: Exception      # 运行异常（如果有）

class HypothesisFeedback(ExperimentFeedback):
    observations: str         # 实验观察
    hypothesis_evaluation: str # 假设评估
    new_hypothesis: str        # 建议的新假设
    acceptable: bool           # 是否可接受
```

**迁移关键点**:
- `HypothesisGen.gen()` 是 LLM 调用的核心，提示词注入 Trace 历史
- `Hypothesis2Experiment.convert()` 决定"本次实验做几个因子/任务"
- `Experiment2Feedback.generate_feedback()` 是评估器，比较当前与 SOTA 的回测指标

---

### 2.5 Developer — 开发者接口

**文件**: `rdagent/core/developer.py`

**意义**: `Developer` 是代码开发的统一抽象。在不同阶段有不同的 Developer 实现（Coder、Runner）。

```python
class Developer(ABC, Generic[ASpecificExp]):
    def develop(self, exp: ASpecificExp) -> ASpecificExp:
        """
        接收实验对象（含 Task 列表），
        对其 in-place 修改（填充 sub_workspace_list 的代码）
        """
```

**常见子类**:
- `FactorCoSTEER` → 负责代码生成（Coder 阶段）
- `QlibFactorRunner` → 负责运行回测（Runner 阶段）

---

### 2.6 Evaluation & Feedback — 评估与反馈

**文件**: `rdagent/core/evaluation.py`

**意义**: 轻量抽象，`Evaluator` 负责把执行结果转化为 `Feedback` 对象。

```python
class Feedback:
    def __bool__(self) -> bool: ...  # True = 成功/接受
    def is_acceptable(self) -> bool: ...
    def finished(self) -> bool: ...

class Evaluator(ABC):
    def evaluate(self, eo: EvaluableObj) -> Feedback: ...
```

---

### 2.7 KnowledgeBase — 知识库

**文件**: `rdagent/core/knowledge_base.py`

**意义**: 用 dill/pickle 持久化的知识存储，可在多次运行间保留学习到的知识。

```python
class KnowledgeBase:
    path: Path  # 持久化路径

    def load() -> None:   # 从磁盘加载
    def dump() -> None:   # 保存到磁盘
```

子类如 `CoSTEERKnowledgeBase` 存储因子/模型的成功/失败经验，用于 RAG 检索。

---

### 2.8 EvolvingFramework — 演化框架

**文件**: `rdagent/core/evolving_framework.py`

**意义**: 为迭代式代码演化提供抽象基础（CoSTEER 的底层框架）。

```python
class EvolvingStrategy(ABC):
    def evolve_iter(self, evo, queried_knowledge, evolving_trace) -> Generator:
        """迭代演化，每次 yield 一个部分实现的 evo"""

class RAGStrategy(ABC):
    """检索增强生成策略"""
    def query(self, evo, evolving_trace) -> QueriedKnowledge: ...
    def generate_knowledge(self, evolving_trace) -> Knowledge: ...
    def dump_knowledge_base(): ...
    def load_dumped_knowledge_base(): ...

@dataclass
class EvoStep:
    evolvable_subjects: EvolvableSubjects  # 当前代码状态
    queried_knowledge: QueriedKnowledge    # 检索到的知识
    feedback: Feedback                     # 本步反馈
```

---

## 3. 工作流引擎（Workflow）

### 3.1 LoopBase & LoopMeta — 循环调度器

**文件**: `rdagent/utils/workflow/loop.py`

**意义**: 基于 Python metaclass + asyncio 的可序列化循环调度框架，支持断点续跑、并行执行、步骤级状态管理。

**核心机制**:

```python
class LoopMeta(type):
    """元类，自动扫描子类中的公开方法，按定义顺序构建 steps 列表"""

class LoopBase:
    steps: list[str]  # 如 ["direct_exp_gen", "coding", "running", "feedback", "record"]

    async def run(self, step_n=None, loop_n=None, all_duration=None):
        """主运行入口，按 steps 顺序执行，支持 async 步骤"""

    @classmethod
    def load(cls, path, checkout=True) -> LoopBase:
        """从磁盘加载检查点（pickle），续跑"""

    def dump(self, path): ...  # 持久化当前状态
```

**关键特性**:
- `skip_loop_error`：指定可跳过的异常类型，失败不中断整体循环
- `withdraw_loop_error`：指定需要回滚的异常类型
- `step_semaphore`：并行度控制（每步最多同时运行 N 个循环）
- 每步输出通过 `prev_out: dict[str, Any]` 传递给下一步

---

### 3.2 RDLoop — R&D 主循环

**文件**: `rdagent/components/workflow/rd_loop.py`

**意义**: RD-Agent 标准 R&D 循环的具体实现，集成了假设生成、代码开发、运行回测、反馈汇总的完整流程。

**步骤序列**:
```
direct_exp_gen → coding → running → feedback → record
```

**各步说明**:

| 步骤 | 方法 | 调用对象 | 输出 |
|------|------|---------|------|
| `direct_exp_gen` | `RDLoop.direct_exp_gen()` | `hypothesis_gen` + `hypothesis2experiment` | `{"propose": Hypothesis, "exp_gen": Experiment}` |
| `coding` | `RDLoop.coding()` | `self.coder.develop(exp)` | Experiment（filled workspaces） |
| `running` | `RDLoop.running()` | `self.runner.develop(exp)` | Experiment（with results） |
| `feedback` | `RDLoop.feedback()` | `self.summarizer.generate_feedback()` | `HypothesisFeedback` |
| `record` | `RDLoop.record()` | `trace.sync_dag_parent_and_hist()` | 更新 Trace |

**配置注入** (`BasePropSetting`):
```python
class BasePropSetting(ExtendedBaseSettings):
    scen: str                  # 场景类路径
    hypothesis_gen: str        # 假设生成器类路径
    hypothesis2experiment: str # 假设→实验转换器类路径
    coder: str                 # 编码器类路径
    runner: str                # 运行器类路径
    summarizer: str            # 反馈汇总器类路径
    knowledge_base: str        # 知识库类路径（可选）
    evolving_n: int = 10       # CoSTEER 演化轮数
```

**用法**:
```python
# 启动新循环
loop = FactorRDLoop(FACTOR_PROP_SETTING)
asyncio.run(loop.run(loop_n=50))

# 续跑
loop = FactorRDLoop.load("path/to/checkpoint")
asyncio.run(loop.run(step_n=5))
```

---

## 4. 组件层（Components）

> **代码路径**: `rdagent/components/`

### 4.1 CoSTEER — 代码演化编码器

**文件**: `rdagent/components/coder/CoSTEER/`

**意义**: RD-Agent 的核心代码生成引擎，实现了"多轮迭代+RAG+知识沉淀"的演化式编码策略（Collaborative Self-Training for Evolving Efficient Research，CoSTEER）。

**子模块结构**:
```
CoSTEER/
├── config.py              # CoSTEERSettings
├── evolvable_subjects.py  # EvolvingItem（待演化代码对象）
├── evolving_strategy.py   # MultiProcessEvolvingStrategy（核心演化逻辑）
├── evaluators.py          # CoSTEERSingleFeedback / CoSTEERMultiFeedback
├── knowledge_management.py # CoSTEERQueriedKnowledge / CoSTEERKnowledgeBase
└── prompts.yaml           # LLM 提示词模板
```

**核心流程**:
```
Experiment(sub_tasks)
    └─ for each task:
        ├─ RAG: 查询知识库（相似成功/失败案例）
        ├─ LLM: implement_one_task(task, queried_knowledge)
        ├─ Evaluate: 运行代码检查语法/逻辑
        └─ UpdateKB: 把新经验存入知识库
```

**关键配置** (`CoSTEERSettings`):
```python
max_loop: int = 3         # 单个任务最多演化轮数
fail_task_trial_limit: int = 5  # 任务失败跳过阈值
```

---

### 4.2 FactorCoder — 因子编码器

**文件**: `rdagent/components/coder/factor_coder/`

**意义**: 基于 CoSTEER 的量化因子代码生成器，生成 Qlib 格式的因子计算代码。

**核心类**:
```python
class FactorTask(Task):
    factor_name: str          # 因子名称
    factor_description: str   # 因子描述
    factor_formulation: str   # 数学公式表达
    variables: dict           # 变量字典（数据字段映射）
    factor_resources: str     # 参考资源

class FactorFBWorkspace(FBWorkspace):
    """因子代码工作区，inject factor.py 等文件"""

class FactorExperiment(Experiment):
    """包含多个 FactorTask 的实验"""

class FactorCoSTEER(Developer):
    """基于 CoSTEER 的因子编码器（Developer 的具体实现）"""
```

**生成代码格式**: 
- `factor.py`：包含 `get_data()` 函数，返回 Qlib Expression 或 DataFrame

**所需数据接口**:
- Qlib 数据格式（`$open`, `$close`, `$volume` 等字段）
- 因子执行模板：`factor_execution_template.txt`

---

### 4.3 ModelCoder — 模型编码器

**文件**: `rdagent/components/coder/model_coder/`

**意义**: 基于 CoSTEER 的量化预测模型代码生成器，生成 PyTorch/sklearn 格式的机器学习模型代码。

**核心类**:
```python
class ModelTask(Task):
    model_type: str        # "Tabular" / "TimeSeries" / "Graph"
    architecture: str      # 模型架构描述
    hyperparameters: dict  # 超参数

class ModelCoSTEER(Developer):
    """基于 CoSTEER 的模型编码器"""
```

**生成代码格式**:
- `model.py`：包含 `Net` 类（PyTorch Module）或 Qlib Model 接口

---

### 4.4 DataScience Coder — 数据科学编码器族

**文件**: `rdagent/components/coder/data_science/`

**意义**: 针对 Kaggle/数据科学竞赛的多阶段专业编码器，每个阶段独立负责一个模块。

```
data_science/
├── raw_data_loader/   # 数据加载器（DataLoaderCoSTEER）
├── feature/           # 特征工程（FeatureCoSTEER）
├── model/             # 模型构建（ModelCoSTEER）
├── ensemble/          # 集成学习（EnsembleCoSTEER）
├── pipeline/          # 完整流程（PipelineCoSTEER）
├── workflow/          # 工作流组合（WorkflowCoSTEER）
└── share/             # 共享组件（DocDev 文档开发）
```

---

### 4.5 KnowledgeManagement — 向量知识库

**文件**: `rdagent/components/knowledge_management/`

**意义**: 基于向量相似度的知识检索，为 CoSTEER 提供 RAG 支持。

```python
# vector_base.py
class KnowledgeMetaData:
    content: str       # 知识内容
    label: str         # 分类标签
    embedding: list    # 向量表示

    def create_embedding(): ...       # 调用 Embedding API
    def split_into_trunk(size=1000): ... # 长文本分块

class VectorBase(KnowledgeBase):
    """向量检索知识库"""
    def search(query_embedding, topk=3) -> list[KnowledgeMetaData]: ...
```

```python
# graph.py  
class UndirectedGraph:
    """图结构知识组织（用于知识推理）"""
```

**依赖**: LLM Embedding API（`text-embedding-3-small` 或兼容接口）

---

### 4.6 DocumentReader — 文档读取器

**文件**: `rdagent/components/document_reader/document_reader.py`

**意义**: 从研究报告（PDF）等文档中提取结构化信息。

**功能**:
- 支持 PDF 解析（依赖 Azure Document Intelligence 或本地解析）
- 输出结构化文本，供因子提取流程使用

**所需配置**:
```ini
AZURE_DOCUMENT_INTELLIGENCE_KEY=...
AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=...
```

---

### 4.7 Loader — 任务/实验加载器

**文件**: `rdagent/components/loader/`

**意义**: 从外部数据源加载 Task 或 Experiment 对象，实现研究任务的预定义导入。

```python
# task_loader.py
class TaskLoader(Loader[Task]):
    """从 JSON/YAML 文件加载任务列表"""

# experiment_loader.py  
class ExperimentLoader(Loader[Experiment]):
    """加载预定义实验"""
```

**Qlib 专用 Loader** (`rdagent/scenarios/qlib/factor_experiment_loader/`):
- `pdf_loader.py`：从研报 PDF 提取因子 Task
- `json_loader.py`：从 JSON 文件加载预定义因子 Task

---

### 4.8 Agent — 智能体基类

**文件**: `rdagent/components/agent/`

**意义**: 提供工具调用型智能体（Tool-use Agent）的基础封装，支持 MCP（Model Context Protocol）工具集成。

```python
class PAIAgent(BaseAgent):
    """基于 pydantic-ai 的 MCP Agent"""
    def __init__(self, system_prompt: str, toolsets: list[str], enable_cache: bool): ...
    def query(self, query: str) -> str: ...
```

- 支持 `RAGAgent`（基于向量检索的问答代理）
- 支持 MCP server 工具调用

---

## 5. 场景层（Scenarios）

> **代码路径**: `rdagent/scenarios/`

### 5.1 Qlib Factor — 量化因子挖掘

**代码路径**: `rdagent/scenarios/qlib/` + `rdagent/app/qlib_rd_loop/factor.py`

**意义**: 自动化挖掘量化 alpha 因子，在 Qlib 框架上回测并筛选。

**完整类链**:

| 角色 | 类 | 文件 |
|-----|----|------|
| 场景 | `QlibFactorScenario` | `scenarios/qlib/experiment/factor_experiment.py` |
| 假设生成 | `QlibFactorHypothesisGen` | `scenarios/qlib/proposal/factor_proposal.py` |
| 假设→实验 | `QlibFactorHypothesis2Experiment` | `scenarios/qlib/proposal/factor_proposal.py` |
| 编码器 | `QlibFactorCoSTEER`→`FactorCoSTEER` | `scenarios/qlib/developer/factor_coder.py` |
| 运行器 | `QlibFactorRunner` | `scenarios/qlib/developer/factor_runner.py` |
| 反馈器 | `QlibFactorExperiment2Feedback` | `scenarios/qlib/developer/feedback.py` |
| 实验对象 | `QlibFactorExperiment` | `scenarios/qlib/experiment/factor_experiment.py` |
| 工作区 | `QlibFBWorkspace` | `scenarios/qlib/experiment/workspace.py` |

**关键配置** (`FactorBasePropSetting`，前缀 `QLIB_FACTOR_`):
```ini
QLIB_FACTOR_TRAIN_START=2008-01-01
QLIB_FACTOR_TRAIN_END=2014-12-31
QLIB_FACTOR_VALID_START=2015-01-01
QLIB_FACTOR_VALID_END=2016-12-31
QLIB_FACTOR_TEST_START=2017-01-01
QLIB_FACTOR_TEST_END=2020-08-01
QLIB_FACTOR_EVOLVING_N=10
```

**所需数据**:
- Qlib 格式股票日线数据（open/high/low/close/volume/vwap 等）
- Qlib 数据路径：通过 `QlibFactorScenario.get_source_data_desc()` 描述

**核心评估指标** (`IMPORTANT_METRICS`):
- `IC`（信息系数）
- `1day.excess_return_with_cost.annualized_return`（年化超额收益）
- `1day.excess_return_with_cost.max_drawdown`（最大回撤）

**启动方式**:
```bash
rdagent fin_factor              # 新建循环
rdagent fin_factor $LOG_PATH    # 续跑
```

---

### 5.2 Qlib Model — 量化模型构建

**代码路径**: `rdagent/scenarios/qlib/` + `rdagent/app/qlib_rd_loop/model.py`

**意义**: 自动化构建量化预测模型（Alpha 模型），在 Qlib 框架上训练和回测。

**完整类链**:

| 角色 | 类 | 文件 |
|-----|----|------|
| 场景 | `QlibModelScenario` | `scenarios/qlib/experiment/model_experiment.py` |
| 假设生成 | `QlibModelHypothesisGen` | `scenarios/qlib/proposal/model_proposal.py` |
| 假设→实验 | `QlibModelHypothesis2Experiment` | `scenarios/qlib/proposal/model_proposal.py` |
| 编码器 | `QlibModelCoSTEER`→`ModelCoSTEER` | `scenarios/qlib/developer/model_coder.py` |
| 运行器 | `QlibModelRunner` | `scenarios/qlib/developer/model_runner.py` |
| 反馈器 | `QlibModelExperiment2Feedback` | `scenarios/qlib/developer/feedback.py` |

**关键配置** (前缀 `QLIB_MODEL_`): 同 Factor，增加模型训练时间窗口。

---

### 5.3 Qlib Quant — 因子+模型联合优化

**代码路径**: `rdagent/scenarios/qlib/` + `rdagent/app/qlib_rd_loop/quant.py`

**意义**: 同时优化因子和模型，使用 Bandit / LLM / 随机策略决定每轮执行因子挖掘还是模型优化。

**特有配置** (前缀 `QLIB_QUANT_`):
```ini
QLIB_QUANT_ACTION_SELECTION=bandit  # 动作选择: bandit/llm/random
```

**Bandit 策略文件**: `rdagent/scenarios/qlib/proposal/bandit.py`
- 基于 UCB（Upper Confidence Bound）在因子/模型两个动作空间中选择

---

### 5.4 Factor from Report — 从研报提取因子

**代码路径**: `rdagent/app/qlib_rd_loop/factor_from_report.py`

**意义**: 解析研究报告 PDF，自动提取因子描述，生成因子实现代码并回测。

**特有配置** (子类 `FactorFromReportPropSetting`):
```ini
report_result_json_file_path=git_ignore_folder/report_list.json
max_factors_per_exp=6    # 每次实验最多实现 N 个因子
report_limit=20          # 最多处理 N 份报告
```

**所需数据**:
- PDF 研究报告文件
- 报告路径 JSON 索引文件（`report_list.json`）

---

### 5.5 DataScience — Kaggle/通用数据科学

**代码路径**: `rdagent/scenarios/data_science/` + `rdagent/app/data_science/`

**意义**: 针对 Kaggle 竞赛和通用数据科学任务的全流程自动化（数据加载→特征工程→建模→集成）。

**循环文件**: `rdagent/scenarios/data_science/loop.py`（`DSLoop`）

**步骤**:
```
propose(exp_gen) → code → running → feedback → record
```

其中 `code` 阶段包括多个 CoSTEER 子阶段：DataLoader / Feature / Model / Ensemble / Workflow

---

### 5.6 General Model — 通用模型实现

**代码路径**: `rdagent/app/general_model/`

**意义**: 从任意描述（如论文）提取并实现机器学习模型，不绑定特定回测框架。

---

### 5.7 Finetune — 模型微调

**代码路径**: `rdagent/app/finetune/` + `rdagent/scenarios/finetune/`

**意义**: 针对 LLM 微调场景的 R&D 自动化（超参数搜索、数据处理优化等）。

---

## 6. LLM 接入层（OAI）

> **代码路径**: `rdagent/oai/`

**意义**: 统一的 LLM 调用封装，支持多种后端（LiteLLM、OpenAI、Azure、本地模型），对上层完全透明。

### 核心类结构

```python
# llm_conf.py
class LLMSettings(ExtendedBaseSettings):
    backend: str = "rdagent.oai.backend.LiteLLMAPIBackend"
    chat_model: str = "gpt-4-turbo"
    embedding_model: str = "text-embedding-3-small"
    # ... 完整 LLM 配置

# llm_utils.py
def APIBackend(*args, **kwargs) -> BaseAPIBackend:
    """工厂函数，根据配置动态返回后端实例"""

# 调用方式（全框架统一）
from rdagent.oai.llm_utils import APIBackend
response = APIBackend().build_messages_and_create_chat_completion(
    user_prompt="...",
    system_prompt="...",
)
```

### 后端实现

| 文件 | 类 | 说明 |
|------|------|------|
| `backend/litellm.py` | `LiteLLMAPIBackend` | **默认后端**，支持 100+ 模型 |
| `backend/pydantic_ai.py` | `PydanticAIBackend` | 用于 Agent 工具调用 |
| `backend/deprec.py` | `DeprecBackend` | 旧版 OpenAI 直接调用 |

### 关键配置（`.env` 或环境变量）

```ini
# LiteLLM 模式（推荐）
CHAT_MODEL=gpt-4-turbo              # 或 deepseek/deepseek-chat 等
EMBEDDING_MODEL=text-embedding-3-small
OPENAI_API_KEY=sk-...
OPENAI_API_BASE=https://api.openai.com/v1

# Azure 模式
CHAT_USE_AZURE=true
CHAT_AZURE_API_BASE=https://xxx.openai.azure.com/
CHAT_AZURE_API_VERSION=2024-02-01

# 缓存（调试用）
DUMP_CHAT_CACHE=true
USE_CHAT_CACHE=true
PROMPT_CACHE_PATH=prompt_cache.db
```

### Prompt 模板系统

**文件**: `rdagent/utils/agent/tpl.py`、`tpl.yaml`

```python
from rdagent.utils.agent.tpl import T

# 加载 prompts.yaml 中的模板并渲染
prompt = T("scenarios.qlib.prompts:factor_background").r(
    runtime_environment=...,
    source_data=...,
)
```

- 模板文件为 `prompts.yaml`，散落在各场景和组件目录下
- 使用 Jinja2 语法，支持变量注入

---

## 7. 运行环境层（Env）

> **代码路径**: `rdagent/utils/env.py`

**意义**: 提供统一的代码执行环境抽象，隔离 Agent 生成的代码与主进程，防止环境污染和安全风险。

### 环境类型

```python
class Env(ABC):
    def run(self, entry: str, workdir: str, env: dict) -> EnvResult: ...

class LocalEnv(Env):
    """本地子进程执行"""

class DockerEnv(Env):
    """Docker 容器执行（隔离环境，推荐生产使用）"""
    conf: DockerEnvConfig  # 镜像名、资源限制等

class QlibDockerEnv(DockerEnv):
    """Qlib 专用 Docker 环境（预装 Qlib + 数据）"""

@dataclass
class EnvResult:
    stdout: str       # 标准输出
    exit_code: int    # 退出码
    running_time: float
```

### Docker 配置

```python
class DockerEnvConfig:
    image: str = "python:3.11"  # Docker 镜像
    mount_path: str             # 工作区挂载路径
    extra_volumes: dict         # 额外挂载（数据目录）
    mem_limit: str              # 内存限制
    cpu_count: int              # CPU 数量
```

**Qlib 数据挂载**: `QlibDockerEnv` 自动挂载 Qlib 数据目录到容器内。

---

## 8. 日志与监控（Log）

> **代码路径**: `rdagent/log/`

**意义**: 结构化日志系统，记录每步执行的完整上下文（假设、代码、结果），支持 Streamlit Web 可视化回放。

### 核心组件

```python
# __init__.py / base.py
rdagent_logger  # 全局 logger 单例

logger.info("message")
logger.log_object(obj, tag="hypothesis")  # 序列化对象到日志
```

**日志目录结构**:
```
LOG_PATH/
└── __session__/
    └── {loop_id}/
        ├── 0_propose/
        ├── 1_coding/
        ├── 2_running/
        ├── 3_feedback/
        └── 4_record/
```

### Streamlit UI

```python
# rdagent/log/ui/app.py         → Qlib 因子/模型日志可视化
# rdagent/log/ui/dsapp.py       → DataScience 日志可视化
# rdagent/log/ui/llm_st.py      → LLM 调用历史查看
# rdagent/log/ui/qlib_report_figure.py → 回测指标图表
```

**启动**:
```bash
rdagent ui --port 19899 --log_dir ./logs
```

---

## 9. 配置系统（Config）

**意义**: 基于 `pydantic-settings` 的层级配置系统，支持环境变量、`.env` 文件、代码默认值三层覆盖。

### 配置层级

```
ExtendedBaseSettings (rdagent/core/conf.py)
    ├── RDAgentSettings          # 全局配置（环境变量前缀为空）
    ├── LLMSettings              # LLM 配置（前缀为空）
    └── BasePropSetting          # RDLoop 配置
        ├── FactorBasePropSetting   # 前缀 QLIB_FACTOR_
        ├── ModelBasePropSetting    # 前缀 QLIB_MODEL_
        └── QuantBasePropSetting    # 前缀 QLIB_QUANT_
```

### 关键全局配置 (`RDAgentSettings`)

```ini
# 工作区路径
WORKSPACE_PATH=./git_ignore_folder/RD-Agent_workspace

# 并行控制
MULTI_PROC_N=1
STEP_SEMAPHORE=1              # 或 {"coding": 3, "running": 2}

# 缓存
CACHE_WITH_PICKLE=true
PICKLE_CACHE_FOLDER_PATH_STR=./pickle_cache/

# MLflow 集成
ENABLE_MLFLOW=false

# 标准输出截断
STDOUT_CONTEXT_LEN=400
STDOUT_LINE_LEN=10000
```

### `.env` 示例文件

**代码路径**: `.env.example`

---

## 10. Web UI 层

**代码路径**: `rdagent/log/ui/` + `web/`（前端 Vue/React）

**意义**: 提供可视化界面查看 Agent 运行状态、实验历史、LLM 对话记录。

### 后端（Streamlit）

| 文件 | 用途 |
|------|------|
| `app.py` | Qlib 因子/模型场景日志 UI |
| `dsapp.py` | DataScience 场景日志 UI |
| `llm_st.py` | LLM 调用历史 |
| `flow.png` | 流程图资源 |
| `ds_trace.py` | DataScience Trace 可视化 |

### 前端（Vue）

**代码路径**: `web/src/`

- 提供与 Agent 的实时交互界面（假设确认、参数调整）
- 通过队列机制（`user_request_q` / `user_response_q`）与 RDLoop 通信

**日志服务器**: `rdagent/log/server/app.py`（FastAPI）

---

## 11. CLI 入口

**文件**: `rdagent/app/cli.py`

**意义**: 统一命令行入口，自动加载 `.env`。

```bash
# 量化场景
rdagent fin_factor            # 因子挖掘
rdagent fin_model             # 模型构建
rdagent fin_quant             # 因子+模型联合
rdagent fin_factor_report     # 从研报提取因子

# 数据科学场景
rdagent data_science

# 其他
rdagent ui                    # 启动 Web UI
rdagent health_check          # 环境健康检查
rdagent grade_summary         # 评分汇总
```

---

## 12. 数据接口与依赖汇总

### Qlib 量化场景数据依赖

| 数据 | 格式 | 用途 | 来源 |
|------|------|------|------|
| 股票日线数据 | Qlib 二进制格式（`.bin`） | 因子计算、回测 | Qlib 数据下载工具 |
| 基准指数数据 | Qlib 格式 | 超额收益计算 | 同上 |
| 成交量/价格字段 | `$open/$high/$low/$close/$volume/$vwap` | 因子表达式 | 包含在日线数据 |
| 研报 PDF | PDF 文件 | `factor_from_report` 场景 | 用户提供 |
| Base Factor JSON | JSON（名称→表达式映射） | 预置基础因子 | 用户可选提供 |

### LLM 服务依赖

| 服务 | 用途 | 必选 |
|------|------|------|
| Chat LLM（GPT-4/DeepSeek等） | 假设生成、代码生成、反馈分析 | ✅ |
| Embedding 模型（text-embedding-3-small等） | RAG 知识检索、因子去重 | ✅ |
| Azure Document Intelligence | PDF 解析（research_from_report 场景） | 可选 |

### 运行时依赖

| 组件 | 用途 | 必选 |
|------|------|------|
| Docker | 代码隔离执行（`DockerEnv`） | 推荐（也可 `LocalEnv`） |
| Qlib | 因子回测框架 | Qlib 场景必选 |
| MLflow | 实验追踪（可选） | 可选 |
| Redis | 并行执行时的状态共享 | 并行时可选 |

---

## 13. 迁移参考：核心/可选/可删减模块

### ✅ 必须保留（核心骨架）

| 模块 | 路径 | 原因 |
|------|------|------|
| Scenario | `rdagent/core/scenario.py` | 场景上下文，所有组件依赖 |
| Task & Experiment | `rdagent/core/experiment.py` | 任务/实验数据结构 |
| Hypothesis & Trace | `rdagent/core/proposal.py` | R&D 记忆核心 |
| Developer | `rdagent/core/developer.py` | 代码开发抽象 |
| Feedback | `rdagent/core/evaluation.py` | 反馈数据结构 |
| LoopBase / RDLoop | `rdagent/utils/workflow/loop.py`, `rdagent/components/workflow/rd_loop.py` | 循环调度引擎 |
| OAI LLM 层 | `rdagent/oai/` | LLM 调用封装 |
| Env | `rdagent/utils/env.py` | 代码执行环境 |
| 配置系统 | `rdagent/core/conf.py` | 配置注入 |

### 🔧 按需选择（场景特定）

| 模块 | 路径 | 适用场景 |
|------|------|---------|
| QlibFactorScenario + 完整类链 | `rdagent/scenarios/qlib/` | 量化因子挖掘 |
| QlibModelScenario + 完整类链 | `rdagent/scenarios/qlib/` | 量化模型构建 |
| FactorCoSTEER | `rdagent/components/coder/factor_coder/` | CoSTEER 代码生成 |
| CoSTEER 知识库 | `rdagent/components/coder/CoSTEER/knowledge_management.py` | 经验积累/RAG |
| FactorFromReport | `rdagent/scenarios/qlib/factor_experiment_loader/` | 研报提取因子 |
| VectorBase / RAG | `rdagent/components/knowledge_management/` | 知识检索增强 |
| DocumentReader | `rdagent/components/document_reader/` | PDF 解析 |

### ❌ 可删减（外围功能）

| 模块 | 路径 | 说明 |
|------|------|------|
| DataScience Coder 族 | `rdagent/components/coder/data_science/` | 仅 Kaggle 场景使用 |
| DataScience Loop | `rdagent/scenarios/data_science/` | 仅 Kaggle 场景 |
| Finetune | `rdagent/scenarios/finetune/` + `rdagent/app/finetune/` | LLM 微调场景 |
| General Model | `rdagent/app/general_model/` | 通用模型提取 |
| Web UI（Vue 前端） | `web/` | 可用 Streamlit 替代 |
| Log Server | `rdagent/log/server/` | 需要 FastAPI 服务端时使用 |
| Benchmark | `rdagent/components/benchmark/` | 基准测试框架 |
| RL 场景 | `rdagent/scenarios/rl/` | 强化学习场景 |

---

### 最小化量化研究工作流建议

基于以上分析，迁移到量化研究工作流时，最小核心依赖链如下：

```
1. 定义 Scenario（描述数据源、背景、运行环境）
     └─ 继承 rdagent/core/scenario.py::Scenario

2. 定义 Task & Experiment（因子/模型任务结构）
     └─ 继承 rdagent/core/experiment.py::Task, Experiment

3. 实现 HypothesisGen（LLM 生成假设）
     └─ 继承 rdagent/core/proposal.py::HypothesisGen
     └─ 参考 rdagent/scenarios/qlib/proposal/factor_proposal.py::QlibFactorHypothesisGen

4. 实现 Hypothesis2Experiment（假设 → 具体任务列表）
     └─ 继承 rdagent/core/proposal.py::Hypothesis2Experiment

5. 实现 Developer (Coder)（代码生成，可直接使用 FactorCoSTEER）
     └─ 继承或复用 rdagent/components/coder/factor_coder/

6. 实现 Developer (Runner)（执行代码，接入自有回测框架）
     └─ 继承 rdagent/core/developer.py::Developer
     └─ 参考 rdagent/scenarios/qlib/developer/factor_runner.py::QlibFactorRunner

7. 实现 Experiment2Feedback（对比回测结果，生成反馈）
     └─ 继承 rdagent/core/proposal.py::Experiment2Feedback
     └─ 参考 rdagent/scenarios/qlib/developer/feedback.py::QlibFactorExperiment2Feedback

8. 组装 BasePropSetting（注入以上 7 个类路径）
     └─ 继承 rdagent/components/workflow/conf.py::BasePropSetting

9. 运行 RDLoop
     └─ rdagent/components/workflow/rd_loop.py::RDLoop
```

**关键替换点**（与 Qlib 解耦）:
- **步骤 6（Runner）**：将 Qlib 回测替换为自有回测引擎，只需让 `develop()` 把回测结果存入 `exp.result`
- **步骤 7（Feedback）**：将 Qlib 指标（IC/IC_IR等）替换为自有评估指标
- **步骤 1（Scenario）**：在 `get_source_data_desc()` 中描述自有数据字段，影响 LLM 生成因子的质量
