# PRD：REST API 服务

## 概述
一个用于管理笔记的简单 REST API。测试 Loki Mode 的仅后端功能。

## 目标用户
需要笔记 API 的开发者。

## API 端点

### 笔记资源

#### GET /api/notes
- 返回所有笔记列表
- 响应：`[{ id, title, content, createdAt }]`

#### GET /api/notes/:id
- 返回单个笔记
- 响应：`{ id, title, content, createdAt }`
- 错误：404 如果未找到

#### POST /api/notes
- 创建新笔记
- 请求体：`{ title, content }`
- 响应：`{ id, title, content, createdAt }`
- 错误：400 如果验证失败

#### PUT /api/notes/:id
- 更新现有笔记
- 请求体：`{ title?, content? }`
- 响应：`{ id, title, content, updatedAt }`
- 错误：404 如果未找到

#### DELETE /api/notes/:id
- 删除笔记
- 响应：204 No Content
- 错误：404 如果未找到

### 健康检查

#### GET /health
- 返回 `{ status: "ok", timestamp }`

## 技术栈
- 运行时：Node.js 18+
- 框架：Express.js
- 数据库：内存（数组）以简化
- 验证：zod 或 joi
- 测试：Jest + supertest

## 要求
- 所有端点上的输入验证
- 正确的 HTTP 状态码
- JSON 错误响应
- 请求日志记录
- 每个端点的单元测试

## 超出范围
- 身份验证
- 数据库持久化
- 速率限制
- API 文档（OpenAPI）
- 部署

## 测试用例
```
POST /api/notes 使用有效数据 → 201 + 笔记对象
POST /api/notes 缺少标题 → 400 + 错误
GET /api/notes → 200 + 数组
GET /api/notes/:id 使用有效 id → 200 + 笔记
GET /api/notes/:id 使用无效 id → 404
PUT /api/notes/:id 使用有效数据 → 200 + 更新的笔记
DELETE /api/notes/:id → 204
GET /health → 200 + 状态对象
```

---

**目的：** 测试后端代理功能、代码审查和 QA，无前端复杂性。
