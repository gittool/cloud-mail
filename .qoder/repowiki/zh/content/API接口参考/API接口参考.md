# API接口参考

<cite>
**本文档引用的文件**   
- [email-api.js](file://mail-worker/src/api/email-api.js)
- [user-api.js](file://mail-worker/src/api/user-api.js)
- [login-api.js](file://mail-worker/src/api/login-api.js)
- [public-api.js](file://mail-worker/src/api/public-api.js)
- [email.js](file://mail-vue/src/request/email.js)
- [user.js](file://mail-vue/src/request/user.js)
- [login.js](file://mail-vue/src/request/login.js)
- [public.js](file://mail-vue/src/request/public.js)
</cite>

## 目录
1. [简介](#简介)
2. [API通用规范](#api通用规范)
3. [用户管理API](#用户管理api)
4. [邮件管理API](#邮件管理api)
5. [登录认证API](#登录认证api)
6. [系统设置API](#系统设置api)
7. [公共开放API](#公共开放api)
8. [前端调用链路示例](#前端调用链路示例)
9. [错误响应格式](#错误响应格式)
10. [分页策略](#分页策略)
11. [速率限制](#速率限制)
12. [API版本控制](#api版本控制)
13. [附录](#附录)

## 简介
本接口参考文档为cloud-mail系统的RESTful API提供完整的技术规范说明。文档涵盖用户、邮件、登录、系统设置等核心功能模块，详细定义各接口的HTTP方法、URL路径、请求参数、请求体结构、响应格式与状态码。特别说明JWT认证机制在请求头中的传递方式与权限校验逻辑。通过分析后端路由注册文件（如email-api.js）与前端请求封装文件（如request/email.js），展示从后端到前端的完整调用链路。文档同时包含错误处理、分页、速率限制等通用规则，并提供curl示例与JavaScript调用片段，帮助开发者快速集成。

## API通用规范

### 认证机制
所有需要身份验证的API接口均采用JWT（JSON Web Token）进行认证。客户端需在HTTP请求头中携带Authorization字段，格式如下：

```
Authorization: Bearer <JWT_TOKEN>
```

JWT令牌由登录接口生成，包含用户身份信息与权限声明，服务端通过`security/user-context.js`解析并验证令牌有效性。

### 权限校验逻辑
系统通过角色-权限模型（Role-Permission）实现细粒度访问控制。每个API请求在进入业务逻辑前，由安全中间件进行权限校验：
- 提取JWT中的用户角色信息
- 查询角色对应的权限列表（`role-service.js`）
- 验证当前请求的API是否在允许的权限范围内
- 若校验失败，返回403 Forbidden状态码

### 响应结构
所有API响应遵循统一的数据封装格式，由`model/result.js`定义：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

其中：
- `code`: 业务状态码，0表示成功
- `message`: 状态描述信息
- `data`: 实际返回的数据内容

**Section sources**
- [model/result.js](file://mail-worker/src/model/result.js#L1-L20)
- [security/user-context.js](file://mail-worker/src/security/user-context.js#L15-L40)

## 用户管理API

### 批量创建用户
- **HTTP方法**: POST
- **URL路径**: `/api/user/batch-create`
- **请求头**: `Authorization: Bearer <token>`
- **请求体**:
  ```json
  {
    "users": [
      {
        "username": "string",
        "email": "string",
        "password": "string"
      }
    ]
  }
  ```
- **响应状态码**:
  - 200: 创建成功
  - 400: 参数校验失败
  - 403: 无权限操作
  - 500: 服务器内部错误
- **权限要求**: ADMIN_ROLE

### 查询用户列表
- **HTTP方法**: GET
- **URL路径**: `/api/user/list`
- **请求头**: `Authorization: Bearer <token>`
- **查询参数**:
  - `page`: 页码（默认1）
  - `size`: 每页数量（默认10）
  - `keyword`: 搜索关键词
- **响应格式**: 分页数据结构
- **权限要求**: USER_MANAGE权限

**Section sources**
- [user-api.js](file://mail-worker/src/api/user-api.js#L25-L80)
- [user.js](file://mail-vue/src/request/user.js#L10-L50)

## 邮件管理API

### 发送邮件
- **HTTP方法**: POST
- **URL路径**: `/api/email/send`
- **请求头**: `Authorization: Bearer <token>`
- **请求体**:
  ```json
  {
    "to": ["user@example.com"],
    "cc": ["user2@example.com"],
    "subject": "邮件主题",
    "content": "邮件正文",
    "attachments": ["file_id_1"]
  }
  ```
- **响应状态码**:
  - 200: 邮件已接收并进入发送队列
  - 400: 参数缺失或格式错误
  - 401: 认证失败
  - 422: 邮件内容包含非法内容
- **异步处理**: 邮件发送由`email-service.js`异步处理，不阻塞API响应

### 查询邮件列表
- **HTTP方法**: GET
- **URL路径**: `/api/email/list`
- **请求头**: `Authorization: Bearer <token>`
- **查询参数**:
  - `page`, `size`: 分页参数
  - `type`: 邮件类型（inbox, sent, draft, trash）
  - `folderId`: 文件夹ID
- **响应格式**: 分页邮件列表
- **权限要求**: EMAIL_READ权限

**Section sources**
- [email-api.js](file://mail-worker/src/api/email-api.js#L30-L100)
- [email.js](file://mail-vue/src/request/email.js#L15-L60)

## 登录认证API

### 用户登录
- **HTTP方法**: POST
- **URL路径**: `/api/login`
- **请求体**:
  ```json
  {
    "username": "string",
    "password": "string"
  }
  ```
- **响应体**:
  ```json
  {
    "code": 0,
    "message": "success",
    "data": {
      "token": "jwt_token_string",
      "userInfo": {
        "id": "user_id",
        "username": "username",
        "email": "email",
        "roles": ["ROLE_USER"]
      }
    }
  }
  ```
- **状态码**:
  - 200: 登录成功
  - 401: 用户名或密码错误
  - 429: 登录尝试过于频繁

### Token刷新
- **HTTP方法**: POST
- **URL路径**: `/api/login/refresh-token`
- **请求头**: `Authorization: Bearer <expired_token>`
- **响应**: 返回新的JWT令牌
- **机制**: 支持过期Token的刷新，避免用户频繁重新登录

**Section sources**
- [login-api.js](file://mail-worker/src/api/login-api.js#L10-L50)
- [login.js](file://mail-vue/src/request/login.js#L5-L25)

## 系统设置API

### 获取系统配置
- **HTTP方法**: GET
- **URL路径**: `/api/setting/system`
- **请求头**: `Authorization: Bearer <token>`
- **权限要求**: ADMIN_ROLE
- **响应数据**: 包含SMTP配置、存储设置、安全策略等系统级参数

### 更新系统配置
- **HTTP方法**: POST
- **URL路径**: `/api/setting/update`
- **请求体**: 配置项键值对
- **审计日志**: 所有配置变更记录操作人与时间戳
- **权限要求**: ADMIN_ROLE

**Section sources**
- [setting-api.js](file://mail-worker/src/api/setting-api.js#L15-L60)

## 公共开放API

### 验证码发送
- **HTTP方法**: POST
- **URL路径**: `/api/public/send-verify-code`
- **请求体**:
  ```json
  {
    "email": "user@example.com",
    "type": "register|reset_password"
  }
  ```
- **速率限制**: 同一IP每小时最多10次
- **安全校验**: 集成Turnstile人机验证（`turnstile-service.js`）

### 用户注册
- **HTTP方法**: POST
- **URL路径**: `/api/public/register`
- **请求体**:
  ```json
  {
    "username": "string",
    "email": "string",
    "password": "string",
    "verifyCode": "string"
  }
  ```
- **邮箱验证**: 注册后需通过验证码完成邮箱绑定

**Section sources**
- [public-api.js](file://mail-worker/src/api/public-api.js#L20-L75)
- [public.js](file://mail-vue/src/request/public.js#L8-L35)

## 前端调用链路示例

### 后端路由注册（email-api.js）
后端使用Hono框架注册邮件相关API路由，定义请求处理逻辑与中间件链。每个路由绑定到具体的service方法，实现关注点分离。

### 前端请求封装（request/email.js）
前端通过axios实例封装邮件API调用，统一处理：
- 请求拦截：自动添加JWT头
- 响应拦截：统一错误处理与结果解包
- 超时设置：默认30秒超时
- baseURL配置：指向mail-worker服务地址

```javascript
// 示例：发送邮件调用
import emailApi from '@/request/email'

emailApi.sendEmail({
  to: ['test@example.com'],
  subject: '测试邮件',
  content: '这是一封测试邮件'
}).then(response => {
  console.log('发送成功', response.data)
}).catch(error => {
  console.error('发送失败', error.message)
})
```

**Section sources**
- [email-api.js](file://mail-worker/src/api/email-api.js#L1-L150)
- [email.js](file://mail-vue/src/request/email.js#L1-L80)

## 错误响应格式

所有错误响应遵循统一格式：

```json
{
  "code": 10001,
  "message": "用户名或密码错误",
  "data": null
}
```

常见错误码：
- `10000`: 参数校验失败
- `10001`: 认证失败
- `10002`: 权限不足
- `10003`: 资源不存在
- `10004`: 操作冲突（如用户名已存在）
- `20000`: 服务器内部错误

业务错误码定义在`error/biz-error.js`中，便于统一管理。

**Section sources**
- [biz-error.js](file://mail-worker/src/error/biz-error.js#L5-L100)
- [result.js](file://mail-worker/src/model/result.js#L25-L50)

## 分页策略

### 分页参数
所有支持分页的接口接受以下查询参数：
- `page`: 当前页码（从1开始）
- `size`: 每页记录数（最大100）

### 分页响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "list": [...],
    "total": 100,
    "page": 1,
    "size": 10
  }
}
```

### 默认与限制
- 默认页码：1
- 默认每页数量：10
- 最大每页数量：100（防止过度消耗资源）

**Section sources**
- [req-utils.js](file://mail-worker/src/utils/req-utils.js#L15-L40)

## 速率限制

系统对关键API实施速率限制，防止滥用：
- 登录接口：同一IP每小时最多10次尝试
- 验证码发送：同一IP每小时最多10次，同一邮箱每天最多5次
- 公共API：基于IP的滑动窗口限流

限流策略由`hono/webs.js`实现，基于Redis或内存存储计数。

**Section sources**
- [webs.js](file://mail-worker/src/hono/webs.js#L20-L60)

## API版本控制

### 版本策略
- 当前版本：v1（通过URL路径 `/api/v1/...` 体现）
- 向后兼容：保证同一主版本内接口兼容性
- 变更通知：重大变更通过开发者公告提前通知

### 兼容性保障
- 不删除已存在的字段
- 不改变现有字段的数据类型
- 新增功能通过新增端点或可选参数实现
- 弃用接口标记并保留至少6个月

**Section sources**
- [hono.js](file://mail-worker/src/hono/hono.js#L10-L30)

## 附录

### curl示例
```bash
# 发送邮件
curl -X POST https://api.cloud-mail.com/api/email/send \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "to": ["user@example.com"],
    "subject": "测试",
    "content": "这是一封测试邮件"
  }'
```

### JavaScript调用片段
```javascript
// 使用fetch调用登录API
async function login(username, password) {
  const response = await fetch('/api/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
  });
  const result = await response.json();
  if (result.code === 0) {
    localStorage.setItem('token', result.data.token);
  }
  return result;
}
```