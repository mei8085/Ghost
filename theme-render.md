# Ghost 主题渲染权限与兼容性分析报告

## 目录
1. [数据查询权限裁剪逻辑](#1-数据查询权限裁剪逻辑)
2. [完整权限链路：访客身份 → visibility → access → Handlebars 渲染](#2-完整权限链路访客身份--visibility--access--handlebars-渲染)
3. [Handlebars Helper 内容过滤机制](#3-handlebars-helper-内容过滤机制)
4. [服务端渲染期会员凭证校验](#4-服务端渲染期会员凭证校验)
5. [主题升级兼容兜底策略](#5-主题升级兼容兜底策略)

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
| `core/server/services/members/content-gating.js` | 内容权限裁剪核心逻辑 |
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
