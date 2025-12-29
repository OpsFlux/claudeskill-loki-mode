---
name: loki-mode
description: Claude Code 的多代理自主创业系统。在 "Loki Mode" 时触发。协调 100 多个专业代理跨工程、QA、DevOps、安全、数据/ML、业务运营、营销、人力资源和客户成功领域。将 PRD 转化为完全部署的、产生收入的产品，无需人工干预。功能包括用于子代理调度的 Task 工具、3 个专业审查员的并行代码审查、基于严重性的问题分类、带有死信处理的分布式任务队列、自动部署到云提供商、A/B 测试、客户反馈循环、事件响应、断路器和自愈。通过分布式状态检查点和带指数退避的自动恢复来处理速率限制。需要 --dangerously-skip-permissions 标志。
---

# Loki Mode - 多代理自主创业系统

## 先决条件

```bash
# 验证 Claude Code 是否已安装
which claude || echo "请先安装 Claude Code"

# 使用自主权限启动
claude --dangerously-skip-permissions

# 启动时验证权限（编排器会检查此项）
# 如果出现权限拒绝错误，系统将停止并显示清晰消息
```

## 技能元数据

| 字段 | 值 |
|-------|-------|
| **触发器** | "Loki Mode" 或 "Loki Mode with PRD at [path]" |
| **跳过时机** | 任务之间需要人工批准、想要先审查计划、单一小任务 |
| **顺序执行于** | writing-plans, pre-dev-task-breakdown |
| **相关技能** | subagent-driven-development, executing-plans |
| **使用的技能** | test-driven-development, requesting-code-review |

## 架构概览

```
                              ┌─────────────────────┐
                              │   ORCHESTRATOR      │
                              │   (Primary Agent)   │
                              └──────────┬──────────┘
                                         │
      ┌──────────────┬──────────────┬────┴────┬──────────────┬──────────────┐
      │              │              │         │              │              │
 ┌────▼────┐   ┌─────▼─────┐  ┌─────▼─────┐ ┌─▼───┐   ┌──────▼──────┐ ┌─────▼─────┐
 │ENGINEERING│  │ OPERATIONS│  │  BUSINESS │ │DATA │   │   PRODUCT   │ │  GROWTH   │
 │  SWARM   │  │   SWARM   │  │   SWARM   │ │SWARM│   │    SWARM    │ │   SWARM   │
 └────┬────┘   └─────┬─────┘  └─────┬─────┘ └──┬──┘   └──────┬──────┘ └─────┬─────┘
      │              │              │          │             │              │
 ┌────┴────┐   ┌─────┴─────┐  ┌─────┴─────┐ ┌──┴──┐   ┌──────┴──────┐ ┌─────┴─────┐
 │Frontend │   │  DevOps   │  │ Marketing │ │ ML  │   │     PM      │ │  Growth   │
 │Backend  │   │  SRE      │  │  Sales    │ │Data │   │  Designer   │ │  Partner  │
 │Database │   │  Security │  │  Finance  │ │  Eng│   │  TechWriter │ │  Success  │
 │Mobile   │   │  Monitor  │  │  Legal    │ │Pipe │   │   i18n      │ │  Community│
 │API      │   │  Incident │  │  HR       │ │line │   │             │ │           │
 │QA       │   │  Release  │  │  Support  │ └─────┘   └─────────────┘ └───────────┘
 │Perf     │   │  Cost     │  │  Investor │
 └─────────┘   │  Compliance│  └───────────┘
               └───────────┘
```

## 关键：代理执行模型

**Claude Code 不支持后台进程。** 代理按顺序执行：

```
ORCHESTRATOR 作为主 Claude Code 会话执行
    │
    ├─► 编排器临时变为每个代理角色
    │   （通过角色提示注入进行上下文切换）
    │
    ├─► OR 为并行工作生成新的 Claude Code 会话：
    │   claude -p "$(cat .loki/prompts/agent-role.md)" --dangerously-skip-permissions
    │   （阻塞直到完成，捕获输出）
    │
    └─► 实现真正的并行性：使用 tmux/screen 会话
        tmux new-session -d -s agent-001 'claude --dangerously-skip-permissions -p "..."'
```

### 并行策略
```bash
# 选项 1：顺序执行（简单、可靠）
for agent in frontend backend database; do
  claude -p "作为 $agent 代理执行..." --dangerously-skip-permissions
done

# 选项 2：通过 tmux 并行（复杂、更快）
tmux new-session -d -s loki-pool
for i in {1..5}; do
  tmux new-window -t loki-pool -n "agent-$i" \
    "claude --dangerously-skip-permissions -p '$(cat .loki/prompts/agent-$i.md)'"
done

# 选项 3：角色切换（推荐）
# 编排器维护代理队列，按任务切换角色
```

## 目录结构

```
.loki/
├── state/
│   ├── orchestrator.json       # 主状态
│   ├── agents/                  # 每个代理的状态文件
│   ├── checkpoints/             # 恢复快照（每小时）
│   └── locks/                   # 基于文件的互斥锁
├── queue/
│   ├── pending.json             # 任务队列
│   ├── in-progress.json         # 活动任务
│   ├── completed.json           # 已完成任务
│   ├── failed.json              # 失败任务（用于重试）
│   └── dead-letter.json         # 永久失败（人工审查）
├── messages/
│   ├── inbox/                   # 每个代理的收件箱
│   ├── outbox/                  # 传出消息
│   └── broadcast/               # 系统范围的公告
├── logs/
│   ├── LOKI-LOG.md             # 主审计日志
│   ├── agents/                  # 每个代理的日志
│   ├── decisions/               # 决策审计跟踪
│   └── archive/                 # 轮换的日志（每日）
├── config/
│   ├── agents.yaml              # 代理池配置
│   ├── infrastructure.yaml      # 云/部署配置
│   ├── thresholds.yaml          # 质量门控、扩展规则
│   ├── circuit-breakers.yaml    # 故障阈值
│   └── secrets.env.enc          # 加密密钥引用
├── prompts/
│   ├── orchestrator.md          # 编排器系统提示
│   ├── eng-frontend.md          # 每个代理的角色提示
│   ├── eng-backend.md
│   └── ...
├── artifacts/
│   ├── releases/                # 版本化发布
│   ├── reports/                 # 生成的报告
│   ├── metrics/                 # 性能数据
│   └── backups/                 # 状态备份
└── scripts/
    ├── bootstrap.sh             # 初始化 .loki 结构
    ├── spawn-agent.sh           # 代理生成助手
    ├── backup-state.sh          # 备份自动化
    ├── rotate-logs.sh           # 日志轮转
    └── health-check.sh          # 系统健康验证
```

## 引导脚本

首次运行时，编排器执行：
```bash
#!/bin/bash
# .loki/scripts/bootstrap.sh

set -euo pipefail

LOKI_ROOT=".loki"

# 创建目录结构
mkdir -p "$LOKI_ROOT"/{state/{agents,checkpoints,locks},queue,messages/{inbox,outbox,broadcast},logs/{agents,decisions,archive},config,prompts,artifacts/{releases,reports,metrics,backups},scripts}

# 初始化队列文件
for f in pending in-progress completed failed dead-letter; do
  echo '{"tasks":[]}' > "$LOKI_ROOT/queue/$f.json"
done

# 初始化编排器状态
cat > "$LOKI_ROOT/state/orchestrator.json" << 'EOF'
{
  "version": "2.1.0",
  "startupId": "",
  "phase": "bootstrap",
  "prdPath": "",
  "prdHash": "",
  "agents": {"active":[],"idle":[],"failed":[],"totalSpawned":0},
  "metrics": {"tasksCompleted":0,"tasksFailed":0,"deployments":0},
  "circuitBreakers": {},
  "lastCheckpoint": "",
  "lastBackup": "",
  "currentRelease": "0.0.0"
}
EOF

# 设置启动 ID（macOS 兼容）
if command -v uuidgen &> /dev/null; then
  STARTUP_ID=$(uuidgen)
else
  STARTUP_ID=$(cat /proc/sys/kernel/random/uuid 2>/dev/null || echo "$(date +%s)-$$")
fi

if [[ "$OSTYPE" == "darwin"* ]]; then
  sed -i '' "s/\"startupId\": \"\"/\"startupId\": \"$STARTUP_ID\"/" "$LOKI_ROOT/state/orchestrator.json"
else
  sed -i "s/\"startupId\": \"\"/\"startupId\": \"$STARTUP_ID\"/" "$LOKI_ROOT/state/orchestrator.json"
fi

echo "引导完成：$LOKI_ROOT 已初始化"
```

## 状态模式

### `.loki/state/orchestrator.json`
```json
{
  "version": "2.1.0",
  "startupId": "uuid",
  "phase": "string",
  "subPhase": "string",
  "prdPath": "string",
  "prdHash": "md5",
  "prdLastModified": "ISO-timestamp",
  "agents": {
    "active": [{"id":"eng-backend-01","role":"eng-backend","taskId":"uuid","startedAt":"ISO"}],
    "idle": [],
    "failed": [{"id":"eng-frontend-02","role":"eng-frontend","failureCount":3,"lastError":"string"}],
    "totalSpawned": 0,
    "totalTerminated": 0
  },
  "circuitBreakers": {
    "eng-frontend": {"state":"closed","failures":0,"lastFailure":null,"cooldownUntil":null},
    "external-api": {"state":"open","failures":5,"lastFailure":"ISO","cooldownUntil":"ISO"}
  },
  "metrics": {
    "tasksCompleted": 0,
    "tasksFailed": 0,
    "tasksInDeadLetter": 0,
    "deployments": 0,
    "rollbacks": 0,
    "incidentsDetected": 0,
    "incidentsResolved": 0,
    "revenue": 0,
    "customers": 0,
    "agentComputeMinutes": 0
  },
  "lastCheckpoint": "ISO-timestamp",
  "lastBackup": "ISO-timestamp",
  "lastLogRotation": "ISO-timestamp",
  "currentRelease": "semver",
  "systemHealth": "green|yellow|red",
  "pausedAt": null,
  "pauseReason": null
}
```

### 代理状态模式（`.loki/state/agents/[id].json`）
```json
{
  "id": "eng-backend-01",
  "role": "eng-backend",
  "status": "active|idle|failed|terminated",
  "currentTask": "task-uuid|null",
  "tasksCompleted": 12,
  "tasksFailed": 1,
  "consecutiveFailures": 0,
  "lastHeartbeat": "ISO-timestamp",
  "lastTaskCompleted": "ISO-timestamp",
  "idleSince": "ISO-timestamp|null",
  "errorLog": ["error1", "error2"],
  "resourceUsage": {
    "tokensUsed": 50000,
    "apiCalls": 25
  }
}
```

### 断路器状态
```
CLOSED (正常) ──► failures++ ──► 达到阈值 ──► OPEN (阻塞)
                                                              │
                                                         冷却过期
                                                              │
                                                              ▼
                                                        HALF-OPEN (测试)
                                                              │
                                          成功 ◄───────────┴───────────► 失败
                                             │                                  │
                                             ▼                                  ▼
                                          CLOSED                              OPEN
```

**断路器配置（`.loki/config/circuit-breakers.yaml`）：**
```yaml
defaults:
  failureThreshold: 5
  cooldownSeconds: 300
  halfOpenRequests: 3

overrides:
  external-api:
    failureThreshold: 3
    cooldownSeconds: 600
  eng-frontend:
    failureThreshold: 10
    cooldownSeconds: 180
```

## 通过 Task 工具生成代理

### 主要方法：Claude Task 工具（推荐）
```markdown
使用 Task 工具调度子代理。每个任务获得新的上下文（无污染）。

**调度实现子代理：**
[Task tool call]
- description: "从计划实现 [任务名称]"
- instructions: |
    1. 从 .loki/queue/in-progress.json 读取任务需求
    2. 遵循 TDD 实施（先测试，后编码）
    3. 验证所有测试通过
    4. 使用常规提交消息提交
    5. 报告：WHAT_WAS_IMPLEMENTED, FILES_CHANGED, TEST_RESULTS
- model: "sonnet"（快速实现）
- working_directory: [项目根目录]
```

### 并行代码审查（3 个审查员同时工作）

**关键：在单个消息中使用 3 个 Task 工具调用调度所有 3 个审查员。**

```markdown
[Task tool call 1: code-reviewer]
- description: "[任务] 的代码质量审查"
- instructions: 审查代码质量、模式、可维护性
- model: "opus"（深度分析）
- context: WHAT_WAS_IMPLEMENTED, BASE_SHA, HEAD_SHA

[Task tool call 2: business-logic-reviewer]
- description: "[任务] 的业务逻辑审查"
- instructions: 审查正确性、边缘情况、需求一致性
- model: "opus"
- context: WHAT_WAS_IMPLEMENTED, REQUIREMENTS, BASE_SHA, HEAD_SHA

[Task tool call 3: security-reviewer]
- description: "[任务] 的安全审查"
- instructions: 审查漏洞、认证问题、数据暴露
- model: "opus"
- context: WHAT_WAS_IMPLEMENTED, BASE_SHA, HEAD_SHA
```

**每个审查员返回：**
```json
{
  "strengths": ["列出优点"],
  "issues": [
    {"severity": "Critical|High|Medium|Low|Cosmetic", "description": "...", "location": "file:line"}
  ],
  "assessment": "PASS|FAIL"
}
```

### 基于严重性的问题处理

| 严重性 | 操作 | 跟踪 |
|----------|--------|----------|
| **Critical** | 阻塞。立即调度修复子代理。重新运行所有 3 个审查员。 | 无（必须修复） |
| **High** | 阻塞。调度修复子代理。重新运行所有 3 个审查员。 | 无（必须修复） |
| **Medium** | 阻塞。调度修复子代理。重新运行所有 3 个审查员。 | 无（必须修复） |
| **Low** | 通过。添加 TODO 注释，提交，继续。 | `# TODO(review): [问题] - [审查员], [日期], 严重性: Low` |
| **Cosmetic** | 通过。添加 FIXME 注释，提交，继续。 | `# FIXME(nitpick): [问题] - [审查员], [日期], 严重性: Cosmetic` |

### 重新审查循环
```
实施 → 审查（3 个并行）→ 汇总
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
        Critical/High/Medium?                           全部通过？
              │                                               │
              ▼                                               ▼
    调度修复子代理                              标记完成
              │                                        添加 TODO/FIXME
              ▼                                        下一个任务
    重新运行所有 3 个审查员 ─────────────────────────────────┘
              │
              └──► 循环直到全部通过
```

### 上下文污染预防
**每个子代理获得新的上下文。绝不：**
- 尝试在编排器上下文中修复（改为调度修复子代理）
- 在子代理调用之间携带状态
- 在同一个子代理中混合实施和审查

### 替代生成方法

**方法 2：顺序子进程（用于没有 Task 工具的环境）**
```bash
claude --dangerously-skip-permissions \
  -p "$(cat .loki/prompts/eng-backend.md)" \
  --output-format json \
  > .loki/messages/outbox/eng-backend-01/result.json
```

**方法 3：通过 tmux 并行（高级，用于真正的并行性）**
```bash
#!/bin/bash
# 并行生成 3 个审查员
tmux new-session -d -s reviewers
tmux new-window -t reviewers -n code "claude -p '$(cat .loki/prompts/code-reviewer.md)' --dangerously-skip-permissions"
tmux new-window -t reviewers -n business "claude -p '$(cat .loki/prompts/business-reviewer.md)' --dangerously-skip-permissions"
tmux new-window -t reviewers -n security "claude -p '$(cat .loki/prompts/security-reviewer.md)' --dangerously-skip-permissions"
# 等待所有完成
```

### 按任务类型选择模型

| 任务类型 | 模型 | 理由 |
|-----------|-------|-----------|
| 实施 | sonnet | 快速，对编码足够好 |
| 代码审查 | opus | 深度分析，捕获细微问题 |
| 安全审查 | opus | 关键，需要彻底性 |
| 业务逻辑审查 | opus | 需要深入理解需求 |
| 文档 | sonnet | 直接的写作 |
| 快速修复 | haiku | 快速迭代 |

### 代理生命周期
```
生成 → 初始化 → 轮询队列 → 声明任务 → 执行 → 报告 → 轮询队列
           │              │                        │          │
           │         断路器打开？               超时？    成功？
           │              │                        │          │
           ▼              ▼                        ▼          ▼
     创建状态    等待退避                  释放    更新状态
                          │                    + 重试         │
                     指数退避                              │
                                                          ▼
                                                    无任务 ──► 空闲（5分钟）
                                                                    │
                                                             空闲 > 30分钟？
                                                                    │
                                                                    ▼
                                                               终止
```

### 动态扩展规则
| 条件 | 操作 | 冷却时间 |
|-----------|--------|----------|
| 队列深度 > 20 | 生成 2 个瓶颈类型的代理 | 5分钟 |
| 队列深度 > 50 | 生成 5 个代理，警告编排器 | 2分钟 |
| 代理空闲 > 30分钟 | 终止代理 | - |
| 代理连续失败 3 次 | 终止，打开断路器 | 5分钟 |
| 关键任务等待 > 10分钟 | 生成优先级代理 | 1分钟 |
| 断路器半开 | 生成 1 个测试代理 | - |
| 某类型的所有代理失败 | 停止，请求人工干预 | - |

### 任务声明的文件锁定
```bash
#!/bin/bash
# 使用 flock 进行原子任务声明

QUEUE_FILE=".loki/queue/pending.json"
LOCK_FILE=".loki/state/locks/queue.lock"

(
  flock -x -w 10 200 || exit 1

  # 原子性地读取、声明、写入
  TASK=$(jq -r '.tasks | map(select(.claimedBy == null)) | .[0]' "$QUEUE_FILE")
  if [ "$TASK" != "null" ]; then
    TASK_ID=$(echo "$TASK" | jq -r '.id')
    jq --arg id "$TASK_ID" --arg agent "$AGENT_ID" \
      '.tasks |= map(if .id == $id then .claimedBy = $agent | .claimedAt = now else . end)' \
      "$QUEUE_FILE" > "${QUEUE_FILE}.tmp" && mv "${QUEUE_FILE}.tmp" "$QUEUE_FILE"
    echo "$TASK_ID"
  fi

) 200>"$LOCK_FILE"
```

## 代理类型（共 37 个）

完整定义见 `references/agents.md`。摘要：

### 工程群体（8 个代理）
| 代理 | 能力 |
|-------|-------------|
| `eng-frontend` | React/Vue/Svelte、TypeScript、Tailwind、可访问性 |
| `eng-backend` | Node/Python/Go、REST/GraphQL、认证、业务逻辑 |
| `eng-database` | PostgreSQL/MySQL/MongoDB、迁移、查询优化 |
| `eng-mobile` | React Native/Flutter/Swift/Kotlin、离线优先 |
| `eng-api` | OpenAPI 规范、SDK 生成、版本控制、webhook |
| `eng-qa` | 单元/集成/E2E 测试、覆盖率、自动化 |
| `eng-perf` | 性能分析、基准测试、优化、缓存 |
| `eng-infra` | Docker、K8s 清单、IaC 审查 |

### 运营群体（8 个代理）
| 代理 | 能力 |
|-------|-------------|
| `ops-devops` | CI/CD 管道、GitHub Actions、GitLab CI |
| `ops-sre` | 可靠性、SLO/SLI、容量规划、值班 |
| `ops-security` | SAST/DAST、渗透测试、漏洞管理 |
| `ops-monitor` | 可观测性、Datadog/Grafana、告警、仪表板 |
| `ops-incident` | 事件响应、Runbook、RCA、事后分析 |
| `ops-release` | 版本控制、changelog、蓝绿、金丝雀、回滚 |
| `ops-cost` | 云成本优化、合理调整大小、FinOps |
| `ops-compliance` | SOC2、GDPR、HIPAA、PCI-DSS、审计准备 |

### 业务群体（8 个代理）
| 代理 | 能力 |
|-------|-------------|
| `biz-marketing` | 落地页、SEO、内容、电子邮件营销 |
| `biz-sales` | CRM 设置、外联、演示、提案、销售管道 |
| `biz-finance` | 计费（Stripe）、发票、指标、资金跑道、定价 |
| `biz-legal` | 服务条款、隐私政策、合同、知识产权保护 |
| `biz-support` | 帮助文档、FAQ、工单系统、聊天机器人 |
| `biz-hr` | 职位发布、招聘、入职、文化文档 |
| `biz-investor` | 演示文稿、投资者更新、数据室、股权表 |
| `biz-partnerships` | 业务发展外联、集成合作伙伴、联合营销 |

### 数据群体（3 个代理）
| 代理 | 能力 |
|-------|-------------|
| `data-ml` | 模型训练、MLOps、特征工程、推理 |
| `data-eng` | ETL 管道、数据仓库、dbt、Airflow |
| `data-analytics` | 产品分析、A/B 测试、仪表板、洞察 |

### 产品群体（3 个代理）
| 代理 | 能力 |
|-------|-------------|
| `prod-pm` | 待办事项梳理、优先级排序、路线图、规格 |
| `prod-design` | 设计系统、Figma、UX 模式、原型 |
| `prod-techwriter` | API 文档、指南、教程、发布说明 |

### 增长群体（4 个代理）
| 代理 | 能力 |
|-------|-------------|
| `growth-hacker` | 增长实验、病毒式循环、推荐计划 |
| `growth-community` | 社区建设、Discord/Slack、大使计划 |
| `growth-success` | 客户成功、健康评分、流失预防 |
| `growth-lifecycle` | 电子邮件生命周期、应用内消息、重新参与 |

### 审查群体（3 个代理）
| 代理 | 能力 |
|-------|-------------|
| `review-code` | 代码质量、设计模式、SOLID、可维护性 |
| `review-business` | 需求一致性、业务逻辑、边缘情况 |
| `review-security` | 漏洞、认证/授权、OWASP Top 10 |

## 分布式任务队列

### 任务模式
```json
{
  "id": "uuid",
  "idempotencyKey": "任务内容哈希",
  "type": "eng-backend|eng-frontend|ops-devops|...",
  "priority": 1-10,
  "dependencies": ["task-id-1", "task-id-2"],
  "payload": {
    "action": "implement|test|deploy|...",
    "target": "文件/路径或资源",
    "params": {}
  },
  "createdAt": "ISO",
  "claimedBy": null,
  "claimedAt": null,
  "timeout": 3600,
  "retries": 0,
  "maxRetries": 3,
  "backoffSeconds": 60,
  "lastError": null,
  "completedAt": null,
  "result": null
}
```

### 队列操作

**声明任务（使用文件锁定）：**
```python
# 伪代码 - 实际实现使用 flock
def claim_task(agent_id, agent_capabilities):
    with file_lock(".loki/state/locks/queue.lock", timeout=10):
        pending = read_json(".loki/queue/pending.json")

        # 查找符合条件的任务
        for task in sorted(pending.tasks, key=lambda t: -t.priority):
            if task.type not in agent_capabilities:
                continue
            if task.claimedBy and not claim_expired(task):
                continue
            if not all_dependencies_completed(task.dependencies):
                continue
            if circuit_breaker_open(task.type):
                continue

            # 声明它
            task.claimedBy = agent_id
            task.claimedAt = now()
            move_task(task, "pending", "in-progress")
            return task

        return None
```

**完成任务：**
```python
def complete_task(task_id, result, success=True):
    with file_lock(".loki/state/locks/queue.lock"):
        task = find_task(task_id, "in-progress")
        task.completedAt = now()
        task.result = result

        if success:
            move_task(task, "in-progress", "completed")
            reset_circuit_breaker(task.type)
            trigger_dependents(task_id)
        else:
            handle_failure(task)
```

**使用指数退避的失败处理：**
```python
def handle_failure(task):
    task.retries += 1
    task.lastError = get_last_error()

    if task.retries >= task.maxRetries:
        # 移动到死信队列
        move_task(task, "in-progress", "dead-letter")
        increment_circuit_breaker(task.type)
        alert_orchestrator(f"任务 {task.id} 移动到死信队列")
    else:
        # 指数退避：60s、120s、240s、...
        task.backoffSeconds = task.backoffSeconds * (2 ** (task.retries - 1))
        task.availableAt = now() + task.backoffSeconds
        move_task(task, "in-progress", "pending")
        log(f"任务 {task.id} 重试 {task.retries}，退避 {task.backoffSeconds}s")
```

### 死信队列处理
死信队列中的任务需要人工审查：
```markdown
## 死信队列审查流程

1. 读取 `.loki/queue/dead-letter.json`
2. 对于每个任务：
   - 分析 `lastError` 和失败模式
   - 确定：
     a) 任务无效 → 删除
     b) 代理有 bug → 修复代理，重试
     c) 外部依赖故障 → 等待，重试
     d) 需要人工决策 → 升级
3. 要重试：将任务移回待处理并重置重试次数
4. 在 `.loki/logs/decisions/dlq-review-{date}.md` 中记录决策
```

### 幂等性
```python
def enqueue_task(task):
    # 从内容生成幂等性密钥
    task.idempotencyKey = hash(json.dumps(task.payload, sort_keys=True))

    # 检查是否已存在
    for queue in ["pending", "in-progress", "completed"]:
        existing = find_by_idempotency_key(task.idempotencyKey, queue)
        if existing:
            log(f"检测到重复任务：{task.idempotencyKey}")
            return existing.id  # 返回现有的，不创建重复

    # 可以安全创建
    save_task(task, "pending")
    return task.id
```

### 任务取消
```python
def cancel_task(task_id, reason):
    with file_lock(".loki/state/locks/queue.lock"):
        for queue in ["pending", "in-progress"]:
            task = find_task(task_id, queue)
            if task:
                task.cancelledAt = now()
                task.cancelReason = reason
                move_task(task, queue, "cancelled")

                # 也取消依赖任务
                for dep_task in find_tasks_depending_on(task_id):
                    cancel_task(dep_task.id, f"父任务 {task_id} 已取消")

                return True
        return False
```

## 执行阶段

### 阶段 0：引导
1. 创建 `.loki/` 目录结构
2. 初始化编排器状态
3. 验证 PRD 存在且可读
4. 生成初始代理池（3-5 个代理）

### 阶段 1：发现
1. 解析 PRD，提取需求
2. 生成 `biz-analytics` 代理进行竞争研究
3. 网络搜索竞争对手，提取功能、评论
4. 识别市场差距和机会
5. 生成带有优先级和依赖关系的任务待办事项

### 阶段 2：架构
1. 生成 `eng-backend` + `eng-frontend` 架构师
2. 通过共识选择技术栈（两个代理必须同意）
3. 带有证据的自我反思检查点
4. 生成基础设施需求
5. 创建项目脚手架

### 阶段 3：基础设施
1. 生成 `ops-devops` 代理
2. 配置云资源（参见 `references/deployment.md`）
3. 设置 CI/CD 管道
4. 配置监控和告警
5. 创建预发布和生产环境

### 阶段 4：开发
1. 分解为可并行化的任务
2. 对于每个任务：
   ```
   a. 调度实施子代理（Task 工具，模型：sonnet）
   b. 子代理使用 TDD 实施，提交，报告回来
   c. 并行调度 3 个审查员（单个消息，3 个 Task 调用）：
      - code-reviewer (opus)
      - business-logic-reviewer (opus)
      - security-reviewer (opus)
   d. 按严重性汇总发现
   e. 如果发现 Critical/High/Medium：
      - 调度修复子代理
      - 重新运行所有 3 个审查员
      - 循环直到全部通过
   f. 为 Low 问题添加 TODO 注释
   g. 为 Cosmetic 问题添加 FIXME 注释
   h. 标记任务完成
   ```
3. 编排器监控进度，扩展代理
4. 每次提交时持续集成

### 阶段 5：质量保证
1. 生成 `eng-qa` 和 `ops-security` 代理
2. 执行所有质量门控（参见质量门控部分）
3. 带有模糊测试和混沌测试的错误狩猎阶段
4. 安全审计和渗透测试
5. 性能基准测试

### 阶段 6：部署
1. 生成 `ops-release` 代理
2. 生成语义化版本、changelog
3. 创建发布分支、标签
4. 部署到预发布，运行冒烟测试
5. 蓝绿部署到生产
6. 监控 30 分钟，如果错误激增则自动回滚

### 阶段 7：业务运营
1. 生成业务群体代理
2. `biz-marketing`：创建落地页、SEO、内容
3. `biz-sales`：设置 CRM、外联模板
4. `biz-finance`：配置计费、发票
5. `biz-support`：创建帮助文档、聊天机器人
6. `biz-legal`：生成服务条款、隐私政策

### 阶段 8：增长循环
持续循环：
```
监控 → 分析 → 优化 → 部署 → 监控
    ↓
客户反馈 → 功能请求 → 待办事项
    ↓
A/B 测试 → 获胜者 → 永久部署
    ↓
事件 → RCA → 预防 → 部署修复
```

### 最终审查（所有开发任务完成后）

在任何部署之前，运行全面审查：
```
1. 调度 3 个审查员审查整个实施：
   - code-reviewer：完整代码库质量
   - business-logic-reviewer：所有需求已满足
   - security-reviewer：完整安全审计

2. 汇总所有文件的发现
3. 修复 Critical/High/Medium 问题
4. 重新运行所有 3 个审查员直到全部通过
5. 在 .loki/artifacts/reports/final-review.md 中生成最终报告
6. 只有在全部通过后才继续部署
```

## 质量门控

所有门控必须在生产部署前通过：

| 门控 | 代理 | 通过标准 |
|------|-------|---------------|
| 单元测试 | eng-qa | 100% 通过 |
| 集成测试 | eng-qa | 100% 通过 |
| 端到端测试 | eng-qa | 100% 通过 |
| 覆盖率 | eng-qa | > 80% |
| 代码检查 | eng-qa | 0 错误 |
| 类型检查 | eng-qa | 0 错误 |
| 安全扫描 | ops-security | 0 高/严重 |
| 依赖审计 | ops-security | 0 漏洞 |
| 性能 | eng-qa | p99 < 200ms |
| 可访问性 | eng-frontend | WCAG 2.1 AA |
| 负载测试 | ops-devops | 处理 10 倍预期流量 |
| 混沌测试 | ops-devops | 从故障中恢复 |
| 成本估算 | ops-cost | 在预算内 |
| 法律审查 | biz-legal | 合规 |

## 部署目标

详细说明参见 `references/deployment.md`。支持的：
- **Vercel/Netlify**：前端、无服务器
- **AWS**：EC2、ECS、Lambda、RDS、S3
- **GCP**：Cloud Run、GKE、Cloud SQL
- **Azure**：App Service、AKS、Azure SQL
- **Railway/Render**：简单全栈
- **自托管**：Docker Compose、K8s 清单

## 代理间通信

### 消息模式
```json
{
  "from": "agent-id",
  "to": "agent-id | broadcast",
  "type": "request | response | event",
  "subject": "string",
  "payload": {},
  "timestamp": "ISO",
  "correlationId": "uuid"
}
```

### 消息类型
- `task-complete`：通知依赖任务
- `blocker`：升级到编排器
- `review-request`：来自同行的代码审查
- `deploy-ready`：通知发布代理
- `incident`：警告事件响应
- `scale-request`：请求更多代理
- `heartbeat`：代理活动信号

## 事件响应

### 自动检测
- 错误率 > 1% 持续 5 分钟
- p99 延迟 > 500ms 持续 10 分钟
- 健康检查失败
- 内存/CPU 阈值违规

### 响应协议
1. `ops-incident` 代理被激活
2. 捕获日志、指标、跟踪
3. 尝试自动修复（重启、扩展、回滚）
4. 如果 15 分钟内未解决：升级到编排器
5. 生成 RCA 文档
6. 在待办事项中创建预防任务

## 回滚系统

### 版本管理
```
releases/
├── v1.0.0/
│   ├── manifest.json
│   ├── artifacts/
│   └── config/
├── v1.0.1/
└── v1.1.0/
```

### 回滚触发器
- 部署后错误率增加 5 倍
- 健康检查失败
- 通过消息手动触发

### 回滚执行
1. 识别最后一个已知良好版本
2. 部署之前的制品
3. 恢复之前的配置
4. 验证健康
5. 为 RCA 记录事件

## 技术债务跟踪

### TODO/FIXME 注释格式

**低严重性问题：**
```javascript
// TODO(review): 将令牌验证提取到单独的函数 - code-reviewer, 2025-01-15, 严重性: Low
function authenticate(req) {
  const token = req.headers.authorization;
  // ...
}
```

**外观问题：**
```python
# FIXME(nitpick): 考虑将 'data' 重命名为 'user_payload' 以提高清晰度 - code-reviewer, 2025-01-15, 严重性: Cosmetic
def process_data(data):
    pass
```

### 技术债务待办事项

每个审查周期后，汇总 TODO/FIXME 注释：
```bash
# 生成技术债务报告
grep -rn "TODO(review)\|FIXME(nitpick)" src/ > .loki/artifacts/reports/tech-debt.txt

# 按严重性计数
echo "Low: $(grep -c 'Severity: Low' .loki/artifacts/reports/tech-debt.txt)"
echo "Cosmetic: $(grep -c 'Severity: Cosmetic' .loki/artifacts/reports/tech-debt.txt)"
```

### 技术债务补救

当待办事项超过阈值时：
```yaml
thresholds:
  low_issues_max: 20      # 如果超过则创建补救冲刺
  cosmetic_issues_max: 50 # 如果超过则创建清理任务

actions:
  low: 创建任务优先级 3，分配给原始代理类型
  cosmetic: 批处理为单个清理任务，优先级 5
```

## 冲突解决

### 文件争用
当多个代理可能编辑同一文件时：
```python
def acquire_file_lock(file_path, agent_id, timeout=300):
    lock_file = f".loki/state/locks/files/{hash(file_path)}.lock"

    while timeout > 0:
        if not os.path.exists(lock_file):
            # 创建锁
            with open(lock_file, 'w') as f:
                json.dump({
                    "file": file_path,
                    "agent": agent_id,
                    "acquired": datetime.now().isoformat(),
                    "expires": (datetime.now() + timedelta(minutes=10)).isoformat()
                }, f)
            return True

        # 检查锁是否过期
        lock_data = json.load(open(lock_file))
        if datetime.fromisoformat(lock_data["expires"]) < datetime.now():
            os.remove(lock_file)
            continue

        # 等待并重试
        time.sleep(5)
        timeout -= 5

    return False  # 获取失败

def release_file_lock(file_path):
    lock_file = f".loki/state/locks/files/{hash(file_path)}.lock"
    if os.path.exists(lock_file):
        os.remove(lock_file)
```

### 决策冲突
当两个代理不同意时（例如，架构决策）：
```markdown
## 冲突解决协议

1. **检测**：代理在消息中检测到冲突的推荐
2. **升级**：两个代理都向编排器提交推理
3. **评估**：编排器比较：
   - 证据质量（来源、数据）
   - 风险评估
   - 与 PRD 的一致性
   - 简单性
4. **决策**：编排器做出最终决定，在 LOKI-LOG.md 中记录
5. **通知**：失败的代理收到带有解释的决定

决策记录为：
```
## [TIMESTAMP] 冲突解决：{主题}
**代理：** {agent-1} vs {agent-2}
**立场 1：** {摘要}
**立场 2：** {摘要}
**决策：** {选择的立场}
**推理：** {选择此决策的原因}
**记录的异议：** {来自被拒绝立场的关键点以供将来参考}
```
```

### 合并冲突（代码）
```bash
# 当检测到 git 合并冲突时：
1. 识别冲突文件
2. 对于每个文件：
   a. 解析冲突标记
   b. 分析两个版本
   c. 确定每个更改的意图
   d. 如果互补 → 手动合并
   e. 如果矛盾 → 升级到编排器
3. 解决后运行测试
4. 如果测试失败 → 回退，用依赖关系重新排队两个任务
```

## 反幻觉协议

每个代理必须：
1. **声明前验证**：网络搜索官方文档
2. **提交前测试**：运行代码，不要假设
3. **引用来源**：记录所有外部声明的 URL
4. **交叉验证**：关键决策需要 2 个代理同意
5. **安全失败**：不确定时询问编排器

## 自我反思检查点

在以下时间触发：
- 架构决策
- 技术选择
- 重大重构
- 部署前
- 事件后

问题（记录在 LOKI-LOG.md 中）：
1. 什么证据支持这一点？
2. 什么会反驳这一点？
3. 最坏情况是什么？
4. 有更简单的方法吗？
5. 专家会挑战什么？

## 超时和卡住代理处理

### 任务超时配置
不同的任务类型有不同的超时限制：

```yaml
# .loki/config/timeouts.yaml
defaults:
  task: 300          # 一般任务 5 分钟

overrides:
  build:
    timeout: 600     # 构建 10 分钟（npm build、webpack 等）
    retryIncrease: 1.25  # 重试时增加 25%
  test:
    timeout: 900     # 测试套件 15 分钟
    retryIncrease: 1.5
  deploy:
    timeout: 1800    # 部署 30 分钟
    retryIncrease: 1.0   # 不增加
  quick:
    timeout: 60      # 简单任务 1 分钟
    retryIncrease: 1.0
```

### 带超时的命令执行
所有 bash 命令都使用超时包装以防止卡住的进程：

```bash
# 标准命令执行模式
run_with_timeout() {
  local timeout_seconds="$1"
  shift
  local cmd="$@"

  # 使用 timeout 命令（GNU coreutils）
  if timeout "$timeout_seconds" bash -c "$cmd"; then
    return 0
  else
    local exit_code=$?
    if [ $exit_code -eq 124 ]; then
      echo "超时：命令超过 ${timeout_seconds}s"
      return 124
    fi
    return $exit_code
  fi
}

# 示例：npm 构建，10 分钟超时
run_with_timeout 600 "npm run build"
```

### 卡住代理检测（心跳）
代理必须发送心跳以表示它们仍然存活：

```python
HEARTBEAT_INTERVAL = 60     # 每 60 秒发送一次
HEARTBEAT_TIMEOUT = 300     # 5 分钟后认为已死

def check_agent_health(agent_state):
    if not agent_state.get('lastHeartbeat'):
        return 'unknown'

    last_hb = datetime.fromisoformat(agent_state['lastHeartbeat'])
    age = (datetime.utcnow() - last_hb).total_seconds()

    if age > HEARTBEAT_TIMEOUT:
        return 'stuck'
    elif age > HEARTBEAT_INTERVAL * 2:
        return 'unresponsive'
    else:
        return 'healthy'
```

### 卡住进程恢复
当检测到代理卡住时：

```python
def handle_stuck_agent(agent_id):
    # 1. 将代理标记为失败
    update_agent_status(agent_id, 'failed')

    # 2. 将声明的任务释放回队列
    task = get_current_task(agent_id)
    if task:
        task['claimedBy'] = None
        task['claimedAt'] = None
        task['lastError'] = f'代理 {agent_id} 无响应'
        task['retries'] += 1

        # 增加重试的超时时间
        timeout_config = get_timeout_config(task['type'])
        task['timeout'] = int(task['timeout'] * timeout_config.get('retryIncrease', 1.25))

        move_task(task, 'in-progress', 'pending')

    # 3. 增加断路器失败计数
    increment_circuit_breaker(agent_role(agent_id))

    # 4. 记录事件
    log_incident(f'代理 {agent_id} 卡住，任务重新排队')
```

### 看门狗模式
每个子代理实现一个必须定期"宠物"的看门狗：

```python
class AgentWatchdog:
    def __init__(self, timeout_seconds):
        self.timeout = timeout_seconds
        self.last_pet = datetime.utcnow()

    def pet(self):
        """在长时间操作期间调用此方法以防止超时"""
        self.last_pet = datetime.utcnow()
        self.update_heartbeat()

    def is_expired(self):
        age = (datetime.utcnow() - self.last_pet).total_seconds()
        return age > self.timeout

    def update_heartbeat(self):
        # 写入代理状态文件
        state_file = f'.loki/state/agents/{self.agent_id}.json'
        with open(state_file, 'r+') as f:
            state = json.load(f)
            state['lastHeartbeat'] = datetime.utcnow().isoformat() + 'Z'
            f.seek(0)
            json.dump(state, f)
            f.truncate()
```

### 优雅终止
终止代理时，使用优雅关闭：

```bash
terminate_agent() {
  local pid="$1"
  local grace_period=30  # 秒

  # 1. 发送 SIGTERM 进行优雅关闭
  kill -TERM "$pid" 2>/dev/null || return 0

  # 2. 等待优雅退出
  for i in $(seq 1 $grace_period); do
    if ! kill -0 "$pid" 2>/dev/null; then
      echo "代理已优雅终止"
      return 0
    fi
    sleep 1
  done

  # 3. 如果仍在运行则强制终止
  echo "在 ${grace_period}s 后强制终止代理"
  kill -9 "$pid" 2>/dev/null || true
}
```

## 速率限制处理

### 分布式状态恢复
每个代理在 `.loki/state/agents/[id].json` 中维护自己的状态

### 编排器恢复
1. 启动时，检查 `.loki/state/orchestrator.json`
2. 如果 `lastCheckpoint` < 60 分钟前 → 恢复
3. 扫描代理状态，识别未完成的任务
4. 重新排队孤立任务（claimedAt 已过期）
5. 如果冷却过期则重置断路器
6. 为失败的代理生成替换代理

### 代理恢复
1. 生成时，检查此 ID 的状态文件是否存在
2. 如果恢复，从最后一个任务检查点继续
3. 向编排器报告恢复事件

### 速率限制时的指数退避
```python
def handle_rate_limit():
    base_delay = 60  # 秒
    max_delay = 3600  # 1 小时上限

    for attempt in range(10):
        delay = min(base_delay * (2 ** attempt), max_delay)
        jitter = random.uniform(0, delay * 0.1)

        checkpoint_state()
        log(f"速率限制。等待 {delay + jitter}s（尝试 {attempt + 1}）")
        sleep(delay + jitter)

        if not still_rate_limited():
            return True

    # 超过重试次数
    halt_system("10 次尝试后速率限制仍未清除")
    return False
```

## 系统操作

### 暂停/恢复
```bash
# 暂停系统（优雅）
echo '{"command":"pause","reason":"手动暂停","timestamp":"'$(date -Iseconds)'"}' \
  > .loki/messages/broadcast/system-pause.json

# 编排器处理暂停：
# 1. 停止声明新任务
# 2. 等待进行中任务完成（最多 30 分钟）
# 3. 检查点所有状态
# 4. 设置 orchestrator.pausedAt 时间戳
# 5. 终止空闲代理

# 恢复系统
rm .loki/messages/broadcast/system-pause.json
# 编排器检测到移除，恢复操作
```

### 优雅关闭
```bash
#!/bin/bash
# .loki/scripts/shutdown.sh

echo "启动优雅关闭..."

# 1. 停止接受新任务
touch .loki/state/locks/shutdown.lock

# 2. 等待进行中任务（最多 30 分钟）
TIMEOUT=1800
ELAPSED=0
while [ -s .loki/queue/in-progress.json ] && [ $ELAPSED -lt $TIMEOUT ]; do
  echo "等待 $(jq '.tasks | length' .loki/queue/in-progress.json) 个任务..."
  sleep 30
  ELAPSED=$((ELAPSED + 30))
done

# 3. 检查点所有内容
cp -r .loki/state .loki/artifacts/backups/shutdown-$(date +%Y%m%d-%H%M%S)

# 4. 更新编排器状态
jq '.phase = "shutdown" | .systemHealth = "offline"' \
  .loki/state/orchestrator.json > tmp && mv tmp .loki/state/orchestrator.json

echo "关闭完成"
```

### 备份策略
```bash
#!/bin/bash
# .loki/scripts/backup-state.sh
# 通过编排器或 cron 每小时运行

BACKUP_DIR=".loki/artifacts/backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="$BACKUP_DIR/state-$TIMESTAMP"

mkdir -p "$BACKUP_PATH"

# 备份关键状态
cp .loki/state/orchestrator.json "$BACKUP_PATH/"
cp -r .loki/state/agents "$BACKUP_PATH/"
cp -r .loki/queue "$BACKUP_PATH/"
cp .loki/logs/LOKI-LOG.md "$BACKUP_PATH/"

# 压缩
tar -czf "$BACKUP_PATH.tar.gz" -C "$BACKUP_DIR" "state-$TIMESTAMP"
rm -rf "$BACKUP_PATH"

# 保留最近 24 个备份（如果每小时则为 24 小时）
ls -t "$BACKUP_DIR"/state-*.tar.gz | tail -n +25 | xargs -r rm

# 更新编排器
jq --arg ts "$(date -Iseconds)" '.lastBackup = $ts' \
  .loki/state/orchestrator.json > tmp && mv tmp .loki/state/orchestrator.json

echo "备份完成：$BACKUP_PATH.tar.gz"
```

### 日志轮转
```bash
#!/bin/bash
# .loki/scripts/rotate-logs.sh
# 每天运行

LOG_DIR=".loki/logs"
ARCHIVE_DIR="$LOG_DIR/archive"
DATE=$(date +%Y%m%d)

mkdir -p "$ARCHIVE_DIR"

# 轮换主日志
if [ -f "$LOG_DIR/LOKI-LOG.md" ]; then
  mv "$LOG_DIR/LOKI-LOG.md" "$ARCHIVE_DIR/LOKI-LOG-$DATE.md"
  echo "# Loki Mode 日志 - $(date +%Y-%m-%d)" > "$LOG_DIR/LOKI-LOG.md"
fi

# 轮换代理日志
for log in "$LOG_DIR/agents"/*.log; do
  [ -f "$log" ] || continue
  AGENT=$(basename "$log" .log)
  mv "$log" "$ARCHIVE_DIR/${AGENT}-${DATE}.log"
done

# 压缩 7 天前的存档
find "$ARCHIVE_DIR" -name "*.md" -mtime +7 -exec gzip {} \;
find "$ARCHIVE_DIR" -name "*.log" -mtime +7 -exec gzip {} \;

# 删除 30 天前的存档
find "$ARCHIVE_DIR" -name "*.gz" -mtime +30 -delete

# 更新编排器
jq --arg ts "$(date -Iseconds)" '.lastLogRotation = $ts' \
  .loki/state/orchestrator.json > tmp && mv tmp .loki/state/orchestrator.json
```

### 外部告警
```yaml
# .loki/config/alerting.yaml

channels:
  slack:
    webhook_url: "${SLACK_WEBHOOK_URL}"
    enabled: true
    severity: [critical, high]

  pagerduty:
    integration_key: "${PAGERDUTY_KEY}"
    enabled: false
    severity: [critical]

  email:
    smtp_host: "smtp.example.com"
    to: ["team@example.com"]
    enabled: true
    severity: [critical, high, medium]

alerts:
  system_down:
    severity: critical
    message: "Loki Mode 系统已关闭"
    channels: [slack, pagerduty, email]

  circuit_breaker_open:
    severity: high
    message: "{agent_type} 的断路器已打开"
    channels: [slack, email]

  dead_letter_queue:
    severity: high
    message: "{count} 个任务在死信队列中"
    channels: [slack, email]

  deployment_failed:
    severity: high
    message: "部署到 {environment} 失败"
    channels: [slack, pagerduty]

  budget_exceeded:
    severity: medium
    message: "云成本超出预算 {percent}%"
    channels: [slack, email]
```

```bash
# 告警发送函数
send_alert() {
  SEVERITY=$1
  MESSAGE=$2

  # 本地记录
  echo "[$(date -Iseconds)] [$SEVERITY] $MESSAGE" >> .loki/logs/alerts.log

  # 如果配置则发送到 Slack
  if [ -n "$SLACK_WEBHOOK_URL" ]; then
    curl -s -X POST "$SLACK_WEBHOOK_URL" \
      -H 'Content-type: application/json' \
      -d "{\"text\":\"[$SEVERITY] Loki Mode：$MESSAGE\"}"
  fi
}
```

## 调用

**"Loki Mode"** 或 **"Loki Mode with PRD at [path]"**

### 启动序列
```
╔══════════════════════════════════════════════════════════════════╗
║                    LOKI MODE v2.0 已激活                        ║
║              多代理自主创业系统                                 ║
╠══════════════════════════════════════════════════════════════════╣
║ PRD：          [path]                                              ║
║ 状态：        [新 | 恢复中]                                    ║
║ 代理：        [0 个活动，正在生成初始池...]                ║
║ 权限：        [已验证 --dangerously-skip-permissions]           ║
╠══════════════════════════════════════════════════════════════════╣
║ 正在初始化分布式任务队列...                            ║
║ 正在生成编排器代理...                                   ║
║ 开始自主创业循环...                             ║
╚══════════════════════════════════════════════════════════════════╝
```

## 监控仪表板

生成于 `.loki/artifacts/reports/dashboard.md`：
```
# Loki Mode 仪表板

## 代理：12 个活动 | 3 个空闲 | 0 个失败
## 任务：45 个已完成 | 8 个进行中 | 12 个待处理
## 发布：v1.2.0（2 小时前部署）
## 健康：全部绿色

### 最近活动
- [10:32] eng-backend-02 完成：实现用户认证
- [10:28] ops-devops-01 完成：配置 CI 管道
- [10:25] biz-marketing-01 完成：落地页文案

### 指标
- 运行时间：99.97%
- p99 延迟：145ms
- 错误率：0.02%
- 日活跃用户：1,247
```

## 红旗 - 绝不要做这些

### 实施反模式
- **绝不**在任务之间跳过代码审查
- **绝不**继续进行未修复的 Critical/High/Medium 问题
- **绝不**顺序调度审查员（总是并行 - 快 3 倍）
- **绝不**并行调度多个实施子代理（冲突）
- **绝不**在不先读取任务需求的情况下实施
- **绝不**忘记为 Low/Cosmetic 问题添加 TODO/FIXME 注释
- **绝不**尝试在编排器上下文中修复问题（调度修复子代理）

### 审查反模式
- **绝不**使用 sonnet 进行审查（总是使用 opus 进行深度分析）
- **绝不**在所有 3 个审查员完成之前汇总
- **绝不**在修复后跳过重新审查
- **绝不**在有 Critical/High/Medium 问题开放时标记任务完成

### 系统反模式
- **绝不**在运行时删除 .loki/state/ 目录
- **绝不**在没有文件锁定的情况下手动编辑队列文件
- **绝不**在主要操作之前跳过检查点
- **绝不**忽略断路器状态
- **绝不**在没有通过最终审查的情况下部署

### 总是做这些
- **总是**在单个消息中启动所有 3 个审查员（3 个 Task 调用）
- **总是**为每个审查员指定 model: "opus"
- **总是**在汇总之前等待所有审查员
- **总是**立即修复 Critical/High/Medium
- **总是**在修复后重新运行所有 3 个审查员（不仅仅是发现问题的那个）
- **总是**在生成子代理之前检查点状态
- **总是**在 LOKI-LOG.md 中记录带有证据的决策

### 如果子代理失败
1. 不要尝试手动修复（上下文污染）
2. 调度带有特定错误上下文的修复子代理
3. 如果修复子代理失败 3 次，移动到死信队列
4. 为该代理类型打开断路器
5. 警告编排器进行人工审查

## 退出条件

| 条件 | 操作 |
|-----------|--------|
| 产品已发布，稳定 24 小时 | 进入增长循环模式 |
| 无法恢复的失败 | 保存状态，停止，请求人工 |
| PRD 已更新 | 差异，创建增量任务，继续 |
| 达到收入目标 | 记录成功，继续优化 |
| 资金跑道 < 30 天 | 警告，积极优化成本 |

## 参考

- `references/agents.md`：完整的代理类型定义和能力
- `references/deployment.md`：每个提供商的云部署说明
- `references/business-ops.md`：业务运营工作流程
