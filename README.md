# Loki Mode（洛基模式）

**Claude Code 的多代理自主创业系统**

[![Claude Code](https://img.shields.io/badge/Claude-Code-orange)](https://claude.ai)
[![Agents](https://img.shields.io/badge/Agents-37-blue)]()
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> 将产品需求文档（PRD）转化为完全部署的、产生收入的产品，无需任何人工干预。

## 什么是 Loki Mode？

Loki Mode 是一个 Claude Code 技能，它协调 6 个群体中的 37 个专业 AI 代理，自主构建、部署和运营一个完整的创业项目。只需说 **"Loki Mode"** 并提供 PRD 即可。

```
PRD → 竞争研究 → 架构设计 → 开发 → 测试 → 部署 → 营销 → 收入
```

## 功能特性

| 类别 | 能力 |
|----------|-------------|
| **多代理系统** | 37 个代理分布在工程、运营、业务、数据、产品和增长群体 |
| **并行代码审查** | 3 个专业审查员（代码、业务、安全）同时运行 |
| **质量门控** | 14 个自动化门控，包括安全扫描、负载测试、可访问性 |
| **部署** | AWS、GCP、Azure、Vercel、Railway，支持蓝绿和金丝雀策略 |
| **业务运营** | 营销、销售、人力资源、法律、财务、投资者关系代理 |
| **可靠性** | 断路器、死信队列、指数退避、状态恢复 |
| **可观测性** | 外部告警（Slack、PagerDuty）、备份/恢复、日志轮转 |

## 代理群体

<img width="5309" height="979" alt="image" src="https://github.com/user-attachments/assets/7d18635d-a606-401f-8d9f-430e6e4ee689" />


### 工程群体（8个）
`eng-frontend` `eng-backend` `eng-database` `eng-mobile` `eng-api` `eng-qa` `eng-perf` `eng-infra`

### 运营群体（8个）
`ops-devops` `ops-sre` `ops-security` `ops-monitor` `ops-incident` `ops-release` `ops-cost` `ops-compliance`

### 业务群体（8个）
`biz-marketing` `biz-sales` `biz-finance` `biz-legal` `biz-support` `biz-hr` `biz-investor` `biz-partnerships`

### 数据群体（3个）
`data-ml` `data-eng` `data-analytics`

### 产品群体（3个）
`prod-pm` `prod-design` `prod-techwriter`

### 增长群体（4个）
`growth-hacker` `growth-community` `growth-success` `growth-lifecycle`

### 审查群体（3个）
`review-code` `review-business` `review-security`

## 安装

### 技能文件结构

```
SKILL.md              # ← 技能文件（必需）- 包含 YAML 前置元数据
references/
├── agents.md         # 代理定义
├── deployment.md     # 部署指南
└── business-ops.md   # 业务工作流程
```

### 对于 Claude.ai（网页版）

1. 访问 [Releases](https://github.com/asklokesh/claudeskill-loki-mode/releases)
2. 下载 `loki-mode-X.X.X.zip` 或 `loki-mode-X.X.X.skill`
3. 进入 **Claude.ai → 设置 → 功能 → 技能**
4. 上传 zip/skill 文件

zip 文件的根目录包含 `SKILL.md`，符合 Claude.ai 的要求。

### 对于 Claude Code（命令行）

**选项 A：从发布版本下载**
```bash
# 下载 Claude Code 版本
cd ~/.claude/skills

# 获取最新版本号
VERSION=$(curl -s https://api.github.com/repos/asklokesh/claudeskill-loki-mode/releases/latest | grep tag_name | cut -d'"' -f4 | tr -d 'v')

# 下载并解压
curl -L -o loki-mode.zip "https://github.com/asklokesh/claudeskill-loki-mode/releases/download/v${VERSION}/loki-mode-claude-code-${VERSION}.zip"
unzip loki-mode.zip && rm loki-mode.zip
# 创建：~/.claude/skills/loki-mode/SKILL.md
```

**选项 B：Git 克隆**
```bash
# 用于个人使用（所有项目）
git clone https://github.com/asklokesh/claudeskill-loki-mode.git ~/.claude/skills/loki-mode

# 仅用于特定项目
git clone https://github.com/asklokesh/claudeskill-loki-mode.git .claude/skills/loki-mode
```

**选项 C：最小化安装（curl）**
```bash
mkdir -p ~/.claude/skills/loki-mode/references
curl -o ~/.claude/skills/loki-mode/SKILL.md https://raw.githubusercontent.com/asklokesh/claudeskill-loki-mode/main/SKILL.md
curl -o ~/.claude/skills/loki-mode/references/agents.md https://raw.githubusercontent.com/asklokesh/claudeskill-loki-mode/main/references/agents.md
curl -o ~/.claude/skills/loki-mode/references/deployment.md https://raw.githubusercontent.com/asklokesh/claudeskill-loki-mode/main/references/deployment.md
curl -o ~/.claude/skills/loki-mode/references/business-ops.md https://raw.githubusercontent.com/asklokesh/claudeskill-loki-mode/main/references/business-ops.md
```

### 验证安装

```bash
# 检查技能是否就位
cat ~/.claude/skills/loki-mode/SKILL.md | head -5
# 应显示 YAML 前置元数据，其中 name 为 loki-mode
```

## 使用方法

### 快速开始（推荐）

使用自主运行器 - 它可以处理所有事情：

```bash
# 使用 PRD 运行（完全自主，自动恢复）
./autonomy/run.sh ./docs/requirements.md

# 交互式运行
./autonomy/run.sh
```

自主运行器将会：
1. 检查所有先决条件（Claude CLI、Python、Git 等）
2. 验证技能安装
3. 初始化 `.loki/` 目录
4. 启动**状态监视器**（每 5 秒更新 `.loki/STATUS.txt`）
5. 启动带有**实时输出**的 Claude Code（查看正在发生的事情）
6. 在速率限制或中断时自动恢复
7. 持续运行直到完成或达到最大重试次数

### 实时输出

不再盯着空白屏幕！Claude 的输出实时显示：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CLAUDE CODE OUTPUT (live)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[你在这里可以看到 Claude 的实时工作过程...]
```

### 状态监视器

在另一个终端中监控任务进度：

```bash
# 实时查看状态更新
watch -n 2 cat .loki/STATUS.txt
```

输出：
```
╔════════════════════════════════════════════════════════════════╗
║                    LOKI MODE 状态                             ║
╚════════════════════════════════════════════════════════════════╝

阶段：开发

任务：
  ├─ 待处理：     10
  ├─ 进行中：    1
  ├─ 已完成：   5
  └─ 失败：      0
```

### 手动模式

如果你更喜欢手动控制：

```bash
# 使用自主权限启动 Claude Code
claude --dangerously-skip-permissions

# 然后说：
> Loki Mode

# 或者使用特定的 PRD：
> Loki Mode with PRD at ./docs/requirements.md
```

## 自主配置

用于自定义自主运行器的环境变量：

```bash
# 自定义设置示例
LOKI_MAX_RETRIES=100 \
LOKI_BASE_WAIT=120 \
LOKI_MAX_WAIT=7200 \
./autonomy/run.sh ./docs/requirements.md
```

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `LOKI_MAX_RETRIES` | 50 | 放弃前的最大重试次数 |
| `LOKI_BASE_WAIT` | 60 | 基础等待时间（秒） |
| `LOKI_MAX_WAIT` | 3600 | 最大等待时间（1 小时） |
| `LOKI_SKIP_PREREQS` | false | 跳过先决条件检查 |

### 自动恢复工作原理

```
./autonomy/run.sh prd.md
         │
         ▼
┌─────────────────────┐
│ 检查先决条件        │ ← Claude CLI、Python、Git 等
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ 初始化 .loki/       │ ← 状态、队列、日志
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ 运行 Claude Code    │◄─────────────────┐
└─────────────────────┘                  │
         │                               │
    退出代码？                            │
         │                               │
    ┌────┴────┐                          │
    ▼         ▼                          │
 成功      速率限制                        │
    │         │                          │
    ▼         ▼                          │
 完成！    等待（指数退避）────────────────┘
```

### 中断后恢复

如果你停止脚本（Ctrl+C）或它崩溃了：

```bash
# 只需再次运行 - 状态会自动保存
./autonomy/run.sh ./docs/requirements.md
```

详细文档请参阅 [autonomy/README.md](autonomy/README.md)。

## 工作原理

### 阶段执行

| 阶段 | 描述 |
|-------|-------------|
| **0. 启动** | 创建 `.loki/` 目录结构，初始化状态 |
| **1. 发现** | 解析 PRD，通过网络搜索进行竞争研究 |
| **2. 架构** | 通过自我反思选择技术栈 |
| **3. 基础设施** | 配置云资源、CI/CD、监控 |
| **4. 开发** | 使用 TDD 实现，并行代码审查 |
| **5. 质量保证** | 14 个质量门控、安全审计、负载测试 |
| **6. 部署** | 蓝绿部署，错误时自动回滚 |
| **7. 业务** | 营销、销售、法律、支持设置 |
| **8. 增长** | 持续优化、A/B 测试、反馈循环 |

### 并行代码审查模式

每个任务同时经过 3 个审查员：

```
实施 → 审查（3 个并行）→ 汇总 → 修复 → 重新审查 → 完成
                │
                ├─ code-reviewer (opus)
                ├─ business-logic-reviewer (opus)
                └─ security-reviewer (opus)
```

### 基于严重性的问题处理

| 严重性 | 操作 |
|----------|--------|
| 严重/高/中 | 阻塞。立即修复。重新审查。 |
| 低 | 添加 `// TODO(review): ...` 注释，继续 |
| 外观问题 | 添加 `// FIXME(nitpick): ...` 注释，继续 |

## 目录结构

运行时，Loki Mode 会创建：

```
.loki/
├── state/          # 编排器和代理状态
├── queue/          # 任务队列（待处理、进行中、已完成、死信）
├── messages/       # 代理间通信
├── logs/           # 审计日志
├── config/         # 配置文件
├── prompts/        # 代理角色提示
├── artifacts/      # 发布、报告、备份
└── scripts/        # 辅助脚本
```

## 配置

### 断路器

```yaml
# .loki/config/circuit-breakers.yaml
defaults:
  failureThreshold: 5
  cooldownSeconds: 300
```

### 外部告警

```yaml
# .loki/config/alerting.yaml
channels:
  slack:
    webhook_url: "${SLACK_WEBHOOK_URL}"
    severity: [critical, high]
```

## 测试用例 PRD

使用 `examples/` 目录中的预构建 PRD 测试技能：

| PRD | 复杂度 | 时间 | 描述 |
|-----|------------|------|-------------|
| `simple-todo-app.md` | 低 | ~10 分钟 | 基础待办应用 - 测试核心功能 |
| `api-only.md` | 低 | ~10 分钟 | 仅 REST API - 测试后端代理 |
| `static-landing-page.md` | 低 | ~5 分钟 | 仅 HTML/CSS - 测试前端/营销 |
| `full-stack-demo.md` | 中 | ~30-60 分钟 | 完整的书签管理器 - 全面测试 |

```bash
# 示例：使用简单待办应用测试
claude --dangerously-skip-permissions
> Loki Mode with PRD at examples/simple-todo-app.md
```

## 运行测试

该技能包含全面的测试套件：

```bash
# 运行所有测试
./tests/run-all-tests.sh

# 运行单个测试套件
./tests/test-bootstrap.sh        # 目录结构、状态初始化
./tests/test-task-queue.sh       # 队列操作、优先级
./tests/test-circuit-breaker.sh  # 故障处理、恢复
./tests/test-agent-timeout.sh    # 超时、卡住进程处理
./tests/test-state-recovery.sh   # 检查点、恢复
```

## 系统要求

- Claude Code，带 `--dangerously-skip-permissions` 标志
- 用于竞争研究和部署的互联网访问
- 云提供商凭证（用于部署阶段）
- Python 3（用于测试套件）

## 对比

| 功能 | 基础技能 | Loki Mode |
|---------|-------------|-----------|
| 代理 | 1 | 37 |
| 群体 | - | 6 |
| 代码审查 | 手动 | 并行 3 审查员 |
| 部署 | 无 | 多云 |
| 业务运营 | 无 | 全栈 |
| 状态恢复 | 无 | 检查点/恢复 |
| 告警 | 无 | Slack/PagerDuty |

## 集成

### Vibe Kanban（可视化仪表板）

可选择与 [Vibe Kanban](https://github.com/BloopAI/vibe-kanban) 集成，提供可视化看板来监控 Loki Mode 的代理：

```bash
# 安装 Vibe Kanban
npx vibe-kanban

# 将 Loki 任务导出到 Vibe Kanban
./scripts/export-to-vibe-kanban.sh
```

优势：
- 所有 37 个代理的可视化进度跟踪
- 需要时手动干预/优先级排序
- 可视化差异的代码审查
- 多项目仪表板

完整设置指南请参阅 [integrations/vibe-kanban.md](integrations/vibe-kanban.md)。

## 贡献

欢迎贡献！请阅读技能说明并提交问题或功能请求。

## 许可证

MIT 许可证 - 详情请参阅 [LICENSE](LICENSE)。

## 致谢

- 灵感来自 [LerianStudio/ring](https://github.com/LerianStudio/ring) 的子代理驱动开发模式
- 为 [Claude Code](https://claude.ai) 生态系统构建

---

**关键词：** claude-code、claude-skills、ai-agents、autonomous-development、multi-agent-system、sdlc-automation、startup-automation、devops、mlops、deployment-automation
