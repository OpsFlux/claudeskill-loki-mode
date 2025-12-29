# 更新日志

Loki Mode 的所有重要更改都将记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)，
本项目遵循 [语义化版本](https://semver.org/spec/v2.0.0.html)。

## [2.5.0] - 2025-12-28

### 新增
- **实时流式输出** - Claude 的输出现在使用 `--output-format stream-json` 实时流式传输
  - 实时解析 JSON 流以显示文本、工具调用和结果
  - 当 Claude 使用工具时显示 `[Tool: name]`
  - 完成时显示 `[Session complete]`
- **Web 仪表板** - 采用 Anthropic 设计语言的视觉任务板
  - 奶油/米色背景，珊瑚色（#D97757）强调色，符合 Anthropic 品牌形象
  - 自动启动于 `http://127.0.0.1:57374` 并在浏览器中打开
  - 显示任务计数和看板风格列（待处理、进行中、已完成、失败）
  - 每 3 秒自动刷新
  - 使用 `LOKI_DASHBOARD=false` 禁用
  - 使用 `LOKI_DASHBOARD_PORT=<port>` 配置端口

### 更改
- 使用 `--output-format stream-json --verbose` 替换 `--print` 模式以实现正确的流式传输
- 基于 Python 的 JSON 解析器实时提取并显示 Claude 的响应
- 简单的 HTML 仪表板替换 Vibe Kanban（无外部依赖）

### 修复
- 实时输出现在真正流式传输（在 2.4.0 中是缓冲到完成后才显示）

## [2.4.0] - 2025-12-28

### 新增
- **实时输出** - Claude 的输出现在使用伪 TTY 实时流式传输
  - 使用 `script` 命令分配 PTY 以实现正确的流式传输
  - 视觉分隔符显示 Claude 何时工作
- **状态监视器** - `.loki/STATUS.txt` 每 5 秒更新一次，包含：
  - 当前阶段
  - 任务计数（待处理、进行中、已完成、失败）
  - 使用 `watch -n 2 cat .loki/STATUS.txt` 监控

### 更改
- 用更简单的状态文件监视器替换 Vibe Kanban 自动启动
- 自主运行器使用 `script` 进行 macOS/Linux 上的正确 TTY 输出

## [2.3.0] - 2025-12-27

### 新增
- **统一自主运行器**（`autonomy/run.sh`）- 处理所有内容的单个脚本：
  - 先决条件检查（Claude CLI、Python、Git、curl、Node.js、jq）
  - 技能安装验证
  - `.loki/` 目录初始化
  - 带自动恢复的自主执行
  - ASCII 艺术横幅和彩色日志
  - 带抖动的指数退避
  - 跨重启的状态持久化
  - 详细文档请参阅 `autonomy/README.md`

### 更改
- 将自主执行移至专用的 `autonomy/` 文件夹（与技能分离）
- 使用新的快速开始方法 `./autonomy/run.sh` 更新 README
- 发布工作流现在包含 `autonomy/` 文件夹

### 已弃用
- `scripts/loki-wrapper.sh` 仍然可用，但现在推荐 `autonomy/run.sh`

## [2.2.0] - 2025-12-27

### 新增
- **Vibe Kanban 集成** - 用于监控代理的可选可视化仪表板：
  - `integrations/vibe-kanban.md` - 完整集成指南
  - `scripts/export-to-vibe-kanban.sh` - 将 Loki 任务导出到 Vibe Kanban 格式
  - 任务状态映射（Loki 队列 → 看板列）
  - 阶段到列的映射，用于可视化进度跟踪
  - 用于调试的元数据保留
  - 参阅 [BloopAI/vibe-kanban](https://github.com/BloopAI/vibe-kanban)

### 文档
- README：添加了 Vibe Kanban 设置的集成部分

## [2.1.0] - 2025-12-27

### 新增
- **自主包装器脚本**（`scripts/loki-wrapper.sh`）- 带自动恢复的真正自主：
  - 监控 Claude Code 进程并在会话结束时检测
  - 在速率限制或中断时自动从检查点恢复
  - 带抖动的指数退避（可通过环境变量配置）
  - `.loki/wrapper-state.json` 中的状态持久化
  - 通过编排器状态或 `.loki/COMPLETED` 标记进行完成检测
  - 使用 SIGINT/SIGTERM 陷阱的优雅关闭处理
  - 可配置：`LOKI_MAX_RETRIES`、`LOKI_BASE_WAIT`、`LOKI_MAX_WAIT`

### 文档
- 在 README 中添加了真正的自主部分，解释包装器使用
- 记录了包装器如何检测会话完成和速率限制

## [2.0.3] - 2025-12-27

### 修复
- **正确的技能文件格式** - 发布工件现在遵循 Claude 的预期格式：
  - `loki-mode-X.X.X.zip` / `.skill` - 用于 Claude.ai（根目录包含 SKILL.md）
  - `loki-mode-claude-code-X.X.X.zip` - 用于 Claude Code（包含 loki-mode/ 文件夹）

### 改进
- **安装说明** - Claude.ai 和 Claude Code 的单独说明
- **SKILL.md** - 已有所需的 YAML 前置元数据，包含 `name` 和 `description`

## [2.0.2] - 2025-12-27

### 修复
- **发布工件结构** - Zip 现在包含 `loki-mode/` 文件夹（不是 `loki-mode-X.X.X/`）
  - 用户可以直接解压到技能目录而无需重命名
  - 仅包含必要的技能文件（无 .git 或 .github 文件夹）

### 改进
- **安装说明** - 使用更清晰的解压步骤更新 README

## [2.0.1] - 2025-12-27

### 改进
- **安装文档** - 全面的安装指南：
  - 解释哪个文件是实际技能（`SKILL.md`）
  - 显示技能文件结构和所需文件
  - 选项 1：从 GitHub Releases 下载（推荐）
  - 选项 2：Git 克隆
  - 选项 3：使用 curl 命令的最小化安装
  - 验证步骤

## [2.0.0] - 2025-12-27

### 新增
- **示例 PRD** - 4 个测试 PRD，供用户在实施前试用：
  - `examples/simple-todo-app.md` - 快速功能测试（~10 分钟）
  - `examples/api-only.md` - 后端代理测试
  - `examples/static-landing-page.md` - 前端/营销测试
  - `examples/full-stack-demo.md` - 全面测试（~30-60 分钟）

- **全面的测试套件** - 6 个测试文件中的 53 个测试：
  - `tests/test-bootstrap.sh` - 目录结构、状态初始化（8 个测试）
  - `tests/test-task-queue.sh` - 队列操作、优先级（8 个测试）
  - `tests/test-circuit-breaker.sh` - 故障处理、恢复（8 个测试）
  - `tests/test-agent-timeout.sh` - 超时、卡住进程处理（9 个测试）
  - `tests/test-state-recovery.sh` - 检查点、恢复（8 个测试）
  - `tests/test-wrapper.sh` - 包装器脚本、自动恢复（12 个测试）
  - `tests/run-all-tests.sh` - 主测试运行器

- **超时和卡住代理处理** - SKILL.md 中的新部分：
  - 每个操作类型的任务超时配置（构建：10 分钟、测试：15 分钟、部署：30 分钟）
  - macOS 兼容的超时包装器，带有 Perl 后备
  - 基于心跳的卡住代理检测
  - 长操作的看门狗模式
  - 使用 SIGTERM/SIGKILL 的优雅终止处理

### 更改
- 使用示例 PRD 和测试说明更新 README
- 测试与 macOS 兼容（当 `timeout` 命令不可用时使用基于 Perl 的超时后备）

## [1.1.0] - 2025-12-27

### 修复
- **macOS 兼容性** - 启动脚本现在可在 macOS 上运行：
  - 在 macOS 上使用 `uuidgen`，在 Linux 上回退到 `/proc/sys/kernel/random/uuid`
  - 修复了 macOS 的 `sed -i` 语法（使用 `sed -i ''`）

- **代理计数** - 修复 README 以显示正确的代理计数（37 个代理）

- **用户名占位符** - 将占位符用户名替换为实际的 GitHub 用户名

## [1.0.1] - 2025-12-27

### 更改
- README 格式的小更新

## [1.0.0] - 2025-12-27

### 新增
- **Loki Mode 技能的首次发布**，适用于 Claude Code

- **多代理架构** - 6 个群体中的 37 个专业代理：
  - 工程群体（8 个代理）：前端、后端、数据库、移动、API、QA、性能、基础设施
  - 运营群体（8 个代理）：devops、安全、监控、事件、发布、成本、SRE、合规
  - 业务群体（8 个代理）：营销、销售、财务、法律、支持、HR、投资者、合作伙伴
  - 数据群体（3 个代理）：ML、工程、分析
  - 产品群体（3 个代理）：PM、设计、技术文档
  - 增长群体（4 个代理）：黑客、社区、成功、生命周期
  - 审查群体（3 个代理）：代码、业务、安全

- **分布式任务队列**，具有：
  - 基于优先级的任务调度
  - 重试的指数退避
  - 失败任务的死信队列
  - 用于重复预防的幂等性密钥
  - 基于文件的原子操作锁定

- **断路器**用于故障隔离：
  - 每个代理类型的故障阈值
  - 自动冷却和恢复
  - 用于测试恢复的半开状态

- **8 个执行阶段**：
  1. 启动 - 初始化 `.loki/` 结构
  2. 发现 - 解析 PRD、竞争研究
  3. 架构 - 技术栈选择
  4. 基础设施 - 云配置、CI/CD
  5. 开发 - TDD 实施，并行代码审查
  6. QA - 14 个质量门控
  7. 部署 - 蓝绿、金丝雀发布
  8. 业务运营 - 营销、销售、法律设置
  9. 增长循环 - 持续优化

- **并行代码审查** - 3 个审查员同时运行：
  - 代码质量审查员
  - 业务逻辑审查员
  - 安全审查员

- **状态恢复** - 基于检查点的恢复，用于速率限制：
  - 自动检查点
  - 孤立任务检测和重新排队
  - 代理心跳监控

- **多平台部署支持**：
  - Vercel、Netlify、Railway、Render
  - AWS（ECS、Lambda、RDS）
  - GCP（Cloud Run、GKE）
  - Azure（Container Apps）
  - Kubernetes（清单、Helm 图表）

- **参考文档**：
  - `references/agents.md` - 完整的代理定义
  - `references/deployment.md` - 云部署指南
  - `references/business-ops.md` - 业务操作工作流程

[2.4.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.0.3...v2.1.0
[2.0.3]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.0.2...v2.0.3
[2.0.2]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.0.1...v2.0.2
[2.0.1]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v2.0.0...v2.0.1
[2.0.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v1.1.0...v2.0.0
[1.1.0]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v1.0.1...v1.1.0
[1.0.1]: https://github.com/asklokesh/claudeskill-loki-mode/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/asklokesh/claudeskill-loki-mode/releases/tag/v1.0.0
