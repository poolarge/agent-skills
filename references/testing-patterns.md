# 测试模式参考

常见测试模式的快速参考。配合 `test-driven-development` 技能使用。

## 目录

- [测试结构（Arrange-Act-Assert）](#测试结构arrange-act-assert)
- [测试命名约定](#测试命名约定)
- [常用断言](#常用断言)
- [Mock 模式](#mock-模式)
- [React/组件测试](#react组件测试)
- [API / 集成测试](#api--集成测试)
- [E2E 测试（Playwright）](#e2e-测试playwright)
- [测试反模式](#测试反模式)

## 测试结构（Arrange-Act-Assert）

```typescript
it('describes expected behavior', () => {
  // Arrange: 设置测试数据和前置条件
  const input = { title: 'Test Task', priority: 'high' };

  // Act: 执行被测试的操作
  const result = createTask(input);

  // Assert: 验证结果
  expect(result.title).toBe('Test Task');
  expect(result.priority).toBe('high');
  expect(result.status).toBe('pending');
});
```

## 测试命名约定

```typescript
// 模式: [单元] [预期行为] [条件]
describe('TaskService.createTask', () => {
  it('creates a task with default pending status', () => {});
  it('throws ValidationError when title is empty', () => {});
  it('trims whitespace from title', () => {});
  it('generates a unique ID for each task', () => {});
});
```

## 常用断言

```typescript
// 相等性
expect(result).toBe(expected);           // 严格相等 (===)
expect(result).toEqual(expected);        // 深度相等（对象/数组）
expect(result).toStrictEqual(expected);  // 深度相等 + 类型匹配

// 真值性
expect(result).toBeTruthy();
expect(result).toBeFalsy();
expect(result).toBeNull();
expect(result).toBeDefined();
expect(result).toBeUndefined();

// 数字
expect(result).toBeGreaterThan(5);
expect(result).toBeLessThanOrEqual(10);
expect(result).toBeCloseTo(0.3, 5);      // 浮点数

// 字符串
expect(result).toMatch(/pattern/);
expect(result).toContain('substring');

// 数组 / 对象
expect(array).toContain(item);
expect(array).toHaveLength(3);
expect(object).toHaveProperty('key', 'value');

// 错误
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(ValidationError);
expect(() => fn()).toThrow('specific message');

// 异步
await expect(asyncFn()).resolves.toBe(value);
await expect(asyncFn()).rejects.toThrow(Error);
```

## Mock 模式

### Mock 函数

```typescript
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ data: 'test' });
mockFn.mockImplementation((x) => x * 2);

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenCalledTimes(3);
```

### Mock 模块

```typescript
// Mock 整个模块
jest.mock('./database', () => ({
  query: jest.fn().mockResolvedValue([{ id: 1, title: 'Test' }]),
}));

// Mock 特定导出
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  generateId: jest.fn().mockReturnValue('test-id'),
}));
```

### 仅在边界处 Mock

```
Mock 这些:                    不要 Mock 这些:
├── Database calls             ├── 内部工具函数
├── HTTP requests              ├── 业务逻辑
├── File system operations     ├── 数据转换
├── External API calls         ├── 验证函数
└── Time/Date (when needed)    └── 纯函数
```

## React/组件测试

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';

describe('TaskForm', () => {
  it('submits the form with entered data', async () => {
    const onSubmit = jest.fn();
    render(<TaskForm onSubmit={onSubmit} />);

    // 通过可访问的角色/标签查找元素（不使用 test ID）
    await screen.findByRole('textbox', { name: /title/i });
    fireEvent.change(screen.getByRole('textbox', { name: /title/i }), {
      target: { value: 'New Task' },
    });
    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({ title: 'New Task' });
    });
  });

  it('shows validation error for empty title', async () => {
    render(<TaskForm onSubmit={jest.fn()} />);

    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    expect(await screen.findByText(/title is required/i)).toBeInTheDocument();
  });
});
```

## API / 集成测试

```typescript
import request from 'supertest';
import { app } from '../src/app';

describe('POST /api/tasks', () => {
  it('creates a task and returns 201', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: 'Test Task' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(201);

    expect(response.body).toMatchObject({
      id: expect.any(String),
      title: 'Test Task',
      status: 'pending',
    });
  });

  it('returns 422 for invalid input', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: '' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(422);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });

  it('returns 401 without authentication', async () => {
    await request(app)
      .post('/api/tasks')
      .send({ title: 'Test' })
      .expect(401);
  });
});
```

## E2E 测试（Playwright）

```typescript
import { test, expect } from '@playwright/test';

test('user can create and complete a task', async ({ page }) => {
  // 导航并认证
  await page.goto('/');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'testpass123');
  await page.click('button:has-text("Log in")');

  // 创建任务
  await page.click('button:has-text("New Task")');
  await page.fill('[name="title"]', 'Buy groceries');
  await page.click('button:has-text("Create")');

  // 验证任务出现
  await expect(page.locator('text=Buy groceries')).toBeVisible();

  // 完成任务
  await page.click('[aria-label="Complete Buy groceries"]');
  await expect(page.locator('text=Buy groceries')).toHaveCSS(
    'text-decoration-line', 'line-through'
  );
});
```

## 测试反模式

| 反模式 | 问题 | 更好的做法 |
|---|---|---|
| 测试实现细节 | 重构时会失败 | 测试输入/输出 |
| 对所有内容做快照测试 | 没人审查快照差异 | 断言具体值 |
| 共享可变状态 | 测试互相污染 | 每个测试独立 setup/teardown |
| 测试第三方代码 | 浪费时间，不是你的 bug | Mock 边界 |
| 为了通过 CI 而跳过测试 | 隐藏了真正的 bug | 修复或删除该测试 |
| 永久使用 `test.skip` | 死代码 | 删除或修复它 |
| 过于宽泛的断言 | 无法捕获回归 | 要具体 |
| 不处理异步错误 | 错误被吞掉，误报通过 | 始终 `await` 异步测试 |