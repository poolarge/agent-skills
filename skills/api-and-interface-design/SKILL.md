---
name: api-and-interface-design
description: 设计稳健的 API 和接口。当创建新的 API 端点、定义接口、设计数据模型或构建 SDK/库时使用。当你需要定义系统或模块之间的契约时使用。
---

# API 与接口设计

## 概述

API 是系统之间的契约。好的 API 易于正确使用，难以错误使用。它们有清晰的类型、一致的约定、恰当的错误信息和允许演进的版本化策略。一旦 API 有消费者，在不破坏他们的情况下变更就变得昂贵——所以第一次就要做对。

## 何时使用

- 创建新的 REST 或 GraphQL API 端点
- 定义 TypeScript 接口或类型
- 设计数据库 schema 或数据模型
- 构建供外部使用的库、SDK 或包
- 在模块或服务之间创建内部 API
- 设计 Webhook 或事件载荷

**何时不使用：** 不影响公共接口的内部实现细节、一次性脚本。

## 设计原则

### 1. 契约优先设计

在实现之前定义契约。契约是消费者和提供者之间的协议——实现是细节。

```
契约定义：
1. 输入和输出类型
2. 错误类型和代码
3. 行为契约（幂等性、排序保证、副作用）
4. 版本化和向后兼容规则
```

### 2. 明确优于隐式

每个行为都应是预期且可发现的：

```typescript
// 好的：意图明确
await api.tasks.create({
  title: 'Buy groceries',
  assigneeId: 'user-123',
  priority: 'high',
});

// 差的：行为隐式且脆弱
await api.create({
  data: { title: 'Buy groceries', uid: 'user-123', pri: 3 },
  type: 'task',
});
```

### 3. 使正确用法容易，错误用法困难

API 应引导消费者走向成功：

```typescript
// 好的：不可能忘记必填字段
function createTask(input: {
  title: string;          // 必填 — 编译器强制
  description?: string;   // 选填
  priority?: TaskPriority; // 选填，默认为 'medium'
}): Promise<Task> { ... }

// 差的：所有东西都是选填的，运行时才知道错误
function createTask(input: Record<string, any>): Promise<any> { ... }
```

### 4. 向后兼容的变更

一旦 API 有消费者，变更必须是增量的：

| 安全（不破坏） | 破坏性（需要版本变更） |
|---|---|
| 添加新的选填字段 | 删除或重命名现有字段 |
| 添加新的端点 | 变更字段类型 |
| 添加新的枚举值* | 变更 URL 结构 |
| 添加新的响应头 | 变更错误格式 |

*新的枚举值对执行严格校验的消费者可能是破坏性变更。评估影响。

### 5. 一致性胜过独创性

在整个 API 中遵循相同的约定：

- 相同的资源命名（复数名词：`/tasks`、`/users`、`/projects`）
- 相同的错误格式（相同的结构，相同的代码）
- 相同的分页风格（基于游标或基于偏移——在整个 API 中选一种）
- 相同的过滤和排序约定
- 相同的认证模式

## REST API 设计

### URL 结构

```
GET    /api/v1/tasks              → 列出任务
POST   /api/v1/tasks              → 创建任务
GET    /api/v1/tasks/:id          → 获取单个任务
PATCH  /api/v1/tasks/:id          → 更新任务
DELETE /api/v1/tasks/:id          → 删除任务

GET    /api/v1/tasks/:id/comments → 列出任务的评论
POST   /api/v1/tasks/:id/comments → 添加评论到任务
```

### 请求/响应格式

```typescript
// 列出端点返回分页结果
interface ListResponse<T> {
  data: T[];
  pagination: {
    cursor?: string;     // 下一页的游标
    hasMore: boolean;    // 是否有更多结果
    total?: number;      // 总计数（可选，可能昂贵）
  };
}

// 单个资源端点返回包装资源
interface SingleResponse<T> {
  data: T;
}

// 错误响应使用一致格式
interface ErrorResponse {
  error: {
    code: string;        // 机器可读：'VALIDATION_ERROR'
    message: string;     // 人类可读：'Title is required'
    details?: Array<{    // 字段级验证错误
      field: string;
      code: string;
      message: string;
    }>;
  };
}
```

### HTTP 状态码

使用正确的状态码——它们是契约的一部分：

| 代码 | 含义 | 何时使用 |
|------|------|----------|
| 200 | 成功 | 成功的 GET、PATCH、DELETE |
| 201 | 已创建 | 成功的 POST |
| 204 | 无内容 | 成功的 DELETE（无响应体） |
| 400 | 错误请求 | 验证错误、格式错误的输入 |
| 401 | 未认证 | 缺少或无效的认证 |
| 403 | 禁止 | 认证但无权限 |
| 404 | 未找到 | 资源不存在 |
| 409 | 冲突 | 重复资源、乐观锁失败 |
| 422 | 无法处理 | 语义验证失败 |
| 429 | 请求过多 | 速率限制 |
| 500 | 服务器错误 | 意外的服务器故障 |

### 分页

**基于游标（对大数据集首选）：**

```typescript
// 请求
GET /api/v1/tasks?cursor=abc123&limit=20

// 响应
{
  data: [...],
  pagination: {
    cursor: "def456",    // 下一页的游标
    hasMore: true,
  }
}
```

**基于偏移（小数据集可接受）：**

```typescript
// 请求
GET /api/v1/tasks?page=2&pageSize=20

// 响应
{
  data: [...],
  pagination: {
    page: 2,
    pageSize: 20,
    total: 156,
    totalPages: 8,
  }
}
```

### 过滤和排序

```typescript
// 过滤 — 使用查询参数
GET /api/v1/tasks?status=pending&assigneeId=user-123

// 排序 — 单个参数带逗号分隔字段
GET /api/v1/tasks?sort=-createdAt,priority
// 负前缀 = 降序，正/无前缀 = 升序

// 字段选择 — 返回字段子集
GET /api/v1/tasks?fields=id,title,status
```

## TypeScript 接口设计

### 类型设计原则

```typescript
// 好的：具体类型，不可能有无效状态
type TaskStatus = 'pending' | 'in_progress' | 'completed';

interface Task {
  id: string;
  title: string;
  status: TaskStatus;
  createdAt: string;     // ISO 8601
  updatedAt: string;     // ISO 8601
  completedAt: string | null;  // 仅当 status = 'completed' 时存在
}

// 差的：宽泛类型，允许无效状态
interface Task {
  id: any;
  title?: string;
  status: string;
  dates: Record<string, Date>;
}
```

### 使无效状态不可表示

设计类型使得无效组合无法编译：

```typescript
// 好的：不可能有带空 completedAt 的已完成任务
type Task =
  | { status: 'pending' | 'in_progress'; completedAt: null }
  | { status: 'completed'; completedAt: string };

// 这无法编译：
const task: Task = { status: 'completed', completedAt: null }; // 类型错误！
```

### 品牌类型防止混淆

```typescript
// 好的：projectId 和 taskId 不可混淆
type ProjectId = string & { readonly __brand: 'ProjectId' };
type TaskId = string & { readonly __brand: 'TaskId' };

function getTask(id: TaskId): Promise<Task> { ... }
function getProject(id: ProjectId): Promise<Project> { ... }

// 这无法编译：
getTask(projectId); // 类型错误！
```

### 输入与输出类型

分离消费者发送的内容和消费者接收的内容：

```typescript
// 消费者发送
interface CreateTaskInput {
  title: string;
  description?: string;
  assigneeId?: string;
  priority?: TaskPriority;
}

// 消费者接收
interface Task {
  id: string;
  title: string;
  description: string | null;
  assigneeId: string | null;
  priority: TaskPriority;
  status: TaskStatus;
  createdAt: string;
  updatedAt: string;
}
```

### Result 类型用于错误

```typescript
// 好的：返回类型使错误显式
type Result<T, E = ApiError> =
  | { ok: true; data: T }
  | { ok: false; error: E };

async function createTask(input: CreateTaskInput): Promise<Result<Task>> {
  try {
    const task = await db.tasks.create(input);
    return { ok: true, data: task };
  } catch (error) {
    return { ok: false, error: toApiError(error) };
  }
}

// 使用方必须处理错误
const result = await createTask(input);
if (!result.ok) {
  // 处理错误
  return;
}
// 使用 result.data
```

## 错误设计

### 错误代码层次

```
VALIDATION_ERROR          → 通用验证失败
VALIDATION_REQUIRED       → 缺少必填字段
VALIDATION_FORMAT         → 格式不正确
VALIDATION_RANGE          → 超出允许范围

AUTH_ERROR                → 通用认证失败
AUTH_INVALID_TOKEN        → token 无效或过期
AUTH_INSUFFICIENT_SCOPE   → token 缺少所需权限

RATE_LIMIT_ERROR          → 通用速率限制
RATE_LIMIT_DAILY          → 达到每日限制
RATE_LIMIT_MINUTE         → 达到每分钟限制
```

### 错误响应

```typescript
// 好的：可操作的错误信息
{
  "error": {
    "code": "VALIDATION_REQUIRED",
    "message": "标题是必填项",
    "details": [
      {
        "field": "title",
        "code": "REQUIRED",
        "message": "标题不能为空"
      }
    ]
  }
}

// 差的：无用错误
{
  "error": "出错了",
  "status": 400
}
```

## 版本化策略

### URL 版本化（最简单）

```
/api/v1/tasks
/api/v2/tasks
```

- 每个版本是一个独立的路由
- V1 保持不变，直到明确弃用
- V2 可以有不同的结构

### 头部版本化（对 API 网关更清晰）

```
GET /api/tasks
Accept-Version: v2
```

### 向后兼容变更不需要新版本

添加新的选填字段、新的端点、新的响应头——这些是增量变更。

## SDK/库设计

### 符合习惯的 API

包装 API 的 SDK 应该感觉像是用宿主语言写的：

```typescript
// TypeScript SDK — 符合习惯
const task = await client.tasks.create({
  title: 'Buy groceries',
});
const tasks = await client.tasks.list({ status: 'pending' });

// 不符合习惯 — 只是原始 HTTP 调用
const task = await client.post('/tasks', { body: { title: 'Buy groceries' } });
```

### 优雅地处理认证

```typescript
// 构造函数注入 — 便于测试
const client = new ApiClient({
  baseUrl: 'https://api.example.com',
  authToken: () => getToken(),  // 函数，所以 token 可以刷新
});
```

### 类型是 SDK

TypeScript SDK 的类型*就是*文档。保持它们准确和完整：

```typescript
// 导出消费者需要的一切
export type { Task, TaskStatus, CreateTaskInput };
export { TaskClient };
```

## 常见合理化说辞

| 合理化说辞 | 现实 |
|---|---|
| "我们稍后加类型" | 你不会的。无类型的 API 会积累隐式依赖，之后不可能安全添加类型。 |
| "直接用 any 就行" | `any` 选择退出类型系统。每次使用都是等待发生的运行时错误。 |
| "版本化太复杂" | v2 比破坏所有 v1 消费者简单。URL 版本化实际上不复杂。 |
| "这只是一个内部 API" | 内部 API 比公共 API 更难变更——你不能控制消费者升级。 |
| "让他们看文档" | 类型*就是*文档。如果编译器无法强制执行，那就不是契约——那是建议。 |

## 危险信号

- 不验证输入的端点
- 返回 200 带错误信息的 API（使用正确的状态码）
- 所有东西都用 `any` 类型
- 没有版本化策略
- 不一致的错误格式
- 破坏向后兼容但不升版本的变更
- 端点在 URL 中暴露内部数据库 ID
- 响应没有一致的包装格式
- 没有速率限制的 API

## 验证

设计任何 API 或接口后：

- [ ] 契约先定义（类型/接口再实现）
- [ ] 输入和输出有明确的类型定义
- [ ] 错误使用一致的格式和代码
- [ ] 使用正确的 HTTP 状态码
- [ ] 列出端点支持分页
- [ ] 有版本化策略
- [ ] 类型使无效状态不可表示
- [ ] API 对消费者有文档（类型即文档）
- [ ] 破坏性变更需要版本变更
