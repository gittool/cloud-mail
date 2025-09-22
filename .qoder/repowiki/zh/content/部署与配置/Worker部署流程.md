# Worker部署流程

<cite>
**本文档引用文件**  
- [index.js](file://mail-worker/src/index.js)
- [webs.js](file://mail-worker/src/hono/webs.js)
- [hono.js](file://mail-worker/src/hono/hono.js)
- [wrangler.toml](file://mail-worker/wrangler.toml)
- [wrangler-dev.toml](file://mail-worker/wrangler-dev.toml)
- [wrangler-test.toml](file://mail-worker/wrangler-test.toml)
- [wrangler-action.toml](file://mail-worker/wrangler-action.toml)
- [init.js](file://mail-worker/src/init/init.js)
- [i18n.js](file://mail-worker/src/i18n/i18n.js)
- [github-action.md](file://doc/github-action.md)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概览](#架构概览)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介
本文档提供Cloudflare Worker部署的完整操作指南，涵盖使用Wrangler CLI进行身份认证、环境初始化、资源绑定验证、版本发布等全流程。详细说明`wrangler.toml`配置文件中关键字段的作用，结合`index.js`入口文件与`webs.js`路由注册机制，解析Hono应用的构建与挂载流程，并涵盖自定义域名部署、预览环境创建、部署回滚等高级操作。同时提供日志解读、常见失败原因及解决方案，并给出性能优化建议。

## 项目结构
本项目包含前端Vue应用（`mail-vue`）和后端Worker服务（`mail-worker`），通过Wrangler统一构建与部署。静态资源由Worker托管，实现前后端一体化部署。

```mermaid
graph TB
subgraph "前端"
VueApp[mail-vue]
Build[打包输出 dist]
end
subgraph "后端"
Worker[index.js]
HonoApp[webs.js + hono.js]
APIs[API模块]
Services[业务服务]
Entities[数据实体]
end
subgraph "部署"
Wrangler[wrangler.toml]
Assets[静态资源绑定]
D1[D1数据库]
KV[KV命名空间]
R2[R2存储桶]
end
Build --> Assets
Worker --> HonoApp
HonoApp --> APIs
APIs --> Services
Services --> Entities
Wrangler --> Worker
Wrangler --> D1
Wrangler --> KV
Wrangler --> R2
```

**图示来源**  
- [wrangler.toml](file://mail-worker/wrangler.toml#L1-L40)
- [index.js](file://mail-worker/src/index.js#L1-L24)
- [webs.js](file://mail-worker/src/hono/webs.js#L1-L21)

**本节来源**  
- [mail-worker](file://mail-worker)
- [mail-vue](file://mail-vue)

## 核心组件
系统以Hono框架为核心构建REST API，通过`index.js`作为Worker入口，`webs.js`注册所有API路由，`wrangler.toml`定义部署配置。支持定时任务执行数据库清理与用户发送计数重置。

**本节来源**  
- [index.js](file://mail-worker/src/index.js#L1-L24)
- [webs.js](file://mail-worker/src/hono/webs.js#L1-L21)
- [wrangler.toml](file://mail-worker/wrangler.toml#L1-L40)

## 架构概览
系统采用模块化设计，前端Vue应用打包后由Worker通过`assets`绑定提供静态资源服务。API请求以`/api/`为前缀被路由至Hono应用处理，其余请求返回前端页面，支持单页应用（SPA）模式。

```mermaid
sequenceDiagram
participant Client as 客户端
participant Worker as Worker入口
participant Router as Hono路由
participant API as API模块
participant Service as 业务服务
participant DB as D1/KV/R2
Client->>Worker : 发送HTTP请求
Worker->>Worker : 判断路径是否以/api/开头
alt 是API请求
Worker->>Router : 重写路径并转发
Router->>API : 匹配对应API
API->>Service : 调用服务层
Service->>DB : 访问数据库或存储
DB-->>Service : 返回数据
Service-->>API : 返回结果
API-->>Router : 返回响应
Router-->>Client : JSON响应
else 静态资源请求
Worker->>Worker : 调用assets.fetch()
Worker-->>Client : 返回HTML/CSS/JS
end
```

**图示来源**  
- [index.js](file://mail-worker/src/index.js#L1-L24)
- [hono.js](file://mail-worker/src/hono/hono.js#L1-L33)
- [webs.js](file://mail-worker/src/hono/webs.js#L1-L21)

## 详细组件分析

### 入口文件分析
`index.js`是Cloudflare Worker的主入口，导出`fetch`处理器和`scheduled`定时任务处理器。所有API请求通过`/api/`前缀识别并转发至Hono应用，其他请求由静态资源处理器响应。

```mermaid
flowchart TD
Start([请求进入]) --> ParseURL["解析URL"]
ParseURL --> CheckAPI{"路径是否以/api/开头?"}
CheckAPI --> |是| Rewrite["重写路径去除/api"]
Rewrite --> Forward["转发至Hono应用"]
Forward --> HonoApp["Hono处理请求"]
HonoApp --> Response["返回JSON响应"]
CheckAPI --> |否| Static["调用assets.fetch()"]
Static --> ReturnHTML["返回前端页面"]
Response --> End([响应客户端])
ReturnHTML --> End
```

**图示来源**  
- [index.js](file://mail-worker/src/index.js#L1-L24)

**本节来源**  
- [index.js](file://mail-worker/src/index.js#L1-L24)

### 路由注册机制
`webs.js`通过导入各API模块实现路由自动注册。每个API模块在导入时向全局Hono实例注册其路由，无需手动挂载，实现松耦合的模块化设计。

```mermaid
classDiagram
class Hono {
+use()
+get()
+post()
+onError()
}
class WebS {
+import '../api/email-api'
+import '../api/user-api'
+...
}
class EmailAPI {
+app.get('/email')
+app.post('/email/send')
}
class UserAPI {
+app.get('/user')
+app.post('/user/login')
}
WebS --> Hono : 使用
EmailAPI --> Hono : 注册路由
UserAPI --> Hono : 注册路由
```

**图示来源**  
- [webs.js](file://mail-worker/src/hono/webs.js#L1-L21)
- [hono.js](file://mail-worker/src/hono/hono.js#L1-L33)

**本节来源**  
- [webs.js](file://mail-worker/src/hono/webs.js#L1-L21)

### 错误处理机制
系统在Hono应用中统一注册错误处理器，识别数据库未绑定等常见错误，并返回结构化JSON响应。同时支持业务异常（BizError）的日志记录。

```mermaid
flowchart TD
ErrorHappens[发生错误] --> ErrorHandler["app.onError()触发"]
ErrorHandler --> CheckType{"错误类型?"}
CheckType --> |BizError| LogOnly["仅记录日志"]
CheckType --> |其他错误| FullLog["完整错误日志"]
CheckType --> |KV未绑定| ReturnKVError["返回KV未绑定提示"]
CheckType --> |D1未绑定| ReturnD1Error["返回D1未绑定提示"]
ReturnKVError --> Response["返回502 JSON"]
ReturnD1Error --> Response
LogOnly --> Response
FullLog --> Response
Response --> Client["客户端收到错误"]
```

**图示来源**  
- [hono.js](file://mail-worker/src/hono/hono.js#L1-L30)

**本节来源**  
- [hono.js](file://mail-worker/src/hono/hono.js#L1-L33)

## 依赖分析
项目通过`wrangler.toml`配置文件声明对D1、KV、R2等Cloudflare资源的绑定，并在运行时通过`env`参数注入。前端依赖通过`build.command`自动安装与构建。

```mermaid
graph LR
A[wrangler.toml] --> B[D1数据库]
A --> C[KV命名空间]
A --> D[R2存储桶]
A --> E[静态资源目录]
A --> F[定时任务]
A --> G[构建命令]
G --> H[pnpm install]
G --> I[pnpm build]
H --> J[安装依赖]
I --> K[构建前端]
J --> K
K --> E
```

**图示来源**  
- [wrangler.toml](file://mail-worker/wrangler.toml#L1-L40)
- [wrangler-dev.toml](file://mail-worker/wrangler-dev.toml#L1-L30)
- [wrangler-test.toml](file://mail-worker/wrangler-test.toml#L1-L40)

**本节来源**  
- [wrangler.toml](file://mail-worker/wrangler.toml#L1-L40)
- [wrangler-dev.toml](file://mail-worker/wrangler-dev.toml#L1-L30)
- [wrangler-test.toml](file://mail-worker/wrangler-test.toml#L1-L40)

## 性能考虑
- **冷启动优化**：减少依赖数量，避免在全局作用域执行耗时操作。
- **缓存策略**：合理使用KV缓存高频数据，减少D1查询压力。
- **静态资源**：通过`assets`绑定直接返回前端资源，降低Worker执行频率。
- **定时任务**：使用`[triggers]`配置每日清理任务，避免手动触发延迟。

## 故障排除指南
### 常见部署失败原因
- **依赖未安装**：确保`build.command`正确执行`pnpm install`。
- **语法错误**：检查TypeScript编译是否通过，避免ES6+语法不兼容。
- **资源未绑定**：确认D1、KV、R2的`binding`名称与代码中一致。
- **环境变量缺失**：开发环境需在`wrangler-dev.toml`中配置`[vars]`。

### 日志解读
- `KV数据库未绑定`：检查`kv`绑定ID是否正确。
- `D1数据库未绑定`：确认`db`绑定配置无误。
- `JWTMismatch`：`/api/init/:secret`接口的`secret`参数与`jwt_secret`环境变量不匹配。

### 初始化失败处理
若初始化失败，可手动访问`https://your-domain/api/init/your-jwt-secret`重新初始化数据库结构。

**本节来源**  
- [hono.js](file://mail-worker/src/hono/hono.js#L1-L30)
- [init.js](file://mail-worker/src/init/init.js#L1-L47)
- [github-action.md](file://doc/github-action.md#L1-L22)

## 结论
本项目通过Wrangler实现Cloudflare Worker的一体化部署，结合Hono框架提供高效API服务。通过多环境配置文件支持开发、测试、生产等不同场景。建议在部署前充分验证资源配置与环境变量，确保系统稳定运行。