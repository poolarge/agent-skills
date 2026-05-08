---
name: code-simplification
description: 简化过度复杂的代码。当重构不必要的复杂代码、减少抽象层数、消除死代码或使代码更易读而不改变行为时使用。
---

# 代码简化

## 概述

更简单的代码更容易理解、测试和维护。每一层抽象、每一个设计模式、每一个间接层都有成本——阅读、调试和修改的成本。在为假想场景过度工程化的代码与太简单无法演进的代码之间，始终选择更简单的选项。

## 何时使用

- 代码比需要的更难阅读
- 有只有一个实现的抽象（过度抽象）
- 函数做太多事（上帝函数）
- 一个概念变更需要修改太多文件（散弹式修改）
- 死代码、注释掉的代码或未使用的导入累积
- 你正想添加"以防万一"的抽象

**何时不使用：** 代码已经清晰简洁，或在简化会破坏必要行为的情况下。

## 简化原则

### 1. 三条规则

**在创建抽象之前等待第三个用例。** 两个相似的事物可以暂时重复。三个就是模式。

```
一个实现  → 直接写
两个实现  → 可以有共享工具函数，不要有抽象
三个实现  → 现在抽象是合理的
```

过早抽象比重复更难修复。重复是显而易见的；不必要的抽象会悄悄滋生。

### 2. 删除代码是最高杠杆的变更

能安全删除的每一行代码都是永远不需要阅读、测试或调试的代码。在添加抽象来组织代码之前，看看能否直接删除它。

### 3. 最少间接层

每个函数调用、接口、包装器和抽象层都是读者必须在脑海中追踪的间接层。最好的代码是从上到下阅读——没有跳跃，没有"这个接口的实现是什么？"的问题。

### 4. 明显优于聪明

```typescript
// 聪明但难阅读
const result = arr.reduce((acc, { type, value }) =>
  ({ ...acc, [type]: [...(acc[type] ?? []), value] }), {});

// 明显且易阅读
const result: Record<string, string[]> = {};
for (const item of arr) {
  if (!result[item.type]) {
    result[item.type] = [];
  }
  result[item.type].push(item.value);
}
```

两者都正确。第二个任何开发者 5 秒内就能理解。第一个需要你脑内执行 reduce。

### 5. 数据优于逻辑

当你可以用数据结构表达某事时，优先选择数据而非逻辑分支：

```typescript
// 逻辑驱动 — 每个新状态需要一个新的 case
function getStatusLabel(status: string): string {
  switch (status) {
    case 'pending': return '待处理';
    case 'in_progress': return '进行中';
    case 'completed': return '已完成';
    case 'cancelled': return '已取消';
    default: return status;
  }
}

// 数据驱动 — 新状态只需加一行
const STATUS_LABELS: Record<string, string> = {
  pending: '待处理',
  in_progress: '进行中',
  completed: '已完成',
  cancelled: '已取消',
};

function getStatusLabel(status: string): string {
  return STATUS_LABELS[status] ?? status;
}
```

## 常见简化模式

### 消除包装器

```typescript
// 不必要的包装器
function getUserById(id: string) {
  return userRepository.findById(id);
}

// 直接使用
// 调用方直接使用 userRepository.findById(id)
```

如果一个函数只是委托给另一个函数，没有添加价值（验证、转换、错误处理），就消除它。让调用方直接使用底层函数。

### 合并条件逻辑

```typescript
// 嵌套条件
if (user) {
  if (user.isActive) {
    if (user.hasPermission('write')) {
      performAction();
    }
  }
}

// 提前返回/守卫子句
if (!user || !user.isActive || !user.hasPermission('write')) {
  return;
}
performAction();
```

### 提取函数

```typescript
// 做太多事的函数
function processOrder(order: Order) {
  // 50 行验证逻辑
  // 30 行价格计算
  // 20 行库存检查
  // 40 行订单创建
}

// 每个做一件事的函数
function processOrder(order: Order) {
  validateOrder(order);
  const total = calculateTotal(order);
  checkInventory(order.items);
  return createOrder(order, total);
}
```

### 消除死代码

```
要删除的代码类型：
- 未使用的导入
- 未使用的变量和函数
- 注释掉的代码（git 记得）
- 不可达代码（return 之后的代码）
- 未使用的函数参数
- 过时的 TODO 注释（创建任务代替）
- 从未被访问的导出
```

### 用具名函数替代回调

```typescript
// 嵌套回调
fetchUser(id, (user) => {
  fetchOrders(user.id, (orders) => {
    fetchItems(orders[0].id, (items) => {
      // 深度嵌套，难阅读
    });
  });
});

// 具名函数（或 async/await）
async function loadUserDashboard(id: string) {
  const user = await fetchUser(id);
  const orders = await fetchOrders(user.id);
  const items = await fetchItems(orders[0].id);
  return { user, orders, items };
}
```

### 减少状态

```typescript
// 太多独立状态变量
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [data, setData] = useState<Task[] | null>(null);

// 单一状态机
type RequestState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Task[] }
  | { status: 'error'; error: string };

const [state, setState] = useState<RequestState>({ status: 'idle' });
```

## 重构安全

### 小步重构

一次做一个简化。每次变更后：

1. 运行测试
2. 确认行为未变
3. 提交

不要在重构的同时改变行为。重构改变结构；行为变更改变功能。把它们分开。

### 测试先于重构

在简化代码之前，确保它有测试。测试确认简化没有改变行为。如果没有测试，先写测试（使用 TDD 红色阶段），然后简化。

## 简化不是...

- **删除必要的复杂性** — 一些问题本身复杂。不要人为简化困难的事情。
- **移除所有注释** — 解释*为什么*的注释很有价值。删除的是陈述代码做什么的注释。
- **缩短变量名** — 描述性名称胜过短名称。`userAuthenticationService` 比 `uas` 好。
- **合并不同的关注点** — 只因为两个函数相似不代表它们应该是同一个。如果它们服务于不同的业务概念，保持分开。

## 常见合理化说辞

| 合理化说辞 | 现实 |
|---|---|
| "我们需要这个抽象以备将来" | 你不会需要的。当未来到来时，需求会不同。到时候再抽象。 |
| "这个模式是最佳实践" | 模式是工具，不是目标。如果模式增加了复杂性而没有增加价值，那它就不是"最佳"的。 |
| "代码需要灵活" | 灵活性有成本。80% 的"灵活"代码从未使用其灵活性。首先为当前需求构建。 |
| "这是企业级" | "企业级"是过度工程化的借口。真正的企业代码首先是可维护的。 |
| "其他开发者会期望这个模式" | 开发者期望可读的代码，不是设计模式目录。清晰胜过符合惯例。 |
| "DRY 原则要求" | DRY 是关于知识重复，不是代码重复。两个碰巧相似的独立概念不是违反 DRY。 |

## 危险信号

- 只有一个实现的接口
- 只被调用一次的函数
- 有 `// 以防万一` 注释的代码
- 需要跳转 5+ 个文件才能理解一个流程
- 有超过 3 个方法的类，每个方法只被一个地方调用
- 代码"感觉"太复杂但"可能需要"
- 配置驱动逻辑的配置比直接写代码更复杂
- 泛型或工具类型比使用它们的具体类型更复杂

## 验证

简化代码后：

- [ ] 行为未变（所有测试仍通过）
- [ ] 代码更易阅读（请他人验证）
- [ ] 没有添加不必要的抽象
- [ ] 死代码已删除
- [ ] 函数和变量有描述性名称
- [ ] 每次简化单独提交
