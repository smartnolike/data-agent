# data-agent

你是一名资深 AI Architect。

请帮我设计一个可扩展的企业级 AI Agent Platform。

技术栈：

- Python FastAPI
- LangGraph
- LangChain
- MCP
- OpenAI Compatible Model
- Spring Boot Gateway（外部调用）
- Redis
- PostgreSQL
- GKE 部署

目标：

构建一个企业智能客服平台，未来不仅支持知识库问答，还支持多个 Agent、多个 Skill、MCP Tool，并可以持续扩展。

要求采用分层架构，而不是把所有逻辑写到一个 Graph。

架构要求：

第一层：

API Layer

负责：

- Chat API
- Session
- Authentication
- Streaming

第二层：

Supervisor（Orchestrator）

Supervisor 本身使用 LangGraph 实现。

Supervisor 负责：

- Intent Recognition
- Context Management
- Memory
- Candidate Agent Selection
- Planner（可选）
- Dispatcher
- Aggregator

Supervisor 不负责业务。

Supervisor 只负责决定：

"谁来完成任务"

第三层：

每一个业务能力都是独立 Agent。

例如：

Knowledge Agent

负责：

Query Rewrite

BM25

Vector Search

Hybrid Search

Rerank

Generate

Customer Agent

负责：

Company Skill

Company API

MCP Tool

Generate

DevOps Agent

负责：

Terraform

GKE

Cloud SQL

IAM

MCP Tool

Generate

要求：

每一个 Agent 都是一个独立 LangGraph。

Agent 可以单独开发。

Agent 可以单独测试。

Agent 可以独立发布。

第四层：

Tool Layer

所有 Tool 都保持 Stateless。

例如：

MCP

Company API

Weather

Calculator

Terraform

SQL

HTTP

GCP SDK

都不要写业务逻辑。

第五层：

Memory

Conversation Memory

Summary Memory

User Profile

Long Memory

要求说明：

哪些放 Redis

哪些放 PostgreSQL

哪些放 VectorDB

请输出：

1.

整体系统架构图（Mermaid）

2.

Supervisor Graph

Mermaid

3.

Knowledge Agent Graph

Mermaid

4.

DevOps Agent Graph

Mermaid

5.

Customer Agent Graph

Mermaid

6.

Python Package 目录结构

例如：

app/
 supervisor/
 agents/
 knowledge/
 customer/
 devops/
 memory/
 tools/
 models/
 graph/
 prompts/

7.

每一个目录职责

8.

Supervisor State 定义

使用 LangGraph State

9.

Agent State 定义

10.

每个 Node 的职责

例如：

IntentNode

PlannerNode

DispatchNode

MergeNode

11.

Skill 注册机制

以后新增一个 Agent

只需要：

新增目录

注册即可

不要修改 Supervisor 核心代码

要求使用 Registry Pattern。

12.

Tool 注册机制

支持 MCP

支持 LangChain Tool

支持普通 Python Tool

统一接口。

13.

Memory 设计

Conversation

Summary

Vector Memory

User Memory

什么时候读取

什么时候写入。

14.

异常处理

Tool Error

Agent Timeout

Retry

Fallback

15.

日志设计

每个 Node

每个 Agent

TraceID

LangSmith

16.

推荐哪些地方使用：

async

cache

stream

checkpoint

human in the loop

17.

最后给出：

推荐的最佳实践。

要求：

- 高内聚
- 低耦合
- 可扩展
- 企业级
- 符合 LangGraph 推荐架构
- 后续可以支持几十个 Agent
- 后续支持 Supervisor + Multi Agent





你是一位资深 AI Agent Architect。

请使用 Python + LangGraph 实现一个企业级多 Agent 框架。

## 技术栈

- Python 3.12
- FastAPI
- LangGraph
- LangChain
- Pydantic
- Async
- MCP（后续接入）
- OpenAI Compatible Model

代码要求：

- 高内聚
- 低耦合
- 可扩展
- 易测试
- 每个 Agent 独立
- 不允许循环依赖

----------------------------------------

整体架构：

Chat API

↓

Supervisor Graph

↓

Business Agent

↓

Tool

Supervisor 只负责调度。

Business Agent 负责业务。

Tool 保持 Stateless。

----------------------------------------

Supervisor(Graph)

Supervisor 本身是一个 LangGraph。

Supervisor 不允许包含任何业务逻辑。

Supervisor 不允许知道：

Bucket 参数

Terraform 参数

CloudSQL 参数

IAM 参数

Supervisor 只负责：

1.Intent Recognition

识别用户意图。

例如：

Knowledge

Customer

DevOps

SQL

2.Task Planner

如果一句话包含多个任务：

例如：

"创建 bucket，再创建 pubsub"

拆成：

Task1

create_bucket

Task2

create_pubsub

Planner 输出：

Task List

3.Dispatch

根据 Task

选择对应 Agent

例如：

DevOps Agent

Knowledge Agent

Customer Agent

4.Aggregator

等待 Agent 返回结果

汇总结果

回复用户

Supervisor 不负责：

参数收集

参数校验

调用 Tool

生成 Terraform

----------------------------------------

DevOps Agent(Graph)

DevOps Agent 是一个独立 LangGraph。

负责整个 DevOps 生命周期。

包括：

Understand Task

↓

Parameter Planner

↓

Parameter Collector

↓

Parameter Validator

↓

Tool Selector

↓

Execute

↓

Return

----------------------------------------

Parameter Planner

Parameter Planner 是 DevOps Agent 最重要的 Node。

负责：

根据 Task

计算：

需要哪些参数。

例如：

Bucket：

required：

name

region

optional：

storage_class

versioning

PubSub：

required：

topic_name

optional：

subscription

retention

Parameter Planner 负责：

1.

计算缺失参数。

2.

判断哪些参数可以使用默认值。

3.

判断哪些参数可以跨 Task 共享。

例如：

Bucket：

region

PubSub：

region

用户已经说：

"放到 asia-east1"

则：

两个 Task

共享：

region

无需再次询问。

4.

决定本轮最多问几个问题。

要求：

不要一次询问超过 3 个问题。

如果缺失参数很多：

采用：

Task Queue

逐 Task 收集。

如果缺失参数较少：

可以跨 Task 合并提问。

例如：

Bucket 缺：

bucket_name

PubSub 缺：

topic_name

则：

一次询问：

Bucket Name

Topic Name

----------------------------------------

Parameter Collector

负责：

向用户提问。

等待用户回复。

更新 State。

如果参数仍不足：

继续询问。

直到：

Parameter Validator

返回：

Enough=True

----------------------------------------

Parameter Validator

负责：

校验：

required 参数是否齐全。

如果：

Enough=False

返回：

Collector

否则：

进入：

Execute。

----------------------------------------

Execute

目前先不要真正调用 MCP。

先返回：

模拟执行。

例如：

Bucket 已创建。

PubSub 已创建。

----------------------------------------

State 设计

Supervisor State：

conversation

tasks

current_task

current_agent

history

DevOps State：

current_task

task_queue

parameter_schema

missing_parameters

collected_parameters

execution_result

----------------------------------------

Task Queue

支持：

多个 Task。

例如：

Task1

Bucket

Task2

PubSub

要求：

支持：

执行完 Task1

自动切换：

Task2

----------------------------------------

目录结构

app/

supervisor/

graph.py

nodes.py

state.py

planner.py

dispatcher.py

agents/

devops/

graph.py

nodes.py

parameter_planner.py

parameter_collector.py

validator.py

executor.py

state.py

tools/

registry.py

memory/

----------------------------------------

要求：

所有 Node

必须是独立函数。

禁止：

一个 Node 超过 200 行。

所有 Prompt

统一放 prompts.py。

所有 Tool

统一放 Tool Registry。

不要把 Prompt 写进代码。

----------------------------------------

最后：

请生成：

完整目录结构

每个文件代码

Graph

State

Node

以及运行示例。

要求：

可以直接运行。

后续方便增加：

Knowledge Agent

Customer Agent

SQL Agent

无需修改 Supervisor 核心代码。




这个 Prompt 实现出来会是一个基础框架，但还不是真正的企业级。

如果是我来做你的这个项目，我会让 Codex 分三次实现，不要一次生成全部。

第一阶段：框架（约 2000 行）

* Supervisor
* DevOps Agent
* State
* Graph
* Registry
* 基础运行

第二阶段：Parameter Planner（约 1000 行）

重点实现：

* 参数 Schema
* 参数共享（region、project 等）
* 多 Task 参数合并
* Task Queue

第三阶段：MCP 接入

重点实现：

* Tool Registry
* MCP Adapter
* Dynamic Tool Loading
* Execute Node

这样生成的代码质量会明显高于一次性让 AI 写完整个系统。

另外，我还可以帮你写一个专门针对 GPT-5 Codex 的 Prompt，利用它比较擅长工程化代码生成的特点，把 Prompt 再优化一版。我觉得效果会比上面这个再好一个档次。




You are the lead architect of an enterprise AI platform.

Do NOT generate implementation code.

Your task is to design the architecture only.

Requirements:

...

最后输出：

- Overall Architecture
- Graph Design
- State Design
- Package Design
- Class Diagram
- Sequence Diagram
- Responsibilities

不要生成任何代码。





Based on the approved architecture,

generate ONLY the project scaffold.

Requirements:

Generate:

folder

__init__.py

graph.py

state.py

nodes.py

planner.py

dispatcher.py

Do NOT implement business logic.

All functions raise NotImplementedError.

Only create interfaces.




Implement ONLY Supervisor.

Do NOT implement any Agent.

Requirements:

Intent Node

Planner Node

Dispatcher Node

Aggregator Node

State

Graph

Router

Unit Tests

No MCP.

No Tool.

No DevOps.



Implement ONLY DevOps Agent.

Do NOT modify Supervisor.

Requirements:

Parameter Planner

Parameter Collector

Validator

Executor

State

Graph

Task Queue

Mock Execute

Unit Tests



Implement ONLY Knowledge Agent.

Do NOT modify Supervisor.

Requirements:

Rewrite

Retriever

BM25

Vector

Hybrid

Generate

State

Graph

Unit Tests



Implement MCP integration.

Requirements:

MCP Adapter

Dynamic Tool Loading

Tool Registry

No business logic.

Support multiple MCP Servers.

Support future authentication provider.










Every component must follow Dependency Injection.

Business logic cannot import implementation classes.

Depend only on interfaces.

Avoid circular dependency.

Avoid singleton.

Prefer composition over inheritance.









Every Graph Node should only do one responsibility.

Each Node should be less than 100 lines.

Every Node must be independently testable.

Every State should be immutable.

Use Pydantic models.

Avoid global variables.

Avoid business logic in Graph definitions.









You are building an enterprise AI platform.

The platform will eventually contain:

- 50+ Agents
- 500+ Tools
- MCP
- LangGraph
- Multi-Agent
- Human In The Loop
- Memory
- RAG
- Workflow
- Supervisor

Every design decision must optimize for long-term scalability instead of short-term simplicity.

Do not overfit to the current requirements.

Design extension points first.

Assume new Agents will be added every week.

Assume Tools are dynamically registered.

Assume multiple LLM providers.

Assume multiple vector databases.

Everything should be pluggable.
