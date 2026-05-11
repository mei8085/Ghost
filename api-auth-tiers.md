# Ghost API 鉴权层级分析报告

## 1. 概述

Ghost 提供 **四套对外接口**，分别服务于不同的用户场景：

| 接口类型 | URL 前缀 | 目标用户 | 主要功能 |
|---------|---------|---------|---------|
| 公开内容接口 (Content API) | `/ghost/api/content/*` | 网站访客、前端应用 | 只读访问公开内容 |
| 会员私域接口 (Members App) | `/members/*` | 订阅会员 | 会员登录、订阅管理、评论 |
| 后台管理接口 (Admin API) | `/ghost/api/admin/*` | 管理员、集成应用 | 完整 CRUD 管理 |
| 会员内容门控 (集成于 Content API) | 同 Content API | 认证会员 | 访问付费/会员专属内容 |

本报告详细分析这几套接口在**凭证类型**、**路由机制**、**权限控制**和**响应裁剪**四个维度的差异。

---

## 2. 凭证类型对比

### 2.1 Content API 凭证体系

Content API 支持 **两种独立的认证方式**，任一通过即可：

#### 方式一: Content API Key（集成凭证）

- **位置**: URL 查询参数 `?key={content_api_key}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/content.js:12`
- **认证流程**:
  1. 从 `req.query.key` 提取密钥
  2. 验证密钥是否存在于数据库
  3. 检查密钥类型是否为 `'content'`
  4. 检查集成限制 (customIntegrations 限制)
  5. 成功后将密钥对象存储在 `req.api_key`

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

#### 方式二: GhostMembers JWT Token（会员凭证）

- **位置**: HTTP Header `Authorization: GhostMembers {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/members/index.js:8`
- **认证流程**:
  1. 从 Authorization 头提取 JWT (scheme 必须为 `GhostMembers`)
  2. 使用 **RS512** 非对称算法验证签名
  3. 验证 audience (站点源 URL) 和 issuer
  4. 成功后将会员信息存储在 `req.member`

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
- `credentialsRequired: false` 意味着未登录用户也可以访问，但不会获得 `req.member`
- **可用于内容门控判断**，决定是否返回付费内容

### 2.2 Content API Key vs GhostMembers Token 对比

| 对比项 | Content API Key | GhostMembers Token |
|-------|----------------|-------------------|
| **传递位置** | URL 查询参数 `?key=` | HTTP Header `Authorization: GhostMembers` |
| **加密方式** | 对称存储（数据库 secret） | RS512 非对称签名 |
| **身份类型** | 集成应用（无用户身份） | 具体会员（有 user/member 身份） |
| **req 对象** | 设置 `req.api_key` | 设置 `req.member` |
| **付费内容访问** | ❌ 不能（无会员身份） | ✅ 可用于内容门控判断 |
| **适用场景** | 前端应用、静态站点生成器 | 会员登录状态、付费内容访问 |
| **过期机制** | 永久有效（可手动撤销） | 短期有效（JWT 过期时间） |

### 2.3 会员私域接口 (Members App) 凭证

Members App 是一个**独立的应用**，使用自己的认证中间件：

**凭证类型一: Session Cookie（浏览器登录）**

- **位置**: Cookie (`ghost-members-ssr` 等)
- **认证中间件**: `loadMemberSession`
- **认证位置**: `ghost/core/core/server/services/members/middleware.js:91`

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

### 2.4 后台管理接口 (Admin API) 凭证

Admin API 支持 **三种认证方式**：

#### 方式一: Admin JWT Token (Header)

- **位置**: HTTP Header `Authorization: Ghost {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/admin.js:48`

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

#### 方式二: Admin JWT Token (URL) - authAdminApiWithUrl

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

#### 方式三: Session Cookie（浏览器登录）

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

---

## 3. 请求路由与权限层

### 3.1 整体路由结构

请求首先通过 Web 入口层路由到不同的应用：

```
HTTP 请求
    ↓
Web 入口路由 (backend.js)
    ├─→ /ghost/api/content/* → Content API 应用
    │       └─ 支持 Content API Key 或 GhostMembers Token
    │
    ├─→ /ghost/api/admin/*   → Admin API 应用
    │       ├─ authAdminApi          (需要认证)
    │       ├─ authAdminApiWithUrl   (URL Token 认证)
    │       └─ publicAdminApi        (无需认证)
    │
    └─→ /members/*           → Members App (独立门户)
            ├─ /members/api/*        (会员相关操作)
            └─ /members/webhooks/*   (Stripe 等 Webhook)
```

### 3.2 Content API 认证链

**位置**: `ghost/core/core/server/web/api/endpoints/content/middleware.js:18`

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

### 3.3 Members App 路由与认证

**位置**: `ghost/core/core/server/web/members/app.js`

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

| 中间件 | 凭证来源 | 用途 |
|-------|---------|------|
| `loadMemberSession` | Cookie | 浏览器登录会员的常规操作 |
| `authMemberByUuid` | URL `?uuid=&key=` | 邮件链接（退订等） |
| `createSessionFromMagicLink` | URL `?token=` | Magic Link 登录 |

### 3.4 Admin API 认证链（三种变体）

Admin API 提供 **三种认证中间件**，适用于不同场景：

#### 3.4.1 authAdminApi（标准认证）

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

#### 3.4.2 authAdminApiWithUrl（URL Token 认证）

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

#### 3.4.3 publicAdminApi（无需认证）

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

### 3.5 Token 权限细粒度检查

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

| 认证方式 | 中间件 | 凭证位置 | 适用场景 | 权限范围 |
|---------|-------|---------|---------|---------|
| JWT Header | `authAdminApi` | `Authorization: Ghost` | 第三方集成 | Staff Token: 继承用户权限；Integration Token: 白名单限制 |
| JWT URL | `authAdminApiWithUrl` | `?token=` | 定时任务、后台任务 | 同 JWT Header，但可忽略有效期 |
| Session | `authAdminApi` | Cookie | 浏览器登录 | 完整用户权限 |
| 无认证 | `publicAdminApi` | 无 | 公开探测 | 仅 `/site` 端点 |

---

## 4. 响应字段按身份裁剪机制

Ghost 使用 **四层机制** 实现响应字段的按身份裁剪：

### 4.1 第一层: API 端点分离

相同的数据通过不同的 Controller 暴露，从源头控制访问范围：

| 数据类型 | Content API | Admin API |
|---------|------------|----------|
| Posts | `postsPublic` (只读) | `posts` (完整 CRUD) |
| Pages | `pagesPublic` (只读) | `pages` (完整 CRUD) |
| Users | `authorsPublic` (仅公开字段) | `users` (完整信息) |
| Settings | `publicSettings` (仅公开设置) | `settings` (所有设置) |

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

### 4.2 第二层: 查询过滤限制

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

| API 类型 | 禁止过滤字段 |
|---------|-------------|
| Content API | `password`, `email` |
| Admin API | `password` |

### 4.3 第三层: 序列化器字段裁剪

这是最精细的裁剪层，通过 `mappers` 和 `clean` 工具实现。

#### 4.3.1 Posts 字段裁剪

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

#### 4.3.2 Authors/Users 字段裁剪

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

### 4.4 第四层: 会员内容门控 (Content Gating)

这是最智能的裁剪层，根据会员身份决定返回哪些内容。

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

| visibility 值 | 可见人群 |
|--------------|---------|
| `public` | 所有人（访客、免费会员、付费会员） |
| `members` | 所有会员（免费 + 付费） |
| `paid` | 仅付费会员 |
| `tiers` | 特定付费层级会员 |

---

## 5. 付费文章在四种身份下的响应对照

假设存在一篇付费文章，配置如下：
- `visibility: 'paid'`（仅付费会员可见）
- HTML 内容包含 `<!--members-only-->` 付费墙标记
- 付费墙前有简介内容，付费墙后有完整正文

下面对比 **访客、免费会员、付费会员、管理员** 四种身份访问同一篇文章的响应差异：

### 5.1 四种身份定义

| 身份 | 认证方式 | `req.member` | `req.user` | `req.api_key` |
|-----|---------|-------------|-----------|--------------|
| **访客 (Guest)** | 无认证 或 Content API Key | `null` 或 无 | 无 | 可能有 (Content API) |
| **免费会员 (Free Member)** | GhostMembers JWT 或 Session | `{status: 'free', products: [...]}` | 无 | 无 |
| **付费会员 (Paid Member)** | GhostMembers JWT 或 Session | `{status: 'paid', products: [...]}` | 无 | 无 |
| **管理员 (Admin)** | Admin Session 或 Staff Token | 无 | `{role: 'admin', ...}` | 可能有 (Admin API) |

### 5.2 响应字段详细对照

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

### 5.3 响应示例对比

#### 访客访问 (Guest)

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

#### 免费会员访问 (Free Member)

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

#### 付费会员访问 (Paid Member)

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

#### 管理员访问 (Admin API)

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

### 5.4 关键差异总结

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

## 6. 鉴权流程总结

### 6.1 Content API 完整流程

```
访客请求 GET /ghost/api/content/posts/{paid-post}
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

### 6.2 Members App 会员数据访问流程

```
会员请求 GET /members/api/member
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

### 6.3 Admin API (Staff Token) 完整流程

```
管理员请求 PUT /ghost/api/admin/posts/{id}
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

### 6.4 authAdminApiWithUrl 定时任务流程

```
定时任务请求 PUT /ghost/api/admin/schedules/posts/{id}?token={jwt}
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

## 7. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Content API Key 认证 | `ghost/core/core/server/services/auth/api-key/content.js` |
| Members JWT 认证 | `ghost/core/core/server/services/auth/members/index.js` |
| Admin API Key 认证 | `ghost/core/core/server/services/auth/api-key/admin.js` |
| Admin Session 认证 | `ghost/core/core/server/services/auth/session/middleware.js` |
| 认证入口编排 | `ghost/core/core/server/services/auth/authenticate.js` |
| 授权检查 | `ghost/core/core/server/services/auth/authorize.js` |
| Members App 入口 | `ghost/core/core/server/web/members/app.js` |
| Members Session 中间件 | `ghost/core/core/server/services/members/middleware.js` |
| Content API 中间件 | `ghost/core/core/server/web/api/endpoints/content/middleware.js` |
| Admin API 中间件 | `ghost/core/core/server/web/api/endpoints/admin/middleware.js` |
| Content API 路由 | `ghost/core/core/server/web/api/endpoints/content/routes.js` |
| Admin API 路由 | `ghost/core/core/server/web/api/endpoints/admin/routes.js` |
| 查询字段限制 | `ghost/core/core/server/api/endpoints/utils/api-filter-utils.ts` |
| Posts 序列化 | `ghost/core/core/server/api/endpoints/utils/serializers/output/mappers/posts.js` |
| 字段清理 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/clean.js` |
| 内容门控 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js` |

---

## 8. 安全要点总结

### 8.1 凭证分层设计

| 凭证类型 | 安全等级 | 适用场景 | 关键特性 |
|---------|---------|---------|---------|
| Content API Key | 低 | 公开集成 | URL 参数，无会员身份 |
| GhostMembers JWT | 高 | 会员登录 | RSA 非对称，包含会员身份 |
| Admin JWT (Header) | 最高 | 第三方管理集成 | HS256 + kid + audience，5 分钟有效期 |
| Admin JWT (URL) | 中 | 定时任务 | 可忽略有效期，仅限内部调用 |
| Session Cookie | 高 | 浏览器登录 | 完整用户权限，支持 2FA |

### 8.2 路由与权限架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Ghost API 架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /ghost/api/content/*                                       │
│  └─ Content API                                             │
│     ├─ 认证: Content API Key 或 GhostMembers JWT            │
│     ├─ 授权: 任一通过即可                                    │
│     └─ 响应: 四层裁剪 (端点→查询→序列化→内容门控)            │
│                                                             │
│  /members/*                                                 │
│  └─ Members App (独立应用)                                   │
│     ├─ 认证: Session Cookie 或 UUID+HMAC                    │
│     ├─ 功能: 登录、订阅、评论、反馈                           │
│     └─ /api/site: 公开探测端点 (无认证)                      │
│                                                             │
│  /ghost/api/admin/*                                         │
│  └─ Admin API                                               │
│     ├─ authAdminApi: JWT Header 或 Session                  │
│     │   └─ Staff Token: 继承用户权限                         │
│     │   └─ Integration Token: 端点白名单限制                 │
│     ├─ authAdminApiWithUrl: URL Token (定时任务)             │
│     └─ publicAdminApi: 无认证 (仅 /site)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 字段裁剪四层防护

1. **端点分离**: `postsPublic` vs `posts` - 从源头限制
2. **查询限制**: 禁止 `password`/`email` 过滤
3. **序列化清理**: 移除 `status`、`email`、`notification` 设置等
4. **内容门控**: 根据 `visibility` 和会员身份动态裁剪

### 8.4 内容门控设计要点

- **不是简单的 403**: 无权限时返回部分内容 + `access: false`，而非拒绝访问
- **付费墙标记**: `<!--members-only-->` 支持部分内容可见
- **精细块级控制**: `<!--kg-gated-block-->` 支持段级别的门控
- **`access` 字段**: 前端可根据此字段判断是否显示升级提示

### 8.5 Admin API 安全边界

- **Staff Token vs Integration Token**: Staff Token 继承用户权限，Integration Token 受白名单限制
- **高风险操作保护**: Staff Token 禁止删除所有内容、转让所有权
- **端点白名单**: Integration Token 只能访问明确允许的资源和方法
- **Token 有效期**: 默认 5 分钟，定时任务可例外
