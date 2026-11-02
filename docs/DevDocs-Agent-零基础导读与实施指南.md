# DevDocs Agent：零基础导读与实施指南

> 需要直接照着修改代码时，请阅读[《逐文件开发实施手册》](DevDocs-Agent-逐文件开发实施手册.md)。后续采用一次对话实现一个功能的方式，当前任务与完成情况以[开发进度](DevDocs-Agent-开发进度.md)为准。本文件保留作为概念导读。

核对日期：2026年11月2日。适用项目：`D:\Agent\ma_Agent`。

这份说明基于当前项目的实际源码、依赖声明、锁文件及三份规划文档。面向已经会 Python 基础语法、初步了解 Agent 和 RAG 的学习者。

本文中的“已有”指已经存在的代码，不代表已运行成功；“新增”“目标”“示意”指接下来需要实现的内容。本次只新增这份指南，没有安装依赖、调用模型、运行项目测试或改造业务代码。

## 1. 我们最终要做什么

你要做一个名为 **DevDocs Agent 的 FastAPI 技术文档助手**。

用户用中文提出 FastAPI 开发问题，程序从提前准备好的官方英文文档和官方示例代码中寻找资料，给出中文解释，并展示实际使用的来源。用户继续追问时，它能理解“这个依赖”“刚才的代码”指的是什么。

例如，用户先问：

> FastAPI 里面 Depends 是怎么用的？

程序检索依赖注入教程，给出解释和来源。接着用户问：

> 如果这个依赖用 yield，应该怎么写？

程序结合上一轮，把问题补成“FastAPI 中使用 yield 的依赖如何定义”，寻找相应文档。用户再问：

> 测试时怎么替换它？把官方示例也找出来。

程序可以先搜索 `dependency_overrides`，再读取相关原文和示例文件，最后回答。

遇到知识库没有覆盖的内容，它应明确说明资料不足，或者请用户补充条件。工具一直失败时，它应在规定次数和时间内结束。

最终演示要看得见四件事：**回答是什么、依据是什么、实际执行了哪些工具、花了多少时间和调用量。**

项目先限定为一份 FastAPI 资料快照、约10篇文档、三个只读工具、一个 Agent、一个本地服务和一个简单界面。模型通过 API 使用，起步不需要 GPU，也不需要训练模型。

FastAPI 在这里有两个角色：一是知识库的资料主题，二是后期给问答程序提供 HTTP 接口的框架。你在“用 FastAPI 做一个回答 FastAPI 问题的服务”。

## 2. 现在做到了哪里

### 2.1 本次实际检查结果

| 检查项 | 当前结果 | 对你的意义 |
| --- | --- | --- |
| 项目根目录 | `D:\Agent\ma_Agent` | 后文命令从这里运行 |
| Git 分支与短提交号 | `main`，`228ace2` | 与旧交接文件里的历史快照不同，以当前记录为准 |
| 阅读前的 Git 状态 | 没有报告工作区修改 | 本指南是本次新增文件 |
| 业务源码目录 | `src/retrieval_graph/` | README 中一些 `retrieval_agent` 路径是旧名称 |
| 默认 `python` | `C:\ProgramData\miniconda3\python.exe`，版本3.8.16 | 不满足 `pyproject.toml` 中 Python >=3.10 的要求 |
| uv | 已可用，版本0.11.2 | 可以用它建立独立项目环境 |
| Git | 当前终端可用 | 可以记录分步改动 |
| `uv.lock` | 存在 | 已有依赖锁文件，无需从零选择全部版本 |
| `.venv`、`.env`、根目录 `.gitignore` | 本次目录检查未发现 | 需要在环境准备阶段创建 |
| 默认 Python 中的相关包 | 未检测到 langgraph、langchain、pytest、langchain-chroma | 这里只说明默认 Python 的情况，不代表整台机器没有其他环境 |
| 安装和运行 | 本次没有安装依赖或运行问答/测试 | 暂时不能说项目已经跑通 |

一次列举 uv 管理的 Python 的辅助检查遇到了默认缓存目录访问被拒绝，因此本文不判断机器上是否另有 Python 3.12。后续由环境安装命令实际确认。

### 2.2 现有代码与目标的差距

当前已经有一个基础检索问答流程：

```text
用户消息 → generate_query → retrieve → respond → 返回回答
```

还有一个简单的文档写入流程：

```text
传入文档 → 附加 user_id → 写入已有检索后端
```

当前代码包含 Elasticsearch、Pinecone、MongoDB 适配，默认选择 Elasticsearch。它还没有本计划需要的 Chroma、本地 FastAPI 文档导入、三个工具、动态工具循环、引用校验、SQLite 会话、业务 API、Streamlit 界面或评测脚本。

根目录 `test.py` 只有两句打印，不是项目的启动入口。真正的测试在 `tests/`。

### 2.3 三份计划怎么读

1. `DevDocs-Agent-开发进度.md` 与 `DevDocs-Agent-逐文件开发实施手册.md`：分别记录当前任务与详细技术实现，是后续逐功能对话的主要依据。
2. `DevDocs-Agent-20天执行计划.md`：作为阶段安排与范围的参考，不要求一天对应一次对话。
3. `DevDocs-Agent-开发交接说明.md`：理解已经选定的约定；其中旧机器路径、提交号只是历史记录。
4. `DevDocs-Agent-项目计划书.md`：作为技术背景资料。

旧计划提出过 Runtime context 迁移、LLM 摘要和额外来源接口。较新的20天计划已经收缩范围：继续用 `RunnableConfig.configurable`；摘要暂缓；来源内容直接随 `/chat` 响应返回。入门阶段按这套收缩后的方案做。

20天按每天3～4小时估算，并非保证完成时间。可以把一天的任务拆成几天，按验收结果前进。

## 3. 先把几个核心概念分清楚

| 概念 | 在本项目里做什么 | 容易混淆的地方 |
| --- | --- | --- |
| 聊天模型 / LLM | 理解问题、改写查询、选择工具、生成答案 | 本身不会自动读取你的本地文件 |
| Embedding 模型 | 把问题和文档变成可比较的数字向量 | 它负责表示文本，不负责写回答 |
| 文档块 / chunk | 文档切分后的可检索片段 | 一块不是一个训练样本 |
| 向量库 / Chroma | 保存向量、正文和元数据，并执行相似检索 | 不是保存聊天历史的主要位置 |
| Retriever | 对外提供“用问题找文档”的接口 | 不应假定它也能写入文档 |
| RAG | 找资料，再把资料交给模型回答 | 不是把资料训练进模型参数 |
| Tool / 工具 | 供模型选择的有类型 Python 函数 | 模型提出名称和参数，由程序真正执行 |
| Agent | 根据已有结果决定继续查找还是回答 | 仍需要代码约束权限、次数和停止条件 |
| LangChain | 提供模型、消息、提示词和存储适配接口 | 是组件工具箱 |
| LangGraph | 安排节点执行顺序、分支、循环和状态更新 | 是流程编排框架 |
| State / 状态 | 存一次会话或当前一轮的中间结果 | 声明字段不会自动实现磁盘持久化 |
| Checkpointer / 检查点 | 保存图状态，供同一会话后续恢复 | 本项目后期使用 SQLite 保存 |
| Prompt / 提示词 | 告诉模型如何改写、查资料、回答 | 不能代替程序校验和次数限制 |
| Token | 模型处理文本的计量单位 | 不等同于字符数或汉字数 |

例如你问“测试时怎么替换依赖”：Embedding 帮你找语义接近的资料；BM25 帮你匹配 `dependency_overrides` 这类技术词；LLM 根据查到的资料解释做法；LangGraph 管理执行次序；Python 代码限制调用次数并核对引用。

## 4. 当前一次提问具体怎样运行

### 4.1 从 `graph.py` 理解三个节点

节点可以先理解为“接收状态、完成一步任务、返回要更新的字段的函数”。

| 节点 | 主要读取什么 | 做什么 | 返回什么 |
| --- | --- | --- | --- |
| `generate_query` | `messages`，追问时还用 `queries` 和配置 | 首轮直接取问题；有历史消息时调用模型生成独立查询 | `{"queries": [查询文本]}` |
| `retrieve` | 最新一条 `queries`，检索配置 | 创建检索器并检索 | `{"retrieved_docs": 文档列表}` |
| `respond` | `messages`、`retrieved_docs`、回答配置 | 将资料和消息放入 Prompt，调用聊天模型 | `{"messages": [模型回答]}` |

例如初始问题是“FastAPI 依赖怎么声明”：

1. 输入消息进入 `messages`。
2. `generate_query` 发现只有一条消息，直接返回原问题作为查询。
3. `retrieve` 把查询交给检索器，查询向量与索引中的文档向量进行比较。
4. 命中资料放入 `retrieved_docs`。
5. `respond` 通过 `utils.format_docs` 把资料变成字符串。
6. `prompts.py` 的回答模板通过 `{retrieved_docs}` 接收这段资料。
7. 模型输出一条回答消息，框架将它合并进状态。

正常情况下，首轮有一次回答生成调用；追问且传入/恢复了历史时，通常多一次查询改写调用。查询的 Embedding 调用另外计算。它们是代码路径上的预期次数，不是本次已经测得的调用记录。

当前流程的顺序固定，没有“模型认为资料不够，于是主动再调用一次读取工具”的分支。这就是后面增加 `agent_graph.py` 的原因。

### 4.2 `index_graph.py` 做了什么

它接受传入的文本或文档，为文档元数据加 `user_id`，然后尝试写入检索后端。写入结束返回 `{"docs": "delete"}`，通过 reducer 清空图状态中的待写文档。

这个 `delete` 清的是**状态里的待写列表**，不表示删除数据库里刚写入的资料。

当前实现通过 retriever 调用 `aadd_documents`。后续应把存储写入和检索职责分开，用存储对象明确的写入接口实现本地适配，不能假定所有 retriever 都有写入方法。

## 5. 详细读懂你正在看的 `state.py`

### 5.1 State 是节点共享的一份工作记录

可以先把它想成一个有固定字段的字典：

```python
# 概念示意，实际项目使用 dataclass 定义结构。
state = {
    "messages": [用户问题, 模型回答],
    "queries": [检索问题1, 检索问题2],
    "retrieved_docs": [命中文档1, 命中文档2],
}
```

节点通常只返回变化的字段。框架负责把返回值并入已有状态，所以 `respond` 返回 `{"messages": [response]}` 时，不需要把全部历史重新返回。

每个字段如何合并，由它的 reducer 决定；没有自定义 reducer 的字段通常用新值替换旧值。

### 5.2 `@dataclass(kw_only=True)`

`dataclass` 帮你生成数据类的初始化方法；`kw_only=True` 表示构造时用字段名传参，例如：

```python
state = State(messages=[])
```

它主要描述数据结构。不要把它当作已经完成所有运行时参数校验；工具和 API 输入还要使用明确的校验模型。

### 5.3 `Annotated` 与 reducer

当前代码：

```python
messages: Annotated[Sequence[AnyMessage], add_messages]
```

分开看：

- `messages`：字段名。
- `Sequence[AnyMessage]`：类型说明，表示消息序列。
- `Annotated[..., add_messages]`：附加框架能读取的元信息，指定用 `add_messages` 合并更新。

`Annotated` 自己不会执行合并。是 LangGraph 读取它，按规则更新状态。

`add_messages` 大体是追加新消息；如果新旧消息 ID 相同，会更新原消息，因而不完全等同于列表相加。

例如原来有“用户提问A”，节点新增“模型回答A”，合并后两条都在。如果下个请求恢复了同一份状态，再加入“用户追问B”，才会形成连续历史。

### 5.4 `queries` 为什么使用 `add_queries`

当前代码：

```python
def add_queries(existing, new):
    return list(existing) + list(new)
```

所以旧值为 `["FastAPI Depends"]`，节点返回 `["FastAPI dependency yield"]` 后，结果是：

```python
["FastAPI Depends", "FastAPI dependency yield"]
```

与消息 reducer 不同，这个函数只是列表拼接，不按 ID 更新，也不主动去重。对话不断增长时，查询历史也可能不断增长。

### 5.5 `retrieved_docs` 为什么会被替换

当前字段是：

```python
retrieved_docs: list[Document] = field(default_factory=list)
```

它没有声明追加 reducer。因此新一轮检索返回 `[文档B, 文档C]` 时，通常替换旧的检索列表，而不是自动与上轮结果合并。

`field(default_factory=list)` 表示每个新状态实例都创建自己的空列表，避免共享一个可变列表。

这里的 `Document` 可以理解为“正文 + metadata”：

```python
Document(
    page_content="文档正文",
    metadata={"source_path": "某个文档路径", "user_id": "local-demo"},
)
```

### 5.6 `InputState` 与 `State` 的区别

`InputState` 只公开消息输入；`State(InputState)` 在它的基础上增加内部查询和检索结果。

用户负责提出问题。查询、检索结果、调用计数等内部字段应由程序产生。这种区分让外部接口更清楚。

### 5.7 `IndexState` 与 `reduce_docs`

`IndexState` 为索引图使用，与问答状态分开。

`reduce_docs` 把几种输入转成 `Document` 列表：字符串变为一篇文档；列表中的字符串、字典等逐项转换；`"delete"` 变为空列表。

注意当前真实实现里的两个细节：

1. 文本生成的 `metadata["id"]` 是随机 UUID，不满足后续“同一资料重复处理得到相同 ID”的目标；而 metadata 里的 ID 也不自动等于向量库使用的记录 ID。
2. 注解允许 `Sequence`，函数实际转换分支主要判断 `list`。不要仅凭类型说明就认定元组等输入都被完整支持。

文件开头的说明也遗留了一些当前并不存在的类和函数名。读代码时以实际定义为准。

### 5.8 后续增加独立 `AgentState`

建议保留基础 RAG 的 `State`，另增 `AgentState`，避免影响对照组。

| 字段 | 保存什么 | 生命周期 |
| --- | --- | --- |
| `messages` | 用户、模型和工具消息 | 会话级 |
| `semantic_query` / `keyword_query` | 当前检索问题与关键词 | 当前轮 |
| `evidence_by_id` | 本轮实际读取的证据、正文与来源 | 当前轮 |
| `tool_calls_used` | 已使用的逻辑工具调用数 | 当前轮 |
| `tool_attempts_used` | 包括重试的实际执行尝试数 | 当前轮 |
| `llm_calls_used` | 包括决策、回答和修正的模型调用数 | 当前轮 |
| `errors` | 结构化错误记录 | 当前轮 |
| `answer` / `citations` / `answer_status` | 最终结果 | 当前轮 |
| `answer_revision_count` | 已修正引用的次数 | 当前轮 |

新一轮开始时，`prepare_turn` 节点必须重置当前轮字段；历史消息继续按会话保留。不能让上一轮的证据和预算直接混到下一轮。

20天版暂不要求 `conversation_summary`。先做好近期完整轮次的输入窗口。

## 6. 功能、技术、文件与验收逐项对应

本节是开发路线。除明确写“已有”的文件外，其余在对应阶段才创建。

### 功能一：建立环境并试通模型

**你要完成：** 能在同一个项目环境里运行 Python；聊天模型能回复；Embedding 返回有效向量；模型能提出符合格式的工具调用。

**技术：** Python 3.12、uv、模型服务 API、python-dotenv、LangChain 模型适配。

**修改：** `configuration.py`、`utils.py` 中的 `load_chat_model`、`retrieval.py` 中的 `make_text_encoder`、`.env.example`。

**新增：** `.gitignore`、`scripts/smoke_models.py`。

**验收：** 短问题有非空回答；两段文本的向量维度相同且数值有效；一个无副作用的测试工具可以收到可解析名称和参数。普通聊天成功后，仍要单独验证工具调用和结构化输出支持。

### 功能二：本地写入与检索

**你要完成：** 将两条自己写的短资料保存到本地，然后根据问题找回它们。重启后还存在，同样资料重复写入不增加条目。

**技术：** `langchain-chroma`、本地持久化目录、稳定记录 ID、元数据过滤。

**修改：** `configuration.py` 增加 `chroma-local`、目录和集合名；`retrieval.py` 增加存储构造、写入与检索接口；`index_graph.py` 改为明确的存储写入。

**新增：** `scripts/smoke_store.py`、本地存储行为测试。

**接口设计示意：** `make_chroma_store`、`upsert_chunks`、`search_dense`。这些名字目前尚未实现。

**验收：** 先只打印检索命中的原文，不急于接回答生成。写入和检索统一使用 `user_id="local-demo"`，防止资料已经存在却被过滤掉。

Chroma 的 LangChain 集成支持通过 `persist_directory` 使用本地持久化，无需为这一模式单独部署云数据库。见[官方 Chroma 集成说明](https://docs.langchain.com/oss/python/integrations/vectorstores/chroma)。

### 功能三：自动准备官方资料与示例

**你要完成：** 脚本读取固定快照的约10篇 Markdown 和它们引用的 Python 示例，生成可检查的本地资料。

**技术：** 文件读取或 HTTP 下载、Markdown 解析、受控代码引用解析、来源注册表、JSONL。

**新增：** `configs/corpus.json`、`src/retrieval_graph/corpus.py`、`scripts/prepare_corpus.py`。

先实现 `load_markdown` 和 `resolve_example_refs`。计划中的快照为 `79cab5bed28b24d92c185d236feb33f5783dfeb3`；这是原计划选定的来源标识，本次没有重新验证远端文件可用性。实现获取脚本时，逐个验证允许列表和引用文件存在。

FastAPI 文档的示例可能只写一个外部代码引用，真实代码在 `docs_src/`。只读 Markdown 而不解析引用，可能得到缺少示例的知识库。

**验收：** 每篇文档的获取状态可见；抽查依赖教程能找到对应示例；原文与代码保留来源关系；未知格式报告出来；解析后的路径不能越出允许资料目录。代码只读取，不执行。

### 功能四：切块、来源标记与建立索引

**你要完成：** 把长文档拆成合适的小块，为它们分配稳定身份，再进行 Embedding 和持久化写入。

**技术：** 按标题切分、长段落二次切分、内容哈希、稳定 ID、批量向量化、索引清单。

**修改：** `corpus.py`、`index_graph.py`、`retrieval.py`。

**新增：** `scripts/build_index.py`、`tests/unit_tests/test_corpus.py`。

初始切分可尝试约700 tokens、重叠约100 tokens；这是起点，后续根据命中结果调整。不能用700个字符直接冒充700 tokens。正文需要保留标题关系，代码尽量不在中间拆断。

每块至少记录：`user_id`、`corpus_id`、`source_commit`、`doc_id`、`chunk_id`、`source_path`、`source_url`、`heading_path`。示例关联可以放在独立注册表中。

`doc_id` 表示原文身份，`chunk_id` 表示片段身份，后续 `evidence_id` 表示某次回答实际使用的证据身份，三者职责不同。

**验收：** 同一资料、切分配置运行两次得到相同 ID；记录数不增长；抽查5块正文和来源正确；索引清单记录模型、维度与切分版本。换 Embedding 模型时重建或切换索引，不能因维度相同就复用旧向量。

### 功能五：先交付基础 RAG

**你要完成：** 在命令行提问，看到回答以及 Top-5 命中片段。

**技术：** 沿用 `generate_query → retrieve → respond`，接入本地存储，将来源放入模型输入和输出展示。

**修改：** `graph.py`、`utils.py`，必要的配置适配。

**新增：** `scripts/chat.py`、`configs/baseline_dense.json`。

**验收：** 至少5个已导入范围内的问题能获得相关资料；打印每块的 ID、标题、路径和正文摘录。先检查检索片段，再评价回答。

保存基线的 Prompt、模型、语料、切分和检索参数。以后调整 Agent 时，不要同时改掉对照组的行为。

### 功能六：把能力封装成三个工具

**你要完成：** 提供可以单独调用、也可供模型选择的函数。

| 工具 | 参数 | 返回内容 | 典型用途 |
| --- | --- | --- | --- |
| `search_docs` | 查询、受限的 `top_k` | 候选资料、摘录、文档 ID、关联示例 ID | 寻找相关章节 |
| `read_document` | `doc_id` | 更完整的文档或章节内容及来源 | 搜索摘录不足以回答 |
| `read_example` | `example_id` | 官方代码文本及来源 | 用户需要官方示例 |

**技术：** Pydantic 参数模型、LangChain 工具接口、ID 注册表、统一错误结构。

**新增：** `schemas.py`、`tools.py`、`tests/unit_tests/test_tools.py`。

统一结果设计为 `ok`、`data`、`evidence_ids`、`error`、`elapsed_ms`；错误说明包含 `code`、`message`、`retryable`。

模型只提供公开 ID，不接受它指定任意本地路径。每次工具真实返回的正文或代码进入本轮证据表。

**验收：** 不接模型，手动完成“搜索 → 读取文档 → 读取示例”；空查询、未知 ID、过大 top_k 均有清楚反馈。工具自己可用之后，再测试模型是否选对。

### 功能七：新增有界 Agent 循环

**你要完成：** 模型根据现有资料选择下一步，程序执行工具并把结果交回模型，最后停止查找并回答。

**技术：** LangGraph 条件边、工具绑定、`AIMessage.tool_calls`、匹配的 `ToolMessage`、预算计数。

**新增：** `agent_graph.py`。

**修改：** `state.py`、`configuration.py`、`prompts.py`、`langgraph.json`、`__init__.py`、`scripts/chat.py`。

目标流程：

```text
prepare_turn
    ↓
decide ──需要资料──→ execute_tools
    ↑                    │
    └────工具结果─────────┘
    │
    └──可以回答/达到上限──→ compose_answer → validate_answer → 结束
```

模型返回工具调用的含义是“建议调用这个工具，参数如下”。真正的名称检查、参数检查和执行由 Python 完成。工具结果通过匹配的 `tool_call_id` 对应回模型请求。

起步每次只允许执行一个工具调用。若模型提出多个，按协议拒绝并提供匹配结果，不能让消息配对断裂。

初始预算可采用：4次逻辑工具调用、6次工具执行尝试、6次模型调用。代码须预留最终回答和必要修正额度；所有调用，包括失败或重试，都按各自规则计数。这些是设计上限，不是实测性能。

**验收：** 找示例的问题能展示真实“搜索 → 读示例 → 回答”；用一直要求工具的替身模型测试，流程也能停止。框架的递归上限仅作兜底，不代替业务预算。

### 功能八：回答结构与引用校验

**你要完成：** 返回回答正文、引用和状态，并阻止不存在的证据 ID 直接通过。

**技术：** Pydantic 结构化输出、确定性 ID 校验、最多一次预算内修正。

**新增：** `citations.py`、引用行为测试。

**修改：** `schemas.py`、`agent_graph.py`、`prompts.py`。

回答结构示意：

```json
{
  "answer": "根据本轮文档得到的解释",
  "citations": ["本轮证据ID"],
  "answer_status": "answered"
}
```

程序用本轮证据表构造来源链接，不直接采用模型生成的 URL。正常事实回答需要证据；资料不足、澄清和失败状态可以没有引用。

**验收：** 替身模型编造 ID 时无法直接返回成功；有效引用能定位到实际读取内容；人工检查回答的关键结论是否真的被引用支持。

引用 ID 有效，只说明来源被读取过，不能自动证明答案正确。不要把链接存在当作消除幻觉。

### 功能九：多轮会话与上下文窗口

**你要完成：** 用相同会话 ID 继续追问，进程重启后能恢复；换会话不会混入历史。

**技术：** `langgraph-checkpoint-sqlite`、`AsyncSqliteSaver`、`configurable.thread_id`、近期完整轮次选择。

**新增：** `memory.py`。

**修改：** `agent_graph.py` 的图构造、`state.py`、命令行入口，后期再接入 API 生命周期。

三种 ID 分清楚：`user_id` 是文档过滤身份，首版固定 `local-demo`；`thread_id` 是会话身份；`request_id` 是某一次请求身份。同一会话的多次请求共用 thread_id，各自有 request_id。

检查点连接在应用或命令行进程生命周期内持有。同一会话请求先串行处理。送给模型的近期窗口，与磁盘里的完整历史分开；裁剪要保留必要条件和工具调用/返回配对。

**验收：** 连续5轮主题与条件正确；新会话不带旧主题；关闭再启动后同一 ID 能追问；新一轮证据和计数重置。

LangGraph 的检查点保存 thread 级状态；异步执行应选匹配的异步检查点实现。见[持久化说明](https://docs.langchain.com/oss/python/langgraph/persistence)和[检查点参考](https://langchain-ai.github.io/langgraph/reference/checkpoints/)。

### 功能十：错误、超时与运行记录

**你要完成：** 网络失败、资料为空、未知工具等情况都可以正常结束，并留下定位依据。

**技术：** 异步超时、请求截止时间、有限重试、JSONL 事件、模型用量元数据。

**新增：** `tracing.py`、`tests/unit_tests/test_agent_limits.py`。

**修改：** `tools.py`、`agent_graph.py`、`configuration.py`。

未知 ID 或错误参数不原样重试；临时网络故障可在预算内重试一次。检查 SDK 的默认重试，避免与业务层重试叠加。

日志记录 request_id、thread_id、模式、语料快照、模型、工具事件、命中 ID、耗时、答案状态和 Token 用量。拿不到用量时写 `unknown`，不要写0。

**验收：** 持续失败和持续调用均会停止；每一步可以从日志定位；密钥不进入日志。初期可试工具10秒、模型30秒、整轮60秒，随后根据实际服务调整。

### 功能十一：BM25 与向量混合检索

**你要完成：** 同时利用语义匹配和精确技术词，改善中文问题查询英文资料时的表现。

**技术：** 跨语言 Embedding、`rank-bm25`、关键词生成、RRF 排名融合。

**修改：** `retrieval.py`、`prompts.py`、`scripts/build_index.py`，新增 hybrid 配置与检索测试。

向量和 BM25 各取 Top-10，按 chunk_id 合并，RRF 排名融合后取 Top-5。初始公式：`Σ 1 / (60 + rank)`，排名从1开始，未命中的一路贡献为0。

中文“怎么替换依赖”不一定命中英文关键词，改写时可额外产生 `FastAPI testing dependency override app.dependency_overrides` 这类关键词，同时保留用户给出的技术词与条件。

**验收：** 能打印两路排名和融合排名；重复块只出现一次；两路均限制到同一 user_id 和语料快照；保留纯向量基线。是否提高命中率，需要实际评测。

### 功能十二：本地 API、界面和评测交付

**你要完成：** 别人可以通过网页使用系统，你可以解释改进的效果与成本。

**技术：** FastAPI、Uvicorn、Streamlit、HTTP、pytest、JSONL 评测数据。

**新增：** `api.py`、`app/streamlit_app.py`、`eval/`、`scripts/evaluate.py`、`reports/`。

**修改：** `README.md`、依赖、配置样例、来源说明。

最小接口：`GET /health` 和 `POST /chat`。后者接收新消息、会话 ID 和模式，返回回答、来源摘录、状态和执行摘要。页面通过 HTTP 调用它。

前端有检查点会话时，每次发送当前新消息即可，不能把全部前端历史再次追加，否则容易重复消息。切换 baseline/agent 模式使用分离会话。

评测先做10条人工核对的问题，再扩到20～30条。比较固定流程 dense、固定流程 hybrid、Agent hybrid，记录来源命中、正确且有依据的回答数、失败行为、耗时和调用量。不要预填效果提升。

**验收：** 网页能问答、追问、新建会话、展示来源和真实工具事件；报告至少分析3个失败；新数据目录能按 README 重建并运行。

## 7. 文件职责速查

| 文件 | 现在/将来负责什么 | 什么时候去改 |
| --- | --- | --- |
| `state.py` | 数据字段与合并规则 | 增加 Agent 状态与轮次重置时 |
| `configuration.py` | 把 configurable 转成配置对象 | 增加模型、存储、预算设置时 |
| `retrieval.py` | Embedding、存储与搜索 | 接 Chroma、混合检索时 |
| `index_graph.py` | 将文档交给存储写入 | 接稳定 ID 与本地写入时 |
| `graph.py` | 基础 RAG 对照流程 | 只做必要的本地适配 |
| `prompts.py` | 查询与回答规范 | 增加 Agent/关键词 Prompt 时；保留基线 |
| `utils.py` | 模型加载、消息与文档格式化 | 服务适配和输出展示时 |
| `corpus.py`（新增） | 读取、代码引用、切分、稳定 ID | 第4～5天 |
| `schemas.py`（新增） | 工具与回答的类型和校验 | 第8天起 |
| `tools.py`（新增） | 三个工具的实际逻辑 | 第8～9天 |
| `agent_graph.py`（新增） | 决策、执行、循环、回答和停止 | 第10～12天 |
| `citations.py`（新增） | 引用属于本轮证据的检查 | 第12天 |
| `memory.py`（新增） | 模型输入历史窗口 | 第13天 |
| `tracing.py`（新增） | 真实事件与用量 | 第7天起逐步增加 |
| `api.py`（新增） | HTTP 服务和资源生命周期 | 第16天 |
| `app/streamlit_app.py`（新增） | 展示与发送请求 | 第17天 |
| `langgraph.json` | 图导出入口 | 新 Agent 可用后增加入口 |
| `pyproject.toml` / `uv.lock` | 依赖声明 / 解析后的版本 | 新增依赖时共同更新 |
| `.env.example` / `.env` | 配置模板 / 本地实际服务设置 | 模型烟测前 |
| `configs/`（新增） | 语料、检索与预算等非秘密设置 | 对应功能实现时 |
| `scripts/`（新增） | 可直接执行的准备、运行和评测入口 | 每阶段按需增加 |

不要先创建几十个空文件来表示完成。每增加一个模块，都配一条能验证其行为的执行方法。

## 8. Windows 环境应该怎样配置

以下是**你接下来执行的步骤**，不是本次已经执行成功的记录。第一次会下载 Python 或依赖，需要网络和相应写入权限。

### 8.1 使用已有项目

当前目录已经是项目，不需要再克隆一遍。

```powershell
Set-Location -LiteralPath 'D:\Agent\ma_Agent'
git status --short
git rev-parse --short HEAD
uv --version
python --version
```

从当前工作起点创建分支即可，不必为了匹配旧文档而重置历史：

```powershell
git switch -c codex/devdocs-agent
```

如果分支已经存在，改用 `git switch codex/devdocs-agent`。保留已有修改，不执行硬重置。

### 8.2 先准备忽略规则

当前未发现根目录 `.gitignore`。创建该文件，初期可包含：

```gitignore
.venv/
.env
.env.*
!.env.example
__pycache__/
*.py[cod]
.pytest_cache/
.mypy_cache/
.ruff_cache/
data/
```

`uv.lock` 应保留并提交。配置样例、评测题和报告也应保留。API Key 只在本地 `.env` 填写。

### 8.3 建立项目的 Python 3.12 环境

当前默认 Python 是3.8.16。让 uv 为项目选择3.12：

```powershell
uv python install 3.12
uv python pin 3.12
uv sync --locked --group dev --extra dev --python 3.12
```

这会准备 `.venv`；`uv python pin` 用 `.python-version` 记录项目选择。以后统一使用 `uv run` 执行，避免误用默认 Conda Python。见[uv 的 Python 版本管理](https://docs.astral.sh/uv/concepts/python-versions/)。

安装命令中的参数：

- `--locked`：要求依赖声明与锁文件一致，不自动改写锁文件。
- `--group dev`：安装项目 `dependency-groups.dev` 中的测试和开发依赖。
- `--extra dev`：安装 `project.optional-dependencies.dev` 中的工具。
- `--python 3.12`：显式选择解释器。

原计划使用 `--frozen`，它也读取现有锁文件，但跳过锁文件是否与声明一致的检查。这里选择 `--locked`，便于尽早发现不一致。若报锁文件不匹配，应检查声明与变更来源，再有意识地运行 `uv lock` 更新并查看差异，不要直接删除锁文件。见[uv 锁定与同步说明](https://docs.astral.sh/uv/concepts/projects/sync/)。

安装完成后核对：

```powershell
uv run --locked python -c "import sys; print(sys.version); print(sys.executable)"
uv run --locked python -m pytest tests/unit_tests
```

解释器应指向项目 `.venv\Scripts\python.exe`，版本为3.12系列。原单元测试主要是配置构造检查，通过它不代表数据库、模型或问答已可用。

原集成测试指定 `elastic-local` 并调用真实模型，暂不作为本地新环境的第一项测试。Windows 下先直接运行 `uv run ...`，不需要为了 Makefile 额外安装 make。

若安装失败，保存第一条实际错误，区分 Python 版本、网络、缓存权限、依赖冲突。看到失败后先解决环境，不继续叠加业务模块。

### 8.4 让 IDE 使用同一解释器

在 IDE 的 Python 解释器选择功能中选：

```text
D:\Agent\ma_Agent\.venv\Scripts\python.exe
```

如果 IDE 中能导入但终端不能，或者反过来，先比较 IDE 的解释器路径与上面 `sys.executable` 的输出。

不必激活虚拟环境，统一用 `uv run` 即可。这样也能减少 Windows 激活脚本相关问题。

### 8.5 准备模型服务配置

复制模板，已有 `.env` 时不要覆盖：

```powershell
if (-not (Test-Path -LiteralPath '.env')) {
    Copy-Item -LiteralPath '.env.example' -Destination '.env'
}
```

当前 `.env.example` 罗列了多家模型和数据库。你只需选择实际要用的一种聊天服务、一种 Embedding 服务；本地 Chroma 路线无需开通列出的全部外部数据库。

你需要确认六个信息：聊天服务提供方、聊天模型名、是否支持工具调用/结构化输出、Embedding 服务与模型名、密钥变量、必要的服务地址。

**当前源码真实读取行为：** `Configuration` 从 `configurable` 读取业务字段；`load_chat_model` 解析 `provider/model` 并调用框架构造器；`make_text_encoder` 目前只显式支持 `openai` 和 `cohere` 两种 Embedding 适配。选择其他服务时，需要按服务协议补适配，不能只改名字。

未来可以在 `.env` 设计 `CHAT_MODEL`、`CHAT_API_KEY`、`CHAT_BASE_URL`、`EMBEDDING_MODEL`、`EMBEDDING_API_KEY`、`EMBEDDING_BASE_URL`，但**这些通用变量目前没有对应的读取逻辑**。若采用它们，要在配置加载和模型构造中显式映射。不能填完就假定可用。

`.env` 文件也不是 Python 自动加载的。已有 `langgraph.json` 为开发服务指定了 `.env`；自己新增的脚本，应在构造客户端前加载项目根目录下的文件，例如：

```python
# 适用于放在项目 scripts/ 下的脚本，需在对应文件创建后使用。
from pathlib import Path
from dotenv import load_dotenv

project_root = Path(__file__).resolve().parents[1]
load_dotenv(project_root / ".env")
```

同理，计划中的 `chroma-local` 目前不在 `retriever_provider` 的允许选项和实现分支中，必须先增加代码。

在尚未选定实际模型服务前，不可能给出已验证的完整服务配置。模型 Key 不需要发到聊天里；在本地填好后，使用最小烟测判断是否可用。模板内历史默认模型名也需要换成你实际试通的模型。

### 8.6 依赖按阶段增加

这些是**做到对应功能时才执行的命令**，会修改 `pyproject.toml` 和 `uv.lock`，不是要求今天全部安装：

```powershell
# 本地向量存储
uv add langchain-chroma

# 若采用 LangChain 标题/递归切分器
uv add langchain-text-splitters

# 会话持久化
uv add langgraph-checkpoint-sqlite

# 混合检索
uv add rank-bm25

# HTTP 服务与界面
uv add fastapi uvicorn streamlit httpx
```

具体 Token 计数库按实际 Embedding/聊天模型选择，再补入依赖。工具 schema 使用的 Pydantic 也应在真正直接使用它时明确声明为项目依赖，不只依赖其他包偶然带入。

每次添加后查看解析结果和锁文件变更，运行受影响验证。当前模板带有多种暂时不用的云后端依赖，先保持基线，若安装冲突再针对真实错误处理。

## 9. 做完后的启动顺序

下面是**目标命令**。当前没有 `scripts/`、`api.py`、`app/streamlit_app.py`，所以现在直接运行会报文件或模块不存在。

```powershell
# 一次性或资料配置变更时：准备资料
uv run python scripts/prepare_corpus.py

# 初次或索引配置变更时：建立索引，会产生 Embedding 用量
uv run python scripts/build_index.py

# 先在命令行验收两种模式
uv run python scripts/chat.py --mode baseline
uv run python scripts/chat.py --mode agent

# 后期：在一个终端启动 API
uv run uvicorn retrieval_graph.api:app --host 127.0.0.1 --port 8000

# 在另一个终端启动界面
uv run streamlit run app/streamlit_app.py
```

文档准备和索引构建应与在线问答分开，不要每次提问或打开页面都重新下载资料、计算全部向量。

## 10. 你今天具体只做什么

今天的目标可以定为“环境清楚，能解释当前三个节点”，先不实现整个系统。

1. 确认当前根目录与 Git 状态，创建开发分支。
2. 新建 `.gitignore`。
3. 通过 uv 准备 Python 3.12 项目环境，选择 IDE 解释器。
4. 安装锁定依赖并运行原单元测试，记录真实结果。
5. 按 `langgraph.json → graph.py → state.py → configuration.py → retrieval.py → index_graph.py → prompts.py / utils.py` 阅读。
6. 在 `docs/day01-baseline.md` 用自己的话回答下面的问题。

自查问题：

- 为什么首轮查询生成通常不调用 LLM，追问会调用？
- `messages`、`queries`、`retrieved_docs` 的合并方式分别是什么？
- 资料是在哪个函数里进入回答 Prompt 的？
- 聊天模型和 Embedding 分别在哪里创建？
- 为什么 `{"docs": "delete"}` 不会删除已写入数据库的资料？
- 为什么有 messages 字段还不能保证关闭程序后继续聊天？

下一步分别验证聊天、Embedding、工具调用。再下一步用两条手写资料验收 Chroma。做到这里，才接入官方文档。

## 11. 只需要补哪些前置知识

不用先完整学完所有 AI 课程。按模块补足马上会用到的知识：

| 知识 | 需要掌握到什么程度 | 最先在哪里用 |
| --- | --- | --- |
| 模块与包、import | 理解 `src/` 包如何被安装与导入 | 环境与入口 |
| 类型注解、dataclass、继承 | 能看懂字段、默认值和类型关系 | state/configuration |
| `Annotated`、reducer | 能解释字段为何追加或覆盖 | state |
| `async def` / `await` | 明白网络等待与异步调用方式 | 模型、检索、工具和 API |
| `with` / `async with` | 明白资源进入、退出和连接生命周期 | 检索器与 SQLite |
| 装饰器 | 知道函数如何暴露为工具或 API | tools/api |
| JSON、JSONL、Pydantic | 能读写数据、验证输入输出 | 工具、评测和 API |
| HTTP、URL、状态码、超时 | 能区分接口地址、鉴权、限流和服务错误 | 模型烟测与服务 |
| pathlib、编码、哈希 | 能读取资料并生成稳定 ID | corpus |
| pytest、替身对象 | 能验证失败和上限，不每次依赖真实 API | 工具与 Agent 验证 |
| Git status/diff/commit | 能查看并保存自己理解的改动 | 每个阶段 |

例如，`cast(...)` 主要帮助类型检查器理解类型，不会把错误数据在运行时自动转换正确；`with` 管理资源，也不意味着程序自动具备持久化。遇到这些语法时，结合它在当前函数里的作用理解。

## 12. 出问题时怎样定位

| 现象 | 优先检查 |
| --- | --- |
| 包导入失败 | 当前 Python 路径是否属于项目 `.venv`，依赖是否安装 |
| 模型完全不返回 | 密钥是否加载、服务地址、模型名、网络和权限 |
| 能聊天但不能工具调用 | 所选服务和模型是否支持协议，消息和 schema 是否兼容 |
| 查不到明明导入过的资料 | 资料是否解析成功、索引集合、user_id、commit 过滤 |
| 搜到的正文缺少示例 | 是否处理 Markdown 外部代码引用 |
| 文档对，回答仍然错 | 实际模型输入、证据完整性、Prompt 和答案结论 |
| 重复运行后资料翻倍 | 是否用了随机 ID，存储写入是否按稳定 ID 更新 |
| 追问忘记主题 | 是否恢复同一 thread_id，是否保留必要历史和条件 |
| 工具一直循环 | 次数限制是否在执行前检查，是否有停止边，是否预留回答预算 |
| 引用链接存在但答案不对 | 人工核对来源正文能否支持具体结论 |

每次只定位链路里的一段：**原始资料 → 规范化文档 → 文档块 → 索引 → 命中片段 → 工具结果 → 模型输入 → 答案与引用**。保留实际输入输出后，再决定改哪一个文件。

最终完成的标志，是你能运行并解释这条完整链路，并展示来源、工具事件、失败处理和真实评测结果。
