# 无障碍检查清单

WCAG 2.1 AA 合规的快速参考。配合 `frontend-ui-engineering` 技能使用。

## 目录

- [基本检查](#基本检查)
- [常见 HTML 模式](#常见-html-模式)
- [测试工具](#测试工具)
- [快速参考：ARIA Live Regions](#快速参考aria-live-regions)
- [常见反模式](#常见反模式)

## 基本检查

### 键盘导航
- [ ] 所有交互元素可通过 Tab 键聚焦
- [ ] 焦点顺序遵循视觉/逻辑顺序
- [ ] 焦点可见（聚焦元素有轮廓/环）
- [ ] 自定义控件有键盘支持（Enter 激活，Escape 关闭）
- [ ] 没有键盘陷阱（用户始终可以从组件 Tab 出去）
- [ ] 页面顶部有跳至内容链接——在键盘聚焦时至少可见
- [ ] 模态框打开时捕获焦点，关闭时返回焦点

### 屏幕阅读器
- [ ] 所有图片有 `alt` 文本（装饰性图片用 `alt=""`)
- [ ] 所有表单输入有关联标签（`<label>` 或 `aria-label`）
- [ ] 按钮和链接有描述性文本（不是"点击这里"）
- [ ] 仅图标按钮有 `aria-label`
- [ ] 页面有一个 `<h1>`，标题不跳级
- [ ] 动态内容变化有通知（`aria-live` 区域）
- [ ] 表格有带 scope 的 `<th>` 标题

### 视觉
- [ ] 文本对比度 ≥ 4.5:1（普通文本）或 ≥ 3:1（大文本，18px+）
- [ ] UI 组件与背景对比度 ≥ 3:1
- [ ] 颜色不是传达信息的唯一方式
- [ ] 文本可缩放至 200% 不破坏布局
- [ ] 没有每秒闪烁超过 3 次的内容

### 表单
- [ ] 每个输入有可见标签
- [ ] 必填字段有标识（不仅靠颜色）
- [ ] 错误信息具体且与字段关联
- [ ] 错误状态通过多于颜色体现（图标、文本、边框）
- [ ] 表单提交错误有汇总且可聚焦
- [ ] 已知字段使用 autocomplete（例如 `type="email" autocomplete="email"`）

### 内容
- [ ] 声明语言（`<html lang="en">`）
- [ ] 页面有描述性 `<title>`
- [ ] 链接与周围文本有区分（不仅靠颜色）
- [ ] 移动端触摸目标 ≥ 44x44px
- [ ] 空状态有意义（不是空白页面）

## 常见 HTML 模式

### 按钮与链接

```html
<!-- 操作使用 <button> -->
<button onClick={handleDelete}>Delete Task</button>

<!-- 导航使用 <a> -->
<a href="/tasks/123">View Task</a>

<!-- 绝不要把 div/span 当按钮 -->
<div onClick={handleDelete}>Delete</div>  <!-- BAD -->
```

### 表单标签

```html
<!-- 显式标签关联 -->
<label htmlFor="email">Email address</label>
<input id="email" type="email" required />

<!-- 隐式包裹 -->
<label>
  Email address
  <input type="email" required />
</label>

<!-- 隐藏标签（优先使用可见标签） -->
<input type="search" aria-label="Search tasks" />
```

### ARIA 角色

```html
<!-- 导航 -->
<nav aria-label="Main navigation">...</nav>
<nav aria-label="Footer links">...</nav>

<!-- 状态消息 -->
<div role="status" aria-live="polite">Task saved</div>

<!-- 警告消息 -->
<div role="alert">Error: Title is required</div>

<!-- 模态对话框 -->
<dialog aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Delete</h2>
  ...
</dialog>

<!-- 加载状态 -->
<div aria-busy="true" aria-label="Loading tasks">
  <Spinner />
</div>
```

### 无障碍列表

```html
<ul role="list" aria-label="Tasks">
  <li>
    <input type="checkbox" id="task-1" aria-label="Complete: Buy groceries" />
    <label htmlFor="task-1">Buy groceries</label>
  </li>
</ul>
```

## 测试工具

```bash
# 自动化审计
npx axe-core          # 程序化无障碍测试
npx pa11y             # CLI 无障碍检查器

# 浏览器中
# Chrome DevTools → Lighthouse → Accessibility
# Chrome DevTools → Elements → Accessibility tree

# 屏幕阅读器测试
# macOS: VoiceOver (Cmd + F5)
# Windows: NVDA（免费）或 JAWS
# Linux: Orca
```

## 快速参考：ARIA Live Regions

| 值 | 行为 | 适用场景 |
|-------|----------|---------|
| `aria-live="polite"` | 在下次停顿时播报 | 状态更新、保存确认 |
| `aria-live="assertive"` | 立即播报 | 错误、时间敏感提醒 |
| `role="status"` | 同 `polite` | 状态消息 |
| `role="alert"` | 同 `assertive` | 错误消息 |

## 常见反模式

| 反模式 | 问题 | 修复 |
|---|---|---|
| `div` 当按钮 | 无法聚焦，无键盘支持 | 使用 `<button>` |
| 缺少 `alt` 文本 | 图片对屏幕阅读器不可见 | 添加描述性 `alt` |
| 仅靠颜色标识状态 | 色盲用户无法感知 | 添加图标、文本或图案 |
| 自动播放媒体 | 令人困惑，无法停止 | 添加控件，不自动播放 |
| 自定义下拉框无 ARIA | 键盘/屏幕阅读器无法使用 | 使用原生 `<select>` 或正确的 ARIA listbox |
| 移除焦点轮廓 | 用户看不到焦点位置 | 样式化轮廓，不要移除 |
| 空链接/按钮 | 播报"链接"但没有描述 | 添加文本或 `aria-label` |
| `tabindex > 0` | 打乱自然 Tab 顺序 | 仅使用 `tabindex="0"` 或 `-1` |