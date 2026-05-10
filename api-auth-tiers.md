# Ghost API 鉴权层级分析报告

## 1. 概述

Ghost 提供三套对外接口，分别服务于不同的用户场景：

1. **公开内容接口 (Content API)** - 面向网站访客和前端应用
2. **会员私域接口 (Members API)** - 面向订阅会员的认证内容访问
3. **后台管理接口 (Admin API)** - 面向管理员和集成应用

本报告详细分析这三套接口在**凭证类型**、**路由机制**、**权限控制**和**响应裁剪**四个维度的差异。

---

## 2. 凭证类型对比

### 2.1 公开内容接口 (Content API)

**凭证类型：API Key (content 类型)**

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
- 适用于只读的公开内容访问

### 2.2 会员私域接口 (Members API)

**凭证类型：JWT Token (GhostMembers)**

- **位置**: HTTP Header `Authorization: GhostMembers {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/members/index.js:8`
- **认证流程**:
  1. 从 Authorization 头提取 JWT (scheme 必须为 `GhostMembers`)
  2. 使用 RS512 算法验证签名
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
- Token 包含会员身份信息
- `credentialsRequired: false` 意味着未登录用户也可以访问，但不会获得 `req.member`

### 2.3 后台管理接口 (Admin API)

**凭证类型：两种方式**

#### 方式一: JWT Token (Ghost 认证)
- **位置**: HTTP Header `Authorization: Ghost {jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/admin.js:48`

#### 方式二: URL Token (用于定时任务等后台任务)
- **位置**: URL 查询参数 `?token={jwt_token}`
- **认证位置**: `ghost/core/core/server/services/auth/api-key/admin.js:67`

**JWT Token 特殊要求**:
- 必须包含 `kid` (Key ID) 头部，用于查找对应的 API Key
- 必须包含 `audience` 声明，匹配请求的 API 路径
- 使用 HS256 算法，密钥为 API Key 的 secret
- 默认有效期 5 分钟 (定时任务可忽略有效期)

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

**特点**:
- 最严格的认证机制
- 支持两种 token 类型：
  - **Staff Token**: 关联具体用户 (`user_id` 存在)，继承用户权限
  - **Integration Token**: 无关联用户，受 endpoint 白名单限制
- 定时任务 URL 可忽略 token 有效期

#### 方式三: Session 认证 (用于浏览器登录)
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

---

## 3. 请求路由与权限层

### 3.1 整体路由结构

请求首先通过 Web 入口层路由到不同的 API 应用：

```
HTTP 请求
    ↓
Web 入口路由 (backend.js)
    ├─→ /ghost/api/content/* → Content API 应用
    ├─→ /ghost/api/admin/*   → Admin API 应用
    └─→ /members/*           → Members 应用 (门户/订阅相关)
```

### 3.2 认证中间件链

#### 3.2.1 Content API 认证链

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
    apiKeyAuth.content.authenticateContentApiKey,  // 尝试 API Key 认证
    members.authenticateMembersToken               // 尝试 Members JWT 认证
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

#### 3.2.2 Admin API 认证链

**位置**: `ghost/core/core/server/web/api/endpoints/admin/middleware.js:100`

```javascript
module.exports.authAdminApi = [
    auth.authenticate.authenticateAdminApi,        // 1. 认证
    auth.authorize.authorizeAdminApi,              // 2. 授权
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
    apiKeyAuth.admin.authenticate,  // 尝试 Admin API Key JWT 认证
    session.authenticate            // 尝试 Session 认证
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

**Token 权限细粒度检查** (`middleware.js:17-91`):

这个中间件根据 token 类型进行额外限制：

1. **用户登录 (Session)**: 无额外限制，使用用户本身的权限系统
2. **Staff Token (有 user_id)**: 
   - 禁止访问 `/db/` DELETE (删除所有内容)
   - 禁止访问 `/users/owner/` PUT (转让所有权)
3. **Integration Token (无 user_id)**:
   - 受 endpoint 白名单限制 (见下文)

**Integration Token 白名单** (`middleware.js:46-75`):
```javascript
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
    // ... 更多
};
```

---

## 4. 响应字段按身份裁剪机制

Ghost 使用三层机制实现响应字段的按身份裁剪：

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

- **Content API**: 禁止按 `password` 和 `email` 字段过滤
- **Admin API**: 仅禁止按 `password` 字段过滤

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

#### 4.3.3 Tags 字段裁剪

**位置**: `clean.js:4-25`

```javascript
const tag = (attrs, frame) => {
    if (localUtils.isContentAPI(frame)) {
        delete attrs.created_at;
        delete attrs.updated_at;
    }
    delete attrs.parent_id;
    delete attrs.parent;
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
- `public`: 所有人可见
- `members`: 所有会员可见 (免费 + 付费)
- `paid`: 仅付费会员可见
- `tiers`: 特定付费层级可见

---

## 5. 鉴权流程总结

### 5.1 Content API 完整流程

```
访客请求 GET /ghost/api/content/posts/
    ↓
1. 检查 ?key= 参数 (Content API Key)
    ├─ 找到 → 设置 req.api_key
    └─ 未找到 → 继续
    ↓
2. 检查 Authorization: GhostMembers header
    ├─ 有效 JWT → 设置 req.member
    └─ 无效/无 → 继续
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
    └─ 根据会员身份执行内容门控
    ↓
6. 返回裁剪后的响应
```

### 5.2 Admin API (Staff Token) 完整流程

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

### 5.3 Members 私域内容访问流程

```
付费会员请求 GET /ghost/api/content/posts/{paid-post}
    ↓
1. Content API Key 或 Members JWT 认证
    ↓
2. 授权通过
    ↓
3. 内容门控检查:
    ├─ 帖子 visibility = 'paid'
    ├─ 会员 status = 'paid'
    └─ checkPostAccess 返回 true
    ↓
4. 返回完整 HTML 内容
    └─ access: true
```

```
免费会员/访客请求 GET /ghost/api/content/posts/{paid-post}
    ↓
1. 认证通过 (API Key 或无 token 访问)
    ↓
2. 授权通过
    ↓
3. 内容门控检查:
    ├─ 帖子 visibility = 'paid'
    ├─ 非付费会员
    └─ checkPostAccess 返回 false
    ↓
4. 内容裁剪:
    ├─ 找到 <!--members-only--> 标记
    ├─ 只返回标记前的内容
    └─ 或完全清空 html/plaintext/excerpt
    ↓
5. 返回裁剪后的内容
    └─ access: false
```

---

## 6. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Content API 认证 | `ghost/core/core/server/services/auth/api-key/content.js` |
| Admin API 认证 | `ghost/core/core/server/services/auth/api-key/admin.js` |
| Members 认证 | `ghost/core/core/server/services/auth/members/index.js` |
| Session 认证 | `ghost/core/core/server/services/auth/session/middleware.js` |
| 认证入口编排 | `ghost/core/core/server/services/auth/authenticate.js` |
| 授权检查 | `ghost/core/core/server/services/auth/authorize.js` |
| Content API 中间件 | `ghost/core/core/server/web/api/endpoints/content/middleware.js` |
| Admin API 中间件 | `ghost/core/core/server/web/api/endpoints/admin/middleware.js` |
| Content API 路由 | `ghost/core/core/server/web/api/endpoints/content/routes.js` |
| Admin API 路由 | `ghost/core/core/server/web/api/endpoints/admin/routes.js` |
| 查询字段限制 | `ghost/core/core/server/api/endpoints/utils/api-filter-utils.ts` |
| Posts 序列化 | `ghost/core/core/server/api/endpoints/utils/serializers/output/mappers/posts.js` |
| 字段清理 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/clean.js` |
| 内容门控 | `ghost/core/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js` |

---

## 7. 安全要点总结

1. **凭证分层**:
   - Content API Key 最简单，适合公开集成
   - Members JWT 使用 RSA 非对称加密，安全性高
   - Admin API JWT 要求最严格，包含 kid 和 audience 验证

2. **授权策略**:
   - Content API: API Key 或 Members Token 任一即可
   - Admin API: 用户身份 (Session/Staff Token) 或 Integration Token
   - Integration Token 受 endpoint 白名单严格限制

3. **字段裁剪**:
   - 四层防护: 端点分离 → 查询限制 → 序列化清理 → 内容门控
   - Content API 永远不返回 email、status 等敏感字段
   - 会员内容根据身份动态裁剪，不是简单的 403 拒绝

4. **内容门控**:
   - 支持 `<!--members-only-->` 付费墙标记
   - 支持 `<!--kg-gated-block-->` 精细块级控制
   - 响应包含 `access` 字段指示是否有权访问完整内容
