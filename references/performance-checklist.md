# 性能检查清单

Web 应用性能的快速参考检查清单。配合 `performance-optimization` 技能使用。

## 目录

- [Core Web Vitals 目标](#core-web-vitals-目标)
- [TTFB 诊断](#ttfb-诊断)
- [前端检查清单](#前端检查清单)
- [后端检查清单](#后端检查清单)
- [测量命令](#测量命令)
- [常见反模式](#常见反模式)

## Core Web Vitals 目标

| 指标 | 良好 | 需改进 | 较差 |
|--------|------|------------|------|
| LCP (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## TTFB 诊断

当 TTFB 较慢（> 800ms）时，在 DevTools Network 瀑布图中逐项检查：

- [ ] **DNS 解析**慢 → 为已知来源添加 `<link rel="dns-prefetch">` 或 `<link rel="preconnect">`
- [ ] **TCP/TLS 握手**慢 → 启用 HTTP/2，考虑边缘部署，验证 keep-alive
- [ ] **服务器处理**慢 → 分析后端，检查慢查询，添加缓存

## 前端检查清单

### 图片
- [ ] 图片使用现代格式（WebP、AVIF）
- [ ] 图片使用响应式尺寸（`srcset` 和 `sizes`）
- [ ] 图片和 `<source>` 元素有明确的 `width` 和 `height`（防止艺术方向场景下的 CLS）
- [ ] 非首屏图片使用 `loading="lazy"` 和 `decoding="async"`
- [ ] Hero/LCP 图片使用 `fetchpriority="high"` 且不延迟加载

### JavaScript
- [ ] 打包体积在 200KB gzipped 以内（初始加载）
- [ ] 路由和重型功能使用动态 `import()` 代码分割
- [ ] 启用 tree shaking（验证依赖提供 ESM 并标记 `sideEffects: false`）
- [ ] `<head>` 中没有阻塞式 JavaScript（使用 `defer` 或 `async`）
- [ ] 重计算卸载到 Web Workers（如适用）
- [ ] 使用相同 props 重渲染的昂贵组件使用 `React.memo()`
- [ ] `useMemo()` / `useCallback()` 仅在性能分析显示有收益的地方使用
- [ ] 长任务（> 50ms）分段处理以保持主线程可用——改善 INP 的主要手段
- [ ] 长时间运行循环中使用 `yieldToMain` 模式，使输入事件能在各段之间执行
- [ ] 在可用处使用现代调度 API：`scheduler.yield()`（优先）、带优先级的 `scheduler.postTask()`、`isInputPending()` 仅在需要时才让出
- [ ] `requestIdleCallback` 用于可延迟的非紧急工作（分析数据刷新、预取、预热）
- [ ] 非关键工作从事件处理器中延迟执行（例如分析、日志），以免延迟交互响应
- [ ] 第三方脚本使用 `async` / `defer` 加载，审计体积，重型脚本使用外观层（聊天 widget、嵌入）

### CSS
- [ ] 关键 CSS 内联或预加载
- [ ] 非关键样式没有渲染阻塞 CSS
- [ ] 生产环境没有 CSS-in-JS 运行时开销（使用提取）

### 字体
- [ ] 限制在 2–3 个字体族，每个 2–3 个字重（每增加一个字重就是额外一次请求）
- [ ] 仅使用 WOFF2 格式（最小、通用支持——跳过 WOFF/TTF/EOT）
- [ ] 尽可能自托管（第三方字体 CDN 增加 DNS + TCP + TLS 往返）
- [ ] LCP 关键字体预加载：`<link rel="preload" as="font" type="font/woff2" crossorigin>`
- [ ] `font-display: swap`（或非关键字体使用 `optional`）以避免 FOIT 阻塞渲染
- [ ] 通过 `unicode-range` 子集化，仅交付每页需要的字形
- [ ] 需要多个字重/样式时考虑可变字体（一个文件替代多个）
- [ ] 用 `size-adjust`、`ascent-override`、`descent-override` 调整回退字体度量，减少字体切换时的 CLS
- [ ] 在使用自定义字体前先考虑系统字体栈

### 网络
- [ ] 静态资源使用长 `max-age` + 内容哈希缓存
- [ ] API 响应在适当位置缓存（`Cache-Control`）
- [ ] 启用 HTTP/2 或 HTTP/3
- [ ] 为已知来源预连接资源（`<link rel="preconnect">`）
- [ ] 关键非图片资源使用 `fetchpriority`（例如关键 `<link rel="preload">`、首屏 `<script>`）——不仅限于 `<img>`
- [ ] 没有不必要的重定向

### 渲染
- [ ] 没有布局抖动（强制同步布局）
- [ ] 动画使用 `transform` 和 `opacity`（GPU 加速）
- [ ] 长列表使用虚拟化（例如 `react-window`）
- [ ] 没有不必要的全页面重渲染
- [ ] 非可见区域使用 `content-visibility: auto` 加 `contain-intrinsic-size` 跳过布局/绘制
- [ ] 没有 `unload` 事件处理器且 HTML 响应没有 `Cache-Control: no-store`——保持前进/后退缓存（bfcache）资格

## 后端检查清单

### 数据库
- [ ] 没有 N+1 查询模式（使用预加载 / joins）
- [ ] 查询有合适的索引
- [ ] 列表端点已分页（绝不 `SELECT * FROM table`)
- [ ] 配置了连接池
- [ ] 启用了慢查询日志

### API
- [ ] 响应时间 < 200ms (p95)
- [ ] 请求处理器中没有同步重计算
- [ ] 批量操作而非单次调用循环
- [ ] 响应压缩（gzip/brotli）
- [ ] 合适的缓存（内存、Redis、CDN）

### 基础设施
- [ ] 静态资源使用 CDN
- [ ] 服务器靠近用户（或边缘部署）
- [ ] 配置了水平扩展（如需要）
- [ ] 负载均衡器有健康检查端点

## 测量命令

### INP 真实用户数据和 DevTools 工作流

1. **先看真实用户数据**——在优化前查看 [CrUX Vis](https://developer.chrome.com/docs/crux/vis) 或你的 RUM 工具中的真实用户 INP
2. **识别慢交互**——打开 DevTools → Performance 面板 → 交互时录制；查看由点击/按键触发的长任务
3. **在中端 Android 设备上测试**——INP 问题通常只在较慢硬件上显现；使用真机或 DevTools CPU 节流（4×–6× 降速）

```bash
# Lighthouse CLI
npx lighthouse https://localhost:3000 --output json --output-path ./report.json

# 打包分析
npx webpack-bundle-analyzer stats.json
# 或 Vite：
npx vite-bundle-visualizer

# 检查打包体积
npx bundlesize

# 代码中的 Web Vitals
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);

# 带交互级别详情的 INP（归因构建）
import { onINP } from 'web-vitals/attribution';
onINP(({ value, attribution }) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } = attribution;
  console.log({ value, interactionTarget, inputDelay, processingDuration, presentationDelay });
});
```

## 常见反模式

| 反模式 | 影响 | 修复 |
|---|---|---|
| N+1 查询 | 数据库负载线性增长 | 使用 joins、includes 或批量加载 |
| 无边界查询 | 内存耗尽、超时 | 始终分页，添加 LIMIT |
| 缺少索引 | 数据增长后读取变慢 | 为过滤/排序列添加索引 |
| 布局抖动 | 卡顿、丢帧 | 批量 DOM 读取，然后批量写入 |
| 未优化图片 | LCP 慢、带宽浪费 | 使用 WebP、响应式尺寸、延迟加载 |
| 大打包体积 | Time to Interactive 慢 | 代码分割、tree shake、审计依赖 |
| 阻塞主线程 | INP 差、UI 无响应 | 用 `scheduler.yield()` / `yieldToMain` 分段长任务，卸载到 Web Workers |
| 内存泄漏 | 内存持续增长，最终崩溃 | 清理监听器、定时器、引用 |