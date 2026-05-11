# Ghost API 鉴权层级分析报告

## 1. 概述

Ghost 提供 **两套独立的 Web 应用**（Backend 和 Frontend），分别挂载不同的接口：

| Web 应用 | 挂载位置 | 包含的接口 | 目标用户 |
|---------|---------|-----------|---------|
| **Backend App** | `backend.js` | Content API、Admin API、Admin UI | 管理员、集成应用、API 调用者 |
| **Frontend App** | `frontend.js` | Members App、Webmentions、Gift Preview、主题路由 | 网站访客、订阅会员 |

### 接口分类

| 接口类型 | URL 前缀 | 挂载位置 | 目标用户 | 凭证要求 | 主要功能 |
|---------|---------|---------|---------|---------|---------|
| 公开内容接口 (Content API) | `/ghost/api/content/*` | `backend.js` → `api/app.js` | 集成应用、认证会员 | **需要 Content API Key 或 GhostMembers Token 之一** | 只读访问公开内容（带凭证的访问） |
| 后台管理接口 (Admin API) | `/ghost/api/admin/*` | `backend.js` → `api/app.js` | 管理员、集成应用 | 需要 Admin JWT 或 Session | 完整 CRUD 管理 |
| 会员私域接口 (Members App) | `/members/*` | `frontend.js` | 订阅会员 | 需要 Session 或 UUID+HMAC | 会员登录、订阅管理、评论 |
| 会员内容门控 | 集成于 Content API | `backend.js` | 认证会员 | 需要 GhostMembers Token（用于身份判断） | 访问付费/会员专属内容 |

**Content API 重要说明**：`authorizeContentApi` 严格要求 `req.api_key` 或 `req.member` 之一存在，两者都没有时直接返回 403。完全无凭证的请求无法通过授权层。

本报告详细分析这几套接口在**凭证类型**、**路由机制**、**权限控制**和**响应裁剪**四个维度的差异。

---

## 2. 请求入口拓扑（基于源码）

### 2.1 完整入口链

Ghost 的 HTTP 请求经过三层入口应用：

```
                    ┌─────────────────┐
                    │   boot.js       │
                    │  (入口挂载点)    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
    ┌─────────▼─────────┐         ┌─────────▼─────────┐
    │   backend.js      │         │   frontend.js     │
    │  (后台应用)        │         │  (前台应用)        │
    └─────────┬─────────┘         └─────────┬─────────┘
              │                             │
    ┌─────────▼─────────┐         ┌─────────┼─────────────────┐
    │   api/app.js      │         │         │                 │
    │  (API 父应用)      │         │         │                 │
    └─────────┬─────────┘         │         │                 │
              │                   │         │                 │
     ┌────────┴────────┐    ┌─────▼─────┐  ┌▼────────┐  ┌────▼─────┐
     │                 │    │  members  │  │webmentions│ │  gift    │
     ▼                 ▼    │   App     │  │           │ │ Preview │
┌─────────┐       ┌─────────┐└───────────┘  └───────────┘ └──────────┘
│ Content │       │ Admin   │
│  API    │       │  API    │
└─────────┘       └─────────┘
```

### 2.2 源码挂载链详解

#### 第一层: boot.js（入口挂载）

**文件**: `ghost/core/core/boot.js:240-257`

```javascript
// ADMIN + API
const backendApp = require('./server/web/parent/backend')();
parentApp.use(vhost(config.getBackendMountPath(), backendApp));

// SITE + MEMBERS
const frontendApp = require('./server/web/parent/frontend')({urlService});
parentApp.use(vhost(config.getFrontendMountPath(), frontendApp));
```

**关键点**:
- `backendApp` 挂载到后台域名/路径
- `frontendApp` 挂载到前台域名/路径
- 使用 `vhost` 中间件实现虚拟主机分离

#### 第二层: backend.js（后台应用）

**文件**: `ghost/core/core/server/web/parent/backend.js:9-20`

```javascript
const {BASE_API_PATH} = require('../../../shared/url-utils');  // BASE_API_PATH = '/ghost/api'

module.exports = () => {
    const backendApp = express('backend');

    // 挂载 API 父应用
    backendApp.lazyUse(BASE_API_PATH, require('../api'));  // /ghost/api/* → api/app.js
    
    // 挂载 Admin UI
    backendApp.use('/ghost/.well-known', require('../well-known'));
    backendApp.use('/ghost', require('../../services/auth/session').createSessionFromToken(), require('../admin')());

    return backendApp;
};
```

**backend.js 挂载的内容**:
| 路径 | 目标应用 | 用途 |
|-----|---------|------|
| `/ghost/api/*` | `../api` (api/app.js) | Content API + Admin API |
| `/ghost/.well-known` | `../well-known` | 公开发现端点 |
| `/ghost/*` | `../admin` | Admin 管理后台 UI |

#### 第三层: api/app.js（API 父应用）

**文件**: `ghost/core/core/server/web/api/app.js:12-35`

```javascript
module.exports = function setupApiApp() {
    const apiApp = express('api');

    // API 版本兼容处理
    apiApp.use(APIVersionCompatibilityService.versionRewrites);
    apiApp.use(APIVersionCompatibilityService.contentVersion);

    // 分发给子 API
    apiApp.lazyUse('/content/', require('./endpoints/content/app'));  // /ghost/api/content/*
    apiApp.lazyUse('/admin/', require('./endpoints/admin/app'));      // /ghost/api/admin/*

    return apiApp;
};
```

**api/app.js 挂载的内容**:
| 路径 | 目标应用 | 用途 |
|-----|---------|------|
| `/ghost/api/content/*` | `./endpoints/content/app` | 公开内容 API |
| `/ghost/api/admin/*` | `./endpoints/admin/app` | 后台管理 API |

#### 第二层: frontend.js（前台应用）

**文件**: `ghost/core/core/server/web/parent/frontend.js:10-25`

```javascript
module.exports = (routerConfig) => {
    const frontendApp = express('frontend');

    // SSL 重定向
    frontendApp.use(shared.middleware.urlRedirects.frontendSSLRedirect);

    // 挂载 Members App (关键: /members 挂载在 frontend.js，不是 backend.js)
    frontendApp.lazyUse('/members', require('../members'));
    
    // 挂载其他前台服务
    frontendApp.lazyUse('/webmentions', require('../webmentions'));
    frontendApp.lazyUse('/gift', require('../gift-preview'));
    
    // 挂载主题路由
    frontendApp.use('/', require('../../../frontend/web')(routerConfig));

    return frontendApp;
};
```

**frontend.js 挂载的内容**:
| 路径 | 目标应用 | 用途 |
|-----|---------|------|
| `/members/*` | `../members` | 会员门户（登录、订阅、评论等） |
| `/webmentions/*` | `../webmentions` | Webmention 接收 |
| `/gift/*` | `../gift-preview` | 礼物卡预览 |
| `/*` | `../../../frontend/web` | 主题渲染、博客页面 |

### 2.3 修正后的路由拓扑总结

```
HTTP 请求
    ↓
┌─────────────────────────────────────────────────────────────────┐
│                     boot.js (入口挂载)                           │
│  ┌─────────────────────────┐    ┌─────────────────────────┐    │
│  │ backend.js (后台应用)     │    │ frontend.js (前台应用)    │    │
│  │ 路径: /ghost/*           │    │ 路径: /members/*         │    │
│  │       /ghost/api/*       │    │       /webmentions/*     │    │
│  │                         │    │       /gift/*            │    │
│  │                         │    │       /* (主题路由)       │    │
│  └──────────┬──────────────┘    └──────────┬──────────────┘    │
└─────────────┼──────────────────────────────┼───────────────────┘
              │                              │
              ▼                              ▼
┌─────────────────────────┐       ┌─────────────────────────┐
│ api/app.js (API 父应用)  │       │ members/app.js          │
│ 路径: /ghost/api/*      │       │ 路径: /members/*        │
└──────────┬──────────────┘       │ 认证: Session Cookie    │
           │                      │       UUID + HMAC       │
           │                      └─────────────────────────┘
    ┌──────┴──────┐
    ▼             ▼
┌─────────┐ ┌─────────┐
│ Content │ │ Admin   │
│  API    │ │  API    │
└─────────┘ └─────────┘
```

**关键修正**:
- ❌ **错误**: `/members/*` 挂在 `backend.js`
- ✅ **正确**: `/members/*` 挂在 `frontend.js`
- ❌ **错误**: `/ghost/api/*` 和 `/members/*` 同属一个应用
- ✅ **正确**: 分属 `backend.js` 和 `frontend.js` 两个独立应用

---

## 3. 凭证类型对比

### 3.1 Backend App 凭证体系（Content API + Admin API）

#### 3.1.1 Content API 凭证

Content API 支持 **两种独立的认证方式**，任一通过即可：

**方式一: Content API Key（集成凭证）**

- **位置**: URL 查询参数 `?key={content_api_key}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/content.js:12`
- **挂载位置**: `backend.js` → `api/app.js` → `content/app.js`

```javascript
// 关键代码位置: content.js:28-42
const apiKey = await models.ApiKey.findOne({secret: key}, {withRelated: ['integration']});

if (!apiKey) {
    return next(new errors.UnauthorizedError({
        message: tpl(messages.unknownContentApiKey),
        code: 'UNKNOWN_CONTENT_API_KEY'
    }));
}

if (apiKey.get('type') !== 'content') {
    return next(new errors.UnauthorizedError({
        message: tpl(messages.invalidApiKeyType),
        code: 'INVALID_API_KEY_TYPE'
    }));
}
```

**特点**:
- 最简单的认证方式
- 密钥以明文形式出现在 URL 中
- 适用于 **第三方集成** 访问公开内容
- **不提供会员身份**，无法访问付费内容

**方式二: GhostMembers JWT Token（会员凭证）**

- **位置**: HTTP Header `Authorization: GhostMembers {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/members/index.js:8`
- **挂载位置**: 虽由 Members App 签发，但用于 Content API 认证

```javascript
// 关键代码位置: members/index.js:13-33
return jwt({
    credentialsRequired: false,
    requestProperty: 'member',
    audience: siteOrigin,
    issuer: membersConfig.issuer,
    algorithms: ['RS512'],
    secret: membersConfig.publicKey,
    getToken(req) {
        if (!req.get('authorization')) {
            return null;
        }
        const [scheme, credentials] = req.get('authorization').split(/\s+/);
        if (scheme !== 'GhostMembers') {
            return null;
        }
        return credentials;
    }
});
```

**特点**:
- 使用非对称加密 (RSA)，安全性高
- Token 包含完整会员身份信息（状态、订阅层级等）
- `credentialsRequired: false` 表示**未提供 token 时不报错**（不直接 401），但最终是否通过由 `authorizeContentApi` 决定
- **可用于内容门控判断**，决定是否返回付费内容

**重要注意**：`credentialsRequired: false` 仅表示 JWT 中间件本身不强制要求 token，**Content API 授权层 (`authorizeContentApi`) 仍然要求 `req.api_key` 或 `req.member` 之一存在，否则返回 403**。

#### 3.1.2 Content API Key vs GhostMembers Token 对比

| 对比项 | Content API Key | GhostMembers Token |
|-------|----------------|-------------------|
| **传递位置** | URL 查询参数 `?key=` | HTTP Header `Authorization: GhostMembers` |
| **加密方式** | 对称存储（数据库 secret） | RS512 非对称签名 |
| **身份类型** | 集成应用（无用户身份） | 具体会员（有 user/member 身份） |
| **req 对象** | 设置 `req.api_key` | 设置 `req.member` |
| **付费内容访问** | ❌ 不能（无会员身份） | ✅ 可用于内容门控判断 |
| **适用场景** | 前端应用、静态站点生成器 | 会员登录状态、付费内容访问 |
| **过期机制** | 永久有效（可手动撤销） | 短期有效（JWT 过期时间） |
| **签发来源** | Ghost Admin 集成页面 | Members App (`/members/api/session`) |

#### 3.1.3 Admin API 凭证

Admin API 支持 **三种认证方式**：

**方式一: Admin JWT Token (Header)**

- **位置**: HTTP Header `Authorization: Ghost {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/admin.js:48`
- **挂载位置**: `backend.js` → `api/app.js` → `admin/app.js`

**JWT 特殊要求**:
- 必须包含 `kid` (Key ID) 头部，用于查找对应的 API Key
- 必须包含 `audience` 声明，匹配请求的 API 路径
- 使用 **HS256** 对称算法，密钥为 API Key 的 secret
- 默认有效期 **5 分钟**

```javascript
// 关键代码位置: admin.js:104-179
const authenticateWithToken = async function apiKeyAuthenticateWithToken(originalUrl, token, ignoreMaxAge) {
    const decoded = jwt.decode(token, {complete: true});
    const apiKeyId = decoded.header.kid;
    
    // 1. 通过 kid 查找 API Key
    const apiKey = await models.ApiKey.findOne({id: apiKeyId}, {withRelated: ['integration']});
    
    // 2. 验证类型为 'admin'
    if (apiKey.get('type') !== 'admin') {
        throw new errors.UnauthorizedError({...});
    }
    
    // 3. 验证 JWT 签名和 audience
    jwt.verify(token, secret, options);
    
    // 4. 如果是 staff token (有 user_id)，获取关联用户
    if (apiKey.get('user_id')) {
        const user = await models.User.findOne({id: apiKey.get('user_id'), status: 'active'}, {require: true});
        result.user = user;
    }
};
```

**方式二: Admin JWT Token (URL) - authAdminApiWithUrl**

- **位置**: URL 查询参数 `?token={jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/admin.js:67`
- **用途**: 定时任务发布、礼物提醒等后台任务
- **特殊**: 可忽略 token 有效期 (`ignoreMaxAge: true`)

```javascript
// 关键代码位置: admin.js:67-77
const authenticateWithUrl = function apiKeyAuthenticateWithUrl(req, res, next) {
    const token = _extractTokenFromUrl(req.originalUrl);
    // CASE: Scheduler publish URLs can have long maxAge but controlled by expiry and neverBefore
    return wrappedAuthenticateWithToken(req, res, next, {token, ignoreMaxAge: true});
};
```

**方式三: Session Cookie（浏览器登录）**

- **位置**: Cookie + Session
- **认证位置**: `ghost/core/core/server/services/auth/session/middleware.js:42`

```javascript
// 关键代码位置: session/middleware.js:42-59
async function authenticate(req, res, next) {
    try {
        const user = await sessionService.getUserForSession(req, res);
        if (user) {
            const isVerified = await sessionService.isVerifiedSession(req, res);
            if (!isVerified) {
                return next();
            }
            req.user = user;
        }
        next();
    } catch (err) {
        next(err);
    }
}
```

**Admin Token 类型细分**:

| Token 类型 | 是否有 user_id | 权限模型 |
|-----------|---------------|---------|
| **Staff Token** | ✅ 有 | 继承关联用户的权限 |
| **Integration Token** | ❌ 无 | 受 endpoint 白名单限制 |

### 3.2 Frontend App 凭证体系（Members App）

Members App 是一个**独立的 Express 应用**，挂在 `frontend.js`，使用自己的认证中间件：

**凭证类型一: Session Cookie（浏览器登录）**

- **位置**: Cookie (`ghost-members-ssr` 等)
- **认证中间件**: `loadMemberSession`
- **认证位置**: `ghost/core/core/server/services/members/middleware.js:91`
- **挂载位置**: `frontend.js` → `members/app.js`

```javascript
// 关键代码位置: middleware.js:91-101
const loadMemberSession = async function loadMemberSession(req, res, next) {
    try {
        const member = await membersService.ssr.getMemberDataFromSession(req, res);
        Object.assign(req, {member});
        res.locals.member = req.member;
        next();
    } catch (err) {
        Object.assign(req, {member: null});
        next();
    }
};
```

**凭证类型二: UUID + HMAC Key（邮件链接）**

- **位置**: URL 查询参数 `?uuid=&key=`
- **认证中间件**: `authMemberByUuid`
- **用途**: 退订链接、邮件中的操作链接
- **认证位置**: `ghost/core/core/server/services/members/middleware.js:123`

```javascript
// 关键代码位置: middleware.js:123-150
const authMemberByUuid = async function authMemberByUuid(req, res, next) {
    const uuid = req.query.uuid;
    const key = req.query.key;
    
    // 验证 HMAC 签名（只有 Ghost 能生成合法 key）
    const memberHmac = crypto.createHmac('sha256', settingsHelpers.getMembersValidationKey())
                            .update(uuid).digest('hex');
    if (memberHmac !== key) {
        throw new errors.UnauthorizedError({...});
    }
};
```

**凭证类型三: Magic Link Token**

- **位置**: URL 查询参数 `?token=`
- **认证中间件**: `createSessionFromMagicLink`
- **用途**: 免密登录链接

### 3.3 凭证与挂载位置对照表

| 凭证类型 | 挂载应用 | 入口文件 | 目标接口 |
|---------|---------|---------|---------|
| Content API Key | Backend App | `backend.js` → `api/app.js` | Content API |
| GhostMembers JWT | Backend App | `backend.js` → `api/app.js` | Content API (内容门控) |
| Admin JWT (Header) | Backend App | `backend.js` → `api/app.js` | Admin API |
| Admin JWT (URL) | Backend App | `backend.js` → `api/app.js` | Admin API (定时任务) |
| Admin Session | Backend App | `backend.js` | Admin UI + Admin API |
| Members Session | Frontend App | `frontend.js` | Members App |
| Members UUID+HMAC | Frontend App | `frontend.js` | Members App (邮件链接) |

---

## 4. 请求路由与权限层

### 4.1 修正后的整体路由结构

```
HTTP 请求
    ↓
boot.js (入口挂载)
    ├─→ Backend App (backend.js)
    │       ├─→ /ghost/api/* → API 父应用 (api/app.js)
    │       │       ├─→ /ghost/api/content/* → Content API
    │       │       │       └─ 认证: **必须提供 Content API Key 或 GhostMembers Token 之一，否则 403**
    │       │       │
    │       │       └─→ /ghost/api/admin/* → Admin API
    │       │               ├─ authAdminApi          (JWT Header 或 Session)
    │       │               ├─ authAdminApiWithUrl   (URL Token)
    │       │               └─ publicAdminApi        (无认证，仅 /site)
    │       │
    │       └─→ /ghost/* → Admin UI
    │
    └─→ Frontend App (frontend.js)
            ├─→ /members/* → Members App (独立门户)
            │       ├─ /members/api/*        (Session 或 UUID+HMAC)
            │       └─ /members/webhooks/*   (无认证)
            │
            ├─→ /webmentions/* → Webmention 接收
            ├─→ /gift/* → Gift Preview
            └─→ /* → 主题路由
```

### 4.2 Backend App - Content API 认证链

**位置**: `ghost/core/core/server/web/api/endpoints/content/middleware.js:18`
**挂载链**: `backend.js` → `api/app.js` → `content/app.js` → `content/routes.js`

```javascript
module.exports.authenticatePublic = [
    shared.middleware.brute.contentApiKey,      // 1. 暴力破解防护
    auth.authenticate.authenticateContentApi,    // 2. 认证 (API Key + Members Token)
    auth.authorize.authorizeContentApi,          // 3. 授权检查
    cors(),                                      // 4. CORS 处理
    shared.middleware.urlRedirects.adminSSLAndHostRedirect,
    shared.middleware.prettyUrls
];
```

**认证层详情** (`authenticate.js:9`):
```javascript
authenticateContentApi: [
    apiKeyAuth.content.authenticateContentApiKey,  // 尝试 API Key 认证 → req.api_key
    members.authenticateMembersToken               // 尝试 Members JWT 认证 → req.member
]
```

**授权层详情** (`authorize.js:11-24`):
```javascript
authorizeContentApi(req, res, next) {
    const hasApiKey = req.api_key && req.api_key.id;
    const hasMember = req.member;
    if (hasApiKey || hasMember) {
        return next();  // 任一认证方式通过即可
    }
    return next(new errors.NoPermissionError({...}));
}
```

**关键点**: Content API 允许 **无认证访问**（当 `credentialsRequired: false` 时），但 `req.member` 为 `null`，内容门控会按访客处理。

### 4.3 Backend App - Admin API 认证链（三种变体）

Admin API 提供 **三种认证中间件**，适用于不同场景：
**挂载链**: `backend.js` → `api/app.js` → `admin/app.js` → `admin/routes.js`

#### 4.3.1 authAdminApi（标准认证）

**位置**: `ghost/core/core/server/web/api/endpoints/admin/middleware.js:100`

```javascript
module.exports.authAdminApi = [
    auth.authenticate.authenticateAdminApi,        // 1. 认证 (JWT Header 或 Session)
    auth.authorize.authorizeAdminApi,              // 2. 授权检查
    apiMw.updateUserLastSeen,                      // 3. 更新用户最后活跃时间
    apiMw.cors,
    shared.middleware.urlRedirects.adminSSLAndHostRedirect,
    shared.middleware.prettyUrls,
    tokenPermissionCheck                           // 4. Token 权限细粒度检查
];
```

**认证层详情** (`authenticate.js:6`):
```javascript
authenticateAdminApi: [
    apiKeyAuth.admin.authenticate,  // 尝试 Admin API Key JWT 认证 → req.api_key + 可能 req.user
    session.authenticate            // 尝试 Session 认证 → req.user
]
```

**授权层详情** (`authorize.js:26-38`):
```javascript
authorizeAdminApi(req, res, next) {
    const hasUser = req.user && req.user.id;        // Session 或 Staff Token
    const hasApiKey = req.api_key && req.api_key.id; // Integration Token
    
    if (hasUser || hasApiKey) {
        return next();
    }
    return next(new errors.NoPermissionError({...}));
}
```

#### 4.3.2 authAdminApiWithUrl（URL Token 认证）

**位置**: `ghost/core/core/server/web/api/endpoints/admin/middleware.js:116`

```javascript
module.exports.authAdminApiWithUrl = [
    auth.authenticate.authenticateAdminApiWithUrl,  // 1. 从 URL ?token= 提取并认证
    auth.authorize.authorizeAdminApi,               // 2. 授权检查
    apiMw.updateUserLastSeen,
    apiMw.cors,
    shared.middleware.urlRedirects.adminSSLAndHostRedirect,
    shared.middleware.prettyUrls,
    tokenPermissionCheck
];
```

**使用场景** (`routes.js`):
```javascript
// 定时任务发布
router.put('/schedules/:resource/:id', mw.authAdminApiWithUrl, http(api.schedules.publish));

// 礼物提醒
router.put('/gifts/flush_reminders', mw.authAdminApiWithUrl, http(api.giftReminders.flushReminders));

// 自动化轮询
router.put('/automations/poll', mw.authAdminApiWithUrl, http(api.automations.poll));
```

**特点**:
- Token 从 URL 查询参数 `?token=` 提取
- 可忽略 token 有效期（`ignoreMaxAge: true`）
- 用于服务器间的后台任务调用

#### 4.3.3 publicAdminApi（无需认证）

**位置**: `ghost/core/core/server/web/api/endpoints/admin/middleware.js:131`

```javascript
module.exports.publicAdminApi = [
    apiMw.cors,
    shared.middleware.urlRedirects.adminSSLAndHostRedirect,
    shared.middleware.prettyUrls,
    tokenPermissionCheck  // 仍然检查，但因为无认证，主要用于完整性
];
```

**使用场景** (`routes.js:19`):
```javascript
// 站点公开信息（用于检测 Ghost 版本等）
router.get('/site', mw.publicAdminApi, http(api.site.read));
```

**特点**:
- 无认证要求，任何人可访问
- 仅用于 `/site` 端点，返回公开站点信息
- 仍然经过 `tokenPermissionCheck`，但因为 `req.api_key` 为 null，不会触发限制

### 4.4 Frontend App - Members App 路由与认证

**位置**: `ghost/core/core/server/web/members/app.js`
**挂载链**: `frontend.js` → `members/app.js`

Members App 是一个**独立的 Express 应用**，使用自己的路由和中间件体系：

```javascript
// 关键代码位置: app.js:21-183
module.exports = function setupMembersApp() {
    const membersApp = express('members');
    
    // 缓存控制
    membersApp.use(shared.middleware.cacheControl('private'));
    
    // CORS
    membersApp.use(corsMiddleware);
    
    // Magic Link 登录处理
    membersApp.use(middleware.createSessionFromMagicLink);
    
    // === 路由配置 ===
    
    // Stripe Webhook (无需认证)
    membersApp.post('/webhooks/stripe', ...);
    
    // 会员数据操作 (Session 或 UUID 认证)
    membersApp.get('/api/member', middleware.getMemberData);
    membersApp.put('/api/member', middleware.updateMemberData);
    
    // 邮件退订 (UUID + HMAC 认证)
    membersApp.get('/api/member/newsletters', middleware.authMemberByUuid, ...);
    
    // 会话管理
    membersApp.get('/api/session', middleware.getIdentityToken);
    membersApp.delete('/api/session', middleware.deleteSession);
    
    // 订阅管理
    membersApp.post('/api/create-stripe-checkout-session', ...);
    membersApp.post('/api/create-stripe-billing-portal-session', ...);
    
    // 评论 (需要 Session)
    membersApp.use('/api/comments', commentRouter());
    
    // 公开站点信息 (无认证，用于 CORS 友好的探测)
    membersApp.get('/api/site', http(api.site.read));
};
```

**Members App 认证中间件对比**:

| 中间件 | 凭证来源 | 用途 | 挂载位置 |
|-------|---------|------|---------|
| `loadMemberSession` | Cookie | 浏览器登录会员的常规操作 | `frontend.js` |
| `authMemberByUuid` | URL `?uuid=&key=` | 邮件链接（退订等） | `frontend.js` |
| `createSessionFromMagicLink` | URL `?token=` | Magic Link 登录 | `frontend.js` |

### 4.5 Admin API Token 权限细粒度检查

**位置**: `ghost/core/core/server/web/api/endpoints/admin/middleware.js:17-91`

这个中间件根据 token 类型进行额外限制：

```javascript
const tokenPermissionCheck = function tokenPermissionCheck(req, res, next) {
    // CASE 1: 用户登录 (Session) 或 无认证 (publicAdminApi)
    if (!req.api_key) {
        return next();  // 无额外限制
    }

    // CASE 2: Staff Token (有 user_id)
    if (req.api_key?.get('user_id')) {
        // 禁止访问高风险操作
        const isDeleteAllContent = req.method === 'DELETE' && (req.path === '/db/' || req.path === '/db');
        const isTransferOwnership = req.method === 'PUT' && (req.path === '/users/owner/' || req.path === '/users/owner');
        
        if (isDeleteAllContent || isTransferOwnership) {
            return next(new errors.NoPermissionError({
                message: tpl(messages.staffTokenBlocked)
            }));
        }
        return next();
    }

    // CASE 3: Integration Token (无 user_id) - 受白名单限制
    const allowlisted = {
        site: ['GET'],
        posts: ['GET', 'PUT', 'DELETE', 'POST'],
        pages: ['GET', 'PUT', 'DELETE', 'POST'],
        images: ['POST'],
        webhooks: ['POST', 'PUT', 'DELETE'],
        tags: ['GET', 'PUT', 'DELETE', 'POST'],
        users: ['GET'],
        members: ['GET', 'PUT', 'DELETE', 'POST'],
        tiers: ['GET', 'PUT', 'POST'],
        settings: ['GET'],
        comments: ['GET', 'POST', 'PUT'],
        // ... 更多
    };

    const match = req.url.match(/^\/([^/?]+)\/?/);
    if (match) {
        const entity = match[1];
        if (allowlisted[entity] && allowlisted[entity].includes(req.method)) {
            return next();
        }
    }

    next(new errors.NoPermissionError({
        message: tpl(messages.apiTokenBlocked),
        statusCode: 403
    }));
};
```

**Admin 认证方式总结**:

| 认证方式 | 中间件 | 凭证位置 | 适用场景 | 权限范围 | 挂载位置 |
|---------|-------|---------|---------|---------|---------|
| JWT Header | `authAdminApi` | `Authorization: Ghost` | 第三方集成 | Staff Token: 继承用户权限；Integration Token: 白名单限制 | `backend.js` |
| JWT URL | `authAdminApiWithUrl` | `?token=` | 定时任务、后台任务 | 同 JWT Header，但可忽略有效期 | `backend.js` |
| Session | `authAdminApi` | Cookie | 浏览器登录 | 完整用户权限 | `backend.js` |
| 无认证 | `publicAdminApi` | 无 | 公开探测 | 仅 `/site` 端点 | `backend.js` |

---

## 5. 响应字段按身份裁剪机制

Ghost 使用 **四层机制** 实现响应字段的按身份裁剪：

### 5.1 第一层: API 端点分离

相同的数据通过不同的 Controller 暴露，从源头控制访问范围：

| 数据类型 | Content API (backend.js) | Admin API (backend.js) | Members App (frontend.js) |
|---------|------------------------|----------------------|--------------------------|
| Posts | `postsPublic` (只读) | `posts` (完整 CRUD) | 不直接暴露 |
| Pages | `pagesPublic` (只读) | `pages` (完整 CRUD) | 不直接暴露 |
| Users | `authorsPublic` (仅公开字段) | `users` (完整信息) | 不直接暴露 |
| Settings | `publicSettings` (仅公开设置) | `settings` (所有设置) | 不直接暴露 |
| Member 数据 | 不直接暴露 | `members` (管理员管理) | `/api/member` (会员自助) |

**示例: Content API 路由** (`routes.js:17-18`):
```javascript
router.get('/posts', mw.authenticatePublic, http(api.postsPublic.browse));
router.get('/posts/:id', mw.authenticatePublic, http(api.postsPublic.read));
```

**示例: Admin API 路由** (`routes.js:29-38`):
```javascript
router.get('/posts', mw.authAdminApi, http(api.posts.browse));
router.post('/posts', mw.authAdminApi, http(api.posts.add));
router.put('/posts/:id', mw.authAdminApi, http(api.posts.edit));
router.delete('/posts/:id', mw.authAdminApi, http(api.posts.destroy));
```

**示例: Members App 路由** (`app.js:53-62`):
```javascript
membersApp.get('/api/member', middleware.getMemberData);
membersApp.put('/api/member', middleware.updateMemberData);
```

### 5.2 第二层: 查询过滤限制

Content API 在查询层就限制了可过滤的字段：

**位置**: `posts-public.js:103-107`
```javascript
query(frame) {
    const options = {
        ...frame.options,
        mongoTransformer: rejectContentApiRestrictedFieldsTransformer
    };
    return postsService.browsePosts(options);
}
```

**限制字段定义** (`api-filter-utils.ts:3-6`):
```typescript
const CONTENT_API_RESTRICTED_FIELDS = new Set([
    'password',
    'email'
]);

const ADMIN_API_RESTRICTED_FIELDS = new Set([
    'password'
]);
```

| API 类型 | 挂载位置 | 禁止过滤字段 |
|---------|---------|-------------|
| Content API | `backend.js` | `password`, `email` |
| Admin API | `backend.js` | `password` |
| Members App | `frontend.js` | 自行管理（不使用此机制） |

### 5.3 第三层: 序列化器字段裁剪

这是最精细的裁剪层，通过 `mappers` 和 `clean` 工具实现。

#### 5.3.1 Posts 字段裁剪

**位置**: `mappers/posts.js:89-112`

```javascript
if (utils.isContentAPI(frame)) {
    date.forPost(jsonModel);
    gating.forPost(jsonModel, frame);  // 会员内容门控
    previewRendering.forPost(jsonModel, frame);
    
    // Content API 总是返回 comments 状态
    if (jsonModel.access) {
        jsonModel.comments = commentsService?.api?.enabled !== 'off';
    } else {
        jsonModel.comments = false;
    }
    
    // 移除源码格式 (只保留 html)
    delete jsonModel.mobiledoc;
    delete jsonModel.lexical;
}
```

**clean.post 额外裁剪** (`clean.js:81-144`):
```javascript
if (localUtils.isContentAPI(frame)) {
    if (!localUtils.isPreview(frame)) {
        delete attrs.status;      // 隐藏发布状态
    }
    delete attrs.email_only;      // 隐藏邮件专属标记
    delete attrs.newsletter;      // 隐藏邮件通讯关联
    delete attrs.email_segment;   // 隐藏邮件受众分段
}
```

#### 5.3.2 Authors/Users 字段裁剪

**位置**: `clean.js:27-79`

```javascript
const author = (attrs, frame) => {
    if (localUtils.isContentAPI(frame)) {
        // 移除时间戳
        delete attrs.created_at;
        delete attrs.updated_at;
        delete attrs.last_seen;
        
        // 移除敏感信息
        delete attrs.status;
        delete attrs.email;
        
        // 移除通知设置
        delete attrs.comment_notifications;
        delete attrs.free_member_signup_notification;
        delete attrs.paid_subscription_started_notification;
        delete attrs.paid_subscription_canceled_notification;
        delete attrs.mention_notifications;
        delete attrs.recommendation_notifications;
        delete attrs.milestone_notifications;
        delete attrs.donation_notifications;
        delete attrs.gift_subscription_purchase_notification;
        
        // 移除后台专用字段
        delete attrs.accessibility;
        delete attrs.tour;
    }
    
    // 所有 API 都移除的字段
    delete attrs.visibility;
    delete attrs.locale;
    
    return attrs;
};
```

### 5.4 第四层: 会员内容门控 (Content Gating)

这是最智能的裁剪层，根据会员身份决定返回哪些内容。**注意：此机制仅适用于 Content API (backend.js)，Members App (frontend.js) 不使用此机制**。

**位置**: `post-gating.js:84-124`

```javascript
const forPost = (attrs, frame) => {
    // 1. 检查会员是否有权访问完整内容
    const memberHasAccess = membersService.contentGating.checkPostAccess(
        attrs, 
        frame.original.context.member
    );
    
    if (!memberHasAccess) {
        // 2. 如果无权访问，裁剪内容
        const paywallIndex = (attrs.html || '').indexOf('<!--members-only-->');
        
        if (paywallIndex !== -1) {
            // 有付费墙标记：只返回付费墙之前的内容
            attrs.html = attrs.html.slice(0, paywallIndex);
            _updateTextAttrs(attrs);  // 更新 plaintext 和 excerpt
        } else {
            // 无付费墙标记：清空所有内容字段
            ['plaintext', 'html', 'excerpt'].forEach((field) => {
                if (attrs[field] !== undefined) {
                    attrs[field] = '';
                }
            });
        }
    }
    
    // 3. 处理精细门控块 (kg-gated-block)
    const hasGatedBlocks = HAS_GATED_BLOCKS_REGEX.test(attrs.html);
    if (hasGatedBlocks) {
        attrs.html = module.exports.stripGatedBlocks(
            attrs.html, 
            frame.original.context.member
        );
        _updateTextAttrs(attrs);
    }
    
    // 4. 设置访问状态字段
    attrs.access = memberHasAccess;
    
    return attrs;
};
```

**内容可见性级别** (基于 `visibility` 字段):

| visibility 值 | 可见人群 | 挂载应用 |
|--------------|---------|---------|
| `public` | 所有人（访客、免费会员、付费会员） | Content API (backend.js) |
| `members` | 所有会员（免费 + 付费） | Content API (backend.js) |
| `paid` | 仅付费会员 | Content API (backend.js) |
| `tiers` | 特定付费层级会员 | Content API (backend.js) |

---

## 6. 付费文章在四种身份下的响应对照

假设存在一篇付费文章，配置如下：
- `visibility: 'paid'`（仅付费会员可见）
- HTML 内容包含 `<!--members-only-->` 付费墙标记
- 付费墙前有简介内容，付费墙后有完整正文

下面对比 **访客、免费会员、付费会员、管理员** 四种身份访问同一篇文章的响应差异：

### 6.1 四种身份定义

| 身份 | 认证方式 | `req.member` | `req.user` | `req.api_key` | 挂载应用 |
|-----|---------|-------------|-----------|--------------|---------|
| **访客 (Guest)** | 无认证 或 Content API Key | `null` 或 无 | 无 | 可能有 (Content API) | Backend (backend.js) |
| **免费会员 (Free Member)** | GhostMembers JWT 或 Session | `{status: 'free', products: [...]}` | 无 | 无 | Backend (backend.js) |
| **付费会员 (Paid Member)** | GhostMembers JWT 或 Session | `{status: 'paid', products: [...]}` | 无 | 无 | Backend (backend.js) |
| **管理员 (Admin)** | Admin Session 或 Staff Token | 无 | `{role: 'admin', ...}` | 可能有 (Admin API) | Backend (backend.js) |

### 6.2 响应字段详细对照

| 字段 | 访客 (Guest) | 免费会员 (Free) | 付费会员 (Paid) | 管理员 (Admin) | 说明 |
|-----|-------------|----------------|----------------|---------------|------|
| **`access`** | `false` | `false` | `true` | `true` | 是否有权访问完整内容 |
| **`visibility`** | `'paid'` | `'paid'` | `'paid'` | `'paid'` | 文章配置的可见性级别 |
| **`html` (付费墙前)** | ✅ 完整返回 | ✅ 完整返回 | ✅ 完整返回 | ✅ 完整返回 | 简介、 teaser 内容 |
| **`html` (付费墙后)** | ❌ 被截断 | ❌ 被截断 | ✅ 完整返回 | ✅ 完整返回 | 付费专属正文 |
| **`plaintext`** | 仅付费墙前内容 | 仅付费墙前内容 | 完整正文 | 完整正文 | 根据 HTML 自动生成 |
| **`excerpt`** | 仅付费墙前摘要 | 仅付费墙前摘要 | 完整摘要 | 完整摘要 | 500 字符限制 |
| **`status`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | Content API 隐藏发布状态 |
| **`email_only`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 邮件专属标记 |
| **`newsletter`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 关联的邮件通讯 |
| **`email_segment`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 邮件受众分段 |
| **`mobiledoc`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 编辑器源码格式 |
| **`lexical`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 新编辑器源码格式 |
| **`comments`** | 取决于 `access` | 取决于 `access` | ✅ `true` | ✅ 取决于配置 | 是否允许评论 |
| **作者 `email`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 作者邮箱 |
| **作者 `status`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 作者账户状态 |
| **作者 `created_at`** | ❌ 被移除 | ❌ 被移除 | ❌ 被移除 | ✅ 保留 | 作者创建时间 |

### 6.3 响应示例对比

#### 访客访问 (Guest) - Backend App (Content API)

```json
{
  "posts": [{
    "id": "post_123",
    "title": "我的付费文章",
    "slug": "my-paid-post",
    "visibility": "paid",
    "access": false,
    "html": "<p>这是付费墙之前的简介内容...</p>",
    "plaintext": "这是付费墙之前的简介内容...",
    "excerpt": "这是付费墙之前的简介内容...",
    "comments": false,
    "authors": [{
      "id": "author_456",
      "name": "张三",
      "slug": "zhangsan"
    }],
    "tiers": [...]
  }]
}
```

**特点**:
- `access: false` - 明确告知无权限
- `html` 被截断在 `<!--members-only-->` 标记处
- `comments: false` - 无权限时不允许评论
- 作者的敏感字段（email、status、created_at）被移除

#### 免费会员访问 (Free Member) - Backend App (Content API)

```json
{
  "posts": [{
    "id": "post_123",
    "title": "我的付费文章",
    "slug": "my-paid-post",
    "visibility": "paid",
    "access": false,
    "html": "<p>这是付费墙之前的简介内容...</p>",
    "plaintext": "这是付费墙之前的简介内容...",
    "excerpt": "这是付费墙之前的简介内容...",
    "comments": false,
    "authors": [{
      "id": "author_456",
      "name": "张三",
      "slug": "zhangsan"
    }],
    "tiers": [...]
  }]
}
```

**特点**:
- 与访客响应 **完全相同**
- `access: false` - 虽然是会员，但不是付费会员
- 付费内容仍然被裁剪

#### 付费会员访问 (Paid Member) - Backend App (Content API)

```json
{
  "posts": [{
    "id": "post_123",
    "title": "我的付费文章",
    "slug": "my-paid-post",
    "visibility": "paid",
    "access": true,
    "html": "<p>这是付费墙之前的简介内容...</p><!--members-only--><p>这是付费专属的完整正文内容...</p>",
    "plaintext": "这是付费墙之前的简介内容... 这是付费专属的完整正文内容...",
    "excerpt": "这是付费墙之前的简介内容...",
    "comments": true,
    "authors": [{
      "id": "author_456",
      "name": "张三",
      "slug": "zhangsan"
    }],
    "tiers": [...]
  }]
}
```

**特点**:
- `access: true` - 有权访问完整内容
- `html` 包含完整内容（付费墙前后都有）
- `comments: true` - 允许评论
- 但 `mobiledoc`、`lexical` 等源码格式仍被移除（Content API 限制）

#### 管理员访问 (Admin API) - Backend App (Admin API)

```json
{
  "posts": [{
    "id": "post_123",
    "title": "我的付费文章",
    "slug": "my-paid-post",
    "visibility": "paid",
    "status": "published",
    "email_only": false,
    "newsletter": {...},
    "email_segment": "status:-free",
    "access": true,
    "html": "<p>这是付费墙之前的简介内容...</p><!--members-only--><p>这是付费专属的完整正文内容...</p>",
    "mobiledoc": "{...}",
    "lexical": "{...}",
    "plaintext": "这是付费墙之前的简介内容... 这是付费专属的完整正文内容...",
    "excerpt": "这是付费墙之前的简介内容...",
    "authors": [{
      "id": "author_456",
      "name": "张三",
      "slug": "zhangsan",
      "email": "zhangsan@example.com",
      "status": "active",
      "created_at": "2024-01-01T00:00:00.000Z"
    }],
    "tiers": [...]
  }]
}
```

**特点**:
- 所有字段完整保留（无 Content API 裁剪）
- `status`、`email_only`、`newsletter`、`email_segment` 可见
- `mobiledoc`、`lexical` 源码格式保留
- 作者的 `email`、`status`、`created_at` 等敏感字段可见
- `access` 字段可能不存在或始终为 `true`（Admin API 不进行内容门控）

### 6.4 关键差异总结

| 对比维度 | 访客/免费会员 | 付费会员 | 管理员 |
|---------|-------------|---------|-------|
| **`access` 字段** | `false` | `true` | `true` 或不存在 |
| **正文内容** | 付费墙前部分 | 完整 | 完整 |
| **源码格式 (mobiledoc/lexical)** | ❌ 移除 | ❌ 移除 | ✅ 保留 |
| **发布状态 (status)** | ❌ 移除 | ❌ 移除 | ✅ 保留 |
| **作者邮箱 (email)** | ❌ 移除 | ❌ 移除 | ✅ 保留 |
| **邮件相关字段** | ❌ 移除 | ❌ 移除 | ✅ 保留 |
| **评论权限** | `false` | `true` | 取决于配置 |

**设计意图**:
1. **访客/免费会员**: 只能看到 teaser 内容，引导升级
2. **付费会员**: 看到完整内容，但看不到后台管理字段
3. **管理员**: 看到所有字段，包括编辑源码和敏感信息

---

## 7. 鉴权流程总结

### 7.1 Content API 完整流程（Backend App）

```
访客请求 GET /ghost/api/content/posts/{paid-post}
    ↓
boot.js → backend.js → api/app.js → content/app.js
    ↓
1. 检查 ?key= 参数 (Content API Key)
    ├─ 找到 → 设置 req.api_key (集成身份，无会员信息)
    └─ 未找到 → 继续
    ↓
2. 检查 Authorization: GhostMembers header
    ├─ 有效 JWT → 设置 req.member (包含会员身份、订阅状态)
    └─ 无效/无 → req.member = null
    ↓
3. 授权检查
    ├─ req.api_key 或 req.member 存在 → 通过
    └─ 都不存在 → 403 NoPermissionError
    ↓
4. 查询层: 禁止 password/email 过滤
    ↓
5. 序列化层:
    ├─ 移除 mobiledoc/lexical 源码格式
    ├─ 移除 status/email_only 等后台字段
    └─ 根据 req.member 执行内容门控
       ├─ req.member = null (访客) → 裁剪付费内容
       ├─ req.member.status = 'free' → 裁剪付费内容
       └─ req.member.status = 'paid' → 返回完整内容
    ↓
6. 返回裁剪后的响应
```

### 7.2 Members App 会员数据访问流程（Frontend App）

```
会员请求 GET /members/api/member
    ↓
boot.js → frontend.js → members/app.js
    ↓
1. loadMemberSession 中间件
    ├─ 从 Cookie 读取会话
    ├─ 查询会员数据
    └─ 设置 req.member
    ↓
2. getMemberData 处理
    ├─ 返回会员个人信息
    ├─ 返回订阅状态
    └─ 返回访问令牌 (用于 Content API)
    ↓
3. 返回会员数据
```

### 7.3 Admin API (Staff Token) 完整流程（Backend App）

```
管理员请求 PUT /ghost/api/admin/posts/{id}
    ↓
boot.js → backend.js → api/app.js → admin/app.js
    ↓
1. 检查 Authorization: Ghost header
    ├─ 提取 JWT，从 kid 查 API Key
    ├─ 验证签名 (HS256) 和 audience
    ├─ API Key 有 user_id → 查询关联用户
    └─ 设置 req.api_key 和 req.user
    ↓
2. 授权检查: req.user 存在 → 通过
    ↓
3. Token 权限检查:
    ├─ Staff Token 允许大部分操作
    └─ 禁止: DELETE /db, PUT /users/owner
    ↓
4. 查询层: 仅禁止 password 过滤
    ↓
5. 序列化层: 保留完整字段 (无 Content API 裁剪)
    ↓
6. 返回完整响应
```

### 7.4 authAdminApiWithUrl 定时任务流程（Backend App）

```
定时任务请求 PUT /ghost/api/admin/schedules/posts/{id}?token={jwt}
    ↓
boot.js → backend.js → api/app.js → admin/app.js
    ↓
1. authenticateAdminApiWithUrl
    ├─ 从 URL ?token= 提取 JWT
    ├─ 验证签名 (忽略 maxAge)
    └─ 设置 req.api_key
    ↓
2. 授权检查: req.api_key 存在 → 通过
    ↓
3. Token 权限检查
    ↓
4. 执行定时发布
    ↓
5. 返回结果
```

---

## 8. 关键文件索引

### 8.1 入口挂载文件

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| 入口挂载点 | `ghost/core/core/boot.js:240-257` | 挂载 backendApp 和 frontendApp |
| Backend 应用入口 | `ghost/core/core/server/web/parent/backend.js` | 挂载 API 和 Admin UI |
| Frontend 应用入口 | `ghost/core/core/server/web/parent/frontend.js` | 挂载 Members App 和主题路由 |
| API 父应用 | `ghost/core/core/server/web/api/app.js` | 分发到 Content API 和 Admin API |

### 8.2 认证相关文件

| 功能模块 | 文件路径 | 挂载位置 |
|---------|---------|---------|
| Content API Key 认证 | `ghost/core/core/server/services/auth/api-key/content.js` | Backend |
| Members JWT 认证 | `ghost/core/core/server/services/auth/members/index.js` | Backend (Content API 用) |
| Admin API Key 认证 | `ghost/core/core/server/services/auth/api-key/admin.js` | Backend |
| Admin Session 认证 | `ghost/core/core/server/services/auth/session/middleware.js` | Backend |
| 认证入口编排 | `ghost/core/core/server/services/auth/authenticate.js` | Backend |
| 授权检查 | `ghost/core/core/server/services/auth/authorize.js` | Backend |
| Members Session 中间件 | `ghost/core/core/server/services/members/middleware.js` | Frontend |

### 8.3 路由与中间件

| 功能模块 | 文件路径 | 挂载位置 |
|---------|---------|---------|
| Members App 入口 | `ghost/core/core/server/web/members/app.js` | Frontend |
| Content API 中间件 | `ghost/core/core/server/web/api/endpoints/content/middleware.js` | Backend |
| Admin API 中间件 | `ghost/core/core/server/web/api/endpoints/admin/middleware.js` | Backend |
| Content API 路由 | `ghost/core/core/server/web/api/endpoints/content/routes.js` | Backend |
| Admin API 路由 | `ghost/core/core/server/web/api/endpoints/admin/routes.js` | Backend |

### 8.4 响应裁剪文件

| 功能模块 | 文件路径 |
|---------|---------|
| 查询字段限制 | `ghost/core/core/server/api/endpoints/utils/api-filter-utils.ts` |
| Posts 序列化 | `ghost/core/core/server/api/endpoints/utils/serializers/output/mappers/posts.js` |
| 字段清理 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/clean.js` |
| 内容门控 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js` |

---

## 9. 安全要点总结

### 9.1 入口拓扑修正总结

| 之前的错误理解 | 源码中的正确实现 |
|--------------|----------------|
| `/members/*` 挂在 `backend.js` | `/members/*` 挂在 `frontend.js` |
| Content API 和 Members App 同属 backend | Content API → backend.js, Members App → frontend.js |
| 统一的认证体系 | Backend 和 Frontend 各有独立认证体系 |

### 9.2 凭证分层设计

| 凭证类型 | 安全等级 | 挂载位置 | 适用场景 | 关键特性 |
|---------|---------|---------|---------|---------|
| Content API Key | 低 | Backend | 公开集成 | URL 参数，无会员身份 |
| GhostMembers JWT | 高 | Backend (Content API 用) | 会员登录 | RSA 非对称，包含会员身份 |
| Admin JWT (Header) | 最高 | Backend | 第三方管理集成 | HS256 + kid + audience，5 分钟有效期 |
| Admin JWT (URL) | 中 | Backend | 定时任务 | 可忽略有效期，仅限内部调用 |
| Admin Session | 高 | Backend | 浏览器登录 | 完整用户权限，支持 2FA |
| Members Session | 高 | Frontend | 会员浏览器操作 | Cookie 会话，独立于 Backend |
| Members UUID+HMAC | 中 | Frontend | 邮件链接 | HMAC 验证，一次性使用 |

### 9.3 修正后的路由与权限架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Ghost Web 架构 (修正后)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    boot.js (入口挂载点)                          │    │
│  │  ┌──────────────────────────┐    ┌──────────────────────────┐   │    │
│  │  │    backend.js            │    │    frontend.js           │   │    │
│  │  │  (后台应用)               │    │  (前台应用)               │   │    │
│  │  └──────────┬───────────────┘    └──────────┬───────────────┘   │    │
│  └─────────────┼───────────────────────────────┼────────────────────┘    │
│                │                               │                         │
│  ┌─────────────▼─────────────┐     ┌───────────▼───────────────────┐    │
│  │  /ghost/api/*             │     │  /members/*                   │    │
│  │  (api/app.js)             │     │  (members/app.js)             │    │
│  │                           │     │                               │    │
│  │  ┌─────────┐ ┌─────────┐  │     │  认证:                       │    │
│  │  │ Content │ │ Admin   │  │     │    • Session Cookie          │    │
│  │  │  API    │ │  API    │  │     │    • UUID + HMAC             │    │
│  │  │         │ │         │  │     │    • Magic Link Token        │    │
│  │  认证:      │ 认证:      │     │                               │    │
│  │  • API Key │ • JWT     │     │  功能:                         │    │
│  │  • Members │ • Session │     │    • 登录/登出                 │    │
│  │    JWT     │ • URL JWT │     │    • 订阅管理                  │    │
│  │            │ • public  │     │    • 评论                      │    │
│  │            │           │     │    • 个人资料                  │    │
│  └───────────┘ └───────────┘     └───────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.4 字段裁剪四层防护

1. **端点分离**: `postsPublic` vs `posts` - 从源头限制
2. **查询限制**: 禁止 `password`/`email` 过滤
3. **序列化清理**: 移除 `status`、`email`、`notification` 设置等
4. **内容门控**: 根据 `visibility` 和会员身份动态裁剪（仅 Content API）

### 9.5 内容门控设计要点

- **不是简单的 403**: 无权限时返回部分内容 + `access: false`，而非拒绝访问
- **付费墙标记**: `<!--members-only-->` 支持部分内容可见
- **精细块级控制**: `<!--kg-gated-block-->` 支持段级别的门控
- **`access` 字段**: 前端可根据此字段判断是否显示升级提示
- **仅适用于 Backend**: Members App (frontend.js) 不使用此机制

### 9.6 Admin API 安全边界

- **Staff Token vs Integration Token**: Staff Token 继承用户权限，Integration Token 受白名单限制
- **高风险操作保护**: Staff Token 禁止删除所有内容、转让所有权
- **端点白名单**: Integration Token 只能访问明确允许的资源和方法
- **Token 有效期**: 默认 5 分钟，定时任务可例外
- **仅在 Backend**: Admin API 完全独立于 Frontend 的 Members App
