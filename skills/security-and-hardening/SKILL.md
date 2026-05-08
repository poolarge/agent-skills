---
name: security-and-hardening
description: 保护代码和基础设施安全。当代码处理用户输入、认证、授权、加密或敏感数据时使用。当设置基础设施、配置访问控制或审计安全漏洞时使用。当安全事件发生或需要遵循合规要求时使用。
---

# 安全与加固

## 概述

默认安全，主动防御。每个系统都有攻击面——用户输入、API 端点、认证流程、数据存储。好的安全不是事后的补充；它内建在每次变更中。每一个接受输入的函数、每一条认证检查、每一条数据查询都是潜在漏洞。把每一个都当回事。

## 何时使用

- 代码处理用户输入或外部数据
- 实现认证或授权
- 处理敏感数据（PII、凭据、金融数据）
- 配置基础设施或部署
- 修复安全漏洞
- 设置 CI/CD 管道和访问控制
- 审计代码库的安全问题

**何时不使用：** 不处理外部输入、认证或敏感数据的纯内部逻辑。

## 安全原则

### 1. 零信任——永远不信任输入

每个外部输入都是恶意的，直到证明不是：

- 用户提交的表单数据
- URL 参数和查询字符串
- HTTP 头（包括 Content-Type）
- API 响应（甚至是"你自己的" API）
- 数据库记录（数据可能被注入）
- 文件上传（名称、类型和内容都不可信）
- WebSocket 消息
- 环境变量（可能被注入）

```typescript
// 好的：验证一切
function handleUserInput(input: unknown): Result<User> {
  // 1. 在类型层面验证
  const parsed = UserInputSchema.safeParse(input);
  if (!parsed.success) {
    return { ok: false, error: 'Invalid input' };
  }

  // 2. 清理输出
  const sanitized = {
    name: sanitizeHtml(parsed.data.name),
    email: parsed.data.email.toLowerCase().trim(),
  };

  return { ok: true, data: sanitized };
}

// 差的：盲目信任
function handleUserInput(input: any): User {
  return input; // XSS、注入、或任何恶意内容直接通过
}
```

### 2. 纵深防御

多层安全，这样一层失效不会让整个系统沦陷：

```
第 1 层：CDN/WAF          → 速率限制、IP 过滤、DDoS 防护
第 2 层：负载均衡器       → TLS 终止、请求大小限制
第 3 层：应用服务器       → 输入验证、认证、授权
第 4 层：业务逻辑         → 领域级权限检查
第 5 层：数据库           → 参数化查询、行级安全
第 6 层：存储             → 静态加密、访问控制
```

### 3. 最小权限

只授予完成工作所需的最少权限，仅此而已：

```typescript
// 好的：检查具体权限
if (!user.hasPermission('tasks:delete', taskId)) {
  throw new ForbiddenError('You cannot delete this task');
}

// 差的：检查宽泛角色
if (user.role !== 'admin') {
  throw new ForbiddenError('Admins only');
}
```

### 4. 安全失败

当安全检查失败时，安全地失败——拒绝访问、记录尝试、不泄露信息：

```typescript
// 好的：通用错误，不泄露信息
catch (error) {
  logger.warn('Authentication failed', { ip: req.ip });
  return res.status(401).json({ error: 'Authentication failed' });
}

// 差的：泄露为什么失败
catch (error) {
  return res.status(401).json({
    error: 'User not found',  // 信息泄露：告诉攻击者用户是否存在
  });
}
```

## 常见漏洞与防御

### 注入攻击（SQL、NoSQL、命令）

```typescript
// 好的：参数化查询
const tasks = await db.query(
  'SELECT * FROM tasks WHERE user_id = $1 AND status = $2',
  [userId, status]
);

// 差的：字符串拼接
const tasks = await db.query(
  `SELECT * FROM tasks WHERE user_id = '${userId}' AND status = '${status}'`
);
```

### XSS（跨站脚本）

```typescript
// 好的：渲染前清理，使用框架的自动转义
return <div>{userContent}</div>;  // React 自动转义

// 好的：当必须渲染 HTML 时
return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />;

// 差的：未清理的 HTML
return <div dangerouslySetInnerHTML={{ __html: userContent }} />;
```

### CSRF（跨站请求伪造）

```typescript
// 使用 CSRF token 的 POST/PUT/DELETE 请求
app.use(csrfProtection);

// 在表单中包含 token
<input type="hidden" name="_csrf" value={csrfToken} />
```

### 认证与会话管理

```typescript
// 好的：安全的会话配置
app.use(session({
  secret: process.env.SESSION_SECRET,  // 长、随机、轮换
  cookie: {
    httpOnly: true,      // JS 无法访问
    secure: true,        // 仅 HTTPS
    sameSite: 'strict',  // 无跨站请求
    maxAge: 3600000,     // 1 小时过期
  },
  resave: false,
  saveUninitialized: false,
}));
```

### 不安全的直接对象引用

```typescript
// 好的：验证所有权
app.get('/api/tasks/:id', auth, async (req, res) => {
  const task = await Task.findById(req.params.id);
  if (!task || task.userId !== req.user.id) {
    return res.status(404).json({ error: 'Not found' });
  }
  return res.json({ data: task });
});

// 差的：无所有权检查
app.get('/api/tasks/:id', auth, async (req, res) => {
  const task = await Task.findById(req.params.id);
  return res.json({ data: task });  // 任何已认证用户都能访问任何任务
});
```

### 敏感数据暴露

```typescript
// 好的：过滤响应中的敏感字段
function toPublicUser(user: User): PublicUser {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    // 不包含: passwordHash, ssn, creditCard
  };
}

// 好的：日志中不包含敏感数据
logger.info('User logged in', { userId: user.id });
// 不包含: logger.info('User logged in', { user })  → 会记录 passwordHash

// 好的：加密存储的敏感数据
const encrypted = await encrypt(sensitiveData, encryptionKey);
await db.secrets.create({ data: encrypted });
```

### 速率限制与暴力破解

```typescript
// 认证端点的速率限制
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 分钟
  max: 5,                     // 每 IP 每 15 分钟 5 次尝试
  message: 'Too many login attempts, please try again later',
});

app.post('/api/auth/login', loginLimiter, handleLogin);
```

## 基础设施安全

### 环境变量与密钥

```bash
# 好的：密钥在环境变量中，永远不在代码中
DATABASE_URL=postgresql://...
SESSION_SECRET=<long-random-string>
ENCRYPTION_KEY=<generated-key>

# 差的：硬编码在代码中
const DATABASE_URL = 'postgresql://user:pass@host:5432/db';  // 绝不这样做
```

**密钥管理规则：**
- 永远不要在代码中硬编码密钥
- 永远不要将 `.env` 文件提交到版本控制
- 使用密钥管理服务（AWS Secrets Manager、HashiCorp Vault）
- 轮换密钥——不要使用同一密钥多年
- 不同环境使用不同密钥

### 依赖安全

```bash
# 定期审计依赖
npm audit
yarn audit
pnpm audit

# 修复已知漏洞
npm audit fix

# 检查过时依赖
npm outdated
```

### Docker 安全

```dockerfile
# 好的：非 root 用户，最小镜像
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]

# 差的：root 用户，完整镜像
FROM node:22
COPY . .
RUN npm install
EXPOSE 3000
CMD ["node", "server.js"]  # 以 root 运行
```

### 网络安全

```
- 对所有外部流量使用 TLS 1.2+
- 内部服务通信使用 mTLS
- 数据库不接受公共连接
- 使用安全组/防火墙规则限制入站流量
- 出站流量仅限必要的外部端点
```

## 安全审查检查清单

### 代码审查

- [ ] 所有用户输入已验证和清理
- [ ] 所有数据库查询使用参数化语句
- [ ] 认证检查在每个受保护的路由上
- [ ] 授权检查验证具体权限，不仅是角色
- [ ] 敏感数据在响应中过滤
- [ ] 日志中不包含敏感数据
- [ ] 错误消息不泄露内部信息
- [ ] 认证端点有速率限制
- [ ] 文件上传有验证和大小限制

### 基础设施审查

- [ ] 无硬编码密钥或凭据
- [ ] `.env` 文件在 `.gitignore` 中
- [ ] 容器以非 root 用户运行
- [ ] 所有外部流量使用 TLS
- [ ] 数据库不接受公共连接
- [ ] 依赖已审计已知漏洞
- [ ] 密钥定期轮换
- [ ] 启用安全头（CSP、HSTS、X-Frame-Options）

### 合规检查（如适用）

- [ ] 数据保留策略已实施
- [ ] 用户同意跟踪已到位
- [ ] 数据导出/删除功能存在（GDPR）
- [ ] 访问日志已维护
- [ ] 灾难恢复计划已测试

## 事件响应

### 事件严重程度

| 级别 | 描述 | 示例 | 响应时间 |
|------|------|------|----------|
| **P0 严重** | 活跃利用、数据泄露 | 生产数据库暴露、RCE 被利用 | 立即 |
| **P1 高** | 可利用漏洞、凭据暴露 | S3 bucket 公开、API 密钥泄露 | < 1 小时 |
| **P2 中** | 潜在可利用 | 过时依赖、弱 CORS 配置 | < 24 小时 |
| **P3 低** | 安全债务 | 缺失安全头、详细错误消息 | 下个冲刺 |

### 事件响应流程

```
1. 遏制 — 停止出血（撤销密钥、阻断 IP、关闭端点）
2. 评估 — 什么被影响了？什么数据被暴露了？
3. 修复 — 修复漏洞
4. 验证 — 确认修复有效
5. 通知 — 通知受影响方（P0/P1 必须）
6. 复盘 — 记录发生原因和预防措施
```

### 事后复盘模板

```markdown
## 安全事后复盘：[事件标题]

### 严重程度：[P0/P1/P2/P3]

### 时间线
- [UTC] 检测
- [UTC] 确认
- [UTC] 遏制
- [UTC] 修复
- [UTC] 验证

### 影响
- 暴露的数据：
- 受影响的用户：
- 持续时间：

### 根本原因
- 漏洞的技术描述

### 发生原因
- 为什么漏洞被引入（缺失检查？配置错误？依赖问题？）

### 预防
- 防止类似事件的一个具体行动

### 经验教训
- 一个改善安全态势的收获
```

## 常见合理化说辞

| 合理化说辞 | 现实 |
|---|---|
| "这是内部 API" | 内部 API 也被攻击（SSRF、被入侵的内部主机）。内部不意味着可信。 |
| "我们稍后再加固" | 你不会的。安全债务像技术债一样累积，但后果更严重。 |
| "我们的用户不会那样做" | 攻击者不是你的用户。假设每个输入都是恶意的。 |
| "这只是管理员工具" | 被入侵的管理员工具 = 被入侵的系统。管理员工具需要*更多*安全，不是更少。 |
| "输入来自我们自己的前端" | 前端可以被绕过。任何 HTTP 客户端都可以调用你的 API。 |
| "框架处理了安全" | 框架提供工具，不是保证。你仍然需要正确使用它们。 |

## 危险信号

- 硬编码密钥、密码或连接字符串
- 没有 `.gitignore` 的 `.env` 文件
- 未经清理的用户输入在 HTML/SQL/命令中
- 没有 authentication 检查的 API 端点
- 返回敏感字段的 API 响应（密码哈希、SSN）
- 以 root 运行的 Docker 容器
- 依赖中有已知漏洞但未修复
- 无速率限制的认证端点
- 内部服务使用 HTTP 而非 HTTPS
- 日志中的敏感数据

## 验证

安全审查后：

- [ ] 所有用户输入已验证和清理
- [ ] 所有查询已参数化
- [ ] 认证和授权在每个受保护的路由上
- [ ] 敏感数据已过滤和加密
- [ ] 无硬编码密钥
- [ ] 依赖已审计
- [ ] 基础设施安全配置已验证
- [ ] 安全头已配置
- [ ] 速率限制已就位
