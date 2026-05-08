# 安全检查清单

Web 应用安全的快速参考。配合 `security-and-hardening` 技能使用。

## 目录

- [预提交检查](#预提交检查)
- [认证](#认证)
- [授权](#授权)
- [输入验证](#输入验证)
- [安全头](#安全头)
- [CORS 配置](#cors-配置)
- [数据保护](#数据保护)
- [依赖安全](#依赖安全)
- [错误处理](#错误处理)
- [OWASP Top 10 快速参考](#owasp-top-10-快速参考)

## 预提交检查

- [ ] 代码中没有密钥 (`git diff --cached | grep -i "password\|secret\|api_key\|token"`)
- [ ] `.gitignore` 包含：`.env`, `.env.local`, `*.pem`, `*.key`
- [ ] `.env.example` 使用占位值（不是真实密钥）

## 认证

- [ ] 密码使用 bcrypt（≥12 rounds）、scrypt 或 argon2 进行哈希
- [ ] Session cookies：`httpOnly`, `secure`, `sameSite: 'lax'`
- [ ] 配置了 session 过期时间（合理的 max-age）
- [ ] 登录端点有速率限制（15分钟内≤10次尝试）
- [ ] 密码重置 token：限时（≤1小时）、一次性
- [ ] 重复失败后锁定账户（可选，附带通知）
- [ ] 敏感操作支持 MFA（可选但推荐）

## 授权

- [ ] 每个受保护的端点都检查认证
- [ ] 每个资源访问都检查所有权/角色（防止 IDOR）
- [ ] 管理端点需要管理员角色验证
- [ ] API keys 限定为最小必要权限
- [ ] JWT tokens 经过验证（签名、过期时间、颁发者）

## 输入验证

- [ ] 所有用户输入在系统边界处验证（API 路由、表单处理器）
- [ ] 验证使用白名单（不是黑名单）
- [ ] 字符串长度有限制（min/max）
- [ ] 数字范围已验证
- [ ] Email、URL 和日期格式使用合适的库验证
- [ ] 文件上传：类型受限、大小受限、内容已验证
- [ ] SQL 查询使用参数化（不使用字符串拼接）
- [ ] HTML 输出已编码（使用框架自动转义）
- [ ] 重定向前验证 URL（防止开放重定向）

## 安全头

```
Content-Security-Policy: default-src 'self'; script-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  (disabled, rely on CSP)
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## CORS 配置

```typescript
// 严格模式（推荐）
cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})

// 生产环境中绝不使用：
cors({ origin: '*' })  // 允许任何来源
```

## 数据保护

- [ ] API 响应中排除敏感字段（`passwordHash`、`resetToken` 等）
- [ ] 不记录敏感数据（密码、token、完整信用卡号）
- [ ] PII 在存储时加密（如果法规要求）
- [ ] 所有外部通信使用 HTTPS
- [ ] 数据库备份已加密

## 依赖安全

```bash
# 审计依赖
npm audit

# 尽可能自动修复
npm audit fix

# 检查严重漏洞
npm audit --audit-level=critical

# 保持依赖更新
npx npm-check-updates
```

## 错误处理

```typescript
// 生产环境：通用错误，不泄露内部信息
res.status(500).json({
  error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' }
});

// 生产环境中绝不使用：
res.status(500).json({
  error: err.message,
  stack: err.stack,         // 泄露内部信息
  query: err.sql,           // 泄露数据库细节
});
```

## OWASP Top 10 快速参考

| # | 漏洞 | 防御措施 |
|---|---|---|
| 1 | 访问控制失效 | 每个端点做认证检查，验证所有权 |
| 2 | 加密失败 | HTTPS，强哈希，代码中无密钥 |
| 3 | 注入 | 参数化查询，输入验证 |
| 4 | 不安全设计 | 威胁建模，规范驱动开发 |
| 5 | 安全配置错误 | 安全头，最小权限，审计依赖 |
| 6 | 脆弱组件 | `npm audit`，保持依赖更新，最小依赖 |
| 7 | 认证失败 | 强密码，速率限制，session 管理 |
| 8 | 数据完整性失败 | 验证更新/依赖，签名构件 |
| 9 | 日志失败 | 记录安全事件，不记录密钥 |
| 10 | SSRF | 验证/白名单 URL，限制出站请求 |