---
name: frontend-ui-engineering
description: 构建在所有条件下都可工作的用户界面。当实现 React、Vue 或其他框架中的 UI 组件、页面或交互功能时使用。当需要响应式设计、可访问性、表单处理、错误状态或客户端状态管理时使用。
---

# 前端 UI 工程

## 概述

构建真实用户使用的真实界面。每个 UI 必须在缓慢的网络、陈旧的数据和意外的交互下都能工作。好的 UI 工程同时考虑愉快路径和出问题时发生什么——加载、错误、空状态和边界情况。

## 何时使用

- 构建或修改 UI 组件、页面或视图
- 实现表单、验证或客户端状态管理
- 添加交互行为（模态框、工具提示、拖放）
- 实现响应式设计或可访问性功能
- 处理加载、错误或空状态

**何时不使用：** 后端逻辑、API 设计、数据库查询、纯服务器端代码。

## 核心原则

### 1. 优先参考源代码

先检查项目现有代码。项目中的模式胜过一般最佳实践：

```bash
# 实现前检查什么
1. 项目是否已使用此组件/模式？找到并遵循它。
2. 项目是否使用组件库？用它，不要重新发明。
3. 项目是否有设计系统？遵循它的 token 和约定。
4. 项目是否有 API 客户端？用它，不要从 fetch() 开始。
```

`source-driven-development` 技能补充了这一点：当你需要在项目中尚未建立的模式时，获取官方文档而非依赖训练数据。

### 2. 始终处理所有状态

每个显示数据的 UI 有五种状态：

```
┌─────────────────────────────────────────┐
│                                         │
│   加载中 ──→ 成功（有数据）──→ 交互中  │
│     │              │            │       │
│     │              ▼            ▼       │
│     │           空状态         刷新中    │
│     │                                    │
│     ▼                                    │
│   错误                                   │
│                                         │
└─────────────────────────────────────────┘
```

**每个数据驱动组件必须处理：**

1. **加载中** — 骨架屏、旋转器或渐进式加载
2. **成功** — 带数据的已填充 UI
3. **空** — 有意义的空状态，不是空白页
4. **错误** — 带重试的错误信息
5. **交互中** — 提交中或刷新中

```tsx
function TaskList({ projectId }: { projectId: string }) {
  const { data, isLoading, error, refetch } = useTasks(projectId);

  if (isLoading) return <TaskListSkeleton />;
  if (error) return <ErrorState error={error} onRetry={refetch} />;
  if (!data?.length) return <EmptyState projectId={projectId} />;
  return <TaskListContent tasks={data} />;
}
```

### 3. 渐进增强

在最佳条件前先为最差条件构建：

- **慢网络：** 在 3G/4G 上加载和使用是否正常
- **无 JavaScript：** 关键内容是否可访问（服务器渲染）
- **禁用样式：** 内容是否可读
- **旧浏览器：** 是否优雅降级
- **无数据：** 空状态是否有帮助

### 4. 可访问性不是附加项

构建可访问的 UI 不是额外步骤——是你构建方式的一部分：

- 使用语义化 HTML 元素（`<button>` 而非可点击的 `<div>`）
- 每个交互元素都可键盘访问
- 颜色对比度满足 WCAG AA（4.5:1 正文，3:1 大文本）
- 图片有替代文本，图标有标签
- 表单输入有关联标签
- 动态内容变化向辅助技术公布

```tsx
// 好的：可访问的按钮
<button
  onClick={handleDelete}
  aria-label={`删除任务: ${task.title}`}
  disabled={isDeleting}
>
  {isDeleting ? '删除中...' : '删除'}
</button>

// 差的：不可访问的按钮
<div onClick={handleDelete} className="cursor-pointer">
  <TrashIcon />
</div>
```

## 组件设计模式

### 组件结构

```
components/
├── ui/                    # 基础组件（Button, Input, Modal）
│   ├── Button.tsx
│   ├── Button.test.tsx
│   └── index.ts
├── features/              # 功能组件（TaskList, UserProfile）
│   ├── tasks/
│   │   ├── TaskList.tsx
│   │   ├── TaskList.test.tsx
│   │   ├── TaskCard.tsx
│   │   └── index.ts
│   └── auth/
│       ├── LoginForm.tsx
│       ├── LoginForm.test.tsx
│       └── index.ts
└── layouts/               # 布局组件
    ├── MainLayout.tsx
    └── AuthLayout.tsx
```

### 属性设计

设计有用且安全的属性：

```tsx
// 好的：清晰的属性，安全默认值
interface TaskCardProps {
  task: Task;
  onStatusChange?: (id: string, status: TaskStatus) => void;
  onDelete?: (id: string) => void;
  variant?: 'default' | 'compact';
}

// 差的：过于宽泛的属性，不安全的默认值
interface TaskCardProps {
  task: any;
  onClick?: (...args: any[]) => void;
  options?: Record<string, unknown>;
}
```

### 复合组件

当组件有需要共享状态的相关部分时，使用复合组件：

```tsx
// 使用方式：
<Tabs>
  <Tabs.List>
    <Tabs.Trigger value="details">详细信息</Tabs.Trigger>
    <Tabs.Trigger value="comments">评论</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="details">...</Tabs.Content>
  <Tabs.Content value="comments">...</Tabs.Content>
</Tabs>
```

## 表单处理

### 表单原则

1. **客户端验证**以获得即时反馈
2. **服务器端验证**作为事实来源（永远不信任客户端）
3. **乐观更新**以获得感知速度（处理错误时回滚）
4. **防抖提交**以防止重复请求

### 表单模式

```tsx
function CreateTaskForm({ projectId }: { projectId: string }) {
  const form = useForm({
    initialValues: { title: '', description: '', assignee: '' },
    validate: {
      title: (v) => (!v?.trim() ? '标题是必填项' : undefined),
    },
    onSubmit: async (values) => {
      await createTask({ ...values, projectId });
    },
  });

  return (
    <form onSubmit={form.handleSubmit}>
      <Input
        label="标题"
        {...form.getFieldProps('title')}
        error={form.errors.title}
        required
      />
      <TextArea
        label="描述"
        {...form.getFieldProps('description')}
      />
      <Button type="submit" loading={form.isSubmitting}>
        创建任务
      </Button>
    </form>
  );
}
```

## 客户端状态管理

### 状态分层

```
┌─────────────────────────────────────┐
│  URL 状态          搜索参数、路由参数        │
│  持久、可分享、可书签                  │
├─────────────────────────────────────┤
│  服务器状态         API 数据缓存         │
│  React Query、SWR、Apollo           │
├─────────────────────────────────────┤
│  表单状态           输入值、验证错误        │
│  React Hook Form、Formik           │
├─────────────────────────────────────┤
│  UI 状态           模态框打开、选中的标签   │
│  useState、useReducer              │
└─────────────────────────────────────┘
```

**将状态放在正确的层级：**

- 能放 URL 吗？放 URL（筛选、分页、选中的标签）
- 是服务器数据吗？用服务器状态库（React Query、SWR）
- 是表单输入吗？用表单库
- 否则：`useState` 或 `useReducer`

### 缓存失效

当相关数据变更时让缓存失效：

```tsx
function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createTask,
    onSuccess: () => {
      // 让此查询的所有缓存失效
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

## 样式约定

### 设计 token

使用设计 token 实现一致的样式：

```css
/* token.css */
:root {
  /* 颜色 */
  --color-primary: #3b82f6;
  --color-danger: #ef4444;
  --color-surface: #ffffff;

  /* 间距 */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-4: 1rem;

  /* 排版 */
  --font-body: 'Inter', system-ui, sans-serif;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --leading-normal: 1.5;
}
```

### 响应式设计

移动优先，渐进增强：

```css
/* 移动优先基础样式 */
.card {
  padding: var(--space-4);
  margin-bottom: var(--space-4);
}

/* 平板及更大 */
@media (min-width: 768px) {
  .card {
    padding: var(--space-6);
  }
}

/* 桌面 */
@media (min-width: 1024px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 300px;
    gap: var(--space-6);
  }
}
```

## 性能

### 关键指标

优化 Core Web Vitals：

| 指标 | 目标 | 衡量内容 |
|------|------|----------|
| **LCP** | < 2.5s | 最大内容绘制 — 感知加载速度 |
| **INP** | < 200ms | 交互到下一次绘制 — 响应性 |
| **CLS** | < 0.1 | 累积布局偏移 — 视觉稳定性 |

### 性能模式

- **代码拆分** — 路由级别和重型组件的懒加载
- **图片优化** — 响应式图片、懒加载、现代格式（WebP、AVIF）
- **列表虚拟化** — 长列表只渲染可见项
- **防抖/节流** — 搜索输入、滚动处理、窗口大小调整
- **乐观更新** — 立即更新 UI，后台同步
- **骨架屏** — 即时视觉反馈而非旋转器

```tsx
// 路由级代码拆分
const TaskDashboard = lazy(() => import('./TaskDashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/tasks" element={<TaskDashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

## 错误边界

将错误隔离到组件树的一部分：

```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <ErrorFallback onReset={() => this.setState({ hasError: false })} />;
    }
    return this.props.children;
  }
}

// 在路由级别使用
<ErrorBoundary>
  <TaskDashboard />
</ErrorBoundary>
```

## 测试

### 组件测试层级

1. **行为测试** — 组件对用户交互做什么（首选）
2. **可访问性测试** — 组件是否可键盘访问，ARIA 是否正确
3. **快照测试** — 组件是否渲染了预期结构（谨慎使用）
4. **视觉回归测试** — 组件是否看起来正确（需要工具）

### 测试模式

```tsx
// 行为测试
it('shows error message and retry button when task loading fails', async () => {
  server.use(
    http.get('/api/tasks', () => HttpResponse.error())
  );

  render(<TaskList projectId="proj-1" />);

  expect(await screen.findByText(/无法加载任务/)).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /重试/ })).toBeInTheDocument();
});

// 交互测试
it('creates a new task when form is submitted', async () => {
  const user = userEvent.setup();
  render(<CreateTaskForm projectId="proj-1" />);

  await user.type(screen.getByLabelText('标题'), '新任务');
  await user.click(screen.getByRole('button', { name: '创建任务' }));

  expect(await screen.findByText('新任务')).toBeInTheDocument();
});
```

## 常见合理化说辞

| 合理化说辞 | 现实 |
|---|---|
| "稍后再加加载状态" | 你不会的。没有加载状态的 UI 在慢网络上是破损的。 |
| "可访问性可以最后做" | 最后做的可访问性意味着重做交互。构建时就要做。 |
| "错误不太可能发生" | 网络错误是最常见的前端错误。每个 API 调用都需要错误处理。 |
| "空状态不重要" | 空状态是你引导用户采取第一个行动的机会。空白页是死胡同。 |
| "这是一个内部工具" | 内部工具用户也是用户。他们也会遇到慢网络和错误。 |
| "我们的用户不用屏幕阅读器" | 你不知道所有用户的状况。而且可访问性有助于所有人（键盘导航、高对比度）。 |

## 危险信号

- 不处理加载、错误和空状态的 UI 组件
- 可点击但不是 `<button>` 元素
- 表单只有客户端验证
- 不为用户提供有用信息的全局错误边界
- 在 3G 上完全破损的 UI
- 没有标签或替代文本的图片/图标
- 无防抖的实时搜索
- 不处理竞态条件的数据获取
- 视觉上不同的组件代码几乎相同（提取共享模式）
- 代码几乎相同但视觉上不同的组件（检查设计是否真的需要不同，或只是样式偏离）

## 验证

构建任何 UI 组件后：

- [ ] 所有五种状态（加载、成功、空、错误、交互中）都已处理
- [ ] 组件可键盘访问
- [ ] 表单有客户端和服务器端验证
- [ ] 响应式设计在移动端、平板和桌面工作
- [ ] 图片有替代文本，图标有标签
- [ ] 在数据变更时缓存正确失效
- [ ] 乐观更新在出错时回滚
- [ ] LCP < 2.5s，INP < 200ms，CLS < 0.1
