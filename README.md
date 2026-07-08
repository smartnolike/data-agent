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