# Vibe Kanban 集成

Loki Mode 可选择与 [Vibe Kanban](https://github.com/BloopAI/vibe-kanban) 集成，为自主执行提供可视化仪表板。

## 为什么将 Vibe Kanban 与 Loki Mode 一起使用？

| 功能 | 仅 Loki Mode | + Vibe Kanban |
|---------|-----------------|---------------|
| 任务可视化 | 基于文件的队列 | 可视化看板 |
| 进度监控 | 日志文件 | 实时仪表板 |
| 手动干预 | 编辑队列文件 | 拖放任务 |
| 代码审查 | 自动化 3 审查员 | + 可视化差异审查 |
| 并行代理 | 后台子代理 | 隔离的 git worktrees |

## 设置

### 1. 安装 Vibe Kanban

```bash
npx vibe-kanban
```

### 2. 在 Loki Mode 中启用集成

运行前设置环境变量：

```bash
export LOKI_VIBE_KANBAN=true
./scripts/loki-wrapper.sh ./docs/requirements.md
```

或创建 `.loki/config/integrations.yaml`：

```yaml
vibe-kanban:
  enabled: true
  sync_interval: 30  # 秒
  export_path: ~/.vibe-kanban/loki-tasks/
```

## 工作原理

### 任务同步流程

```
Loki Mode                          Vibe Kanban
    │                                   │
    ├─ 创建任务 ──────────────────► 任务出现在看板上
    │                                   │
    ├─ 代理声明任务 ─────────────► 状态："进行中"
    │                                   │
    │ ◄─────────────────── 用户暂停 ─┤（可选干预）
    │                                   │
    ├─ 任务完成 ────────────────► 状态："完成"
    │                                   │
    └─ 查看结果 ◄─────────────── 用户查看差异
```

### 任务导出格式

Loki Mode 以 Vibe Kanban 兼容格式导出任务：

```json
{
  "id": "loki-task-eng-frontend-001",
  "title": "实现用户身份验证 UI",
  "description": "创建带有验证的登录/注册表单",
  "status": "todo",
  "agent": "claude-code",
  "tags": ["eng-frontend", "phase-4", "priority-high"],
  "metadata": {
    "lokiPhase": "DEVELOPMENT",
    "lokiSwarm": "engineering",
    "lokiAgent": "eng-frontend",
    "createdAt": "2025-01-15T10:00:00Z"
  }
}
```

### 将 Loki 阶段映射到看板列

| Loki 阶段 | 看板列 |
|------------|---------------|
| BOOTSTRAP | 待办事项 |
| DISCOVERY | 计划中 |
| ARCHITECTURE | 计划中 |
| INFRASTRUCTURE | 进行中 |
| DEVELOPMENT | 进行中 |
| QA | 审查中 |
| DEPLOYMENT | 部署中 |
| BUSINESS_OPS | 完成 |
| GROWTH | 完成 |

## 导出脚本

添加此内容以将 Loki Mode 任务导出到 Vibe Kanban：

```bash
#!/bin/bash
# scripts/export-to-vibe-kanban.sh

LOKI_DIR=".loki"
EXPORT_DIR="${VIBE_KANBAN_DIR:-~/.vibe-kanban/loki-tasks}"

mkdir -p "$EXPORT_DIR"

# 导出待处理任务
if [ -f "$LOKI_DIR/queue/pending.json" ]; then
    python3 << EOF
import json
import os

with open("$LOKI_DIR/queue/pending.json") as f:
    tasks = json.load(f)

export_dir = os.path.expanduser("$EXPORT_DIR")

for task in tasks:
    vibe_task = {
        "id": f"loki-{task['id']}",
        "title": task.get('payload', {}).get('description', task['type']),
        "description": json.dumps(task.get('payload', {}), indent=2),
        "status": "todo",
        "agent": "claude-code",
        "tags": [task['type'], f"priority-{task.get('priority', 5)}"],
        "metadata": {
            "lokiTaskId": task['id'],
            "lokiType": task['type'],
            "createdAt": task.get('createdAt', '')
        }
    }

    with open(f"{export_dir}/{task['id']}.json", 'w') as out:
        json.dump(vibe_task, out, indent=2)

print(f"导出 {len(tasks)} 个任务到 {export_dir}")
EOF
fi
```

## 实时同步（高级）

对于实时同步，在 Loki Mode 旁边运行监视器：

```bash
#!/bin/bash
# scripts/vibe-sync-watcher.sh

LOKI_DIR=".loki"

# 监视队列更改并同步
while true; do
    # 在 macOS 上使用 fswatch，在 Linux 上使用 inotifywait
    if command -v fswatch &> /dev/null; then
        fswatch -1 "$LOKI_DIR/queue/"
    else
        inotifywait -e modify,create "$LOKI_DIR/queue/" 2>/dev/null
    fi

    ./scripts/export-to-vibe-kanban.sh
    sleep 2
done
```

## 结合使用的优势

### 1. 可视化进度跟踪
看板上看到所有 37 个 Loki 代理作为任务移动。

### 2. 安全隔离
Vibe Kanban 在隔离的 git worktrees 中运行每个代理，非常适合 Loki 的并行开发。

### 3. 人工介入选项
暂停自主执行，可视化查看更改，然后继续。

### 4. 多项目仪表板
如果对多个项目运行 Loki Mode，在一个 Vibe Kanban 实例中查看所有项目。

## 使用场景对比

| 场景 | 推荐 |
|----------|----------------|
| 完全自主、无监控 | 仅 Loki Mode + 包装器 |
| 需要可视化进度仪表板 | 添加 Vibe Kanban |
| 想要手动任务优先级排序 | 使用 Vibe Kanban 重新排序 |
| 合并前代码审查 | 使用 Vibe Kanban 的差异查看器 |
| 多个并发 PRD | Vibe Kanban 用于项目切换 |

## 未来集成想法

- [ ] 双向同步（Vibe → Loki）
- [ ] Vibe Kanban MCP 服务器用于代理通信
- [ ] 工具间共享的代理配置文件
- [ ] 统一日志仪表板
