# AI Agent Terraform JSON 生成平台需求文档 v0.1

## 1. 项目背景

Terraform 是基础设施即代码领域的事实标准之一，但直接编写 Terraform 配置仍然有较高门槛，尤其是在多云资源、网络拓扑、安全策略、模块依赖、变量输出、环境隔离等场景中，用户容易写出结构不完整或不可验证的配置。

本项目希望构建一个 AI Agent，让用户通过自然语言描述基础设施需求，由 Agent 自动选择预定义 Skill，生成结构化、可校验、可迭代修改的 Terraform JSON 配置。

项目重点不是让大模型自由生成 Terraform，而是让大模型负责理解需求、规划资源、选择 Skill、解释错误和修复建议；具体 Terraform JSON 结构由确定性的 Skill、模板和校验流程生成。

## 2. 项目目标

### 2.1 核心目标

- 支持用户使用自然语言描述基础设施需求。
- 自动解析用户意图，生成标准化资源计划。
- 根据资源计划选择并调用 Terraform Skill。
- 生成 Terraform JSON 文件，例如 `main.tf.json`、`variables.tf.json`、`outputs.tf.json`。
- 对生成结果进行结构校验、Terraform CLI 校验和自动修复。
- 支持多轮对话修改同一套基础设施定义。
- 支持 Skill 插件化扩展，后续可以新增 AWS、Azure、GCP、Kubernetes 等资源能力。

### 2.2 非目标

- 第一阶段不直接执行 `terraform apply`。
- 第一阶段不做完整云账号资源发现。
- 第一阶段不支持复杂企业级审批流。
- 第一阶段不追求覆盖所有 Terraform Provider 资源。

## 3. 推荐技术栈

### 3.1 后端语言

推荐使用 Python。

原因：

- LangGraph Python 生态成熟。
- Pydantic 适合定义结构化输入输出。
- Terraform、OPA、Conftest、JSON Schema 等工具易于集成。
- 后续接入 RAG、文档检索、异步任务也比较方便。

### 3.2 核心框架

| 模块 | 推荐技术 | 用途 |
| --- | --- | --- |
| Agent 编排 | LangGraph | 定义多节点状态机和循环修复流程 |
| LLM 调用 | LangChain 或原生 SDK | 调用模型、结构化输出、Tool Calling |
| Schema | Pydantic | 定义 Intent、Plan、Skill Input、Skill Output |
| 模板生成 | Jinja2 或纯 Python dict | 生成 Terraform JSON 片段 |
| JSON 校验 | jsonschema / Pydantic | 校验中间结构和最终结构 |
| Terraform 校验 | Terraform CLI | 执行 `terraform init -backend=false`、`terraform validate` |
| 策略检查 | OPA / Conftest | 检查安全策略、合规规则，可后续引入 |
| API 服务 | FastAPI | 对外提供 REST API |
| 任务队列 | Celery / RQ / Arq | 后续处理长任务，例如 validate、plan |
| 存储 | PostgreSQL / SQLite | 存储会话、项目、版本、生成记录 |
| 文件存储 | 本地目录 / S3 | 存储生成的 Terraform 文件 |

### 3.3 MVP 推荐组合

MVP 阶段建议保持简单：

- Python
- FastAPI
- LangGraph
- Pydantic
- Jinja2
- Terraform CLI
- SQLite

## 4. 用户角色

### 4.1 平台使用者

通过自然语言描述基础设施需求，查看生成结果，继续通过对话修改配置。

### 4.2 平台管理员

维护 Skill、Provider 版本、默认安全策略、模板和校验规则。

### 4.3 DevOps / Infra 工程师

审查 Agent 生成的 Terraform JSON，将结果接入 GitOps、CI/CD 或 Terraform Cloud。

## 5. 典型使用流程

### 5.1 首次生成

用户输入：

```text
帮我生成一个 AWS 生产环境 VPC，CIDR 是 10.0.0.0/16，跨 3 个 AZ，每个 AZ 有 public subnet 和 private subnet，需要 NAT Gateway，后续用于 EKS。
```

Agent 流程：

1. 解析用户意图。
2. 判断云厂商为 AWS。
3. 识别需要 VPC、Subnet、Route Table、Internet Gateway、NAT Gateway、Outputs。
4. 检查缺失参数，例如 region、命名前缀、是否单 NAT 或多 NAT。
5. 如参数缺失且影响较大，向用户追问。
6. 生成资源计划。
7. 调用对应 Skill。
8. 合并 Terraform JSON。
9. 执行校验。
10. 输出文件和说明。

### 5.2 多轮修改

用户输入：

```text
把 NAT Gateway 改成每个 AZ 一个，另外加一个 RDS MySQL，只允许 EKS 节点访问。
```

Agent 应基于已有状态：

- 找到当前项目中的 VPC 和 private subnet。
- 修改 NAT Gateway 拓扑。
- 新增 RDS、DB Subnet Group、Security Group。
- 更新依赖关系和 outputs。
- 再次校验。

## 6. 系统总体架构

```text
Client / UI / API
        |
        v
FastAPI Service
        |
        v
LangGraph Agent Workflow
        |
        +--> Intent Parser
        +--> Missing Info Resolver
        +--> Resource Planner
        +--> Skill Router
        +--> Skill Executor
        +--> Terraform JSON Merger
        +--> Static Validator
        +--> Terraform CLI Validator
        +--> Repair Node
        +--> Finalizer
        |
        v
Project Store / File Store / Skill Registry
```

## 7. LangGraph 状态设计

Agent 的所有节点共享一个 `AgentState`。

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

class UserMessage(BaseModel):
    role: Literal["user", "assistant", "system"]
    content: str

class ParsedIntent(BaseModel):
    cloud: str | None = None
    region: str | None = None
    environment: str | None = None
    resources: list[dict[str, Any]] = Field(default_factory=list)
    constraints: list[str] = Field(default_factory=list)
    assumptions: list[str] = Field(default_factory=list)

class ResourcePlan(BaseModel):
    project_name: str
    provider: str
    region: str
    resources: list[dict[str, Any]]
    dependencies: list[dict[str, str]] = Field(default_factory=list)

class SkillCall(BaseModel):
    skill_name: str
    input: dict[str, Any]
    reason: str

class TerraformFragment(BaseModel):
    terraform: dict[str, Any] = Field(default_factory=dict)
    provider: dict[str, Any] = Field(default_factory=dict)
    variable: dict[str, Any] = Field(default_factory=dict)
    resource: dict[str, Any] = Field(default_factory=dict)
    data: dict[str, Any] = Field(default_factory=dict)
    output: dict[str, Any] = Field(default_factory=dict)
    locals: dict[str, Any] = Field(default_factory=dict)

class ValidationResult(BaseModel):
    ok: bool
    source: str
    errors: list[str] = Field(default_factory=list)
    warnings: list[str] = Field(default_factory=list)

class AgentState(BaseModel):
    project_id: str
    messages: list[UserMessage] = Field(default_factory=list)
    intent: ParsedIntent | None = None
    missing_questions: list[str] = Field(default_factory=list)
    plan: ResourcePlan | None = None
    skill_calls: list[SkillCall] = Field(default_factory=list)
    fragments: list[TerraformFragment] = Field(default_factory=list)
    final_tf_json: dict[str, Any] = Field(default_factory=dict)
    validation_results: list[ValidationResult] = Field(default_factory=list)
    repair_attempts: int = 0
    files: dict[str, str] = Field(default_factory=dict)
```

## 8. LangGraph 节点设计

### 8.1 `parse_intent_node`

职责：

- 将用户自然语言转换为结构化 `ParsedIntent`。
- 提取云厂商、region、环境、资源类型、数量、网络拓扑、安全约束、命名偏好。
- 识别用户是在新建项目还是修改已有项目。

输入：

- `messages`
- 当前项目已有状态

输出：

- `intent`

实现方式：

- 使用 LLM structured output。
- 输出必须符合 Pydantic schema。
- 对不确定字段使用 `null`，不要编造。

关键逻辑：

```python
def parse_intent_node(state: AgentState) -> AgentState:
    prompt = build_parse_intent_prompt(state.messages, state.final_tf_json)
    intent = llm.with_structured_output(ParsedIntent).invoke(prompt)
    state.intent = intent
    return state
```

优化点：

- 加入资源同义词映射，例如 ECS/EC2、对象存储/S3、数据库/RDS。
- 根据历史项目状态判断用户是否在“增量修改”。
- 对高风险默认值不要自动假设，例如公网数据库、开放 `0.0.0.0/0` 入站。

### 8.2 `resolve_missing_info_node`

职责：

- 判断生成 Terraform 是否缺少关键参数。
- 将缺失参数分为必须追问和可以默认。

必须追问示例：

- 云厂商不明确。
- region 缺失且用户没有默认配置。
- 数据库是否允许公网访问。
- 生产环境 NAT Gateway 是单个还是每 AZ 一个。

可以默认示例：

- 命名前缀：从项目名生成。
- 默认 tag：`Environment`、`ManagedBy`。
- VPC DNS support：默认开启。

输出：

- `missing_questions`

路由：

- 如果存在必须追问的问题，进入 `ask_user_node`。
- 否则进入 `plan_resources_node`。

关键逻辑：

```python
def resolve_missing_info_node(state: AgentState) -> AgentState:
    questions = []
    intent = state.intent

    if not intent.cloud:
        questions.append("你希望使用哪个云厂商？例如 AWS、Azure 或 GCP。")

    if not intent.region:
        questions.append("你希望部署在哪个 region？")

    if contains_database(intent) and not has_database_access_policy(intent):
        questions.append("数据库是否需要公网访问？如果不需要，我会默认只允许应用安全组访问。")

    state.missing_questions = questions
    return state
```

优化点：

- 每轮最多追问 3 个最关键问题。
- 对低风险缺省值写入 `assumptions`，最终展示给用户。
- 后续支持用户级默认配置，例如默认云厂商、默认 region。

### 8.3 `ask_user_node`

职责：

- 将缺失问题返回给用户。
- 暂停图执行，等待用户下一轮输入。

实现方式：

- API 层返回 `needs_input` 状态。
- LangGraph 可以使用 interrupt/human-in-the-loop 模式。

输出示例：

```json
{
  "status": "needs_input",
  "questions": [
    "你希望部署在哪个 AWS region？",
    "生产环境 NAT Gateway 是每个 AZ 一个，还是为了省成本只创建一个？"
  ]
}
```

优化点：

- 问题要带默认建议。
- 对生产环境默认建议高可用，对开发环境默认建议低成本。

### 8.4 `plan_resources_node`

职责：

- 将 intent 转换成标准资源计划 `ResourcePlan`。
- 明确每个资源的名称、类型、数量、依赖、输入参数。
- 不生成 Terraform JSON，只生成计划。

输入：

- `intent`
- 用户默认配置
- 现有 Terraform 状态

输出：

- `plan`

计划示例：

```json
{
  "project_name": "prod-eks-network",
  "provider": "aws",
  "region": "us-east-1",
  "resources": [
    {
      "id": "vpc.main",
      "kind": "aws_vpc",
      "name": "main",
      "cidr_block": "10.0.0.0/16"
    },
    {
      "id": "subnet.public",
      "kind": "aws_subnet_group",
      "tier": "public",
      "az_count": 3
    },
    {
      "id": "nat_gateway.main",
      "kind": "aws_nat_gateway",
      "mode": "per_az"
    }
  ],
  "dependencies": [
    {"from": "subnet.public", "to": "vpc.main"},
    {"from": "nat_gateway.main", "to": "subnet.public"}
  ]
}
```

实现方式：

- 先用规则补全常见基础设施模式。
- 再用 LLM 对复杂需求做结构化规划。
- 最终通过 Pydantic 校验。

优化点：

- 资源计划是后续所有 Skill 的唯一输入来源。
- 对依赖关系做拓扑排序。
- 对命名规则做统一处理，避免不同 Skill 生成的名字不一致。

### 8.5 `skill_router_node`

职责：

- 根据 `ResourcePlan` 选择要调用的 Skill。
- 为每个 Skill 准备输入。
- 确定 Skill 调用顺序。

输入：

- `plan`
- Skill Registry

输出：

- `skill_calls`

Skill 选择规则：

```text
aws_vpc                 -> aws_vpc_skill
aws_subnet_group        -> aws_subnet_skill
aws_internet_gateway    -> aws_igw_skill
aws_nat_gateway         -> aws_nat_gateway_skill
aws_security_group      -> aws_security_group_skill
aws_rds                 -> aws_rds_skill
aws_s3                  -> aws_s3_skill
aws_iam_role            -> aws_iam_skill
```

实现方式：

```python
def skill_router_node(state: AgentState) -> AgentState:
    calls = []
    for resource in topological_sort(state.plan.resources, state.plan.dependencies):
        skill = skill_registry.find(provider=state.plan.provider, kind=resource["kind"])
        calls.append(SkillCall(
            skill_name=skill.name,
            input=skill.build_input(resource, state.plan),
            reason=f"Generate {resource['kind']}"
        ))
    state.skill_calls = calls
    return state
```

优化点：

- Skill Registry 不建议让 LLM 动态决定，优先用规则映射。
- 当多个 Skill 可用时，用 provider、版本、环境、资源类型过滤。
- Skill 调用前做 input schema 校验。

### 8.6 `skill_executor_node`

职责：

- 执行所有 Skill。
- 收集每个 Skill 输出的 Terraform fragment。
- 记录 Skill 执行错误。

输入：

- `skill_calls`

输出：

- `fragments`

Skill 接口建议：

```python
from abc import ABC, abstractmethod
from pydantic import BaseModel

class TerraformSkill(ABC):
    name: str
    provider: str
    supported_kinds: list[str]
    input_model: type[BaseModel]

    @abstractmethod
    def generate(self, data: BaseModel) -> TerraformFragment:
        pass
```

实现方式：

```python
def skill_executor_node(state: AgentState) -> AgentState:
    fragments = []
    for call in state.skill_calls:
        skill = skill_registry.get(call.skill_name)
        validated_input = skill.input_model.model_validate(call.input)
        fragment = skill.generate(validated_input)
        fragments.append(fragment)
    state.fragments = fragments
    return state
```

优化点：

- Skill 内部尽量使用 Python dict 生成 Terraform JSON，而不是纯文本拼接。
- 每个 Skill 配套单元测试和 golden file。
- Skill 不直接读写最终文件，只返回 fragment。

### 8.7 `terraform_merge_node`

职责：

- 将多个 Terraform fragment 合并为完整 Terraform JSON。
- 处理同名资源冲突、变量冲突、provider 配置合并。
- 输出标准 Terraform JSON dict。

输入：

- `fragments`

输出：

- `final_tf_json`

Terraform JSON 结构：

```json
{
  "terraform": {},
  "provider": {},
  "variable": {},
  "locals": {},
  "data": {},
  "resource": {},
  "output": {}
}
```

合并规则：

- `terraform.required_providers` 深度合并。
- `provider` 同 provider 只允许一个默认配置，alias provider 可以多个。
- `resource.<type>.<name>` 不允许重复。
- `variable.<name>` 不允许类型冲突。
- `output.<name>` 不允许重复，除非内容相同。

实现方式：

```python
def merge_resource(target: dict, incoming: dict) -> None:
    for resource_type, named_resources in incoming.items():
        target.setdefault(resource_type, {})
        for name, body in named_resources.items():
            if name in target[resource_type] and target[resource_type][name] != body:
                raise ValueError(f"Resource conflict: {resource_type}.{name}")
            target[resource_type][name] = body
```

优化点：

- 资源合并必须确定性排序，方便 Git diff。
- 出现冲突时不要覆盖，进入修复节点。
- 对 Terraform interpolation 保持字符串原样，例如 `"${aws_vpc.main.id}"`。

### 8.8 `static_validate_node`

职责：

- 在运行 Terraform CLI 前做快速静态检查。
- 检查 JSON 格式、资源结构、命名规范、安全默认值。

检查项：

- JSON 可序列化。
- 顶层 key 是否合法。
- resource 层级是否为 `resource_type -> resource_name -> config`。
- 是否存在重复资源名。
- 是否存在明显危险规则，例如生产环境开放 SSH 到 `0.0.0.0/0`。
- 是否存在缺失引用。

输出：

- `validation_results`

实现方式：

```python
def static_validate_node(state: AgentState) -> AgentState:
    errors = []
    warnings = []

    if "resource" not in state.final_tf_json:
        warnings.append("No resources generated.")

    errors.extend(validate_resource_shape(state.final_tf_json))
    warnings.extend(validate_security_baseline(state.final_tf_json, state.plan.environment))

    state.validation_results.append(ValidationResult(
        ok=not errors,
        source="static",
        errors=errors,
        warnings=warnings
    ))
    return state
```

优化点：

- 静态校验失败不一定需要 LLM 修复，优先由规则修复。
- 安全问题可作为 warning 或 hard error，由环境决定。

### 8.9 `terraform_cli_validate_node`

职责：

- 将 Terraform JSON 写入临时工作目录。
- 执行 Terraform CLI 校验。
- 收集错误信息。

执行命令：

```bash
terraform init -backend=false
terraform validate -json
```

输出：

- `ValidationResult(source="terraform_cli")`

实现方式：

```python
def terraform_cli_validate_node(state: AgentState) -> AgentState:
    workdir = create_project_workspace(state.project_id)
    write_json(workdir / "main.tf.json", state.final_tf_json)

    init_result = run(["terraform", "init", "-backend=false"], cwd=workdir)
    validate_result = run(["terraform", "validate", "-json"], cwd=workdir)

    errors = parse_terraform_validate_errors(validate_result.stdout)
    state.validation_results.append(ValidationResult(
        ok=validate_result.returncode == 0,
        source="terraform_cli",
        errors=errors
    ))
    return state
```

优化点：

- Terraform init 可能需要下载 provider，CI 或沙箱环境要处理缓存。
- provider 版本要锁定，避免今天能过、明天失败。
- CLI 错误信息要转换成结构化 repair hints。

### 8.10 `policy_validate_node`

职责：

- 执行安全和合规策略检查。
- MVP 可选，建议 v0.2 加入。

检查示例：

- 生产环境禁止开放数据库公网访问。
- 生产环境禁止 SSH 开放到全网。
- S3 bucket 默认开启加密和阻止 public access。
- RDS 必须开启 deletion protection。
- IAM Policy 禁止 `Action: "*"` 且 `Resource: "*"`。

技术选择：

- OPA/Rego
- Conftest
- 自定义 Python 规则

优化点：

- 策略检查分为 error 和 warning。
- 给每条策略规则配置文档链接和修复建议。

### 8.11 `repair_node`

职责：

- 根据静态校验、Terraform CLI 校验和策略检查结果修复生成内容。
- 限制最大修复次数，避免无限循环。

输入：

- `final_tf_json`
- `validation_results`
- `repair_attempts`

输出：

- 更新后的 `final_tf_json`
- `repair_attempts + 1`

修复优先级：

1. 确定性规则修复。
2. Skill 重新生成局部 fragment。
3. LLM 根据错误和 schema 生成 JSON Patch。

推荐使用 JSON Patch，而不是让 LLM 重写完整 Terraform：

```json
[
  {
    "op": "add",
    "path": "/resource/aws_security_group_rule/db_ingress",
    "value": {
      "type": "ingress",
      "from_port": 3306,
      "to_port": 3306,
      "protocol": "tcp",
      "security_group_id": "${aws_security_group.db.id}",
      "source_security_group_id": "${aws_security_group.app.id}"
    }
  }
]
```

实现方式：

```python
MAX_REPAIR_ATTEMPTS = 3

def repair_node(state: AgentState) -> AgentState:
    if state.repair_attempts >= MAX_REPAIR_ATTEMPTS:
        return state

    errors = collect_latest_errors(state.validation_results)

    patch = llm.with_structured_output(JsonPatchList).invoke(
        build_repair_prompt(state.final_tf_json, errors)
    )

    state.final_tf_json = apply_json_patch(state.final_tf_json, patch)
    state.repair_attempts += 1
    return state
```

优化点：

- LLM 只能输出 patch，不能输出解释性文本。
- Patch 应先 dry-run，再应用。
- 修复后重新进入 `static_validate_node` 和 `terraform_cli_validate_node`。

### 8.12 `finalize_node`

职责：

- 将最终 Terraform JSON 拆分为文件。
- 生成摘要、假设、warning、使用说明。
- 持久化项目版本。

输出文件建议：

```text
generated/
  main.tf.json
  variables.tf.json
  outputs.tf.json
  providers.tf.json
  README.md
```

拆分规则：

- `terraform`、`provider` 写入 `providers.tf.json`。
- `variable` 写入 `variables.tf.json`。
- `output` 写入 `outputs.tf.json`。
- `resource`、`data`、`locals` 写入 `main.tf.json`。

输出摘要示例：

```json
{
  "status": "success",
  "files": [
    "providers.tf.json",
    "main.tf.json",
    "variables.tf.json",
    "outputs.tf.json"
  ],
  "warnings": [
    "已默认开启 VPC DNS support。",
    "生产环境 NAT Gateway 使用 per_az 模式。"
  ]
}
```

## 9. 路由逻辑设计

LangGraph 可以采用以下路由：

```text
START
  -> parse_intent_node
  -> resolve_missing_info_node
  -> route_missing_info

route_missing_info:
  has_questions -> ask_user_node -> END
  no_questions  -> plan_resources_node

plan_resources_node
  -> skill_router_node
  -> skill_executor_node
  -> terraform_merge_node
  -> static_validate_node
  -> route_static_validation

route_static_validation:
  failed -> repair_node
  passed -> terraform_cli_validate_node

terraform_cli_validate_node
  -> route_cli_validation

route_cli_validation:
  failed and repair_attempts < 3 -> repair_node -> static_validate_node
  failed and repair_attempts >= 3 -> finalize_failed_node
  passed -> policy_validate_node

policy_validate_node
  -> route_policy_validation

route_policy_validation:
  hard_failed -> repair_node
  passed_or_warning -> finalize_node
```

## 10. Skill 体系设计

### 10.1 Skill 目录结构

```text
skills/
  aws/
    vpc/
      skill.py
      schema.py
      tests/
        test_vpc_skill.py
        golden_basic_vpc.json
    subnet/
    nat_gateway/
    security_group/
    ec2/
    rds/
    s3/
    iam/
```

### 10.2 Skill 元数据

```python
class SkillMetadata(BaseModel):
    name: str
    provider: str
    version: str
    supported_kinds: list[str]
    required_provider_versions: dict[str, str]
    description: str
```

示例：

```json
{
  "name": "aws_vpc_skill",
  "provider": "aws",
  "version": "0.1.0",
  "supported_kinds": ["aws_vpc"],
  "required_provider_versions": {
    "aws": "~> 5.0"
  },
  "description": "Generate AWS VPC Terraform JSON."
}
```

### 10.3 Skill 输出规范

每个 Skill 只输出 fragment：

```python
class TerraformFragment(BaseModel):
    terraform: dict = {}
    provider: dict = {}
    variable: dict = {}
    locals: dict = {}
    data: dict = {}
    resource: dict = {}
    output: dict = {}
```

### 10.4 首批 Skill 建议

MVP 建议支持 AWS：

- `aws_provider_skill`
- `aws_vpc_skill`
- `aws_subnet_skill`
- `aws_route_table_skill`
- `aws_igw_skill`
- `aws_nat_gateway_skill`
- `aws_security_group_skill`
- `aws_ec2_skill`
- `aws_rds_skill`
- `aws_s3_skill`
- `aws_output_skill`

## 11. Terraform JSON 生成原则

### 11.1 生成策略

- 优先使用 Python dict，不拼接 JSON 字符串。
- 所有资源名使用统一命名器。
- 所有可复用值尽量放入 `variable` 或 `locals`。
- 所有重要资源输出 `output`。
- 所有资源默认添加 tags。

### 11.2 命名规则

```text
<environment>-<project>-<component>
```

示例：

```text
prod-payment-vpc
prod-payment-private-subnet-a
prod-payment-rds
```

Terraform resource name 使用下划线：

```text
main
private_a
app_sg
db_sg
```

### 11.3 默认安全基线

- RDS 默认不公网暴露。
- S3 默认开启 block public access。
- Security Group 默认最小入站权限。
- 生产环境默认启用高可用配置。
- IAM 默认最小权限，不生成 `*:*` 策略。

## 12. API 设计

### 12.1 创建或继续会话

```http
POST /api/projects/{project_id}/messages
```

请求：

```json
{
  "message": "创建一个 AWS VPC，跨 3 个 AZ，需要 private subnet 和 NAT Gateway"
}
```

响应：

```json
{
  "status": "success",
  "project_id": "proj_123",
  "summary": "已生成 AWS VPC、Subnet、Route Table、NAT Gateway。",
  "files": [
    "providers.tf.json",
    "main.tf.json",
    "variables.tf.json",
    "outputs.tf.json"
  ],
  "warnings": []
}
```

需要用户补充时：

```json
{
  "status": "needs_input",
  "questions": [
    "你希望部署在哪个 AWS region？"
  ]
}
```

### 12.2 获取生成文件

```http
GET /api/projects/{project_id}/files
```

### 12.3 下载项目

```http
GET /api/projects/{project_id}/archive
```

### 12.4 校验项目

```http
POST /api/projects/{project_id}/validate
```

## 13. 数据模型

### 13.1 Project

```text
id
name
provider
region
environment
created_at
updated_at
```

### 13.2 ConversationMessage

```text
id
project_id
role
content
created_at
```

### 13.3 ProjectVersion

```text
id
project_id
version
intent_json
plan_json
terraform_json
validation_json
created_at
```

### 13.4 GeneratedFile

```text
id
project_id
version_id
path
content
created_at
```

## 14. 校验与修复策略

### 14.1 校验层级

1. Pydantic schema 校验。
2. Terraform JSON shape 校验。
3. 引用关系校验。
4. Terraform CLI 校验。
5. 策略校验。

### 14.2 修复策略

- Schema 错误：优先由代码修复。
- 资源冲突：回到 `plan_resources_node` 或要求用户确认。
- Terraform CLI 错误：LLM 输出 JSON Patch。
- 策略错误：规则修复或向用户解释风险。

### 14.3 最大修复次数

默认最多 3 次。

超过后返回失败结果，并输出：

- 当前生成文件。
- 错误列表。
- 建议用户补充的信息。

## 15. MVP 范围

### 15.1 必须实现

- 单项目多轮对话。
- AWS Provider。
- VPC、Subnet、IGW、NAT Gateway、Security Group、EC2、RDS、S3。
- Terraform JSON 生成。
- Terraform CLI validate。
- 最多 3 次自动修复。
- 输出文件下载。

### 15.2 可以延后

- 多云支持。
- 成本估算。
- OPA/Conftest。
- Terraform plan。
- Terraform Cloud 集成。
- GitHub PR 自动提交。
- 图形化拓扑展示。

## 16. 开发里程碑

### Phase 1：核心原型

- 搭建 FastAPI 项目。
- 定义 Pydantic schema。
- 实现 LangGraph 主流程。
- 实现 AWS VPC、Subnet、IGW、NAT Gateway Skill。
- 生成单个 `main.tf.json`。
- 支持静态校验。

### Phase 2：可用 MVP

- 增加 EC2、RDS、S3、Security Group Skill。
- 支持文件拆分。
- 接入 Terraform CLI validate。
- 实现 repair loop。
- 增加项目版本存储。

### Phase 3：工程化增强

- 增加策略校验。
- 增加 Skill 单元测试和 golden file。
- 增加 Web UI。
- 增加生成结果 diff。
- 支持导出 zip。

### Phase 4：高级能力

- 接入 Terraform provider 文档 RAG。
- 支持 Terraform plan。
- 支持 GitHub PR。
- 支持 Azure/GCP。
- 支持企业策略包。

## 17. 风险与应对

| 风险 | 说明 | 应对 |
| --- | --- | --- |
| LLM 编造 Terraform 字段 | 模型可能生成不存在的参数 | Skill 使用确定性模板，CLI validate |
| Provider 版本变化 | 新旧参数不兼容 | 锁定 provider 版本 |
| 自动修复误改 | LLM 重写完整 JSON 容易引入新问题 | 使用 JSON Patch，限制作用范围 |
| 安全默认值不合理 | 可能生成公网开放资源 | 策略校验和环境感知默认值 |
| Skill 扩展混乱 | 多人维护 Skill 容易不一致 | 统一 Skill 接口、schema、测试 |
| Terraform init 依赖网络 | 下载 provider 可能失败 | 配置 provider cache，CI 预热 |

## 18. 推荐项目目录

```text
terraform_agent/
  app/
    main.py
    api/
      routes.py
    agent/
      graph.py
      state.py
      nodes/
        parse_intent.py
        resolve_missing_info.py
        plan_resources.py
        skill_router.py
        skill_executor.py
        merge.py
        validate_static.py
        validate_terraform.py
        repair.py
        finalize.py
    skills/
      base.py
      registry.py
      aws/
        provider.py
        vpc.py
        subnet.py
        route_table.py
        nat_gateway.py
        security_group.py
        ec2.py
        rds.py
        s3.py
    terraform/
      writer.py
      merger.py
      validator.py
      naming.py
    storage/
      models.py
      repository.py
    llm/
      client.py
      prompts.py
  tests/
    skills/
    agent/
    golden/
  generated/
  pyproject.toml
  README.md
```

## 19. 关键实现建议

### 19.1 LLM 只做结构化输出

所有 LLM 节点都必须使用 schema 限制输出，避免自由文本进入核心流程。

### 19.2 Skill 不依赖对话上下文

Skill 只接收结构化 input，不直接读用户消息。这样 Skill 可以单测，也可以被非 AI 系统复用。

### 19.3 资源计划是核心中间产物

不要从用户意图直接生成 Terraform。先生成 ResourcePlan，再路由 Skill。

### 19.4 修复使用 JSON Patch

不要让模型重写整个 Terraform JSON。错误越局部，修复越稳定。

### 19.5 每个 Skill 必须有 golden test

例如：

```text
输入：basic_vpc_input.json
输出：golden_basic_vpc.tf.json
```

这样后续改模板时可以稳定回归。

## 20. 下一步建议

建议下一步先实现最小闭环：

```text
自然语言输入
  -> ParsedIntent
  -> ResourcePlan
  -> AWS VPC Skill
  -> Terraform JSON
  -> 静态校验
  -> 输出 main.tf.json
```

完成后再加入 Terraform CLI validate 和 repair loop。这样可以尽快验证 Agent + Skill 的核心设计是否成立。


# 项目需求：基于 GitHub Markdown 知识库的 AI Chat Assistant（MVP）

## 项目背景

公司内部有大量技术文档存储在 GitHub Repository 中，文档格式主要为 Markdown（.md）。

希望开发一个 AI Chat Assistant，用户通过聊天窗口提问，系统能够从 GitHub Repository 中检索相关 Markdown 文档内容，并基于检索结果结合大模型生成回答。

当前阶段仅实现 MVP（最小可行产品），不使用向量数据库，不使用 Embedding，不使用 RAG 向量检索。

目标是验证：

用户问题 -> 文档检索 -> AI 回答

这条链路是否可行。

---

## 技术栈要求

### 前端

- Vue3
- Element Plus
- Chat 风格界面

### 后端

- Python 3.12
- FastAPI

### AI框架

- LangChain

当前阶段不要使用：

- LangGraph
- DeepAgent

因为暂时不存在多Agent协作场景。

---

## 数据来源

GitHub Repository

示例：

docs/
├── gke.md
├── network.md
├── iam.md
├── terraform.md
└── troubleshooting.md

系统启动时读取本地同步的 Repo 内容。

支持递归扫描所有 Markdown 文件：

*.md

---

## Markdown 解析

使用 LangChain MarkdownHeaderTextSplitter。

要求：

- 保留 Markdown 标题层级
- 按章节进行切分
- 每个 Chunk 保留来源文件路径
- 每个 Chunk 保留标题信息

例如：

# GKE

## Create Cluster

内容...

切分后：

Chunk:
- file: gke.md
- section: GKE/Create Cluster
- content: 内容...

---

## 检索方案

禁止：

- Embedding
- PGVector
- Milvus
- Chroma
- FAISS

采用：

BM25

推荐库：

rank-bm25

流程：

用户问题
↓
BM25 检索
↓
返回 Top 5 Chunk
↓
送入 LLM

---

## LLM

使用 Google Vertex AI

模型：

Gemini 2.5 Flash

要求：

根据检索结果回答问题。

如果知识库中没有相关内容：

返回：

“当前知识库中未找到相关信息。”

禁止模型编造答案。

---

## Prompt 设计

System Prompt：

你是公司内部技术知识助手。

你只能根据提供的知识库内容回答问题。

如果知识库中没有答案，请明确说明未找到相关信息。

禁止编造、猜测或补充未经知识库验证的内容。

回答时优先引用知识库中的内容。

---

## API设计

POST /chat

Request：

{
  "question": "如何创建GKE集群？"
}

Response：

{
  "answer": "...",
  "sources": [
    {
      "file": "gke.md",
      "section": "Create Cluster"
    }
  ]
}

---

## 前端功能

聊天窗口

支持：

- 用户提问
- AI回答
- 显示引用来源

例如：

回答：

创建GKE集群需要先创建VPC网络...

来源：

gke.md
network.md

---

## 项目目录结构

backend/
├── app.py
├── loader.py
├── splitter.py
├── retriever.py
├── llm.py
├── prompt.py
└── models.py

frontend/
├── src
├── views
├── components
└── api

docs/
├── *.md

---

## MVP目标

实现以下能力：

1. 用户在聊天窗口输入问题
2. 系统检索 GitHub Repo 中 Markdown 内容
3. 返回最相关的 Top K 文档片段
4. 调用 Gemini 生成答案
5. 展示引用来源
6. 不使用任何向量数据库

---

## 后续扩展（非本期）

V2：
- GitHub Webhook 自动同步 Repo
- 文档增量更新

V3：
- Hybrid Search（BM25 + Embedding）

V4：
- LangGraph Agent
- Terraform JSON 自动生成
- 配置校验 Agent
- 人工审批流

