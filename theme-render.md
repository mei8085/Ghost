# Ghost 主题渲染权限与兼容性分析报告

## 目录
0. [Private Site 私密站点模式：全站级访问控制](#0-private-site-私密站点模式全站级访问控制)
1. [数据查询权限裁剪逻辑](#1-数据查询权限裁剪逻辑)
2. [完整权限链路：访客身份 → visibility → access → Handlebars 渲染](#2-完整权限链路访客身份--visibility--access--handlebars-渲染)
3. [Handlebars Helper 内容过滤机制](#3-handlebars-helper-内容过滤机制)
4. [服务端渲染期会员凭证校验](#4-服务端渲染期会员凭证校验)
5. [主题升级兼容兜底策略](#5-主题升级兼容兜底策略)

---

## 0. Private Site 私密站点模式：全站级访问控制

### 0.1 概述

**启用条件**：设置 `is_private = true` + `password`（访问密码）

Private Site 是 Ghost 的**全站级访问控制**机制，与 Members 内容权限系统完全独立。启用后：
- 所有未认证访客将被重定向到 `/private/` 页面
- 需要输入正确的访问密码才能进入站点
- 与 `posts.visibility` 和 `member` 权限裁剪是**层叠关系**（先过 private site 这一关，再经过 members 权限检查）

### 0.2 请求管线中的挂载位置

**文件**：`ghost/core/core/frontend/web/site.js` 第 89-95 行

```javascript
// setupMiddleware 在 loadMemberSession 之前执行
config.get('apps:internal').forEach((appName) => {
    const app = require(path.join(config.get('paths').internalAppPath, appName));
    if (Object.prototype.hasOwnProperty.call(app, 'setupMiddleware')) {
        app.setupMiddleware(siteApp);  // 这里挂载 private-blogging 的中间件
    }
});

// loadMemberSession 在第 109 行
siteApp.use(membersService.middleware.loadMemberSession);
```

**private-blogging/index.js** 第 49-56 行：
```javascript
setupMiddleware: function setupMiddleware(siteApp) {
    siteApp.use(middleware.checkIsPrivate);       // 第一步：检查是否开启 private 模式
    siteApp.use(middleware.filterPrivateRoutes);  // 第二步：路由过滤与认证
},

setupErrorHandling: function setupErrorHandling(siteApp) {
    siteApp.use(middleware.handle404);  // 错误处理：404 时的重定向
}
```

**完整管线顺序**：
```
HTTP 请求
   │
   ├─► privateBlogging.checkIsPrivate    ← 设置 res.isPrivateBlog
   ├─► privateBlogging.filterPrivateRoutes ← 重定向未认证用户
   │
   ├─► loadMemberSession                  ← 加载会员（仅在通过 private 检查后才会到达）
   ├─► themeMiddleware
   ├─► siteRoutes
   │      └─► fetch-data → post-gating → visibility/access 裁剪（第1-2节逻辑）
   │
   └─► privateBlogging.handle404         ← 404 时的二次检查
```

### 0.3 ghost-private 会话机制

**文件**：`ghost/core/core/frontend/apps/private-blogging/lib/middleware.js` 第 70-75 行

```javascript
// checkIsPrivate 中间件
checkIsPrivate: function checkIsPrivate(req, res, next) {
    let isPrivateBlog = settingsCache.get('is_private');

    if (!isPrivateBlog) {
        res.isPrivateBlog = false;
        return next();
    }

    res.isPrivateBlog = true;

    // 初始化 cookie-session
    return session({
        name: 'ghost-private',           // Cookie 名称
        maxAge: (30 * 24 * 60 * 60 * 1000), // 30天，单位：毫秒
        signed: false,                   // ⚠️ 不使用签名（见下文安全分析）
        sameSite: 'none'
    })(req, res, next);
},
```

**Session 存储内容**：
```javascript
// doLoginToPrivateSite 第 168-175 行
const hasher = crypto.createHash('sha256');
const salt = Date.now().toString();
// ... 验证密码成功后
hasher.update(submittedAccessCode + salt, 'utf8');
req.session.token = hasher.digest('hex');  // SHA256(password + salt)
req.session.salt = salt;                   // 时间戳盐值
```

**Session 验证**：
```javascript
// verifySessionHash 第 27-37 行
function verifySessionHash(salt, hash) {
    const accessCode = settingsCache.get('password');

    if (!salt || !hash || !hasAccessCode(accessCode)) {
        return false;
    }

    let hasher = crypto.createHash('sha256');
    hasher.update(accessCode + salt, 'utf8');
    return hasher.digest('hex') === hash;
}
```

**安全设计分析**：

| 特性 | 实现方式 | 说明 |
|------|----------|------|
| 防重放 | `salt = Date.now()` | 使用登录时的时间戳作为盐，每次登录生成不同的 token |
| 防篡改 | `signed: false` | ⚠️ **注意**：Session 数据不使用 HMAC 签名 |
| 存储位置 | Cookie | 整个 session 对象存储在 Cookie 中（cookie-session） |
| 验证方式 | 服务器端重算哈希 | 不存储 session，通过重新计算 SHA256(password+salt) 验证 |

**⚠️ 重要安全注意**：
- `signed: false` 意味着 session cookie **没有签名保护**
- 但由于 token 本身就是 `SHA256(password + salt)`，即使攻击者能修改 cookie，也需要知道 password 才能伪造有效 token
- 与 `ghost-members-ssr` 使用的 `Keygrip` 签名机制不同（见第 4.3.3 节）

### 0.4 访问放行与重定向规则

**文件**：`middleware.js` 第 78-125 行 `filterPrivateRoutes`

#### 0.4.1 无条件放行的路径

| 路径 | 处理方式 | 原因 |
|------|----------|------|
| `/private/` | 直接放行 `next()` | 登录页面本身必须可访问 |
| `/robots.txt` | 直接返回定制版 | 防止搜索引擎索引私有内容 |

**robots.txt 内容**：
```
User-agent: *
Disallow: /
```

#### 0.4.2 Private RSS Feed（特殊处理）

```javascript
// 第 107-112 行
let isPrivateRSS = new RegExp(`/${settingsCache.get('public_hash')}/rss(/)?$`);
if (isPrivateRSS.test(req.path)) {
    req.url = req.url.replace(settingsCache.get('public_hash') + '/', '');
    return next();
}
```

**设计目的**：
- `public_hash` 是一个随机哈希（站点设置）
- RSS 阅读器无法输入密码，但可以在 URL 中携带 secret
- 例如：`/777aaa/rss/` → 重写为 `/rss/` 并放行

#### 0.4.3 正常认证流程

```javascript
// 第 114-124 行
privateBlogging.authenticatePrivateSession(req, res, function onSessionVerified() {
    // CASE: 认证通过后，禁止普通 RSS（但 private RSS 已在上面放行）
    if (req.path.match(/\/rss\/$/)) {
        return next(new errors.NotFoundError({message: 'Page not found.'}));
    }
    next();  // 继续正常请求处理
});
```

**authenticatePrivateSession** 逻辑（第 127-140 行）：
```javascript
authenticatePrivateSession: function authenticatePrivateSession(req, res, next) {
    const hash = req.session.token || '';
    const salt = req.session.salt || '';
    const isVerified = verifySessionHash(salt, hash);

    if (isVerified) {
        return next();  // 认证通过
    } else {
        // 认证失败，重定向到登录页，保留原 URL
        let redirectUrl = urlUtils.urlFor({relativeUrl: privateRoute});
        redirectUrl += '?r=' + encodeURIComponent(req.url);
        return res.redirect(redirectUrl);
    }
}
```

**重定向 URL 安全验证**（第 39-57 行）：
```javascript
function getRedirectUrl(query) {
    try {
        const redirect = decodeURIComponent(query.r || '/');
        const parsedUrl = new URL(redirect, config.get('url'));
        const pathname = parsedUrl.pathname;
        const search = parsedUrl.search;

        const base = new URL(config.get('url'));
        const target = new URL(pathname, config.get('url'));
        
        // 关键：确保不会重定向到外部域名（防钓鱼）
        if (target.host !== base.host) {
            return '/';
        }
        return pathname + search;
    } catch (e) {
        return '/';
    }
}
```

### 0.5 登录流程

**文件**：`lib/router.js` 第 29-42 行

```javascript
privateRouter
    .route('/')
    .get(
        middleware.redirectPrivateToHomeIfLoggedIn,  // 已登录则直接跳首页
        _renderer
    )
    .post(
        bodyParser.urlencoded({extended: true}),
        middleware.redirectPrivateToHomeIfLoggedIn,
        web.shared.middleware.brute.privateBlog,      // 防暴力破解
        middleware.doLoginToPrivateSite,
        _renderer
    );
```

**登录处理**（middleware.js 第 160-184 行）：
```javascript
doLoginToPrivateSite: function doLoginToPrivateSite(req, res, next) {
    const submittedAccessCode = req.body && req.body.password;
    const accessCode = settingsCache.get('password');
    const forward = getRedirectUrl(req.query);

    if (hasAccessCode(accessCode) && hasAccessCode(submittedAccessCode) && accessCode === submittedAccessCode) {
        // 密码正确：设置 session 并重定向
        const hasher = crypto.createHash('sha256');
        const salt = Date.now().toString();
        hasher.update(submittedAccessCode + salt, 'utf8');
        req.session.token = hasher.digest('hex');
        req.session.salt = salt;
        return res.redirect(urlUtils.urlFor({relativeUrl: forward}));
    } else {
        // 密码错误：返回登录页并显示错误
        res.error = {message: 'Incorrect access code.'};
        return next();
    }
}
```

### 0.6 404 特殊处理

**文件**：`middleware.js` 第 186-205 行

```javascript
handle404: function handle404(err, req, res, next) {
    // 非私有站点：继续下一个错误处理器
    if (!res.isPrivateBlog) {
        return next(err);
    }
    
    // 非 404 错误：直接抛出
    if (err.statusCode !== 404) {
        return next(err);
    }
    
    // 关键安全设计：404 时也要检查认证
    // 未认证用户：重定向到 /private/（防止通过 404 泄露路径存在性）
    // 已认证用户：显示正常的 404 页面
    return privateBlogging.authenticatePrivateSession(req, res, function onSessionVerified() {
        return next(err);
    });
}
```

**设计意图**：防止攻击者通过尝试不同路径并观察是 404 还是重定向来推断站点的内容结构。

### 0.7 Frontend Caching 策略

**文件**：`ghost/core/core/frontend/web/middleware/frontend-caching.js`

#### 0.7.1 请求管线中的位置

```javascript
// site.js 第 122-130 行
// 在 checkIsPrivate/filterPrivateRoutes 之后，siteRoutes 之前
siteApp.use(async function frontendCaching(req, res, next) {
    try {
        const middleware = await mw.frontendCaching.getMiddleware();
        return middleware(req, res, next);
    } catch {
        return next();
    }
});
```

**关键点**：`checkIsPrivate` 先设置 `res.isPrivateBlog`，然后 `frontendCaching` 基于此标志决定缓存策略。

#### 0.7.2 缓存策略决策

**核心判断**（第 53-57 行）：
```javascript
const shouldCacheMembersContent = config.get('cacheMembersContent:enabled');

if (res.isPrivateBlog || (req.member && !shouldCacheMembersContent)) {
    return shared.middleware.cacheControl('private')(req, res, next);
}
```

**完整决策树**：

```
HTTP 请求
   │
   ├─► checkIsPrivate 设置 res.isPrivateBlog
   │
   └─► frontend-caching 决策
         │
         ├─► 预览请求? ──────► Cache-Control: noCache
         │
         ├─► res.isPrivateBlog?
         │    ├─► YES ─────────► Cache-Control: private (完全不缓存)
         │    │
         │    └─► NO ──────────► 继续判断
         │
         ├─► req.member 存在?
         │    ├─► NO ─────────► Cache-Control: public, max-age=...
         │    │
         │    └─► YES ─────────► 继续判断
         │
         └─► shouldCacheMembersContent?
              ├─► NO ──────────► Cache-Control: private
              │
              └─► YES ─────────► 继续判断
                   │
                   ├─► 多活跃订阅? ──► Cache-Control: private
                   │
                   └─► 单订阅/免费 ──► Cache-Control: public, X-Member-Cache-Tier: <tierId>
```

#### 0.7.3 cacheControl 配置

**文件**：`ghost/core/core/server/web/shared/middleware/cache-control.js` 第 25-29 行

```javascript
const profiles = {
    public: 'public, max-age=<value>, stale-while-revalidate=<value>',
    noCache: 'no-cache, max-age=0, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0',
    private: 'no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0'
};
```

**Private Site 模式下的实际响应头**：
```
Cache-Control: no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0
```

**这意味着**：
- `private`：仅可被用户的浏览器缓存，不可被共享缓存（CDN）存储
- `no-store`：**完全禁止**在任何缓存中存储响应
- `no-cache`：即使缓存了，每次使用前必须重新验证
- `must-revalidate`：缓存过期后必须回源验证

**设计目的**：
1. 防止私有站点的内容被 CDN/代理缓存泄露
2. 确保每次请求都经过 Private Site 认证检查
3. 与 `ghost-private` 会话机制配合，实现即时访问控制

### 0.8 密码变更后会话整体失效机制

**文件**：`ghost/core/core/frontend/apps/private-blogging/lib/middleware.js` 第 27-37 行

#### 0.8.1 验证流程

```javascript
function verifySessionHash(salt, hash) {
    const accessCode = settingsCache.get('password');  // 每次验证重新读取密码

    // 第 30-32 行：密码为空 → 直接返回 false
    if (!salt || !hash || !hasAccessCode(accessCode)) {
        return false;
    }

    // 第 34-36 行：用当前密码重算哈希，与 session 中的 hash 比较
    let hasher = crypto.createHash('sha256');
    hasher.update(accessCode + salt, 'utf8');
    return hasher.digest('hex') === hash;
}
```

#### 0.8.2 失效原理

**登录时写入的 session**：
```
session = {
    token: SHA256(oldPassword + salt),  // 旧密码计算的哈希
    salt: "1715412345678"               // 登录时的时间戳
}
```

**密码变更后的验证**：
```
访问时 verifySessionHash(session.salt, session.token):

1. 读取 settingsCache.get('password') → 得到 newPassword（新密码）
2. 计算 SHA256(newPassword + session.salt)
3. 与 session.token（SHA256(oldPassword + salt)）比较
4. 不相等 → return false → 认证失败 → 重定向到 /private/
```

#### 0.8.3 三种失效场景

| 场景 | 验证结果 | 用户体验 |
|------|----------|----------|
| **管理员变更密码** | `SHA256(新密码+salt)` ≠ `session.token` | 所有已登录用户被踢下线，需重新输入新密码 |
| **管理员清除密码** (`password: ''`) | `hasAccessCode('')` → `false` | 所有已登录用户被踢下线 |
| **用户自行登出** | cookie 被清除 | 下次访问需重新登录 |

#### 0.8.4 测试验证

**文件**：`test/unit/frontend/apps/private-blogging/middleware.test.js` 第 415-428 行

```javascript
it('authenticatePrivateSession should redirect when stored access code is empty', function () {
    const salt = Date.now().toString();
    settingsStub.withArgs('password').returns('');  // 密码被清除
    req.url = '/welcome';
    req.session = {
        token: hash('', salt),
        salt
    };

    privateBlogging.authenticatePrivateSession(req, res, next);
    sinon.assert.notCalled(next);           // 不调用 next()
    sinon.assert.called(res.redirect);      // 触发重定向
    sinon.assert.calledWith(res.redirect, '/private/?r=%2Fwelcome');
});
```

#### 0.8.5 设计优势

| 特性 | 说明 |
|------|------|
| **无状态失效** | 不需要在服务器端维护 session 黑名单 |
| **即时生效** | 密码变更立即影响所有现有会话，无需等待 cookie 过期 |
| **零额外存储** | 不需要数据库记录 active sessions |
| **自动处理** | 与 `settingsCache` 集成，密码更新后自动生效 |

**对比 Members 系统**：
- `ghost-private`：通过**哈希比对**实现无状态失效
- `ghost-members-ssr`：通过 **transient_id 数据库查询**实现（可在数据库层面撤销）

### 0.9 与 Members 权限系统的职责边界

| 维度 | Private Site | Members Visibility / Access |
|------|-------------|----------------------------|
| **控制层级** | 全站级 | 单篇文章级 |
| **触发位置** | 请求管线最前端（`filterPrivateRoutes`） | API 序列化层（`post-gating.js`） |
| **认证方式** | 统一访问密码 | 会员系统（Magic Link / 订阅） |
| **会话 Cookie** | `ghost-private`（cookie-session） | `ghost-members-ssr`（Keygrip 签名） |
| **会话时长** | 30 天（`30 * 24 * 60 * 60 * 1000` ms） | 约 6 个月（`1000 * 60 * 60 * 24 * 184` ms） |
| **效果** | 未通过 → 重定向到 `/private/` | 未通过 → 裁剪内容 + 设置 `access=false` |
| **用户感知** | 必须先输入密码才能看到任何内容 | 能看到文章列表，但会员专享内容被裁剪 |

**层叠关系**：

```
┌──────────────────────────────────────────────────────────────────────┐
│                    HTTP 请求                                          │
│                        │                                             │
│              ┌─────────▼─────────┐                                   │
│              │  Private Site     │                                   │
│              │  (checkIsPrivate) │                                   │
│              │                   │                                   │
│              │  is_private=false │──────┐                              │
│              │       OR          │      │ 继续                          │
│              │  已认证通过        │      │                              │
│              └───────────────────┘      │                              │
│                        │               │                              │
│              ┌─────────▼─────────┐     │                              │
│              │  认证失败         │◄────┘                              │
│              │  重定向到 /private│                                    │
│              └───────────────────┘                                    │
│                        │                                             │
│              ┌─────────▼─────────┐                                   │
│              │  loadMemberSession│  ← 仅在通过 Private 检查后执行        │
│              │  (Members 会员)   │                                   │
│              └───────────────────┘                                    │
│                        │                                             │
│              ┌─────────▼─────────┐                                   │
│              │  fetch-data       │                                   │
│              │  post-gating      │  ← 按 members visibility 裁剪内容    │
│              │  (第1-2节逻辑)    │                                   │
│              └───────────────────┘                                    │
│                        │                                             │
│              ┌─────────▼─────────┐                                   │
│              │  主题渲染          │                                   │
│              │  {{#has visibility}}│                                   │
│              │  {{#if access}}    │                                   │
│              └───────────────────┘                                    │
└──────────────────────────────────────────────────────────────────────┘
```

**启用组合场景**：

1. **只启用 Private Site**：全站需要密码，所有通过认证的用户看到相同内容（post-gating 仍生效，但 member=null，所以 members/paid 文章会被裁剪）

2. **只启用 Members**：公开内容可见，会员专享内容需要登录

3. **同时启用两者**：必须先输入 Private Site 密码 → 然后再登录会员账号 → 才能看到付费内容

---

## 1. 数据查询权限裁剪逻辑

### 1.1 核心权限裁剪模块

位置：`ghost/core/core/server/services/members/content-gating.js`

Ghost 实现了一个专门的内容权限裁剪模块，负责根据会员身份决定是否允许访问特定内容。

#### 1.1.1 核心函数

**`checkPostAccess(post, member)`** - 检查文章访问权限

访问规则：
- **public**：所有用户可访问（无论是否登录）
- **members**：已登录会员可访问
- **paid**：付费会员可访问（排除 free 状态）
- **tiers**：特定会员层级（tier）可访问

权限判断流程：

```
1. 检查 post.visibility === 'public' → 允许访问
2. 检查 member 是否存在 → 不存在则拒绝
3. 检查 post.visibility === 'members' → 允许访问
4. 检查 paid → 转换为 NQL: 'status:-free'
5. 检查 tiers → 转换为 NQL: 'product:tier-slug'
6. 使用 NQL 查询匹配 member 对象
7. 返回 PERMIT_ACCESS / BLOCK_ACCESS
```

代码实现关键片段：
```javascript
function checkPostAccess(post, member) {
    if (post.visibility === 'public') {
        return PERMIT_ACCESS;
    }

    if (!member) {
        return BLOCK_ACCESS;
    }

    if (post.visibility === 'members') {
        return PERMIT_ACCESS;
    }

    let visibility = post.visibility === 'paid' ? 'status:-free' : post.visibility;
    if (visibility === 'tiers') {
        if (!post.tiers) {
            return BLOCK_ACCESS;
        }
        visibility = post.tiers.map((product) => {
            return `product:'${product.slug}'`;
        }).join(',');
    }

    if (visibility && member.status && nql(visibility, {expansions: MEMBER_NQL_EXPANSIONS, transformer: rejectUnknownKeys}).queryJSON(member)) {
        return PERMIT_ACCESS;
    }

    return BLOCK_ACCESS;
}
```

#### 1.1.2 Gated Block 访问控制

**`checkGatedBlockAccess(gatedBlockParams, member)`** - 检查门控区块访问权限

支持的参数：
- `nonMember: true/false` - 是否仅对非会员显示
- `memberSegment: NQL查询` - 会员细分筛选

允许的 memberSegment 值：
- `""` - 空字符串（所有已登录用户都拒绝）
- `"status:free"` - 免费会员
- `"status:-free"` - 付费会员
- `"status:free,status:-free"` - 所有会员

判断逻辑：
```
1. nonMember=true 且 未登录 → 允许
2. nonMember=false 或 memberSegment 未设置 且 已登录 → 拒绝
3. memberSegment + 已登录 → 使用 NQL 匹配
   - 空 memberSegment 或无效键 → 拒绝
   - NQL 匹配成功 → 允许
```

### 1.2 数据序列化时的内容裁剪

位置：`ghost/core/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js`

在数据序列化输出阶段，Ghost 会根据会员权限对文章内容进行裁剪：

#### 1.2.1 裁剪流程

```
1. 默认设置 access = true
2. 调用 checkPostAccess(attrs, frame.original.context.member) 检查权限
3. 无权限时：
   - 查找 <!--members-only--> 分隔符
   - 找到则只保留分隔符之前的内容
   - 没找到则清空 plaintext、html、excerpt
4. 处理 gated blocks（内部 HTML 区块）
5. 根据权限更新 access 字段
6. 替换 Transistor 嵌入的 {uuid} 占位符
```

#### 1.2.2 Gated Block 处理

Gated Block 格式：
```html
<!--kg-gated-block:begin nonMember:true memberSegment:"status:free"-->
    门控内容
<!--kg-gated-block:end-->
```

处理逻辑：
```javascript
const stripGatedBlocks = function (html, member) {
    return html.replace(GATED_BLOCK_REGEX, (match, params, content) => {
        const gatedBlockParams = module.exports.parseGatedBlockParams(params);
        const checkResult = membersService.contentGating.checkGatedBlockAccess(gatedBlockParams, member);

        if (checkResult === PERMIT_ACCESS) {
            return content;  // 保留内容，移除包裹注释
        } else {
            return '';  // 完全移除
        }
    });
};
```

---

## 2. 完整权限链路：访客身份 → visibility → access → Handlebars 渲染

### 2.1 整体数据流概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HTTP 请求进入                                │
│                              ↓                                      │
│               site.js - 前端站点应用入口                             │
│                              ↓                                      │
│               membersService.middleware.loadMemberSession           │
│                              ↓                                      │
│                    从 Cookie 读取 transient_id                      │
│                    验证签名 → 查询会员数据                            │
│                              ↓                                      │
│           req.member, res.locals.member = member 对象 / null        │
│                              ↓                                      │
│               routing controllers (entry.js, collection.js)         │
│                              ↓                                      │
│           fetch-data.js: processQuery(query, slugParam, locals)     │
│           query.options.context = {member: locals.member}           │
│                              ↓                                      │
│           api.postsPublic.browse(query.options)                     │
│                              ↓                                      │
│           serializers/output/mappers/posts.js                       │
│           gating.forPost(jsonModel, frame)                          │
│           使用 frame.original.context.member                        │
│                              ↓                                      │
│           设置 attrs.access = true/false                            │
│           根据权限裁剪 html/excerpt                                  │
│                              ↓                                      │
│           渲染层：renderer.js → res.render(template, data)          │
│                              ↓                                      │
│           Handlebars 模板渲染                                        │
│           - {{#has visibility="..."}} - 判断文章公开级别              │
│           - {{#foreach posts visibility="..."}} - 循环过滤          │
│           - {{#if access}} - 控制内容展示                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 阶段一：请求管线与会话建立

**文件**：`ghost/core/core/frontend/web/site.js`, `ghost/core/core/server/services/members/middleware.js`

```javascript
// members-ssr.js 第91-101行
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

**关键点**：
- 失败时设置 `member = null`，不是抛出异常中断请求
- 未登录用户仍可访问公开内容

### 2.3 阶段二：数据查询与上下文传递

**文件**：`ghost/core/core/frontend/services/data/fetch-data.js`

```javascript
// fetch-data.js 第57行
function processQuery(query, slugParam, locals) {
    const api = require('../proxy').api;
    // ...
    query.options.context = {member: locals.member};
    return (api[query.controller] || api[query.resource])[query.type](query.options);
}
```

**文件**：`ghost/core/core/frontend/services/proxy.js` 第57行
```javascript
api: require('../../server/api').endpoints,  // 直接使用后端 API
```

**关键点**：
- `locals.member` 来自 `res.locals.member`（即中间件加载的会员）
- 所有 API 查询都会带上 `context.member`
- 使用的是内部 API 端点（postsPublic），不是 HTTP Content API

### 2.4 阶段三：输出序列化与权限裁剪

**文件**：`ghost/core/core/server/api/endpoints/utils/serializers/output/mappers/posts.js`

```javascript
// posts.js 第89-92行
if (utils.isContentAPI(frame)) {
    date.forPost(jsonModel);
    gating.forPost(jsonModel, frame);  // ← 关键：权限裁剪入口
    previewRendering.forPost(jsonModel, frame);
}
```

**文件**：`ghost/core/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js` 第85-120行

```javascript
// 第85-88行：默认 access = true
if (!Object.prototype.hasOwnProperty.call(frame.options, 'columns') || (frame.options.columns.includes('access'))) {
    attrs.access = true;
}

// 第90行：关键 - 使用 frame.original.context.member
const memberHasAccess = membersService.contentGating.checkPostAccess(attrs, frame.original.context.member);

// 第92-105行：无权限时裁剪内容
if (!memberHasAccess) {
    const paywallIndex = (attrs.html || '').indexOf('<!--members-only-->');
    if (paywallIndex !== -1) {
        attrs.html = attrs.html.substring(0, paywallIndex);
        // ... 更新 plaintext, excerpt
    } else {
        attrs.plaintext = '';
        attrs.html = '';
        attrs.excerpt = '';
    }
}

// 第107-110行：处理 gated blocks
if (hasGatedBlocks) {
    attrs.html = module.exports.stripGatedBlocks(attrs.html, frame.original.context.member);
}

// 第114-117行：替换 Transistor {uuid} 占位符
const member = frame.original.context.member;
if (member && member.uuid && attrs.html) {
    attrs.html = attrs.html.replace(/%7Buuid%7D/gi, member.uuid);
}

// 第119-122行：最终设置 access
if (!Object.prototype.hasOwnProperty.call(frame.options, 'columns') || (frame.options.columns.includes('access'))) {
    attrs.access = memberHasAccess;
}
```

### 2.5 阶段四：模板渲染与条件分支

**文件**：`ghost/core/core/frontend/services/rendering/renderer.js`

```javascript
module.exports = function renderer(req, res, data) {
    setContext(req, res, data);       // 设置请求上下文
    templates.setTemplate(req, res, data);  // 选择模板
    res.render(res._template, data, function (err, html) {
        res.send(html);
    });
};
```

此时 `data` 中的每篇文章已经包含：
- **`visibility`**：文章原始公开级别（public/members/paid/tiers）
- **`access`**：当前访客实际是否有权限（true/false）
- **`html`**：已裁剪后的内容（无权限时只有摘要或空）
- **`tiers`**：已按 visibility 过滤后的可用层级

### 2.6 闭环示例：主题模板中的使用

```handlebars
{{!-- 场景：会员专享文章列表页 --}}

{{#foreach posts}}
    <article>
        {{!-- 第一步：基于 visibility 判断文章级别 --}}
        {{#has visibility="members"}}
            <span class="badge">会员专享</span>
        {{/has}}
        
        {{#has visibility="paid"}}
            <span class="badge premium">付费专享</span>
        {{/has}}
        
        <h2>{{title}}</h2>
        
        {{!-- 第二步：基于 access 判断实际能否查看完整内容 --}}
        {{#if access}}
            {{content}}  {{!-- 有权限：显示完整内容 --}}
        {{else}}
            {{excerpt}}  {{!-- 无权限：显示摘要 --}}
            <p>
                {{#unless @member}}
                    <a href="#/portal/signin">登录查看全文</a>
                {{else}}
                    <a href="#/portal/plan">升级会员查看</a>
                {{/unless}}
            </p>
        {{/if}}
    </article>
{{/foreach}}
```

### 2.7 visibility 与 access 的区别

| 属性 | 来源 | 含义 | 变化性 |
|------|------|------|--------|
| `visibility` | 数据库 `posts.visibility` 字段 | 文章**配置**的公开级别 | 固定，由作者设置 |
| `access` | `post-gating.js` 动态计算 | 当前访客**实际**是否有权限 | 动态，取决于访客身份 |

**关系**：
```
access = checkPostAccess(post.visibility, req.member)
```

---

## 3. Handlebars Helper 内容过滤机制

### 3.1 {{#has}} Helper - 条件判断

位置：`ghost/core/core/frontend/helpers/has.js`

`has` helper 是主题中控制内容可见性的核心工具，支持多种属性检查：

#### 3.1.1 支持的属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `tag` | 标签匹配 | `{{#has tag="news"}}` |
| `author` | 作者匹配 | `{{#has author="john"}}` |
| `visibility` | 内容可见性 | `{{#has visibility="paid"}}` |
| `slug` | 别名匹配 | `{{#has slug="welcome"}}` |
| `id` | ID匹配 | `{{#has id="xxx"}}` |
| `number` | 序号匹配（1-based） | `{{#has number="3"}}` 或 `{{#has number="nth:3"}}` |
| `index` | 索引匹配（0-based） | `{{#has index="2"}}` |
| `any` | 任一属性存在 | `{{#has any="twitter,facebook"}}` |
| `all` | 所有属性存在 | `{{#has all="@site.twitter,@site.facebook"}}` |

#### 3.1.2 visibility 属性的核心作用

```handlebars
{{#foreach posts}}
    {{#has visibility="public"}}
        <h2>公开文章: {{title}}</h2>
    {{/has}}
    
    {{#has visibility="members"}}
        <h2>会员专享: {{title}}</h2>
        {{#unless access}}
            <p>请登录查看完整内容</p>
        {{/unless}}
    {{/has}}
    
    {{#has visibility="paid"}}
        <h2>付费专享: {{title}}</h2>
    {{/has}}
{{/foreach}}
```

实现原理（第143-145行）：
```javascript
visibility: function () {
    return attrs.visibility && evaluateStringMatch(attrs.visibility, self.visibility, true) || false;
}
```

注意：`has` helper 只是**条件判断**，实际的内容过滤发生在**数据序列化阶段**（见第1节和第2节）。

### 3.2 {{#foreach}} Helper - 循环过滤

位置：`ghost/core/core/frontend/helpers/foreach.js`

`foreach` helper 在循环遍历数据时会自动根据 `visibility` 属性过滤项目：

#### 3.2.1 自动过滤逻辑

```javascript
// 第27-44行
let visibility = options.hash.visibility;

// 对 Post 和 Newsletter 类型，默认 visibility = 'all'
if (_.isArray(items) && items.length > 0 && checks.isPost(items[0])) {
    visibility = visibility || 'all';
}

// 过滤掉不应该在主题中显示的项目
items = ghostHelperUtils.visibility.filter(items, visibility);
```

#### 3.2.2 visibility 过滤选项

来自 `@tryghost/helpers` 包：
- `public` - 只显示公开项目
- `members` - 只显示会员项目
- `paid` - 只显示付费项目
- `all` - 显示所有（默认）

### 3.3 {{tags}} 和 {{authors}} Helper - 资源过滤

这两个 helper 也支持 visibility 过滤，用于过滤标签和作者：

#### 3.3.1 Tags Helper（第35行）

```handlebars
{{tags visibility="public"}}
```

```javascript
return ghostHelperUtils.visibility.filter(tagsList, options.hash.visibility, processTag);
```

#### 3.3.2 Authors Helper（第42行）

```handlebars
{{authors visibility="public"}}
```

```javascript
return utils.visibility.filter(authorsList, visibility, processAuthor);
```

### 3.4 access 字段 - 细粒度控制

经过 `post-gating.js` 处理后，每个 post 对象会包含 `access` 字段：

```handlebars
{{#foreach posts}}
    <article>
        <h2>{{title}}</h2>
        {{#if access}}
            <div>{{content}}</div>
        {{else}}
            <div>{{excerpt}}</div>
            <p><a href="#/portal/signin">登录阅读全文</a></p>
        {{/if}}
    </article>
{{/foreach}}
```

---

## 4. 服务端渲染期会员凭证校验

### 4.1 整体请求处理流程

位置：`ghost/core/core/frontend/web/site.js`

```
请求进入
    ↓
offersService.middleware (优惠处理)
    ↓
customRedirects.middleware (自定义重定向)
    ↓
静态资源服务 (favicon, public files, images, media)
    ↓
membersService.middleware.loadMemberSession ← 关键：加载会员会话
    ↓
themeMiddleware (主题中间件)
    ↓
frontendCaching (前端缓存)
    ↓
memberPageViewMiddleware (会员页面浏览追踪)
    ↓
siteRoutes (路由处理)
    ↓
主题渲染
```

### 4.2 会员会话加载中间件

位置：`ghost/core/core/server/services/members/middleware.js` 第91-101行

```javascript
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

**重要特性**：即使会话加载失败，也会继续执行（设置 member=null），确保未登录用户也能访问公开内容。

### 4.3 MembersSSR 核心实现

位置：`ghost/core/core/server/services/members/members-ssr.js`

#### 4.3.1 会话 Cookie 配置

```javascript
// members-ssr.js 第26行
const SIX_MONTHS_MS = 1000 * 60 * 60 * 24 * 184;  // 单位：毫秒

// 第72-78行
this.sessionCookieOptions = {
    signed: true,           // 使用签名防篡改
    httpOnly: true,         // 禁止 JS 访问
    sameSite: 'lax',        // 跨站保护
    maxAge: cookieMaxAge,   // 默认 SIX_MONTHS_MS = 1000 * 60 * 60 * 24 * 184
    path: cookiePath
};
```

**重要修正**：`maxAge` 单位是**毫秒**，不是秒。

计算：
- `1000 * 60 * 60 * 24 * 184` = 约 **6.07 个月**（按30天/月计算）
- 实际值：184 天 = 约 6 个月零 4 天

#### 4.3.2 会话验证流程

**`getMemberDataFromSession(req, res)`**

```
1. 读取签名 Cookie: 'ghost-members-ssr'
2. 验证签名有效性
3. 提取 transient_id（临时会话ID）
4. 通过 transient_id 查询会员数据
5. 返回完整 member 对象
```

实现代码：
```javascript
async getMemberDataFromSession(req, res) {
    const transientId = this._getSessionCookies(req, res);
    const member = await this._getMemberIdentityDataFromTransientId(transientId);
    return member;
}
```

#### 4.3.3 Cookie 签名验证详解

**签名机制**：使用 Node.js `cookies` 库 + `Keygrip`

```javascript
// members-ssr.js 第87-90行
this.cookiesOptions = {
    keys: Array.isArray(cookieKeys) ? cookieKeys : [cookieKeys],  // 支持密钥轮换
    secure: cookieSecure
};

// 第101-103行
_getCookies(req, res) {
    return createCookies(req, res, this.cookiesOptions);
}
```

**Cookie 名称**：
```javascript
// service.js 第138行
cookieName: 'ghost-members-ssr',
```

**签名验证流程**：
```javascript
// members-ssr.js 第105-137行
_getSessionCookies(req, res) {
    const cookies = this._getCookies(req, res);
    
    // 第一步：尝试读取签名 cookie
    const value = cookies.get(this.sessionCookieName, {signed: true});
    
    if (!value) {
        // 第二步：清理孤立的未签名 cookie（兼容性处理）
        const unsignedValue = cookies.get(this.sessionCookieName, {signed: false});
        if (unsignedValue) {
            cookies.set(this.sessionCookieName, null, {...this.sessionCookieOptions, signed: false});
        }
        throw new BadRequestError({...});
    }
    return value;
}
```

**签名 Cookie 的工作原理**：

当设置 `signed: true` 时，实际会存储两个 cookie：
1. **`ghost-members-ssr`** = `transient_id` 值
2. **`ghost-members-ssr.sig`** = `HMAC(ghost-members-ssr=<value>, cookieKeys[0])`

读取时 `cookies.get(name, {signed: true})` 会：
1. 读取主 cookie 值
2. 读取 `.sig` 后缀的签名 cookie
3. 使用 `Keygrip` 验证签名是否匹配
4. 支持密钥轮换（依次尝试 keys 数组中的每个密钥）

### 4.4 会员登录流程（Magic Link）

位置：`middleware.js` 第356-456行

#### 4.4.1 Magic Link 验证与会话创建

```
用户点击邮件中的 magic link
    ↓
解析 URL 中的 token 参数
    ↓
exchangeTokenForSession(req, res)
    ↓
验证 JWT token 有效性
    ↓
获取会员数据
    ↓
(可选) 执行 GeoIP 定位
    ↓
设置会话 Cookie: ghost-members-ssr = transient_id
    ↓
(可选) 设置访问缓存 Cookie: ghost-access + ghost-access-hmac
    ↓
重定向到目标页面
```

#### 4.4.2 访问缓存 Cookie（用于 CDN 缓存）

位置：`middleware.js` 第40-79行

```javascript
const setAccessCookies = function setAccessCookies(member, req, res, freeTier) {
    if (!member) {
        // 清除旧 cookie
        return;
    }
    
    const hmacSecret = config.get('cacheMembersContent:hmacSecret');
    // ...
    const activeSubscription = member.subscriptions?.find(sub => sub.status === 'active');
    const cookieTimestamp = Math.floor(Date.now() / 1000);
    const memberTier = activeSubscription && activeSubscription.tier.id || freeTier.id;
    const memberTierAndTimestamp = `${memberTier}:${cookieTimestamp}`;
    const memberTierHmac = crypto.createHmac('sha256', hmacSecretBuffer)
        .update(memberTierAndTimestamp).digest('hex');
    
    // 设置 cookie:
    // ghost-access = tierId:timestamp
    // ghost-access-hmac = HMAC(tierId:timestamp)
};
```

这些 Cookie 用于 CDN 层按会员层级进行缓存区分。

### 4.5 UUID 认证方式（邮件链接）

位置：`middleware.js` 第123-165行

用于邮件中的免登录链接（如取消订阅）：

```javascript
const authMemberByUuid = async function authMemberByUuid(req, res, next) {
    const uuid = req.query.uuid;
    const key = req.query.key;
    
    // 验证 HMAC: key = HMAC(uuid, membersValidationKey)
    const memberHmac = crypto.createHmac('sha256', settingsHelpers.getMembersValidationKey())
        .update(uuid).digest('hex');
    if (memberHmac !== key) {
        throw new errors.UnauthorizedError(...);
    }
    
    // 查询会员
    const member = await membersService.api.memberBREADService.read({uuid});
    Object.assign(req, {member});
    next();
};
```

### 4.6 请求上下文传递

会员数据通过以下方式传递到渲染层：

| 位置 | 用途 | 关键路径 |
|------|------|----------|
| `req.member` | Express 请求层 | 中间件 → 控制器 |
| `res.locals.member` | Handlebars 模板层 | 模板中通过 `@member` 访问 |
| `frame.original.context.member` | API 序列化层 | `post-gating.js` 权限判断 |
| `query.options.context.member` | API 查询层 | `fetch-data.js` 传递 |

**模板中访问会员信息**：
```handlebars
{{#if @member}}
    <p>欢迎回来, {{@member.email}}!</p>
    <p>会员等级: {{@member.status}}</p>
{{else}}
    <p>请 <a href="#/portal/signin">登录</a> 查看完整内容</p>
{{/if}}
```

---

## 5. 主题升级兼容兜底策略

### 5.1 主题验证系统

位置：`ghost/core/core/server/services/themes/validate.js`

Ghost 使用 **gscan** 工具进行主题兼容性验证。

#### 5.1.1 验证流程

```
主题加载
    ↓
gscan.check() 或 gscan.checkZip()
    ↓
检查参数：
  - checkVersion: 对应 Ghost 主版本号 (v5, v6 等)
  - labs: 当前启用的实验特性
  - skipChecks: 是否跳过检查
    ↓
gscan.format() 格式化结果
    ↓
生产环境移除警告
    ↓
缓存验证结果
```

#### 5.1.2 错误等级

| 等级 | 影响 | 处理方式 |
|------|------|----------|
| **fatal errors** | 主题无法正常工作 | 启动时记录错误日志，**但继续运行** |
| **errors** | 部分功能异常 | 记录警告日志，主题仍可激活 |
| **warnings** | 建议改进 | 开发环境显示，生产环境隐藏 |

判断是否可激活：
```javascript
const canActivate = function canActivate(checkedTheme) {
    return !checkedTheme.results.hasFatalErrors;
};
```

### 5.2 启动时的容错机制

位置：`ghost/core/core/server/services/themes/activate.js` 第17-44行

#### 5.2.1 启动时激活流程

```javascript
module.exports.loadAndActivate = async (themeName, options = {}) => {
    try {
        const loadedTheme = await themeLoader.loadOneTheme(themeName);
        const checkedTheme = await validate.check(themeName, loadedTheme, {skipChecks});

        if (!validate.canActivate(checkedTheme)) {
            // CASE 1: 有致命错误 - 记录错误但继续
            logging.error(validate.getThemeValidationError('activeThemeHasFatalErrors', ...));
        } else if (checkedTheme.results.error.length) {
            // CASE 2: 有非致命错误 - 记录警告
            logging.warn(validate.getThemeValidationError('activeThemeHasErrors', ...));
        }

        // 无论如何都尝试激活主题
        await activator.activateFromBoot(themeName, loadedTheme, checkedTheme);
    } catch (err) {
        if (err instanceof errors.NotFoundError) {
            // CASE 3: 主题文件缺失 - 记录错误但不退出
            err.message = tpl(messages.activeThemeIsMissing, ...);
        }
        // CASE 4: 其他未知错误 - 记录错误但不退出
        logging.error(err);
        // 不抛出异常，确保 admin 面板仍可访问
    }
};
```

**关键设计决策**：即使主题有严重问题，也不会导致整个 Ghost 实例崩溃，管理员仍可通过后台更换主题。

### 5.3 API 激活时的严格检查

位置：`activate.js` 第46-62行

通过后台 API 激活主题时检查更严格：

```javascript
module.exports.activate = async (themeName) => {
    const loadedTheme = list.get(themeName);
    
    // 使用 checkSafe - 有 fatal error 会抛出异常
    const checkedTheme = await validate.checkSafe(themeName, loadedTheme);
    
    await activator.activateFromAPI(themeName, loadedTheme, checkedTheme);
    return validate.getErrorsFromCheckedTheme(checkedTheme);
};
```

`checkSafe` 的实现（validate.js 第124-142行）：
```javascript
const checkSafe = async function checkSafe(themeName, theme, isZip) {
    const checkedTheme = await check(themeName, theme, {isZip});

    if (canActivate(checkedTheme)) {
        return checkedTheme;
    }

    // 不可激活则抛出异常
    if (isZip) {
        fs.remove(checkedTheme.path);  // 清理临时文件
    }

    throw getThemeValidationError('themeHasErrors', themeName, checkedTheme);
};
```

### 5.4 版本兼容性检查

#### 5.4.1 主题版本检查

位置：`validate.js` 第50-52行

```javascript
const ghostVersion = require('@tryghost/version');
const checkedVersion = `v${ghostVersion.safe.split('.')[0]}`;
// 例如: Ghost 5.80.2 → checkedVersion = 'v5'
```

主题的 `package.json` 中可以指定兼容的 Ghost 版本：
```json
{
  "name": "my-theme",
  "version": "1.0.0",
  "engines": {
    "ghost": ">=5.0.0"
  },
  "config": {
    "posts_per_page": 10
  }
}
```

#### 5.4.2 API 版本匹配

位置：`ghost/core/core/server/web/api/middleware/version-match.js`

```javascript
module.exports = function checkVersionMatch(req, res, next) {
    const clientVersion = req.get('X-Ghost-Version');
    const serverVersion = res.locals.version.match(/^(\d+\.)?(\d+\.)?(\d+)/)[0];

    if (clientVersion) {
        const constraint = '^' + clientVersion + '.0';
        
        // 规则：
        // - 客户端 minor 版本 < 服务器: 允许（向前兼容）
        // - 客户端 minor 版本 > 服务器: 拒绝
        // - 主版本不同: 总是拒绝
        if (!semver.satisfies(serverVersion, constraint)) {
            return next(new errors.VersionMismatchError(...));
        }
    }
    next();
};
```

### 5.5 主题激活桥接

位置：`ghost/core/core/server/services/themes/activation-bridge.js`

三种激活方式：

| 方式 | 触发时机 | 验证严格度 |
|------|----------|------------|
| `activateFromBoot` | 服务启动时 | 宽松（记录错误但继续） |
| `activateFromAPI` | 后台 API 激活 | 严格（checkSafe） |
| `activateFromAPIOverride` | API 覆盖激活 | 严格 |

激活后执行：
1. `customThemeSettings.api.activateTheme()` - 激活自定义主题设置
2. `bridge.activateTheme()` - 通知前端引擎切换主题

### 5.6 兼容性兜底策略总结

1. **启动容错**：主题问题不会导致服务崩溃
2. **分级错误**：fatal/error/warning 三级错误处理
3. **双轨验证**：启动时宽松检查，API 激活时严格检查
4. **版本适配**：按 Ghost 主版本号验证主题兼容性
5. **管理员可恢复**：即使当前主题故障，后台仍可访问

---

## 附录

### A. 关键文件清单

| 文件路径 | 功能说明 |
|----------|----------|
| `core/frontend/apps/private-blogging/lib/middleware.js` | Private Site 全站访问控制核心逻辑 |
| `core/frontend/apps/private-blogging/index.js` | Private Site 中间件挂载与路由注册 |
| `core/server/services/members/content-gating.js` | Members 内容权限裁剪核心逻辑 |
| `core/server/api/endpoints/utils/serializers/output/utils/post-gating.js` | API 输出时的内容裁剪 |
| `core/server/api/endpoints/utils/serializers/output/mappers/posts.js` | 帖子输出序列化（调用 gating） |
| `core/frontend/services/data/fetch-data.js` | 前端数据获取（传递 member 上下文） |
| `core/server/services/members/middleware.js` | 会员中间件（会话加载等） |
| `core/server/services/members/members-ssr.js` | 服务端渲染会员会话管理 |
| `core/frontend/helpers/has.js` | Handlebars {{#has}} 条件 helper |
| `core/frontend/helpers/foreach.js` | Handlebars {{#foreach}} 循环 helper |
| `core/server/services/themes/validate.js` | 主题兼容性验证 |
| `core/server/services/themes/activate.js` | 主题激活流程 |
| `core/frontend/web/site.js` | 前端站点请求处理管线 |

### B. 会员身份状态流转

```
未登录 (member = null)
    ↓ 点击 magic link
验证 token → 获取 transient_id → 设置 cookie
    ↓
已登录免费会员 (member.status = 'free')
    ↓ 升级订阅
已登录付费会员 (member.status = 'paid', member.products = [...])
    ↓ 登出 / cookie 过期
回到未登录状态
```

### C. 内容可见性矩阵

| 文章 visibility | 未登录 | 免费会员 | 付费会员 | 特定 Tier 会员 |
|-----------------|--------|----------|----------|---------------|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `members` | ❌ | ✅ | ✅ | ✅ |
| `paid` | ❌ | ❌ | ✅ | 取决于 tier |
| `tiers: [tierA]` | ❌ | ❌ | 取决于 tier | ✅ (有 tierA) |

### D. 权限链路时序图

```
HTTP 请求
   │
   ├─► site.js (Express app)
   │      │
   │      ├─► middleware.loadMemberSession()
   │      │      │
   │      │      ├─► members-ssr._getSessionCookies()
   │      │      │      └─► cookies.get('ghost-members-ssr', {signed: true})
   │      │      │            └─► Keygrip 验证签名
   │      │      │
   │      │      └─► members-ssr._getMemberIdentityDataFromTransientId()
   │      │             └─► Members API 查询
   │      │
   │      └─► res.locals.member = member / null
   │
   ├─► routing controller (entry.js / collection.js)
   │      │
   │      └─► fetch-data.fetchData()
   │             │
   │             └─► processQuery()
   │                    ├─► query.options.context = {member: locals.member}
   │                    └─► api.postsPublic.browse(query.options)
   │
   ├─► API 层 (posts-public.js)
   │      │
   │      └─► serializers.output.mappers.posts()
   │             │
   │             └─► gating.forPost(jsonModel, frame)
   │                    ├─► checkPostAccess(attrs, frame.original.context.member)
   │                    ├─► 裁剪 html/excerpt（如无权限）
   │                    ├─► stripGatedBlocks(attrs.html, member)
   │                    └─► attrs.access = memberHasAccess
   │
   └─► rendering.renderer()
          │
          └─► res.render(template, data)
                 │
                 └─► Handlebars 模板执行
                        ├─► {{#has visibility="..."}}
                        ├─► {{#foreach posts visibility="..."}}
                        └─► {{#if access}}
```
