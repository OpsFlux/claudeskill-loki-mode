# PRD：全栈演示应用

## 概述
一个展示 Loki Mode 端到端功能的完整全栈应用。一个带有标签的简单书签管理器。

## 目标用户
想要保存和组织书签的用户。

## 功能

### 核心功能
1. **添加书签** - 保存带有标题和可选标签的 URL
2. **查看书签** - 带有搜索/过滤的所有书签列表
3. **编辑书签** - 更新标题、URL 或标签
4. **删除书签** - 删除书签
5. **标签管理** - 创建、查看和按标签过滤

### 用户流程
1. 用户打开应用 → 看到书签列表
2. 点击"添加书签" → 出现表单
3. 输入 URL、标题、标签 → 提交
4. 书签出现在列表中
5. 可以按标签过滤或按标题搜索
6. 可以编辑或删除任何书签

## 技术栈

### 前端
- React 18 with TypeScript
- Vite 用于打包
- TailwindCSS 用于样式
- React Query 用于数据获取

### 后端
- Node.js 18+
- Express.js
- SQLite with better-sqlite3
- zod 用于验证

### 结构
```
/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── types/
│   │   └── App.tsx
│   ├── package.json
│   └── vite.config.ts
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   ├── db/
│   │   └── index.ts
│   ├── package.json
│   └── tsconfig.json
└── README.md
```

## API 端点

### 书签
- `GET /api/bookmarks` - 列出所有（查询：`?tag=`、`?search=`）
- `POST /api/bookmarks` - 创建新的
- `PUT /api/bookmarks/:id` - 更新
- `DELETE /api/bookmarks/:id` - 删除

### 标签
- `GET /api/tags` - 列出所有标签及计数

## 数据库架构
```sql
CREATE TABLE bookmarks (
  id INTEGER PRIMARY KEY,
  url TEXT NOT NULL,
  title TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tags (
  id INTEGER PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE bookmark_tags (
  bookmark_id INTEGER REFERENCES bookmarks(id),
  tag_id INTEGER REFERENCES tags(id),
  PRIMARY KEY (bookmark_id, tag_id)
);
```

## 要求
- 全面使用 TypeScript
- 前端和后端的输入验证
- 带用户反馈的错误处理
- 加载状态
- 空状态
- 响应式设计

## 测试
- 后端：Jest + supertest 用于 API 测试
- 前端：基本组件测试（可选）
- E2E：手动测试清单

## 超出范围
- 用户身份验证
- 导入/导出
- 浏览器扩展
- 云部署
- 实时同步

## 成功标准
- 所有 CRUD 操作正常工作
- 搜索和过滤正常工作
- 无控制台错误
- 测试通过
- 代码审查通过（所有 3 个审查员）

---

**目的：** Loki Mode 全功能的全面测试，包括前端、后端、数据库和代码审查代理。预期完整执行约 30-60 分钟。
