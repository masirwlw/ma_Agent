# DevDocs Agent 开发交接说明

更新时间：2026年10月30日。用途：在当前或新的Codex开发聊天中恢复项目目标、真实目录、源码状态与执行计划，无需复制全部历史对话。

## 1. 项目目标与用户背景

用户准备申请业务方向的AI Agent应用开发实习。岗位涉及Agent架构、Prompt工程、工具调用编排、RAG、评测、幻觉控制、多轮稳定性、工具可靠性，以及推理和训练基础设施。

用户希望通过第一个入门项目理解每个技术环节，后续再开发更深入的业务项目。用户已决定采用现成的官方技术文档，避免自行制作小说语料或大规模标注数据。计划在 **20天** 完成一个最小完整版本；执行计划暂按每天3～4小时设计，该时长是规划假设，并非用户已经承诺的可用时间。

项目名称：**DevDocs Agent：支持工具调用、证据引用与多轮追问的技术文档助手**。

底座为 `langchain-ai/retrieval-agent-template`，目标语料为FastAPI官方文档与其引用的示例代码。保留上游来源与许可证，独立贡献以实际实现、测试和实验为准。

## 2. 真实目录与已经核对的状态

**实际源码根目录：**

```text
D:\gxh\CODE\2026_code\retrieval-agent-template
```

这一层包含 `pyproject.toml`、`langgraph.json`、`src/`、`tests/` 和 `.git/`，应作为Codex主要工作目录。

2026年10月30日已通过只读命令确认：

| 项目 | 已核对结果 |
| --- | --- |
| Git根目录 | `D:/gxh/CODE/2026_code/retrieval-agent-template` |
| HEAD | `1d0c84cf19a60278f2238ead42e5f9189191cfd4` |
| 当前分支 | `main` |
| 受跟踪的源码变更 | 本次检查未发现 |
| 未跟踪文件 | 根目录中的旧版项目计划书MD、PDF |
| 实施进度 | 源码已获取，尚未开始本次计划中的功能改造 |
| 运行验证 | 本次未安装依赖、调用真实模型或运行项目测试 |

后续聊天开始时应重新检查Git状态，保留用户之后新增的修改，不把这份快照当作永远不变的当前状态。

**当前规划聊天的工作目录：**

```text
C:\Users\admin\Documents\ChatGPT\Multi-agent
```

该聊天当前配置的写入范围未包含上述D盘源码目录。本次只读取了D盘源码，新的计划与交接文件保存于C盘规划目录。尚未修改D盘业务代码，也未替用户更改Codex项目的文件夹设置。

## 3. 如何在Codex里继续

不需要删除现有项目、重新下载代码或手动汇总全部聊天。可在当前Codex本地项目的菜单中选择“编辑项目”，添加实际源码文件夹，并将它设为主目录。新聊天以主目录作为默认工作目录；若当前聊天仍沿用旧目录，就在设好主目录的项目中新开开发聊天。

仅在终端执行 `Set-Location` 不等于改变Codex项目的文件夹配置或文件权限。新开发聊天开始时，应确认当前工作目录、Git根目录和可编辑范围指向源码目录。

以上操作依据官方说明，界面标签可能因版本或语言略有差异：[OpenAI官方项目与聊天说明](https://learn.chatgpt.com/docs/projects?surface=app)。

将本文件与《DevDocs-Agent-20天执行计划.md》复制到源码根目录下的 `docs/`，或直接附加给新聊天。旧版《DevDocs-Agent-项目计划书.md》保留作技术背景；时间安排与范围以20天执行计划为准。

## 4. 目标功能与范围

20天内应形成一条可演示、可排错、可评估的完整链路：

1. 自动读取一份固定commit的FastAPI官方Markdown文档与引用示例。
2. 清洗、切块、稳定ID、来源元数据、Embedding、本地Chroma持久化。
3. 保留模板的固定RAG流程作对照；新增BM25与向量的RRF混合检索。
4. 增加三个只读工具：`search_docs`、`read_document`、`read_example`。
5. 一个有界Agent循环，明确参数校验、调用上限、超时与有限重试。
6. 回答附本轮真实证据的引用；证据不足可澄清或说明无法支持结论。
7. 多轮会话、异步SQLite检查点、近期完整轮次裁剪与证据预算。
8. 最小FastAPI服务与Streamlit展示；查看来源、真实工具轨迹及用量。
9. 逐步形成20～30条经人工核对来源的评测题，保留开发与最终用例。
10. 对比报告、失败分析、README、演示与真实贡献说明。

LLM摘要、Reranker、流式输出、多Agent、任意代码执行、多租户鉴权、GPU训练与大规模部署均为后续扩展，不纳入20天必做项。这个项目主要对应岗位的应用、RAG、工具与评测要求，不能宣称已经具备亿级用户或异构计算研发能力。

## 5. 已确认的源码结构

真实包目录是 `src/retrieval_graph/`；部分README描述仍使用旧目录名，开发以实际文件为准。

| 文件 | 当前作用 |
| --- | --- |
| `graph.py` | `generate_query → retrieve → respond`固定流程；不是完整的动态工具Agent |
| `index_graph.py` | 为传入文档附加 `user_id` 并写入后端 |
| `retrieval.py` | Embedding和Elasticsearch、Pinecone、MongoDB适配；尚无Chroma本地适配 |
| `state.py` | 消息、历史查询和当前检索文档 |
| `configuration.py` | 用户、模型、检索与Prompt配置；通过 `from_runnable_config`读取 |
| `prompts.py` | 查询与回答Prompt |
| `utils.py` | 消息解析、资料格式化、模型加载 |
| `__init__.py` | 导出基础图与索引图 |
| `langgraph.json` | 现有图入口为 `indexer`、`retrieval_graph` |

## 6. 开发时必须保留的约定

- **身份一致：** 第一版本地单用户固定 `user_id="local-demo"`，索引、问答和评测一致。语料ID与commit单独管理，不能漏掉身份过滤后误以为检索失效。
- **配置一致：** 20天版先延续 `RunnableConfig.configurable` 与上游配置读取方式。`thread_id`用于检查点；不在入门期同时做运行时context的全面迁移。
- **接口清楚：** 文档写入使用存储对象的明确写入接口，检索使用搜索接口，不假定每种retriever都有 `aadd_documents`。
- **可比基线：** 原固定图保留；共同的本地适配与资料处理如实记录。冻结基线Prompt和参数，避免改Agent时连对照组一起改变。
- **引用边界：** 引用ID只来自当前轮实际读取的证据。存在有效引用不等于语义正确，后者通过人工评测核对。
- **预算有代码保证：** 调用和重试有独立计数；只执行单个工具调用，预留最终回答额度。
- **会话资源：** 异步图使用匹配的异步SQLite检查点，由服务生命周期持有连接，避免每个请求重建内存状态。
- **路径可迁移：** 业务代码不硬编码 `D:\gxh\...`；本地数据目录通过项目根目录或配置确定。
- **如实记录：** 没有运行过的测试、没有实测的提升和Token用量不能写成已验证。缺失用量标未知，不标0。
- **分步学习：** 每次实现一个可解释的变化，留下验收与失败记录；不一次生成完整系统后跳过理解。

## 7. 下一步从哪里开始

当前进度是 **Day 1之前：源码已获取并完成只读核对**。

下一步应：

1. 将D盘源码目录设为Codex项目主目录，确认当前开发聊天的编辑范围。
2. 阅读该源码目录及父目录的适用 `AGENTS.md`；本次检查的父目录链与源码中未发现该文件，但以后可能新增。
3. 重新确认Git状态，在保留用户修改的前提下创建开发分支。
4. 按20天计划完成Day 1：环境准备、原单元测试与代码结构笔记。
5. Day 2分别验证聊天模型、Embedding与最小工具调用。Day 3再加入本地Chroma。

尚未确定用户使用哪个模型服务，也未确认API配置和可用额度。先询问可用服务或读取不含秘密的配置说明，不要求用户把API Key直接发在聊天中。

## 8. 可直接用于新聊天的开场消息

```text
我要继续DevDocs Agent入门项目。

源码目录：D:\gxh\CODE\2026_code\retrieval-agent-template
底座：langchain-ai/retrieval-agent-template
已核对起始提交：1d0c84cf19a60278f2238ead42e5f9189191cfd4
目标：20天完成基于FastAPI官方资料的文档Agent，用来学习RAG、工具调用、会话、可靠性与评测。

请先阅读我附加或放在docs目录中的：
1. DevDocs-Agent-开发交接说明.md
2. DevDocs-Agent-20天执行计划.md

然后检查真实工作目录、AGENTS.md、Git状态与当前进度。
目前的快照是：源码已获取，尚未开始功能改造；请重新核对，不覆盖任何已有修改。
从Day 1开始，每次只完成当天任务，解释改了哪些文件、为什么改、如何验收。
先做本地可用版本，统一user_id为local-demo，保留原RAG对照。
未经实际运行，不声称测试已通过或指标已提升。
```

## 9. 资料位置

规划聊天中的交付目录：

```text
C:\Users\admin\Documents\ChatGPT\Multi-agent\plans
```

文件包括本交接说明、20天执行计划，以及此前的完整项目计划书。若在新项目内继续，先复制或附加这些文件；新聊天不保证自动继承本聊天的全部对话。

上游参考：[固定提交](https://github.com/langchain-ai/retrieval-agent-template/commit/1d0c84cf19a60278f2238ead42e5f9189191cfd4)、[核心图源码](https://github.com/langchain-ai/retrieval-agent-template/blob/1d0c84cf19a60278f2238ead42e5f9189191cfd4/src/retrieval_graph/graph.py)。
