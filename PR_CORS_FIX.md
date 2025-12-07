# PR: fix: 统一 CORS 处理，修复预检请求 401/500 错误

## 问题背景

在 Obsidian（基于 Electron）或浏览器环境中调用 `/v1beta/models/gemini-2.5-flash:streamGenerateContent?alt=sse` 时，客户端会先发送 OPTIONS 预检请求。由于预检请求不携带认证头，导致以下问题：

1. **401 错误**：预检请求被认证守卫拦截
2. **500 错误**：流式响应尝试追加 CORS 头时，因 `headers` 不可变抛出异常

## 解决方案

在路由层统一处理 CORS，避免预检请求进入业务守卫链。

### 新增 `src/app/v1/_lib/cors.ts`

- 动态回显 `Origin` 和 `Access-Control-Request-Headers`
- 预检请求直接返回 `204 No Content`
- 添加 `Access-Control-Expose-Headers` 暴露 `x-request-id`、`x-ratelimit-*` 等响应头
- Vary 头用 `append` 追加，避免覆盖已有值

### 路由层集成

- `/v1` 和 `/v1beta` 路由统一调用 `registerCors(app)`
- 业务请求（POST/GET 等）仍完整走鉴权/限流/代理链

## 技术细节

**为什么需要处理 OPTIONS？**

Electron 应用和浏览器对跨域请求会自动发送 OPTIONS 预检。当请求携带自定义头（`Authorization`、`x-api-key`、`anthropic-version` 等）时触发。若预检被拒绝，实际业务请求不会发送。

**当前配置**：
```typescript
"Access-Control-Allow-Origin": "*"  // 允许所有来源
```

## 安全考虑

**安全性未降低**：
- 预检请求放行 ≠ 业务请求放行
- 实际 API 调用仍需有效的 API Key
- 原有认证/限流/会话守卫链完全不变

## 变更文件

- ✨ 新增：`src/app/v1/_lib/cors.ts`
- 🔧 修改：`src/app/v1/[...route]/route.ts`
- 🔧 修改：`src/app/v1beta/[...route]/route.ts`
