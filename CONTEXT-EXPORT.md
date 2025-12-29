# Loki Mode - 会话上下文导出

**日期：** 2025-12-28
**版本：** 2.5.0
**仓库：** https://github.com/asklokesh/claudeskill-loki-mode

---

## 项目概述

**Loki Mode** 是一个 Claude Code 技能，提供多代理自主创业系统。它协调 6 个群体中的 37 个专业代理，将 PRD 从想法转化为完全部署的产品。

### 核心功能
- 37 个代理分布在 6 个群体（工程、运营、业务、数据、产品、增长）
- 用于子代理调度的 Task 工具，带有新上下文
- 分布式任务队列（待处理、进行中、已完成、失败、死信）
- 每个代理故障处理的断路器
- 通过心跳监控的超时/卡住代理检测
- 通过 `.loki/state/` 中的检查点进行状态恢复
- 带速率限制自动恢复的自主执行

---

## 文件结构

```
claudeskill-loki-mode/
├── SKILL.md                    # 主技能文件（需要 YAML 前置元数据）
├── VERSION                     # 当前版本：2.5.0
├── CHANGELOG.md                # 完整的版本历史
├── README.md                   # 主文档
├── references/
│   ├── agents.md               # 37 个代理定义
│   ├── deployment.md           # 云部署指南
│   └── business-ops.md         # 业务操作工作流程
├── examples/
│   ├── simple-todo-app.md      # 用于测试的简单 PRD
│   ├── api-only.md             # 仅后端 PRD
│   ├── static-landing-page.md  # 前端/营销 PRD
│   └── full-stack-demo.md      # 完整书签管理器 PRD
├── tests/
│   ├── run-all-tests.sh        # 主测试运行器（53 个测试）
│   ├── test-bootstrap.sh       # 8 个测试
│   ├── test-task-queue.sh      # 8 个测试
│   ├── test-circuit-breaker.sh # 8 个测试
│   ├── test-agent-timeout.sh   # 9 个测试
│   ├── test-state-recovery.sh  # 8 个测试
│   └── test-wrapper.sh         # 12 个测试
├── scripts/
│   ├── loki-wrapper.sh         # 传统包装器（已弃用）
│   └── export-to-vibe-kanban.sh # 可选的 Vibe Kanban 导出
├── integrations/
│   └── vibe-kanban.md          # Vibe Kanban 集成指南
├── autonomy/
│   ├── run.sh                  # ⭐ 主入口点 - 处理所有内容
│   └── README.md               # 自主文档
└── .github/workflows/
    └── release.yml             # GitHub Actions 发布工作流
```

---

## 如何使用

### 快速开始（推荐）
```bash
./autonomy/run.sh ./docs/requirements.md
```

### run.sh 做什么
1. 检查先决条件（Claude CLI、Python、Git、curl）
2. 验证技能安装
3. 初始化 `.loki/` 目录
4. 启动状态监视器（每 5 秒更新 `.loki/STATUS.txt`）
5. 运行带有实时输出的 Claude Code
6. 使用指数退避在速率限制时自动恢复
7. 持续运行直到完成或达到最大重试次数

### 监控进度
```bash
# 在另一个终端
watch -n 2 cat .loki/STATUS.txt
```

---

## 关键技术细节

### Claude Code 调用
自主运行器通过 stdin 传递提示以实现实时输出：
```bash
echo "$prompt" | claude --dangerously-skip-permissions
```

**重要：** 使用 `-p` 标志无法正确流式传输输出。通过 stdin 传输可显示交互式输出。

### 状态文件
- `.loki/state/orchestrator.json` - 当前阶段、指标
- `.loki/autonomy-state.json` - 重试计数、状态、PID
- `.loki/queue/*.json` - 任务队列
- `.loki/STATUS.txt` - 人类可读状态（每 5 秒更新）
- `.loki/logs/*.log` - 执行日志

### 环境变量
| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `LOKI_MAX_RETRIES` | 50 | 最大重试次数 |
| `LOKI_BASE_WAIT` | 60 | 基础等待时间（秒） |
| `LOKI_MAX_WAIT` | 3600 | 最大等待时间（1 小时） |
| `LOKI_SKIP_PREREQS` | false | 跳过先决条件检查 |

---

## 版本历史摘要

| 版本 | 主要更改 |
|---------|-------------|
| 2.5.0 | 实时流式输出（stream-json）、采用 Anthropic 设计的 Web 仪表板 |
| 2.4.0 | 实时输出修复（stdin 管道）、STATUS.txt 监视器 |
| 2.3.0 | 统一自主运行器（`autonomy/run.sh`） |
| 2.2.0 | Vibe Kanban 集成 |
| 2.1.0 | 带自动恢复的自主包装器 |
| 2.0.x | 测试套件、macOS 兼容性、发布工作流 |
| 1.x.x | 初始技能，包含代理、部署指南 |

---

## 已知问题和解决方案

### 1. "自主运行时输出为空白"
**原因：** 使用 `-p` 标志无法流式传输输出
**解决方案：** 使用 stdin 管道：`echo "$prompt" | claude --dangerously-skip-permissions`

### 2. "Vibe Kanban 未显示任务"
**原因：** Vibe Kanban 是 UI 驱动的，不会自动读取 JSON 文件
**解决方案：** 使用 `.loki/STATUS.txt` 进行监控，或单独运行 Vibe Kanban

### 3. "macOS 上未找到 timeout 命令"
**原因：** macOS 没有 GNU coreutils
**解决方案：** 测试脚本中的基于 Perl 的后备

### 4. "TTY raw mode 错误"
**原因：** 在非交互模式下运行 Claude
**解决方案：** 最新提交（008ed86）添加了 `--no-input` 标志

---

## Git 配置

**提交者：** asklokesh（永远不要使用 Claude 作为共同作者）

**提交格式：**
```
简短描述 (vX.X.X)

详细的更改要点列表
```

---

## 测试套件

运行所有测试：
```bash
./tests/run-all-tests.sh
```

6 个测试套件中的 53 个测试 - 所有测试都应该通过。

---

## 待处理/未来工作

1. **正确的 Vibe Kanban 集成** - Vibe Kanban 不读取文件，需要 API 集成
2. **更好的实时输出** - 当前 stdin 管道有效，但可能有边缘情况
3. **任务可视化** - 可以添加简单的 TUI 用于任务监控

---

## 优先阅读的重要文件

开始新会话时，阅读这些文件：
1. `SKILL.md` - 实际的技能说明
2. `autonomy/run.sh` - 主入口点
3. `VERSION` 和 `CHANGELOG.md` - 当前状态
4. 本文件（`CONTEXT-EXPORT.md`）- 完整上下文

---

## 用户偏好

- 始终使用 `asklokesh` 作为提交者
- 永远不要使用 Claude 作为共同作者
- 保持技能文件整洁，自主分离
- 推送前测试
- 实时输出很重要 - 用户希望看到正在发生的事情

---

## 最后已知状态

- **版本：** 2.5.0
- **最新提交：**（待推送）
- **测试：** 所有 53 个测试通过
- **添加的功能：** 通过 stream-json 实时流式输出、采用 Anthropic 设计的 Web 仪表板
