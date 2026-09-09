# octo-server PM Agent — 知识库

> 本知识库覆盖 octo-server 项目的九大知识领域，用于 AINOL Agent 实操考核。
> 每条结论附带 `来源: <相对路径>#L<起>-L<止>` 格式引用，可核验。

---

## 目录

1. [认证与身份](#1-认证与身份)
2. [鉴权模型](#2-鉴权模型)
3. [配置](#3-配置)
4. [业务模块清单](#4-业务模块清单)
5. [API与错误约定](#5-api与错误约定)
6. [IM控制面](#6-im控制面)
7. [Bot与Agent](#7-bot与agent)
8. [存储与外部依赖](#8-存储与外部依赖)
9. [构建与发布](#9-构建与发布)

---

# octo-server 知识库文档

> 本文档分析 `/root/.openclaw/workspace/octo-server` Go 项目，覆盖认证与身份、鉴权模型、配置、业务模块清单四大知识领域。每条结论附带源码引用。

---

## 1. 认证与身份

octo-server 有 **四条认证路径**，各自服务于不同的调用方和场景。

### 1.1 Token 认证（用户会话）

**核心机制：** 用户登录后，服务端生成随机 token 写入 Redis，后续请求通过 `token` 请求头或 `Authorization: Bearer` 头携带该 token，由 `AuthMiddleware` 解析。

**Token 解析链路：**

1. `BearerTokenCompat` 中间件将 `Authorization: Bearer <token>` 回填到 `token` 头（来源: `main.go#L283-L294`）
2. `CacheTokenParser.Parse` 从 Redis 读取 token 对应的 payload，解码为 `TokenInfo`（来源: `pkg/auth/parser.go#L114-L170`）
3. 若注入了 `TokenValidator`，则走 `Validate` 方法：读取 Redis payload + PTTL，校验 v3 token 的绝对过期时间和 session generation（来源: `pkg/auth/validator.go#L57-L113`）
4. `LanguageResolver` 实时解析用户语言偏好（Redis 热缓存 → DB），覆盖 token 中的快照（来源: `pkg/auth/parser.go#L131-L145`）
5. `RoleResolver` 实时解析用户系统角色（Redis 热缓存 → DB），防止降权后旧 token 仍携带 admin 角色（来源: `pkg/auth/parser.go#L147-L162`）

**Token 格式版本：**

- **v2 格式：** `v2:` 前缀 + JSON `{"uid","name","role","lang"}`（来源: `pkg/auth/tokeninfo.go#L96-L108`）
- **v3 格式：** `v3:` 前缀 + JSON 含 `issued_at`, `expires_at`, `session_generation`, `session_revision` 等安全字段（来源: `pkg/auth/tokeninfo.go#L110-L137`）
- **Legacy 格式：** `uid@name[@role]` 字符串，由旧版本写入，`Decode` 向后兼容（来源: `pkg/auth/tokeninfo.go#L159-L173`）

**Token 校验器（v3 安全特性）：**

- v3 token 要求有限 Redis TTL + 绝对过期时间 + session generation 匹配（来源: `pkg/auth/validator.go#L71-L93`）
- v1/v2 legacy token 在 rollout 策略下可继续读取，但 `SessionModeEnforce` 模式下会被拒绝（来源: `pkg/auth/session_v3.go#L290-L306`）

**用在哪：** 所有需要登录用户身份的 HTTP API 端点（`/v1/user/*`, `/v1/groups/*`, `/v1/space/*` 等），通过 `ctx.AuthMiddleware(r)` 挂载到路由组。

**关键文件：**
- Token 解析器：`pkg/auth/parser.go#L114-L170`
- Token 校验器：`pkg/auth/validator.go#L57-L113`
- Token 编解码：`pkg/auth/tokeninfo.go#L96-L173`
- Session v3 生命周期：`pkg/auth/session_v3.go#L195-L420`
- Session 运行时：`pkg/auth/runtime.go#L43-L120`
- 中间件注入：`main.go#L206-L230`

### 1.2 Bot Token 认证（Bot API）

**核心机制：** Bot 通过 `Authorization: Bearer <token>` 头携带 token，`authBot` 中间件按 token 前缀路由：

- **`bf_` 前缀 → User Bot：** 查询 `robot` 表（来源: `modules/bot_api/auth.go#L40-L55`）
- **`app_` 前缀 → App Bot：** 先查内存 Registry（O(1)），未命中再查 `app_bot` 表，写入缓存（来源: `modules/bot_api/auth.go#L57-L100`）

**用在哪：** 所有 `/v1/bot/*` 端点，包括 sendMessage、events、syncMessages、heartbeat、register 等（来源: `modules/bot_api/bot_api.go#L298-L435`）。

**关键文件：**
- 统一鉴权中间件：`modules/bot_api/auth.go#L26-L50`
- User Bot 认证：`modules/bot_api/auth.go#L40-L55`
- App Bot 认证：`modules/bot_api/auth.go#L57-L100`
- 路由注册：`modules/bot_api/bot_api.go#L298-L435`

### 1.3 User API Key 认证（`uk_` 前缀）

**核心机制：** 自动化客户端通过 `Authorization: Bearer uk_<...>` 头携带 User API Key，`authUserAPIKey` 中间件查询 `user_api_key` 表验证。

**流程：**
1. 提取 Bearer token，检查 `uk_` 前缀（来源: `modules/botfather/api_user.go#L36-L44`）
2. 查询 `apiKeyService.AuthByKey(token)` 验证 key 有效性（status=1）（来源: `modules/botfather/api_user.go#L45-L55`）
3. 检查 client_id 是否允许（botfather 自有 client 或已启用的 integration client）（来源: `modules/botfather/api_user.go#L56-L72`）
4. 将 `uid`、`api_key_uid`、`api_key_space_id` 写入 gin context（来源: `modules/botfather/api_user.go#L73-L78`）

**用在哪：** `/v1/user/bots/*` 端点（BotFather 的 User Bot 管理），以及通过 `authtree` 贡献到 TreeUserKey 的复用路由（如 `/v1/user/users/:uid`, `/v1/user/user/search` 等）。

**关键文件：**
- 认证中间件：`modules/botfather/api_user.go#L36-L80`
- 路由注册：`modules/botfather/api_user.go#L105-L121`
- Authtree 挂载：`modules/botfather/api_user.go#L121`

### 1.4 Webhook Token 认证（URL 路径内 token）

**核心机制：** Incoming Webhook 推送端点不使用 `AuthMiddleware`，而是将 token 嵌入 URL 路径：`/v1/incoming-webhooks/:webhook_id/:token`。

**流程：**
1. 请求到达推送路由（无 AuthMiddleware，四层限流由粗到细）（来源: `modules/incomingwebhook/api.go#L226-L260`）
2. handler 内通过 `webhook_id` + `token` 查询 DB，校验 token hash 匹配且 webhook 处于启用状态（来源: `modules/incomingwebhook/api.go#L1314-L1338`）
3. 鉴权失败的 IP 会被 `ipFailureGateMiddleware` 在打 DB 前快速切断（来源: `modules/incomingwebhook/api.go#L237-L241`）

**用在哪：** 所有 incoming webhook 推送形态（native、GitHub、企业微信、GitLab、飞书、Multica/Octo），共享同一组中间件链与限流桶（来源: `modules/incomingwebhook/api.go#L248-L260`）。

**关键文件：**
- 路由注册与限流链：`modules/incomingwebhook/api.go#L207-L260`
- 推送 handler 中的 token 校验：`modules/incomingwebhook/api.go#L1314-L1338`

### 1.5 内部认证（Internal Middleware）

部分模块有独立的内部认证中间件，用于服务间调用：
- `bot_mention` 的 `internalAuthMiddleware`（来源: `modules/bot_mention/api.go#L75`）
- `notify` 的 `internalAuthMiddleware`（来源: `modules/notify/api.go#L218`）
- `bot_task` 的 `sourceAuthMiddleware`（来源: `modules/bot_task/api.go#L211`）
- `internal_resolve` 的 `internalAuthMiddleware`（来源: `modules/internal_resolve/api.go#L162`）

### 1.6 无 Cookie / 无 WebSocket 握手

项目代码中**不存在 Cookie 认证路径**和 **WebSocket 握手认证路径**。IM 长连接由 WuKongIM 独立处理，octo-server 只负责 HTTP API 层的认证。`BearerTokenCompat` 中间件将标准 `Authorization: Bearer` 头适配为 octo-lib 的自定义 `token` 头（来源: `main.go#L283-L294`）。

---

## 2. 鉴权模型

### 2.1 系统级角色（Manager RBAC）

**角色定义：** 系统角色存储在 `user` 表的 `role` 字段中，由 `RoleResolver` 在每次请求时实时解析（Redis 热缓存 → DB），不依赖 token 中的快照。

**内置角色：**

| 角色 | 值 | 来源 |
|---|---|---|
| 普通用户 | `""` | 无角色 |
| 管理员 | `admin` | octo-lib `wkhttp.Admin` |
| 超级管理员 | `superAdmin` | octo-lib `wkhttp.SuperAdmin` |
| Dashboard 只读 | `dashboardReader` | `pkg/auth/manager_roles.go#L10` |
| Market 管理员 | `marketAdmin` | `pkg/auth/manager_roles.go#L36` |

**角色策略函数：**
- `IsManagerConsoleRole(role)` — 判断角色是否可建立管理控制台会话（来源: `pkg/auth/manager_roles.go#L43-L48`）
- `CanAdminMarketplace(role)` — 判断角色是否可管理平台市场（MCP/Skill/Expert）（来源: `pkg/auth/manager_roles.go#L54-L57`）
- `CanReadManagerDashboard(role)` — 判断角色是否可读运营仪表盘（来源: `pkg/auth/manager_roles.go#L63-L68`）

**角色覆盖关系：** `superAdmin` > `admin` > `dashboardReader` / `marketAdmin`。`dashboardReader` 和 `marketAdmin` 是**刻意不注册到 octo-lib `CheckLoginRole`** 的固定角色，因此不会意外获得 admin/superAdmin 端点权限（来源: `pkg/auth/manager_roles.go#L8-L12`）。

**关键文件：**
- 角色定义与策略：`pkg/auth/manager_roles.go#L1-L90`
- 角色实时解析：`pkg/auth/parser.go#L147-L162`
- 角色服务实现：`modules/user/role_service.go`

### 2.2 群组级 RBAC（Group Manager）

**机制：** 群管理员和创建者通过 `group_member` 表的 `role` 字段标识，由 `QueryIsGroupManagerOrCreator(groupNo, uid)` 查询。

**鉴权点举例：**
- 修改群信息：`isManager, err := g.db.QueryIsGroupManagerOrCreator(groupNo, loginUID)` → `if !isManager` 拒绝（来源: `modules/group/api.go#L1313-L1319`）
- 移除群成员：`if !isManager && loginUID != memberUID` 拒绝（来源: `modules/group/api.go#L3081-L3087`）
- 群全员禁言：`if !isManagerOrCreator` 拒绝（来源: `modules/group/api.go#L3843-L3849`）
- 设置 Bot 管理员：`if !isManagerOrCreator` 拒绝（来源: `modules/group/api.go#L4436-L4442`）
- `CheckLoginRole()` 检查系统级角色（非群级），用于系统管理端点（来源: `modules/group/api.go#L3183`）

**覆盖关系：** 群级 RBAC 与系统级 RBAC 独立运作。系统管理员（admin/superAdmin）通过 `CheckLoginRole()` 管理系统级端点；群管理员通过 `QueryIsGroupManagerOrCreator` 管理群级操作。两者不互相覆盖。

### 2.3 Space 级 ACL（SpaceMiddleware）

**机制：** `SpaceMiddleware` 是 opt-in 中间件，route group 级别注入。从请求的 `space_id` query 参数或 `X-Space-ID` 头提取 Space ID，校验当前用户是否为该 Space 的成员。

**流程：**
1. 提取 `space_id`（query 优先，header 其次）（来源: `pkg/space/middleware.go#L152-L157`）
2. 无 `space_id` 则跳过（opt-in 语义）（来源: `pkg/space/middleware.go#L158-L160`）
3. Redis 缓存查成员身份（正向 60s / 否定 30s）（来源: `pkg/space/middleware.go#L169-L175`）
4. 缓存未命中则查 DB `CheckMembership`（来源: `pkg/space/middleware.go#L183-L184`）
5. 非成员返回 403（来源: `pkg/space/middleware.go#L196-L198`）

**缓存失效：** `InvalidateMembershipCache` 在成员被移除时调用。DEL 失败时写否定缓存兜底（TTL 30s），防止正向条目活满 60s 导致隔离失效（来源: `pkg/space/middleware.go#L109-L130`）。

**用在哪：** 挂在需要 Space 隔离的路由组上，如 `/v1/user/pinned`（来源: `modules/user/api.go#L362`）、`/v1/user/space`（来源: `modules/user/api.go#L371`）、`/v1/space`（来源: `modules/space/api.go#L82`）、频道消息操作（来源: `modules/channel/api.go#L62`）等。

### 2.4 Authtree（非会话认证树）

**机制：** `pkg/authtree` 让同一个 handler 在多棵认证树下复用，每棵树有自己的凭证和租户语义。

**两棵树：**

| 树 | 凭证前缀 | 挂载方 | 租户语义 |
|---|---|---|---|
| `TreeUserKey` | `uk_` | modules/botfather | Space（冻结在 key 中） |
| `TreeBotToken` | `bf_` / `app_` | modules/bot_api | 无（bot 无 space_member 行） |

**租户作用域声明（Route.Tenant）：**
- `ScopeBoundSpace`：handler 读 `pkg/space.GetSpaceID`，MountOn 自动注入 `BoundSpaceContext`（来源: `pkg/authtree/authtree.go#L196-L199`）
- `ScopeRouteGuard`：handler 不读请求 Space，靠路由级 guard 拦截跨租户访问（来源: `pkg/authtree/authtree.go#L175-L183`）
- `ScopeUnscoped`：刻意不限制租户，handler 自有 friendship / group membership 等边界（来源: `pkg/authtree/authtree.go#L185-L189`）

**覆盖关系：** Authtree 树的中间件在 session AuthMiddleware **之后**或**替代**它。TreeUserKey 上的路由通过 `authUserAPIKey` 鉴权（非 session token），TreeBotToken 上的路由通过 `authBot` 鉴权。一棵树内的路由不受另一棵树的鉴权影响。

**关键文件：**
- Authtree 核心：`pkg/authtree/authtree.go#L1-L416`
- TreeUserKey 挂载：`modules/botfather/api_user.go#L105-L121`
- TreeBotToken 挂载：`modules/bot_api/bot_api.go#L438-L439`
- 贡献路由示例（user 模块）：`modules/user/api.go#L302-L314`

### 2.5 Bot / Agent 身份门禁

**Bot API 身份断言：** `authBot` 中间件写入 `robot_id`、`bot_kind`、`app_bot_scope` 等上下文键。`requireBotIdentity` 断言身份已写入，但**刻意不写 `c.Set("uid", robotID)`**，避免 handler 误用 bot ID 作为登录用户 UID（来源: `modules/bot_api/bot_api.go#L345-L360`）。

**App Bot Scope 门禁：** `appBotScopeGuard` 为复用路由补上 App Bot 授权规则：
- DM 读：scope=space 的 App Bot 须证明对端在其绑定 Space 内（来源: `modules/bot_api/authtree_guard.go`）
- 群/子区读：一律拒 App Bot（DM-only 策略）

**OBO（On-Behalf-Of）门禁：** OBO v2 fan-out 路径有多层门禁：friend-gate（好友关系校验）、channel-access（频道读权限校验）、group-membership（群成员校验），均通过 override seam 可测试（来源: `modules/bot_api/bot_api.go#L127-L175`）。

### 2.6 谁覆盖谁 — 总结

```
请求进入
  │
  ├─ 全局中间件：trace_id → i18n → tracer → HTTP metrics → access log → BearerTokenCompat → 全局 per-IP 限流
  │
  ├─ AuthMiddleware（session token）→ 写入 uid/role/language 到 context
  │    │
  │    ├─ SpaceMiddleware（opt-in）→ 校验 Space 成员身份
  │    │
  │    └─ CheckLoginRole() / CheckLoginRoleIsSuperAdmin() → 系统级 RBAC
  │
  ├─ authBot（bot token）→ 写入 robot_id/bot_kind，不走 AuthMiddleware
  │    │
  │    └─ requireBotIdentity → 断言 bot 身份
  │
  ├─ authUserAPIKey（uk_ token）→ 写入 uid/api_key_uid/api_key_space_id，不走 AuthMiddleware
  │    │
  │    └─ enforceKeySpace → 校验请求 Space 与 key 绑定 Space 一致
  │
  └─ Webhook URL token → 不走 AuthMiddleware，handler 内校验
```

---

## 3. 配置

### 3.1 `configs/tsdd.yaml` 配置段总览

| 配置段 | 用途 | 是否必填 | 来源 |
|---|---|---|---|
| `mode` | 运行模式（debug/release） | 是 | `configs/tsdd.yaml#L2` |
| `adminpwd` | 管理员密码 | 否 | `configs/tsdd.yaml#L3` |
| `addr` | API 监听地址 | 否（有默认） | `configs/tsdd.yaml#L4` |
| `grpcAddr` | Webhook gRPC 监听地址 | 否 | `configs/tsdd.yaml#L5` |
| `appName` | 项目名称 | 否 | `configs/tsdd.yaml#L6` |
| `rootDir` | 数据根目录 | 否 | `configs/tsdd.yaml#L7` |
| `messageSaveAcrossDevice` | 消息跨设备保存 | 否 | `configs/tsdd.yaml#L8` |
| `welcomeMessage` | 欢迎消息模板 | 否 | `configs/tsdd.yaml#L9` |
| `phoneSearchOff` | 关闭手机号搜索 | 否 | `configs/tsdd.yaml#L10` |
| `onlineStatusOn` | 开启在线状态 | 否 | `configs/tsdd.yaml#L11` |
| `groupUpgradeWhenMemberCount` | 群自动升级阈值 | 否 | `configs/tsdd.yaml#L12` |
| `eventPoolSize` | 事件池大小 | 否 | `configs/tsdd.yaml#L13` |
| `webhookSecretKey` | Webhook HMAC-SHA256 签名密钥 | 否（留空则不验证） | `configs/tsdd.yaml#L17-L19` |
| `wukongIM` | 悟空 IM 连接配置 | 否（但生产通常需要） | `configs/tsdd.yaml#L22-L24` |
| `db` | MySQL + Redis 连接 | 否（有默认值，但生产必须配置） | `configs/tsdd.yaml#L27-L34` |
| `external` | 外网访问配置 | 否 | `configs/tsdd.yaml#L37-L44` |
| `logger` | 日志配置 | 否 | `configs/tsdd.yaml#L47-L51` |
| `smsCode` | 测试短信验证码 | 否（release 模式禁止） | `configs/tsdd.yaml#L55` |
| `smsProvider` / `aliyunSMS` / `aliyunInternationalSMS` / `uniSMS` | 短信服务商 | 否 | `configs/tsdd.yaml#L56-L72` |
| `fileService` | 文件服务类型（minio/aliyunOSS/seaweedFS/qiniu/tencentCOS） | 否 | `configs/tsdd.yaml#L120` |
| `minio` | MinIO 配置 | 否（取决于 fileService） | `configs/tsdd.yaml#L121-L145` |
| `oss` | 阿里云 OSS 配置 | 否 | `configs/tsdd.yaml#L146-L151` |
| `seaweed` | SeaweedFS 配置 | 否 | `configs/tsdd.yaml#L152-L154` |
| `qiniu` | 七牛云配置 | 否 | `configs/tsdd.yaml#L155-L160` |
| `push` | 推送配置（APNs/HMS/小米/vivo/OPPO/Firebase） | 否 | `configs/tsdd.yaml#L163-L196` |
| `register` | 注册控制 | 否 | `configs/tsdd.yaml#L199-L202` |
| `account` | 内置系统账户 | 否 | `configs/tsdd.yaml#L205-L210` |
| `avatar` | 头像配置 | 否 | `configs/tsdd.yaml#L213-L217` |
| `shortNo` | 短号配置 | 否 | `configs/tsdd.yaml#L220-L223` |
| `robot` | 机器人配置 | 否 | `configs/tsdd.yaml#L226-L229` |
| `gitee` / `github` | 第三方 OAuth 登录 | 否 | `configs/tsdd.yaml#L232-L239` |
| `cache` | 缓存配置（token 前缀/过期时间等） | 否（有默认值） | `configs/tsdd.yaml#L242-L250` |

### 3.2 必填项

`configs/tsdd.yaml` 中**唯一未注释的必填项**是 `mode`（来源: `configs/tsdd.yaml#L2`）。`smsCode` 虽未注释但在 release 模式下会触发启动校验失败（来源: `configs/tsdd.yaml#L55`）。

其余所有配置项均以注释形式给出默认值。生产部署实际需要的配置项：
- `db.mysqlAddr` / `db.redisAddr` — 数据库和 Redis 连接
- `wukongIM.apiURL` / `wukongIM.managerToken` — IM 引擎连接
- `external.baseURL` — 外网访问地址
- `fileService` + 对应存储配置 — 文件服务

### 3.3 启动时校验

**配置加载流程：**
1. Viper 读取 YAML 文件（来源: `main.go#L102-L106`）
2. 设置环境变量前缀 `TS_`，支持环境变量覆盖（来源: `main.go#L143-L145`）
3. `validateTokenExpireConfig` 校验 `cache.tokenExpire`：必须是合法 Go duration，> 0，≥ 1ms，≤ 720h（30天）（来源: `main.go#L146-L148`, `internal/tokenlifecycle/config.go#L20-L52`）
4. `ValidateTestCodeConfig` 校验：release 模式下禁止配置 `smsCode`（万能验证码后门）（来源: `main.go#L160-L161`, `modules/base/common/testcode.go#L42-L48`）
5. `ValidateRuntimeLocales` 校验 i18n 运行时语言配置（来源: `main.go#L202-L203`）
6. Redis Lua 支持探测：`tokenStore.Probe(ctx)` 验证 Redis 支持 Lua 脚本（来源: `main.go#L218-L219`）
7. Session rollout 初始化：`InitializeSessionRollout` 在模块迁移后、HTTP 服务前运行，读取/创建 rollout 控制状态（来源: `pkg/auth/runtime.go#L73-L130`）

**关键文件：**
- 配置加载与校验入口：`main.go#L141-L161`
- Token 过期校验：`internal/tokenlifecycle/config.go#L20-L52`
- TestCode 校验：`modules/base/common/testcode.go#L42-L48`
- Session rollout 初始化：`pkg/auth/runtime.go#L73-L130`

---

## 4. 业务模块清单

### 4.1 通过 `register.AddModule` 注册的模块

| 模块名 | 目录 | 职责 | 来源 |
|---|---|---|---|
| `user` | `modules/user/` | 用户注册、登录、个人信息、设备管理、扫码登录、第三方 OAuth | `modules/user/1module.go#L29-L103` |
| `friend` | `modules/user/` | 好友关系管理（与 user 同文件注册） | `modules/user/1module.go#L105-L175` |
| `user_manager` | `modules/user/` | 用户管理后台（管理员端） | `modules/user/1module.go#L177-L186` |
| `group` | `modules/group/` | 群组创建、成员管理、群设置、群管理员、黑名单、扫码入群 | `modules/group/1module.go#L23-L53` |
| `group` (第二个注册) | `modules/group/` | 群相关的 IM 数据源和事件（与上面同名但不同注册块） | `modules/group/1module.go#L184` |
| `message` | `modules/message/` | 消息发送、历史、撤回、回复 | `modules/message/1module.go#L28-L38` |
| `conversation` | `modules/message/` | 会话列表、未读数 | `modules/message/1module.go#L40-L48` |
| `sidebar` | `modules/message/` | 侧边栏消息 | `modules/message/1module.go#L68-L100` |
| `conversation_ext_thread_auth` | `modules/message/` | 子区会话扩展鉴权（无 SetupAPI） | `modules/message/1module.go#L102` |
| `conversation_ext` | `modules/conversation_ext/` | 会话扩展功能 | `modules/conversation_ext/1module.go#L74-L90` |
| `conversation_ext_follow` | `modules/conversation_ext/` | 关注会话扩展 | `modules/conversation_ext/1module.go#L92-L106` |
| `channel` | `modules/channel/` | 频道信息查询、消息清理、故事线 | `modules/channel/1module.go#L17-L19` |
| `space` | `modules/space/` | Space（组织空间）创建、成员管理、邀请 | `modules/space/1module.go#L32-L42` |
| `space_manager` | `modules/space/` | Space 管理后台 | `modules/space/1module.go#L44-L54` |
| `botfather` | `modules/botfather/` | User Bot 生命周期管理、User API Key、文档端点 | `modules/botfather/1module.go#L17-L20` |
| `bot_api` | `modules/bot_api/` | Bot API 网关（消息发送/同步/事件/文件/卡片/OBO） | `modules/bot_api/1module.go#L14-L17` |
| `app_bot` | `modules/app_bot/` | App Bot 管理（创建/发布/连接/速率限制） | `modules/app_bot/1module.go#L14-L17` |
| `robot` | `modules/robot/` | 机器人服务和事件队列 | `modules/robot/1module.go#L18-L22` |
| `incomingwebhook` | `modules/incomingwebhook/` | 群入站 Webhook（推送/适配器/限流） | `modules/incomingwebhook/1module.go#L14-L22` |
| `file` | `modules/file/` | 文件上传/下载/预签名 URL | `modules/file/1module.go#L15-L19` |
| `common` | `modules/common/` | 公共 API（系统设置等） | `modules/common/1module.go#L16-L26` |
| `common` (manager) | `modules/common/` | 公共管理 API（无模块名） | `modules/common/1module.go#L28-L34` |
| `notify` | `modules/notify/` | 通知服务（卡片通知构建与降级） | `modules/notify/1module.go#L14-L20` |
| `notification` | `modules/notification/` | 通知模块 | `modules/notification/1module.go#L17-L22` |
| `webhook` | `modules/webhook/` | 出站 Webhook（消息事件推送到外部） | `modules/webhook/1module.go#L15-L20` |
| `thread` | `modules/thread/` | 子区（Thread）管理 | `modules/thread/1module.go#L25-L58` |
| `project` | `modules/project/` | 项目管理（Space 内的项目维度） | `modules/project/1module.go#L22-L26` |
| `qrcode` | `modules/qrcode/` | 二维码生成 | `modules/qrcode/1module.go#L9-L12` |
| `sticker` | `modules/sticker/` | 表情包管理 | `modules/sticker/1module.go#L14-L20` |
| `category` | `modules/category/` | 用户分类 | `modules/category/1module.go#L15-L20` |
| `search` | `modules/search/` | 搜索服务 | `modules/search/1module.go#L15-L20` |
| `messages_search` | `modules/messages_search/` | 消息搜索（Elasticsearch 后端） | `modules/messages_search/1module.go#L14-L18` |
| `backup` | `modules/backup/` | 数据备份 | `modules/backup/1module.go#L14-L18` |
| `report` | `modules/report/` | 举报功能 | `modules/report/1module.go#L18-L24` |
| `report_manager` | `modules/report/` | 举报管理后台 | `modules/report/1module.go#L31-L37` |
| `workplace` | `modules/workplace/` | 工作台 | `modules/workplace/1module.go#L17-L23` |
| `workplace_manager` | `modules/workplace/` | 工作台管理后台 | `modules/workplace/1module.go#L29-L35` |
| `oidc` | `modules/oidc/` | OIDC 身份认证集成 | `modules/oidc/1module.go#L14-L20` |
| `openapi` | `modules/openapi/` | OpenAPI 文档服务 | `modules/openapi/1module.go#L14-L20` |
| `opanalytics` | `modules/opanalytics/` | 运营分析 | `modules/opanalytics/1module.go#L17-L22` |
| `statistics` | `modules/statistics/` | 统计模块 | `modules/statistics/1module.go#L9-L12` |
| `bot_mention` | `modules/bot_mention/` | Bot @提及处理 | `modules/bot_mention/1module.go#L9-L13` |
| `bot_task` | `modules/bot_task/` | Bot 任务管理 | `modules/bot_task/1module.go#L9-L13` |
| `bot_provision` | `modules/bot_provision/` | Bot 预配 | `modules/bot_provision/1module.go#L9-L13` |
| `card_template_catalog` | `modules/card_template_catalog/` | 卡片模板目录 | `modules/card_template_catalog/1module.go#L14-L20` |
| `conversation_ext` | `modules/conversation_ext/` | 会话扩展（关注/置顶等） | `modules/conversation_ext/1module.go#L74-L90` |
| `integration` | `modules/integration/` | 集成管理 | `modules/integration/1module.go#L17-L22` |
| `internal_resolve` | `modules/internal_resolve/` | 内部身份解析（供其他服务调用） | `modules/internal_resolve/1module.go#L16-L21` |
| `usersecret` | `modules/usersecret/` | 用户密钥管理 | `modules/usersecret/1module.go#L14-L20` |
| `voice_adapter` | `modules/voice_adapter/` | 语音适配器（转写/上下文） | `modules/voice_adapter/1module.go#L15-L27` |
| `ai_team` | `modules/ai_team/` | AI 团队功能 | `modules/ai_team/1module.go#L14-L18` |
| `agentmailgateway` | `modules/agentmailgateway/` | Agent 邮件网关 | `modules/agentmailgateway/1module.go#L9-L13` |

### 4.2 未通过 `register.AddModule` 注册的目录（库模块）

以下目录存在于 `modules/` 下但**没有** `1module.go` 或未调用 `register.AddModule`，它们作为库被其他模块导入使用：

| 目录 | 用途 | 被谁使用 |
|---|---|---|
| `botidentity` | Bot 身份解析器 | `main.go` 直接导入（来源: `main.go#L30`, `main.go#L562`） |
| `cardtrust` | 卡片发送方身份判定（LRU 缓存） | `modules/messages_search`, `modules/webhook` 等导入 |
| `source` | 用户来源标签 | `modules/user` 导入 |
| `base` | 基础服务（app/event/common/elastic） | 多模块依赖 |

### 4.3 可能未启用 / 用不到的模块

以下模块虽然注册了，但在特定部署中可能不活跃：

- **`backup`** — 需要手动触发或配置定时任务，不参与常规请求处理。
- **`statistics`** — 无 SetupAPI，可能仅注册数据源不提供 HTTP 端点（来源: `modules/statistics/1module.go#L9-L12`，注册块中无 `SetupAPI`）。
- **`agentmailgateway`** — 依赖外部邮件服务配置，未配置时无实际效果。
- **`ai_team`** — 依赖 AI 后端服务配置，未配置时无实际效果。
- **`opanalytics`** — 运营分析，可能需要额外配置才激活。
- **`openapi`** — 仅提供 OpenAPI 文档，不影响业务逻辑。

### 4.4 模块间依赖关系（关键路径）

```
main.go
  ├─ modules/botidentity  (直接导入，IdentityResolver)
  ├─ modules/user         (登录/注册/用户管理)
  ├─ modules/bot_api      (Bot API 网关)
  ├─ modules/botfather    (User Bot 管理 + User API Key)
  ├─ modules/notify       (通知服务)
  ├─ modules/project      (项目管理)
  ├─ modules/common       (公共设置)
  ├─ modules/internal_resolve (内部身份解析)
  ├─ modules/card_template_catalog (卡片模板)
  ├─ pkg/auth             (Token 解析/校验/Session)
  ├─ pkg/authtree         (非会话认证树)
  ├─ pkg/space            (Space 中间件/成员校验)
  └─ ... (其他模块通过 register.AddModule 自动注册)
```

---

## 附录：关键文件索引

| 文件 | 行数 | 职责 |
|---|---|---|
| `main.go` | 1052 | 入口、配置加载、中间件装配、模块启动 |
| `pkg/auth/parser.go` | 227 | Token 解析（Redis → Decode → Resolver） |
| `pkg/auth/validator.go` | 143 | Token 校验（v3 TTL/generation） |
| `pkg/auth/tokeninfo.go` | 206 | Token 编解码（v2/v3/legacy） |
| `pkg/auth/session_v3.go` | ~600 | Session 生命周期（issue/revoke/reuse） |
| `pkg/auth/runtime.go` | ~200 | Session 运行时（Redis pool/rollout） |
| `pkg/auth/manager_roles.go` | 90 | 管理员角色定义与策略 |
| `pkg/authtree/authtree.go` | 416 | 非会话认证树（TreeUserKey/TreeBotToken） |
| `pkg/space/middleware.go` | 227 | Space 成员校验中间件 |
| `modules/user/api.go` | 5318 | 用户 API（登录/注册/设备/好友等） |
| `modules/bot_api/auth.go` | 187 | Bot API 鉴权 |
| `modules/bot_api/bot_api.go` | 532 | Bot API 路由 |
| `modules/botfather/api_user.go` | 783 | User API Key 认证与路由 |
| `modules/incomingwebhook/api.go` | 1821 | Incoming Webhook 推送 |
| `modules/group/api.go` | 5308 | 群组 API |
| `configs/tsdd.yaml` | 259 | 配置模板 |
| `internal/tokenlifecycle/config.go` | 52 | Token 过期校验 |
| `modules/base/common/testcode.go` | 50 | 测试验证码校验 |

# octo-server 知识库文档（Part 2：§5–§9）

> 本文档覆盖 octo-server Go 项目中 API 与错误约定、IM 控制面、Bot 与 Agent、存储与外部依赖、构建与发布五大知识领域。每条结论均附源码引用。

---

## §5. API 与错误约定

### 5.1 统一响应体

octo-server 的 HTTP 错误响应采用**兼容双格式**：同时输出新格式 `error` 对象和旧格式 `msg`/`status` 字段，确保旧客户端不 break。

错误响应 JSON 结构（由 `ErrorRenderer.Render` 统一生成）：

```json
{
  "error": {
    "code": "err.server.thread.not_found",
    "message": "Thread not found.",
    "details": { "field": "group_no" },
    "http_status": 404
  },
  "msg": "Thread not found.",
  "status": 400
}
```

- `error.code`：i18n 错误码 ID，全局唯一稳定 key（如 `err.server.thread.not_found`）
- `error.message`：经过 i18n 本地化翻译后的消息文案
- `error.details`：经过 `SafeDetailKeys` 白名单过滤后的安全详情
- `error.http_status`：语义 HTTP 状态码（响应 body 内的真实状态）
- `msg`：兼容旧客户端的消息字段
- `status`：兼容旧客户端的传输状态码

成功响应无统一包装，直接由 `c.Response(data)` 输出业务 JSON。

`来源: pkg/i18n/renderer.go#L30-L67`
`来源: pkg/httperr/respond.go#L22-L77`

### 5.2 错误码分类

所有 user-visible 错误码必须通过 `codes.Register` 注册，ID 遵循 `err.<namespace>.<domain>.<reason>` 命名约定：

- **`err.shared.*`**：跨模块通用错误（鉴权、限流、参数、内部），在 `pkg/i18n/codes/shared.go` 的 `init()` 中注册
- **`err.server.<module>.*`**：业务专属错误，在各模块对应的 `pkg/errcode/<module>.go` 文件中注册

命名规则由 `idPattern` 正则强制校验：`^err\.(shared|server)\.[a-z0-9_]+(\.[a-z0-9_]+)*$`

`Code` 结构体字段说明：

| 字段 | 作用 |
|---|---|
| `ID` | 全局唯一稳定 key |
| `HTTPStatus` | canonical HTTP status（401/403/429/...） |
| `DefaultMessage` | en-US 源文案 |
| `DefaultMessages` | 极端故障兜底翻译 |
| `SafeDetailKeys` | details 字段白名单，防止敏感信息泄露 |
| `Internal` | `true` 时不输出实际 message，改用占位文案 |

`来源: pkg/i18n/codes/registry.go#L28-L43`（命名规则）
`来源: pkg/i18n/codes/registry.go#L60-L77`（Code 结构体）
`来源: pkg/i18n/codes/registry.go#L79-L101`（Register 函数）
`来源: pkg/i18n/codes/shared.go#L22-L109`（shared 错误码注册）

shared 命名空间已注册的错误码包括：

| ID | HTTPStatus | 说明 |
|---|---|---|
| `err.shared.auth.required` | 401 | 需要登录 |
| `err.shared.auth.token_missing` | 401 | token 缺失 |
| `err.shared.auth.token_invalid` | 401 | token 无效 |
| `err.shared.auth.token_expired` | 401 | token 过期 |
| `err.shared.auth.forbidden` | 403 | 无权限 |
| `err.shared.rate.limited` | 429 | 限流 |
| `err.shared.param.invalid` | 400 | 参数无效 |
| `err.shared.not_found` | 404 | 资源不存在 |
| `err.shared.internal` | 500 | 内部错误（Internal=true） |

`来源: pkg/i18n/codes/shared.go#L22-L109`

server 命名空间按模块分文件注册，共 38 个文件、约 4591 行：

`来源: pkg/errcode/*.go`（38 个文件）

### 5.3 HTTP 状态码映射

存在**两种传输状态模式**，由调用方选择：

1. **`ResponseErrorL`**（兼容模式，默认）：HTTP 传输状态固定为 **400**，但 body 内 `error.http_status` 和 `status` 字段保留语义状态码。用于有旧客户端依赖 400 的遗留端点。

2. **`ResponseErrorLWithStatus`**（语义模式）：HTTP 传输状态等于 `Code.HTTPStatus`（如 404/409/410/429/500），body 内字段一致。用于无旧客户端的新端点（OIDC 绑定、Bot bind/unbind、通知暂停、Bot mention ingress 等）。

`来源: pkg/httperr/respond.go#L22-L48`（两个函数签名及文档）
`来源: pkg/httperr/respond.go#L57-L77`（respondL 实现，transportStatus 选择逻辑）

`ResponseErrorLWithStatus` 当前消费者列表（文档化在函数注释中）：

| 消费者 | 端点 |
|---|---|
| OIDC 自服务绑定 | `modules/oidc/api_bind.go` |
| Octo-link Bot bind/unbind | `modules/botfather/api_user.go` |
| 通知暂停 | `modules/notification/api.go` |
| Bot mention ingress | `modules/bot_mention/api.go` |

`来源: pkg/httperr/respond.go#L33-L48`

### 5.4 关键文件索引

| 文件 | 作用 |
|---|---|
| `pkg/i18n/codes/registry.go` | 错误码注册表核心（Code 结构、Register/Lookup/All） |
| `pkg/i18n/codes/shared.go` | `err.shared.*` 通用错误码注册 |
| `pkg/errcode/*.go` | `err.server.<module>.*` 业务错误码按模块分文件注册 |
| `pkg/httperr/respond.go` | `ResponseErrorL` / `ResponseErrorLWithStatus` 门面函数 |
| `pkg/i18n/renderer.go` | `ErrorRenderer` 实现，生成兼容双格式 JSON 响应 |

---

## §6. IM 控制面

### 6.1 octo-server 与 WuKongIM 的分工边界

octo-server 与 WuKongIM（悟空IM）构成**业务层 + 消息传输层**架构：

**WuKongIM 负责**（消息传输层）：
- WebSocket 长连接维护与消息推送
- 消息路由与分发
- 在线状态管理
- IM Token 签发与校验

**octo-server 负责**（业务层）：
- 消息持久化（通过 webhook 接收 IM 消息并入库）
- 群组/Space/Thread 业务逻辑
- 用户管理与好友关系
- Bot 注册与管理
- 推送通知（APNs/FCM/华为/小米等）

**IM → Server 方向（webhook 回调）**：

WuKongIM 通过两种方式回调 octo-server：

1. **gRPC webhook**：WuKongIM 调用 `WebhookService.SendWebhook` gRPC 接口，传递事件类型和数据。octo-server 启动 gRPC server 监听 `cfg.GRPCAddr`，处理 `EventMsgNotify`（消息通知）、`EventMsgOffline`（离线消息）、`EventOnlineStatus`（在线状态）三类事件。

2. **HTTP webhook**：WuKongIM 以 HTTP POST 调用 `/v1/webhook` 或 `/v2/webhook`，通过 query 参数 `event` 指定事件类型。也支持 `/v1/webhook/message/notify` 直达消息通知端点。

`来源: modules/webhook/api.go#L39-L54`（Webhook 结构体）
`来源: modules/webhook/api.go#L193-L211`（gRPC server 创建与注册）
`来源: modules/webhook/api.go#L217-L225`（SendWebhook 实现）
`来源: modules/webhook/api.go#L150-L158`（HTTP webhook 路由注册）
`来源: modules/webhook/api.go#L377-L392`（handleEvent 事件分发）

**Server → IM 方向（API 调用）**：

octo-server 通过 `config.Context` 的方法调用 WuKongIM API：

- `ctx.SendMessage()` — 发送消息，底层调用 WuKongIM 的消息发送 API
- `ctx.SendCMD()` — 发送 CMD 指令（如同步消息、刷新会话等）
- `ctx.UpdateIMToken()` — 更新 IM Token
- `ctx.EventBegin()` — 开启事件（配合 `wkevent` 包）

这些方法封装在 `octo-lib` 库的 `config.Context` 中，octo-server 通过依赖注入获取。

`来源: modules/message/api.go#L775`（SendMessage 调用示例）
`来源: modules/message/api.go#L1131`（SendCMD 调用示例）
`来源: modules/bot_api/register.go#L334-L341`（UpdateIMToken 调用示例）
`来源: modules/message/api.go#L1106-L1112`（EventBegin 调用示例）

### 6.2 wkevent 事件机制

`wkevent` 定义了事件类型枚举和事件持久化接口：

- `wkevent.Message` — 消息事件
- `wkevent.CMD` — CMD 事件
- `wkevent.None` — 无事件

`Event.Begin()` 在数据库事务内创建事件记录，`Event.Commit()` 在事务提交后异步投递到 WuKongIM。

`来源: pkg/wkevent/*.go`（事件类型定义和 Event 接口）

### 6.3 关键文件索引

| 文件 | 作用 |
|---|---|
| `modules/webhook/api.go` | webhook gRPC/HTTP 服务端，处理 IM 回调事件 |
| `pkg/wkhook/webhook.proto` | gRPC webhook 服务 protobuf 定义 |
| `pkg/wkhook/webhook.pb.go` | protobuf 生成代码 |
| `pkg/wkhook/webhook_grpc.pb.go` | gRPC 生成代码 |
| `pkg/wkevent/*.go` | 事件类型定义和 Event 接口 |
| `configs/tsdd.yaml` | WuKongIM 连接配置（apiURL, managerToken） |

---

## §7. Bot 与 Agent

### 7.1 四大 Bot 模块的关系

octo-server 中有四个核心 Bot 模块，各自职责不同：

#### `app_bot`（应用 Bot 管理）

**职责**：管理 App Bot 的全生命周期 CRUD（创建、查询、更新、删除、发布/下线、Token 轮换）。

**特点**：
- App Bot token 前缀为 `app_`，UID 格式为 `app_<id>_bot`
- 支持两种 scope：`platform`（平台级）和 `space`（Space 级）
- 管理端点走 `/v1/admin/app_bot`（管理员），业务端点走 `/v1/space/:space_id/app_bot`（Space 管理员/所有者）
- 有自己的 DB 表 `app_bot` 和独立的 SQL 迁移

`来源: modules/app_bot/app_bot.go#L27-L34`（token/UID 前缀常量）
`来源: modules/app_bot/app_bot.go#L117-L137`（路由注册）
`来源: modules/app_bot/1module.go#L8-L22`（模块注册）

#### `botfather`（Bot 管理中枢）

**职责**：User Bot 的创建/管理/命令处理，User API Key 认证与授权，Robot Apply 审批流程，欢迎消息，文档端点。

**特点**：
- 原 Bot API 端点（`/v1/bot/*`）已迁移至 `bot_api` 模块
- 现保留：User Bot CRUD（`/v1/user/bots/*`）、Robot Apply（`/v1/robot/apply*`）、文档端点（`/v1/bot/skill.md` 等）、Runtime onboarding（`/v1/runtime-onboarding`）
- User API Key 认证（`uk_` 前缀）供外部 Agent/CLI 使用
- 监听消息事件和用户注册事件，发送欢迎消息
- 管理 BotFather 系统用户

`来源: modules/botfather/api.go#L29-L51`（BotFather 结构体和构造函数）
`来源: modules/botfather/api.go#L84-L106`（路由注册，含迁移说明）
`来源: modules/botfather/api_user.go#L105-L127`（User API Key 路由）
`来源: modules/botfather/api_apply.go#L526-L534`（Apply 路由）
`来源: modules/botfather/1module.go#L12-L25`（模块注册）

#### `bot_provision`（Bot 供给接口）

**职责**：为 octo-fleet/daemon 提供 Bot 铸造和 Token 获取的跨服务接口。

**端点**：
- `POST /v1/bot/mint` — Web session 认证，铸造 Bot OBO 身份，返回 `bot_uid`
- `GET /v1/bot/:uid/token` — Daemon api_key Bearer 认证，返回 `bot_token`

**设计决策**：放在 `bot_provision` 而非 `botfather`，因为它是 octo-server 与 octo-fleet/daemon 之间的新契约面，便于未来弃用 botfather 的旧 runtime/bot API。

`来源: modules/bot_provision/bot_api.go#L1-L16`（模块设计说明）
`来源: modules/bot_provision/bot_api.go#L48-L96`（mintBot 实现）
`来源: modules/bot_provision/bot_api.go#L194-L199`（路由注册）
`来源: modules/bot_provision/jwt.go#L20-L27`（BotProvision 结构体）
`来源: modules/bot_provision/1module.go#L7-L17`（模块注册）

#### `botidentity`（Bot 身份解析）

**职责**：解析 Bot UID 背后的权威生命周期身份。只读取 `robot` 和 `app_bot` 两张表，`user.robot` 表仅为展示元数据，不作为授权源。

**核心类型**：
- `Kind`：`user_bot` 或 `app_bot`
- `Identity`：包含 UID、Kind、CreatorUID、AppScope、AppSpaceID
- `Resolver.Resolve(uid)` → `*Identity`

**设计原则**：跨表唯一性不变量——如果同一 UID 同时出现在 `robot` 和 `app_bot` 表中（`ErrAmbiguousIdentity`），调用方必须 fail-closed。

`来源: modules/botidentity/resolver.go#L1-L11`（包文档）
`来源: modules/botidentity/resolver.go#L18-L27`（Kind 常量和错误）
`来源: modules/botidentity/resolver.go#L35-L46`（Identity 结构体）
`来源: modules/botidentity/resolver.go#L81-L94`（Resolver 和 Resolve 方法）

### 7.2 `bot_api` 模块（Bot 对外 API）

`bot_api` 是 Bot 面向外部适配器/集成的主要 API 表面：

- 认证方式：Bot token（`bf_` 前缀为 User Bot，`app_` 前缀为 App Bot）
- 核心端点：`/v1/bot/register`（注册/鉴权）、`/v1/bot/heartbeat`（心跳）、`/v1/bot/sendMessage`（发消息）等
- 限流：三层（per-IP strict → authBot 鉴权 → per-bot business）
- 支持 OBO（On-Behalf-Of）模式：用户授权 Bot 代表自己行动

`来源: modules/bot_api/bot_api.go#L297-L497`（路由注册）
`来源: modules/bot_api/register.go#L303-L317`（register 处理逻辑）

### 7.3 Agent 会话如何起

Agent 会话通过 Bot 注册流程启动：

1. **Bot 注册**：Agent 通过 `POST /v1/bot/register` 携带 Bot token 注册
2. **Token 鉴权**：Server 查询 `robot` 表验证 bot_token，区分 User Bot（`bf_`）和 App Bot（`app_`）
3. **IM Token 签发**：Server 调用 `ctx.UpdateIMToken` 向 WuKongIM 为该 Bot UID 注册 IM Token
4. **Agent 运行时上报**：注册请求可携带 `agent_platform`、`agent_version`、`plugin_version`、`agent_hosting` 字段，写入 `robot` 表的 agent 列
5. **返回连接信息**：Server 返回 `BotRegisterResp`（含 `robot_id`、`im_token`、`ws_url`、`api_url`、`owner_uid`）
6. **Agent 建连**：Agent 使用返回的 `im_token` 和 `ws_url` 连接 WuKongIM WebSocket

`agent_hosting` 字段标识 Agent 运行时托管形态：
- `self_hosted` — 自托管
- `octo_hosted` — 平台托管
- `none` — 撤回之前上报的形态

`来源: modules/bot_api/register.go#L303-L317`（register 入口分发）
`来源: modules/bot_api/register.go#L319-L389`（registerUserBot 完整流程）
`来源: modules/bot_api/register.go#L21-L57`（agent_hosting 常量和设计说明）
`来源: modules/bot_api/register.go#L392-L435`（applyAgentReport 持久化）

### 7.4 关键文件索引

| 文件 | 作用 |
|---|---|
| `modules/app_bot/app_bot.go` | App Bot CRUD API |
| `modules/app_bot/1module.go` | App Bot 模块注册 |
| `modules/botfather/api.go` | BotFather 路由与初始化 |
| `modules/botfather/api_user.go` | User Bot CRUD + API Key 认证路由 |
| `modules/botfather/api_apply.go` | Robot Apply 审批流程 |
| `modules/bot_provision/bot_api.go` | Bot mint + token 获取跨服务接口 |
| `modules/botidentity/resolver.go` | Bot 身份权威解析器 |
| `modules/bot_api/bot_api.go` | Bot 对外 API 路由注册 |
| `modules/bot_api/register.go` | Bot 注册与 Agent 运行时上报 |
| `modules/bot_api/obo_api.go` | OBO（On-Behalf-Of）授权管理 |
| `modules/bot_api/obo_fanout.go` | OBO 消息扇出 |

---

## §8. 存储与外部依赖

### 8.1 表怎么建/怎么迁移

**迁移框架**：使用 `rubenv/sql-migrate` 库，SQL 文件以 `-- +migrate Up` / `-- +migrate Down` 标记方向。

**迁移文件组织**：每个模块在自身 `sql/` 目录下维护迁移文件，文件名采用 14 位时间戳前缀（`YYYYMMDDHHMMSS_<name>.sql`），通过 `//go:embed sql` 编译进二进制。

**模块注册模式**（以 `app_bot` 为例）：

```go
//go:embed sql
var sqlFS embed.FS

func init() {
    register.AddModule(func(ctx interface{}) register.Module {
        return register.Module{
            Name: "app_bot",
            SetupAPI: func() register.APIRouter { return NewAppBot(ctx.(*config.Context)) },
            SQLDir: register.NewSQLFS(sqlFS),
        }
    })
}
```

`来源: modules/app_bot/1module.go#L8-L22`
`来源: modules/bot_api/1module.go#L8-L22`
`来源: modules/botfather/1module.go#L12-L25`

**迁移执行**：`NewMySQL` 在启动时调用 `Migration(sqlDir, session)`，通过 `FileDirMigrationSource` 递归扫描 SQL 文件，按 ID 排序后用 `migrate.Exec` 执行。迁移记录存储在 `gorp_migrations` 表中。

`来源: pkg/db/mysql.go#L16-L35`（NewMySQL，含迁移调用）
`来源: pkg/db/mysql.go#L41-L56`（Migration 函数）
`来源: pkg/db/mysql.go#L58-L110`（FileDirMigrationSource 递归扫描）

**迁移兼容层**：`RewriteLegacyMigrationIDs` 处理从旧命名格式（`<module>-<YYYYMMDD>-<NN>.sql`）到新时间戳前缀格式的迁移。它在启动时、`migrate.Exec` 之前运行，将 `gorp_migrations` 表中的旧 ID 更新为新 ID。

`来源: pkg/db/migrate_compat.go#L26-L44`（RewriteLegacyMigrationIDs 函数及注释）

**当前迁移文件统计**：全项目 208 个 SQL 迁移文件，分布在各模块 `sql/` 目录下。

`来源: find /root/.openclaw/workspace/octo-server -name "*.sql" -path "*/sql/*" | wc -l`

**建表示例**（app_bot 表）：

```sql
CREATE TABLE IF NOT EXISTS app_bot (
  id           VARCHAR(40) PRIMARY KEY,
  uid          VARCHAR(40) UNIQUE NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  scope        VARCHAR(20) NOT NULL DEFAULT 'platform',
  space_id     VARCHAR(40) DEFAULT NULL,
  status       TINYINT NOT NULL DEFAULT 0,
  token        VARCHAR(100) UNIQUE NOT NULL,
  created_by   VARCHAR(40) NOT NULL,
  created_at   DATETIME NOT NULL DEFAULT NOW(),
  updated_at   DATETIME NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  INDEX idx_scope_status (scope, status),
  INDEX idx_space_status (space_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`来源: modules/app_bot/sql/20260505000001_app_bot_legacy01.sql#L1-L19`

### 8.2 Redis 缓存什么

octo-server 的 Redis 使用通过 `pkg/redis` 统一入口（chokepoint 模式），所有生产代码必须通过 `octoredis.NewInstrumentedClient` / `InstrumentedClientFromOptions` 构造客户端，禁止直接 `rd.NewClient`。

`来源: pkg/redis/*.go`（chokepoint 守卫测试 `TestNoRawRedisClientOutsideChokepoint`）

Redis 在 main.go 中用于以下场景：

| 用途 | 构造位置 | 说明 |
|---|---|---|
| 全局限流 | main.go L302 | 令牌桶 Lua 脚本限流，多副本共享配额 |
| 卡片动作分发队列 | main.go L749 | `cardactiondispatch.NewRedisQueue`，DLQ + 活跃队列 |
| Bot API per-bot 限流 | `bot_api/registry_redis.go` | per-bot 令牌桶 |
| Bot API App Bot 注册表 | `bot_api/registry_redis.go` | 多实例 Bot 注册表 |
| 模块级缓存 | 各模块 | 如 `incomingwebhook/ratelimit.go`、`bot_mention/idempotency.go` 等 |

`来源: main.go#L302-L306`（限流 Redis）
`来源: main.go#L749-L767`（卡片动作分发 Redis）

配置层支持 Redis TLS（用于 AWS ElastiCache / Azure Cache 等托管服务）：

`来源: pkg/db/redis.go#L10-L16`（NewRedisFromConfig 自动应用 TLS）
`来源: configs/tsdd.yaml`（redisAddr, redisPass, redisTLS 等配置项）

### 8.3 对象存储怎么接

**统一接口**：`modules/file/service.go` 定义 `IService` 接口，包含 `UploadFile`、`DownloadURL`、`PresignedPutURL`、`PresignedGetURL` 等方法。

**后端分发**：`NewService` 根据 `config.FileService` 配置值分发到具体实现：

| 配置值 | 实现文件 | 预签名 PUT | 预签名 GET |
|---|---|---|---|
| `minio` | `service_minio.go` | ✅ | ✅ |
| `tencentCOS` | `service_cos.go` | ✅ | ✅ |
| `aliyunOSS` | `service_oss.go` | ✅ | ✅ |
| `qiniu` | `service_qiniu.go` | ❌（token+form 模式） | ✅ |
| `seaweedFS` | `service_seaweedfs.go` | ❌ | ❌ |
| `awsS3`（本地常量） | `service_s3.go` | ✅ | ✅ |
| （fallback） | `service_seaweedfs.go` | — | — |

`来源: modules/file/service.go#L32-L57`（IUploadService / IService 接口）
`来源: modules/file/service.go#L59-L101`（NewService 分发逻辑）

**预签名 URL 契约**：浏览器直传 PUT 时，`contentType` 和 `contentDisposition` 必须与签名时一致（SigV4 canonical-headers 覆盖），否则存储后端返回 `403 SignatureDoesNotMatch`。

`来源: modules/file/service.go#L47-L58`（PresignedPutURL / PresignedGetURL 接口）
`来源: configs/tsdd.yaml`（fileService / minio / oss / qiniu / seaweed 配置及 SigV4 契约注释）

### 8.4 关键文件索引

| 文件 | 作用 |
|---|---|
| `pkg/db/mysql.go` | MySQL 连接创建与 SQL 迁移执行 |
| `pkg/db/migrate_compat.go` | 迁移 ID 兼容层（旧→新重写） |
| `pkg/db/redis.go` | Redis 连接工厂 |
| `pkg/redis/*.go` | Redis 客户端 chokepoint + 插桩 |
| `pkg/cache/cache.go` | 缓存接口定义 |
| `modules/file/service.go` | 文件服务统一接口与后端分发 |
| `modules/file/service_minio.go` | MinIO 后端实现 |
| `modules/file/service_oss.go` | 阿里云 OSS 后端实现 |
| `modules/file/service_cos.go` | 腾讯云 COS 后端实现 |
| `modules/file/service_qiniu.go` | 七牛云后端实现 |
| `modules/file/service_seaweedfs.go` | SeaweedFS 后端实现 |
| `modules/file/service_s3.go` | AWS S3 后端实现 |
| `configs/tsdd.yaml` | 存储/Redis/对象存储配置模板 |

---

## §9. 构建与发布

### 9.1 go build / Dockerfile / Dockerfile.ghcr 的区别

#### 直接 `go build`

本地开发使用 `go build ./...`。需要注意：

- 项目依赖私有 sibling 仓库 `octo-lib` 和 `octo-adapters`，预览期需在 `go.mod` 中添加 `replace` 指令指向本地路径。
- CI 中使用 `CGO_ENABLED=0 go build -v ./...` 验证编译。

`来源: BUILDING.md#L1-L24`（本地构建说明）
`来源: .github/workflows/ci.yml#L99`（CI 构建步骤）

CI 中的 `go build` 命令：

```bash
CGO_ENABLED=0 go build -v ./...
```

`来源: .github/workflows/ci.yml#L99`

#### `Dockerfile`（主 Dockerfile）

**用途**：从源码完整构建 octo-server 容器镜像，是 `make build` 和 Docker Hub 发布的入口。

**多阶段构建**：
- **build 阶段**（`golang:1.25`）：`go mod download` → `COPY . .` → `CGO_ENABLED=0 GOOS=linux go build` 产出静态二进制 `app`
- **prod 阶段**（`alpine:3.21`）：仅拷贝二进制 + assets + configs + CA 证书 + 时区

**版本信息嵌入**：通过 `-ldflags` 注入 `main.Commit`、`main.CommitDate`、`main.Version`、`main.TreeState`，`git describe --tags` 失败时 fallback 到 `dev`。

**关键配置**：
- `GOPROXY=https://goproxy.cn,direct` — 中国大陆友好的 Go 模块代理
- `CGO_ENABLED=0` + `-installsuffix cgo` — 纯静态编译
- `ENTRYPOINT ["/home/app"]`

`来源: Dockerfile#L1-L48`

#### `Dockerfile.ghcr`（GHCR 发布用）

**用途**：用于 GitHub Container Registry 多架构发布，**不从源码构建**，而是直接拷贝预编译的二进制。

**特点**：
- 基础镜像：`debian:bookworm-slim`（而非 Alpine）
- 使用 `ARG TARGETARCH` 接收多架构构建参数
- 直接 `COPY --chmod=0755 linux_${TARGETARCH} main` — 二进制由外部 CI 交叉编译后传入
- 包含 `assets` 和 `configs` 目录
- `CMD ["/app/main"]`

**与主 Dockerfile 的区别**：

| 维度 | Dockerfile | Dockerfile.ghcr |
|---|---|---|
| 基础镜像 | `golang:1.25` → `alpine:3.21` | `debian:bookworm-slim` |
| 构建方式 | 多阶段从源码编译 | 直接拷贝预编译二进制 |
| 多架构 | 单架构（`GOOS=linux`） | `TARGETARCH` 多架构 |
| 二进制名 | `app` | `main` |
| 工作目录 | `/home` | `/app` |
| 用途 | 本地构建 / Docker Hub | GHCR 多架构发布 |

`来源: Dockerfile.ghcr#L1-L17`

### 9.2 与 octo-deployment 的关系

`octo-deployment` 是 OCTO 生态的**唯一官方 OOTB 部署仓库**，包含完整的 docker-compose 栈（octo-server + admin + web + matter + smart-summary + WuKongIM + MySQL + Redis + MinIO + nginx）。

**本仓库的 Makefile 中 `run-dev` / `stop-dev` 已退役**，指向 `octo-deployment`：

```makefile
run-dev:
	@echo "run-dev has been retired — the bundled docker-compose stack moved to"
	@echo "  https://github.com/Mininglamp-OSS/octo-deployment"
```

`来源: Makefile#L20-L27`（run-dev 退役说明）

**Makefile 中的 `push` / `deploy` / `deploy-v2` 目标**是遗留的私有 Aliyun 注册表推送命令（且有一个 stale tag bug：`make push` 推送的是 `wukongchatserver:latest` 而非 `octo-server:latest`），不是规范的发布渠道。

`来源: Makefile#L1-L10`（build/push/deploy 目标）
`来源: BUILDING.md#L26-L45`（Docker 构建与 octo-deployment 关系说明）

**Docker Hub 发布**：通过 `.github/workflows/docker-publish.yml` 自动触发，在 `v*` Git tag push 时构建多架构镜像（`linux/amd64`, `linux/arm64`），发布到 `mininglamposs/octo-server`。

`来源: .github/workflows/docker-publish.yml#L1-L30`（触发条件与并发控制）
`来源: BUILDING.md#L35-L44`（Docker Hub 发布说明）

### 9.3 发布流程

发布遵循 org-wide OCTO release process：

1. **选择 main 上的 commit**，确认 CI green
2. **推送 tag**：`git tag -a v1.7.0 -m "Release v1.7.0" <sha>` → `git push origin v1.7.0`
3. **运行 Release Publish workflow**：`.github/workflows/release-publish.yml`，传入 tag 和 CI run ID
4. **Changelog**：由 release-drafter 自动从 PR 标题生成（Conventional Commits 格式）

`来源: RELEASING.md#L1-L52`

### 9.4 关键文件索引

| 文件 | 作用 |
|---|---|
| `Dockerfile` | 主 Dockerfile，多阶段从源码构建 |
| `Dockerfile.ghcr` | GHCR 发布用，拷贝预编译二进制 |
| `Makefile` | make build/push/deploy + i18n/lint 工具链 |
| `BUILDING.md` | 构建指南（本地/Docker/依赖说明） |
| `RELEASING.md` | 发布流程（SemVer/tag/CI） |
| `.github/workflows/ci.yml` | CI 流水线（go build/test/lint） |
| `.github/workflows/docker-publish.yml` | Docker Hub 多架构镜像发布 |
| `.github/workflows/release-drafter.yml` | 自动 changelog 草稿 |
| `.github/workflows/release-publish.yml` | GitHub Release 发布 |
| `configs/tsdd.yaml` | 运行时配置模板 |
