# AI 工程知识库产品详细设计方案

## 1. 背景与目标

本产品面向拥有私有 GitLab 仓库、线上服务器、运维日志和研发协作记录的技术团队。产品通过 Chat 形式与用户交互，但底层不是普通问答机器人，而是一个持续索引代码、理解系统结构、可受控连接运行环境、并能沉淀排障经验的 AI 工程知识平台。

产品核心目标：

1. 将私有 GitLab 仓库加工成 AI 易读、可检索、可增量更新的工程上下文，避免 AI 每次直接扫描原始代码。
2. 用户通过 Chat 提问、排查、总结和发起修复任务，降低使用门槛。
3. AI 可以在授权范围内连接服务器、日志系统、监控系统和发布系统，协助排查线上 bug。
4. 将用户沟通记录、排查过程、证据、根因、修复方法和后续 runbook 自动沉淀为可复用知识。
5. 对权限、安全、审计、生产环境操作进行严格控制，避免 AI 越权访问或执行高风险操作。

一句话定位：

> 一个连接私有 GitLab、运行环境和团队排障记录的 AI 工程知识平台。它持续把代码库加工成 agent 可读上下文，用户通过 Chat 查询、排障和修复问题，系统自动把沟通、证据、根因和修复方案沉淀为可复用知识。

## 2. 产品边界

### 2.1 要做什么

产品优先解决研发团队的工程知识和线上排障问题：

1. 私有代码库持续索引。
2. 代码结构、模块职责、调用关系、接口、配置、测试和 owner 识别。
3. Chat 式代码问答、系统理解和 bug 排查。
4. 服务器、Kubernetes、日志、指标、CI/CD、GitLab MR 的受控查询。
5. 排障记录和修复方法沉淀。
6. 可复用 runbook 生成和审核。
7. 文档和代码知识的新鲜度管理。
8. Agent 工具接口，例如 MCP、REST API、CLI。

### 2.2 暂时不做什么

第一阶段不建议做：

1. 通用个人笔记产品。
2. 替代 Notion、Confluence 的完整编辑器。
3. 大而全的企业搜索。
4. 无审核自动修改生产环境。
5. 无来源的 AI 结论。
6. 仅依赖向量库的简单 RAG 问答。
7. 面向所有行业的通用知识库。

### 2.3 核心差异化

相比普通知识库，本产品的差异化在于：

1. 代码仓库被持续加工成结构化工程上下文。
2. AI 默认读取索引、摘要、图谱和上下文包，而不是每次扫原始代码。
3. Chat 背后有工具调用、权限守卫和任务编排。
4. 可连接真实运行环境做只读诊断。
5. 排障过程自动沉淀成组织知识。
6. 知识与代码、部署、日志、MR、事故、owner 之间有关系图谱。

## 3. 典型用户和场景

### 3.1 目标用户

主要用户：

1. 后端工程师。
2. DevOps / SRE。
3. 技术负责人。
4. 平台工程团队。
5. 新加入项目的研发人员。
6. 客服技术支持或二线支持团队。

### 3.2 核心场景

#### 场景 A：理解代码库

用户：

```text
订单服务的支付流程是怎么走的？
```

系统行为：

1. 识别服务：order-service、payment-service。
2. 查询模块摘要、API route、调用关系、相关文档和历史 MR。
3. 返回流程说明、关键文件、关键函数、依赖服务和风险点。
4. 附上来源 commit、文件路径和相关 MR。

#### 场景 B：定位代码位置

用户：

```text
退款重试逻辑在哪里？失败后会不会重复退款？
```

系统行为：

1. 查询 symbol index 和 module summary。
2. 找到 RefundRetryJob、RefundService、RefundRepository。
3. 查询幂等逻辑和相关测试。
4. 如有必要读取少量真实源码片段。
5. 返回逻辑说明和准确来源。

#### 场景 C：线上 bug 排查

用户：

```text
生产环境订单支付失败率升高，帮我排查。
```

系统行为：

1. 识别服务和环境。
2. 请求确认排查范围和只读权限。
3. 查询最近部署、错误日志、指标、相关 MR、历史相似事故。
4. 输出证据、时间线、可能根因和建议修复步骤。
5. 自动生成 investigation record。

#### 场景 D：修复建议和 MR 草稿

用户：

```text
按你的建议生成修复方案，并开一个 MR 草稿。
```

系统行为：

1. 明确当前操作是代码变更，不再只依赖摘要。
2. checkout 指定分支或 commit。
3. 读取真实相关源码。
4. 修改代码、补充测试、运行验证。
5. 生成 MR 描述、风险说明和回滚建议。

#### 场景 E：知识沉淀

用户：

```text
把这次排查过程整理成 runbook。
```

系统行为：

1. 汇总对话、工具调用、日志证据、最终结论。
2. 生成结构化排障记录。
3. 生成 runbook 草稿。
4. 关联服务、错误码、文件、MR、owner。
5. 等待人工审核发布。

## 4. 总体架构

### 4.1 架构概览

```text
Private GitLab
  |
  | webhook / scheduled sync
  v
GitLab Connector
  |
  v
Code Intelligence Pipeline
  |
  +--> Metadata Store
  +--> Symbol Index
  +--> Search Index
  +--> Vector Index
  +--> Knowledge Graph
  +--> Generated Summaries

Runtime Systems
  |
  +--> SSH / Server Connector
  +--> Kubernetes Connector
  +--> Logs Connector
  +--> Metrics Connector
  +--> CI/CD Connector
  |
  v
Diagnostic Gateway

User
  |
  v
Chat UI
  |
  v
Chat Orchestrator
  |
  +--> Intent Router
  +--> Agent Planner
  +--> Context Engine
  +--> Tool Executor
  +--> Permission Guard
  +--> Memory Writer
  |
  v
Knowledge Capture Engine
  |
  +--> Investigation Records
  +--> Runbooks
  +--> Fix Records
  +--> Team Knowledge Notes
```

### 4.2 模块划分

| 模块 | 职责 |
| --- | --- |
| GitLab Connector | 同步仓库、分支、commit、MR、Issue、pipeline 和权限 |
| Code Intelligence Engine | 解析代码、提取符号、生成摘要、分析依赖和影响范围 |
| Context Engine | 根据任务组装 AI 可用上下文包 |
| Chat Orchestrator | 管理对话、识别意图、规划工具调用、返回结果 |
| Diagnostic Gateway | 连接服务器、日志、指标、K8s、发布系统 |
| Permission Guard | 做 repo、服务、环境、日志和工具级权限控制 |
| Tool Executor | 执行结构化工具调用并记录审计日志 |
| Knowledge Capture Engine | 从对话和排查过程生成结构化知识 |
| Review UI | 审核 runbook、知识更新和高风险操作 |
| Admin Console | 管理连接器、权限、工具策略和审计记录 |

## 5. GitLab 仓库预处理设计

### 5.1 为什么不让 AI 每次读原始代码

直接让 AI 每次读仓库原始代码会带来：

1. 大仓库读取慢。
2. token 成本高。
3. 容易漏掉关键上下文。
4. 无法稳定复用历史理解。
5. 难以实现权限过滤。
6. 无法追踪回答基于哪个 commit。
7. 代码更新后缺乏增量维护机制。

正确方式是将仓库加工成多层索引：

```text
原始代码
  -> 文件元数据
  -> 语法结构
  -> 符号索引
  -> 调用关系
  -> 模块摘要
  -> 服务画像
  -> 知识图谱
  -> AI 上下文包
```

### 5.2 GitLab 同步机制

同步来源：

1. Repository files。
2. Branches。
3. Commits。
4. Merge Requests。
5. Issues。
6. Tags / Releases。
7. Pipelines。
8. Code owners。
9. GitLab group / project 权限。

触发方式：

1. Webhook：push、merge request、pipeline、tag。
2. 定时同步：兜底同步，避免 webhook 丢失。
3. 手动触发：管理员或项目 owner 可强制重建。

同步状态：

```text
repo_id
branch
last_seen_commit
last_indexed_commit
sync_status
index_status
last_error
updated_at
```

### 5.3 增量索引流程

```text
GitLab push event
  |
  v
Fetch changed commits
  |
  v
Compute changed files
  |
  v
Classify changes
  |
  +--> code file
  +--> test file
  +--> config file
  +--> doc file
  +--> dependency file
  +--> CI/CD file
  |
  v
Re-parse affected files
  |
  v
Update symbol index
  |
  v
Update dependency graph
  |
  v
Update embeddings
  |
  v
Update module summaries
  |
  v
Mark affected knowledge freshness
```

增量策略：

1. 文件 hash 未变则跳过。
2. 小文件直接解析，大文件按语言和结构切分。
3. 只重新生成受影响模块摘要。
4. 只更新受影响 chunk 的 embedding。
5. 调用关系变化时更新 graph edge。
6. 对关联文档打上 freshness warning。

### 5.4 文件级索引

每个文件保存：

```text
repo_id
branch
commit
path
language
file_type
size
hash
last_modified_at
last_author
owners
imports
exports
defined_symbols
referenced_symbols
related_tests
related_docs
summary
```

文件类型建议：

```text
source_code
test_code
config
documentation
ci_cd
schema
migration
script
lockfile
binary
generated
```

### 5.5 符号级索引

符号包括：

1. class。
2. function。
3. method。
4. interface。
5. enum。
6. API route。
7. event handler。
8. database table。
9. migration。
10. config key。
11. cron job。
12. queue topic。

符号字段：

```text
symbol_id
repo_id
branch
commit
name
qualified_name
symbol_type
file_path
start_line
end_line
signature
docstring
visibility
calls
called_by
imports
exports
related_tests
related_configs
summary
```

示例：

```json
{
  "repo": "payment-service",
  "path": "src/refund/refund.service.ts",
  "symbol": "RefundService.retryRefund",
  "type": "method",
  "signature": "retryRefund(refundId: string): Promise<RefundResult>",
  "calls": [
    "RefundRepository.findById",
    "PaymentGatewayClient.refund",
    "RefundRepository.updateStatus"
  ],
  "related_tests": [
    "test/refund/refund.service.spec.ts"
  ],
  "summary": "对失败退款进行幂等重试，调用第三方支付网关并更新退款状态。"
}
```

### 5.6 模块摘要

模块摘要用于让 AI 快速理解目录或服务，不需要每次读取大量源码。

摘要粒度：

1. 目录级摘要。
2. 包级摘要。
3. 服务级摘要。
4. API 分组摘要。
5. 领域模块摘要。

摘要内容：

```text
模块职责
核心入口
关键类和函数
依赖模块
对外接口
数据表
配置项
消息队列
定时任务
相关测试
常见风险
最近重要变更
```

模块摘要示例：

```text
模块：src/refund/

职责：
处理退款创建、退款状态同步、退款重试和第三方网关结果落库。

核心入口：
- RefundController.createRefund
- RefundService.createRefund
- RefundRetryJob.handle

关键约束：
- 退款必须幂等，不能对同一 payment_id 重复发起实际退款。
- 第三方网关超时不代表退款失败，需要通过状态同步确认。
- 生产环境 timeout 配置来自 PAYMENT_GATEWAY_TIMEOUT_MS。

相关测试：
- test/refund/refund.service.spec.ts
- test/refund/refund-retry.job.spec.ts

风险：
- 修改 retryRefund 时必须检查幂等键。
- 修改网关 timeout 时需要关注历史事故 INC-2025-041。
```

### 5.7 服务画像

服务画像描述一个服务在运行系统中的位置。

字段：

```text
service_name
repo
language
framework
entry_points
runtime
deploy_target
owners
dependencies
upstream_services
downstream_services
databases
queues
external_apis
health_check
log_locations
metrics
dashboards
runbooks
recent_deployments
known_incidents
```

### 5.8 知识新鲜度

每条生成知识都需要记录来源和新鲜度。

新鲜度信号：

1. 相关代码文件发生变化。
2. 相关 symbol 签名发生变化。
3. 相关 API route 发生变化。
4. 相关配置发生变化。
5. 相关测试发生变化。
6. 相关服务近期发生事故。
7. 文档长时间未更新。
8. owner 变更。

新鲜度状态：

```text
fresh
possibly_stale
stale
needs_review
deprecated
```

当相关代码更新后，系统可以提示：

```text
docs/payment/refund.md 可能已过期。
原因：
- src/refund/refund.service.ts 在 commit a81f2c9 中修改。
- RefundService.retryRefund 签名发生变化。
- 文档最后更新时间为 2026-04-12。
建议：
- 生成文档更新草稿。
- 指派 payment-team 审核。
```

## 6. Context Engine 设计

### 6.1 职责

Context Engine 的目标是：根据用户当前任务，组装最小充分上下文，让 AI 不需要盲目读取所有代码。

输入：

```text
user_id
conversation_id
task
repo / service / environment
mentioned_files
mentioned_errors
time_range
permissions
```

输出：

```text
context_pack
sources
constraints
tool_suggestions
confidence
missing_information
```

### 6.2 检索顺序

建议检索顺序：

1. 解析用户意图和实体。
2. 查询权限范围。
3. 查服务画像。
4. 查模块摘要。
5. 查 symbol index。
6. 查知识图谱。
7. 查历史事故和 runbook。
8. 查向量索引。
9. 必要时读取少量真实源码。
10. 生成 context pack。

### 6.3 Context Pack 结构

```json
{
  "task": "排查生产环境退款失败",
  "scope": {
    "services": ["payment-service"],
    "environment": "prod",
    "time_range": "last_1h"
  },
  "relevant_code": [
    {
      "repo": "payment-service",
      "path": "src/refund/refund.service.ts",
      "symbols": ["RefundService.createRefund", "RefundService.retryRefund"],
      "reason": "退款失败核心逻辑"
    }
  ],
  "service_context": {
    "owners": ["payment-team"],
    "deploy_target": "kubernetes/prod",
    "logs": ["loki:payment-service"],
    "metrics": ["prometheus:payment_refund_errors_total"]
  },
  "constraints": [
    "生产环境默认只读排查",
    "退款逻辑必须保持幂等",
    "修改代码前必须读取真实源码"
  ],
  "history": [
    {
      "type": "incident",
      "id": "INC-2025-041",
      "summary": "支付网关 timeout 配置过低导致退款失败"
    }
  ],
  "sources": [
    {
      "type": "module_summary",
      "id": "summary-payment-refund",
      "commit": "a81f2c9"
    }
  ]
}
```

### 6.4 是否读取真实源码的策略

默认策略：

| 场景 | 是否读取真实源码 |
| --- | --- |
| 解释模块职责 | 不需要，优先用模块摘要和服务画像 |
| 查函数位置 | 不需要，优先用 symbol index |
| 查调用关系 | 不需要，优先用 graph |
| 分析 bug 根因 | 可能需要，读取相关少量源码 |
| 生成修复方案 | 需要，必须读取真实源码 |
| 修改代码 | 需要，必须 checkout 并读取真实文件 |
| 回答安全/资金/生产相关问题 | 需要更多证据，不能只用摘要 |

## 7. Chat 与 Agent 编排设计

### 7.1 Chat 不等于简单问答

用户通过 Chat 操作系统，但 Chat 背后需要有：

1. 意图识别。
2. 任务规划。
3. 上下文检索。
4. 工具调用。
5. 权限校验。
6. 证据收集。
7. 结果解释。
8. 知识沉淀。

### 7.2 主要意图类型

```text
code_question
architecture_question
bug_investigation
log_analysis
incident_summary
fix_suggestion
code_change_request
runbook_generation
knowledge_search
knowledge_update
permission_request
```

### 7.3 Agent 执行状态

Chat UI 应展示 agent 正在执行的步骤：

```text
正在识别相关服务...
正在查询 payment-service 服务画像...
正在读取最近部署记录...
正在查询生产错误日志...
正在检索历史相似事故...
正在生成根因假设...
正在整理排查记录...
```

### 7.4 工具调用模型

所有工具调用都应结构化，不建议直接暴露裸 shell。

工具示例：

```text
code.search(query, repo, filters)
code.get_file_summary(repo, path, commit)
code.get_symbol(repo, symbol, commit)
code.get_call_graph(repo, symbol, depth)
code.get_related_tests(repo, path_or_symbol)

gitlab.get_recent_mrs(repo, filters)
gitlab.get_mr_diff(repo, mr_id)
gitlab.get_pipeline_status(repo, ref)
gitlab.create_mr(repo, branch, title, description)

service.get_profile(service)
service.get_dependencies(service)
service.get_recent_deployments(service, environment)

logs.search(service, environment, query, time_range)
logs.count_errors(service, environment, filters, time_range)
logs.get_samples(service, environment, filters, time_range)

metrics.query(promql, time_range)
metrics.get_service_dashboard(service, environment)

k8s.get_pods(namespace, selector)
k8s.describe_pod(namespace, pod)
k8s.get_logs(namespace, pod, container, time_range)

server.get_status(host)
server.list_processes(host, filters)
server.tail_logs(host, path, time_range)

incident.search_similar(query, service, error_code)
knowledge.create_investigation(data)
knowledge.create_runbook_draft(data)
knowledge.link_entities(source, target, relation)
```

### 7.5 回答格式

高质量回答应包含：

1. 结论。
2. 证据。
3. 推理过程摘要。
4. 风险和不确定性。
5. 建议下一步。
6. 来源引用。
7. 是否已沉淀为知识。

示例：

```text
初步结论：
支付失败率升高大概率与 v1.38.2 中网关 timeout 配置变更有关。

关键证据：
1. 10:21 发布 payment-service v1.38.2。
2. 10:24 后 PAYMENT_GATEWAY_TIMEOUT 错误明显增加。
3. MR !428 将 PAYMENT_GATEWAY_TIMEOUT_MS 从 5000 改为 1000。
4. 历史事故 INC-2025-041 有相似模式。

建议：
1. 短期回滚 timeout 配置。
2. 补充慢网关响应测试。
3. 更新 payment gateway timeout runbook。

不确定性：
尚未确认第三方支付网关本身是否在同一时间段发生异常。
```

## 8. 服务器排障设计

### 8.1 基本原则

1. 默认只读。
2. 所有操作必须经过权限校验。
3. 所有工具调用必须记录审计。
4. 生产环境写操作必须人工审批。
5. 尽量使用结构化工具，不暴露裸 shell。
6. 日志和命令输出需要脱敏。
7. AI 结论必须保留证据链。

### 8.2 连接方式

推荐支持：

1. SSH 只读账号。
2. Kubernetes API。
3. Docker API。
4. 日志平台，例如 ELK、OpenSearch、Loki。
5. 指标平台，例如 Prometheus、Grafana、Datadog。
6. Sentry 或错误追踪系统。
7. GitLab CI/CD。
8. 发布平台。
9. 数据库只读账号。

MVP 优先级：

1. GitLab。
2. Kubernetes logs 或 SSH 日志查询，二选一。
3. Prometheus 或现有日志平台，二选一。
4. GitLab pipeline 和 MR 查询。

### 8.3 命令白名单

如果必须使用 SSH，建议只允许白名单命令。

允许：

```bash
pwd
ls
cat
tail
grep
rg
journalctl
docker ps
docker logs
kubectl get
kubectl describe
kubectl logs
systemctl status
df -h
free -m
ps
top
curl health endpoint
```

禁止：

```bash
rm
mv
cp 到系统目录
chmod
chown
kill
pkill
systemctl restart
systemctl stop
docker stop
docker rm
kubectl delete
kubectl apply
kubectl edit
数据库 update/delete/drop
写配置文件
执行未知脚本
```

### 8.4 排障标准流程

用户提出 bug 后，agent 应遵循固定流程：

```text
1. 明确问题描述
2. 明确环境和时间范围
3. 识别相关服务
4. 查询服务画像
5. 查询最近部署
6. 查询错误日志
7. 查询关键指标
8. 查询依赖服务状态
9. 查询相关 MR / commit
10. 查询历史相似事故
11. 形成根因假设
12. 验证或排除假设
13. 给出修复建议
14. 生成排查记录
15. 生成 runbook 草稿
```

### 8.5 排障中的证据链

每个结论应能追溯到证据：

```text
结论：payment-service timeout 配置过低导致失败率升高。

证据：
- Deployment: payment-service v1.38.2 at 2026-06-09 10:21
- Log: PAYMENT_GATEWAY_TIMEOUT spike from 10:24
- MR: !428 changed timeout from 5000ms to 1000ms
- Metric: gateway latency p95 around 1800ms
- Similar incident: INC-2025-041
```

### 8.6 高风险操作审批

操作分级：

| 等级 | 示例 | 策略 |
| --- | --- | --- |
| L0 | 查询知识库、查索引 | 自动执行 |
| L1 | 查日志、查指标、查服务状态 | 自动执行，记录审计 |
| L2 | 读取生产日志、查询敏感配置 | 需要权限，可能需要二次确认 |
| L3 | 创建 MR、生成配置变更 | 用户确认 |
| L4 | 回滚、重启、修改生产配置 | 审批流 |
| L5 | 删除数据、执行破坏性命令 | 默认禁止 |

## 9. 知识沉淀设计

### 9.1 沉淀对象

系统应沉淀以下知识：

1. 用户问题。
2. 对话摘要。
3. 排查时间线。
4. 工具调用记录。
5. 日志证据。
6. 指标证据。
7. 相关代码和 MR。
8. 根因假设。
9. 最终根因。
10. 修复方法。
11. 验证结果。
12. 后续预防措施。
13. runbook 草稿。
14. FAQ。

### 9.2 Investigation Record

排查记录是核心知识资产。

字段：

```text
id
title
status
created_by
created_at
updated_at
service
environment
time_range
severity
symptoms
user_question
conversation_id
timeline
tools_used
evidence
hypotheses
root_cause
fix_summary
verification
related_repos
related_files
related_symbols
related_mrs
related_deployments
related_incidents
owner
review_status
runbook_id
```

示例：

```json
{
  "type": "bug_investigation",
  "title": "payment-service gateway timeout spike",
  "service": "payment-service",
  "environment": "prod",
  "time_range": {
    "from": "2026-06-09T10:20:00+08:00",
    "to": "2026-06-09T10:50:00+08:00"
  },
  "symptoms": [
    "支付失败率升高",
    "PAYMENT_GATEWAY_TIMEOUT 错误增加"
  ],
  "evidence": [
    {
      "source": "deployment",
      "summary": "payment-service v1.38.2 在 10:21 发布"
    },
    {
      "source": "logs",
      "summary": "10:24 后 timeout 错误增加"
    },
    {
      "source": "gitlab_mr",
      "summary": "MR !428 修改 timeout 配置"
    }
  ],
  "root_cause": "网关 timeout 配置过低，遇到第三方延迟升高时触发大量超时。",
  "fix": "将 timeout 恢复到 5000ms，补充慢响应测试。",
  "related_files": [
    "src/payment/PaymentGatewayClient.ts",
    "config/prod.yml"
  ],
  "related_mrs": ["!428", "!431"],
  "runbook_candidate": true
}
```

### 9.3 Runbook 生成

Runbook 草稿结构：

```text
# 问题名称

## 适用场景

## 典型症状

## 影响范围

## 快速判断

## 排查步骤

## 常见根因

## 修复方法

## 验证方式

## 回滚方案

## 相关服务

## 相关日志查询

## 相关指标

## 历史案例

## Owner
```

Runbook 必须经过人工审核后才成为正式知识。

### 9.4 从对话到知识的流程

```text
Conversation
  |
  v
Conversation Summarizer
  |
  v
Extract entities
  |
  +--> service
  +--> environment
  +--> error_code
  +--> file
  +--> MR
  +--> command
  +--> evidence
  |
  v
Generate Investigation Record
  |
  v
Generate Runbook Draft
  |
  v
Human Review
  |
  v
Publish Knowledge
  |
  v
Update Knowledge Graph
```

### 9.5 知识状态

```text
draft
needs_review
approved
published
stale
deprecated
rejected
```

## 10. 数据模型设计

### 10.1 核心实体

```text
User
Team
Permission
Repo
Branch
Commit
MergeRequest
Issue
File
Symbol
Module
Service
Environment
Deployment
Server
LogSource
MetricSource
ToolCall
Conversation
Message
Investigation
Evidence
Hypothesis
Fix
Runbook
KnowledgeNote
AuditLog
```

### 10.2 关键关系

```text
Team owns Service
Repo contains File
File defines Symbol
Symbol calls Symbol
Module contains Symbol
Service implemented_by Repo
Service deployed_to Environment
Deployment uses Commit
MergeRequest contains Commit
Commit changes File
Investigation affects Service
Investigation references Evidence
Investigation produces Fix
Investigation creates Runbook
Conversation linked_to Investigation
Runbook applies_to Service
KnowledgeNote derived_from Conversation
ToolCall belongs_to Conversation
AuditLog records ToolCall
```

### 10.3 存储建议

MVP 简化方案：

```text
PostgreSQL
- 用户、团队、权限
- repo、commit、file、symbol
- conversation、message
- investigation、runbook
- audit log

pgvector
- 文档 chunk embedding
- 代码摘要 embedding
- investigation embedding
- runbook embedding

Object Storage
- 仓库快照
- 日志样本
- 工具输出附件

Redis / Queue
- 索引任务
- 摘要任务
- webhook 任务
```

后续增强：

```text
Search Engine
- OpenSearch / Elasticsearch 做关键词、日志和代码搜索

Graph DB
- Neo4j / Memgraph / PostgreSQL graph schema 做复杂依赖关系

Data Warehouse
- 分析 agent 使用、知识命中率、故障趋势
```

## 11. 权限与安全设计

### 11.1 权限原则

1. 用户只能检索自己有权限访问的仓库、服务、日志和环境。
2. 权限过滤必须发生在检索阶段，而不是生成答案后。
3. AI 工具调用必须绑定用户身份。
4. 生产环境默认只读。
5. 写操作必须有审批和审计。
6. 敏感数据需要脱敏。
7. 所有回答需要可追溯来源。

### 11.2 权限维度

```text
repo_permission
branch_permission
service_permission
environment_permission
server_permission
log_permission
metric_permission
tool_permission
knowledge_permission
write_permission
approval_permission
```

### 11.3 权限继承

优先继承源系统权限：

1. GitLab group / project 权限。
2. GitLab protected branch 权限。
3. Kubernetes namespace 权限。
4. 日志系统权限。
5. 监控系统权限。
6. 内部 SSO / IAM 权限。

### 11.4 审计日志

每次工具调用记录：

```text
tool_call_id
conversation_id
user_id
tool_name
arguments
target_resource
permission_result
started_at
finished_at
status
output_summary
output_reference
risk_level
approval_id
```

### 11.5 敏感信息处理

需要识别并脱敏：

1. API key。
2. token。
3. password。
4. private key。
5. cookie。
6. 用户手机号、邮箱、身份证等 PII。
7. 客户订单敏感信息。
8. 数据库连接串。

日志和代码进入模型上下文前必须经过脱敏管道。

### 11.6 Prompt Injection 防护

风险来源：

1. Issue 内容。
2. MR 描述。
3. 文档内容。
4. 日志内容。
5. 网页或第三方系统返回。

防护策略：

1. 把外部内容标记为 untrusted content。
2. 工具调用必须由系统策略控制，不能由文档内容诱导。
3. 不允许外部内容修改系统指令。
4. 对高风险操作强制人工确认。
5. 对工具参数做 schema 校验。

## 12. UI 设计

### 12.1 信息架构

左侧导航：

```text
Chats
Investigations
Services
Repos
Runbooks
Knowledge
Reviews
Admin
```

中间主区域：

```text
Chat 对话
Agent 执行步骤
结论和建议
引用来源
待确认操作
```

右侧上下文面板：

```text
当前服务
当前环境
相关代码
相关日志
相关指标
最近部署
历史相似事故
执行过的工具
权限状态
```

### 12.2 Chat 页面

Chat 页面需要支持：

1. 多轮对话。
2. 任务状态展示。
3. 工具调用展开。
4. 来源引用。
5. 操作确认按钮。
6. 一键生成 investigation。
7. 一键生成 runbook。
8. 关联服务和 repo。

### 12.3 Investigation 页面

展示：

1. 问题标题。
2. 状态。
3. 时间线。
4. 影响服务。
5. 环境。
6. 证据列表。
7. 根因假设。
8. 最终根因。
9. 修复步骤。
10. 相关 MR。
11. 相关 runbook。
12. 审核状态。

### 12.4 Service 页面

展示：

1. 服务画像。
2. 代码仓库。
3. owner。
4. 依赖服务。
5. API 列表。
6. 数据库和队列。
7. 部署环境。
8. 日志入口。
9. 指标和 dashboard。
10. 历史事故。
11. runbook。
12. 最近变更。

### 12.5 Review 页面

用于审核：

1. Runbook 草稿。
2. 知识更新。
3. AI 生成模块摘要。
4. 高风险工具调用。
5. 文档过期提示。

## 13. Agent 工具接口设计

### 13.1 为什么需要 MCP / API / CLI

产品自身有 Chat UI，但也应该被外部 agent 使用，例如 Codex、Claude Code、内部自动化 agent。建议提供三类接口：

1. MCP Server：给 AI agent 标准化调用。
2. REST API：给内部系统集成。
3. CLI：给开发者和 coding agent 在本地或 CI 中调用。

### 13.2 MCP 工具建议

```text
kb.search
kb.get_context
kb.get_service_profile
kb.get_module_summary
kb.get_symbol
kb.find_owner
kb.find_runbook
kb.search_investigations
kb.create_investigation
kb.create_runbook_draft
kb.check_freshness

code.search
code.get_related_files
code.get_call_graph
code.get_related_tests

ops.search_logs
ops.query_metrics
ops.get_recent_deployments
ops.get_service_status
```

### 13.3 CLI 示例

```bash
kb context "排查 payment-service 退款失败"
kb service payment-service
kb code symbol RefundService.retryRefund
kb stale-docs --repo payment-service --diff origin/main
kb incident search "PAYMENT_GATEWAY_TIMEOUT"
kb runbook draft --from-investigation INC-2026-006
```

## 14. MVP 设计

### 14.1 MVP 目标

用最小版本验证：

1. 私有 GitLab 仓库能否被持续加工成可用 AI 上下文。
2. Chat 是否能显著提升代码理解和排障效率。
3. 只读服务器/日志连接是否能支撑真实 bug 排查。
4. 排查记录沉淀是否能在后续问题中复用。

### 14.2 MVP 范围

第一版建议只做：

1. GitLab group 接入。
2. 仓库文件、commit、MR 同步。
3. 文件级索引。
4. 简单 symbol 提取。
5. 模块摘要生成。
6. pgvector 语义检索。
7. Chat 问答。
8. 只读日志查询。
9. 最近部署/MR 查询。
10. Investigation record 生成。
11. Runbook 草稿生成。
12. 基础权限和审计。

### 14.3 MVP 不做

1. 自动生产修复。
2. 复杂图数据库。
3. 全语言深度 AST。
4. 完整企业搜索。
5. 多租户复杂计费。
6. 全量可视化架构图。
7. 自动发布 runbook。

### 14.4 MVP 技术选型建议

```text
Frontend:
- React / Next.js
- Chat UI + Investigation UI + Admin UI

Backend:
- Node.js / Python / Go 均可
- REST API
- Worker queue

Storage:
- PostgreSQL
- pgvector
- Redis
- Object Storage

Connectors:
- GitLab API
- GitLab Webhook
- Kubernetes logs 或 SSH read-only
- Prometheus 或现有日志系统

AI Layer:
- 模型供应商可插拔
- Tool calling
- Context pack builder
- Prompt templates
```

### 14.5 90 天路线图

#### 第 1-2 周：需求验证和原型

1. 访谈 5-10 个研发团队。
2. 选择一个真实 GitLab group。
3. 明确最常见的 3 类 bug。
4. 定义权限边界。
5. 设计数据模型和工具列表。

#### 第 3-5 周：代码索引

1. GitLab connector。
2. 仓库同步。
3. 文件索引。
4. symbol 提取。
5. 模块摘要。
6. 语义检索。

#### 第 6-8 周：Chat 和排障

1. Chat UI。
2. Intent router。
3. Context engine。
4. 日志查询工具。
5. 最近部署和 MR 查询。
6. 排障工作流。

#### 第 9-10 周：知识沉淀

1. Investigation record。
2. Runbook draft。
3. 历史排查检索。
4. Review UI。

#### 第 11-12 周：安全和试点

1. 权限继承。
2. 审计日志。
3. 敏感信息脱敏。
4. 试点接入。
5. 指标收集。

## 15. 质量指标

### 15.1 产品指标

```text
代码问题回答命中率
回答引用覆盖率
排障平均耗时降低比例
历史事故复用次数
runbook 草稿采纳率
用户追问次数
知识过期发现数
MR / commit 关联准确率
```

### 15.2 技术指标

```text
仓库索引延迟
增量索引耗时
检索 P95 延迟
Chat 首 token 延迟
工具调用成功率
权限过滤准确率
日志脱敏召回率
摘要过期检测准确率
```

### 15.3 安全指标

```text
越权访问次数
高风险命令拦截次数
未审批写操作次数
敏感信息泄露次数
审计日志覆盖率
```

## 16. 风险与应对

### 16.1 代码摘要不准确

风险：

AI 基于过期或错误摘要回答，导致误导用户。

应对：

1. 摘要绑定 commit 和来源文件。
2. 高风险问题必须读取真实源码。
3. 摘要有 freshness 状态。
4. 用户可以反馈摘要错误。
5. 重要摘要需要 owner 审核。

### 16.2 权限泄露

风险：

用户通过 Chat 得到无权访问的代码、日志或事故信息。

应对：

1. 检索前做权限过滤。
2. 工具调用绑定用户身份。
3. 继承 GitLab 和运行环境权限。
4. 所有输出引用做权限校验。
5. 审计所有访问。

### 16.3 AI 执行危险操作

风险：

AI 在服务器上执行破坏性命令。

应对：

1. 默认只读。
2. 使用结构化工具替代裸 shell。
3. 命令白名单。
4. 高风险操作审批。
5. 生产写操作默认禁止。

### 16.4 排障结论不可靠

风险：

AI 给出未经验证的根因。

应对：

1. 要求输出证据链。
2. 明确区分事实、假设和建议。
3. 支持假设验证步骤。
4. 保留不确定性。
5. 人工确认最终 root cause。

### 16.5 索引成本过高

风险：

大仓库、多仓库导致索引成本和延迟过高。

应对：

1. 增量索引。
2. 文件 hash 跳过。
3. 按语言和文件类型过滤。
4. 对 generated、vendor、lockfile 做排除。
5. 摘要按模块而不是全文件频繁生成。

## 17. 推荐下一步

建议下一步先产出四份更具体的子文档：

1. `code-index-schema.md`：代码索引和数据表设计。
2. `agent-tools-spec.md`：Chat 背后的工具协议和权限等级。
3. `bug-investigation-workflow.md`：线上排障标准流程和记录结构。
4. `mvp-roadmap.md`：第一版迭代拆分、工期和验收标准。

如果只做一个工程原型，优先顺序应为：

```text
GitLab connector
  -> 文件和 symbol 索引
  -> 模块摘要
  -> Chat 查询
  -> 日志查询工具
  -> 排障记录沉淀
```

这条路径能最快验证产品核心假设：AI 不直接扫原始代码，也能基于持续索引的工程上下文完成有用的代码理解和线上 bug 排查。
