# Ghost 主题渲染权限与兼容性分析报告

## 目录
1. [数据查询权限裁剪逻辑](#1-数据查询权限裁剪逻辑)
2. [Handlebars Helper 内容过滤机制](#2-handlebars-helper-内容过滤机制)
3. [服务端渲染期会员凭证校验](#3-服务端渲染期会员凭证校验)
4. [主题升级兼容兜底策略](#4-主题升级兼容兜底策略)

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
2. 调用 checkPostAccess 检查权限
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

## 2. Handlebars Helper 内容过滤机制

### 2.1 {{#has}} Helper - 条件判断

位置：`ghost/core/core/frontend/helpers/has.js`

`has` helper 是主题中控制内容可见性的核心工具，支持多种属性检查：

#### 2.1.1 支持的属性

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

#### 2.1.2 visibility 属性的核心作用

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

注意：`has` helper 只是**条件判断**，实际的内容过滤发生在**数据查询阶段**（见第1节）。

### 2.2 {{#foreach}} Helper - 循环过滤

位置：`ghost/core/core/frontend/helpers/foreach.js`

`foreach` helper 在循环遍历数据时会自动根据 `visibility` 属性过滤项目：

#### 2.2.1 自动过滤逻辑

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

#### 2.2.2 visibility 过滤选项

来自 `@tryghost/helpers` 包：
- `public` - 只显示公开项目
- `members` - 只显示会员项目
- `paid` - 只显示付费项目
- `all` - 显示所有（默认）

### 2.3 {{tags}} 和 {{authors}} Helper - 资源过滤

这两个 helper 也支持 visibility 过滤，用于过滤标签和作者：

#### 2.3.1 Tags Helper（第35行）

```handlebars
{{tags visibility="public"}}
```

```javascript
return ghostHelperUtils.visibility.filter(tagsList, options.hash.visibility, processTag);
```

#### 2.3.2 Authors Helper（第42行）

```handlebars
{{authors visibility="public"}}
```

```javascript
return utils.visibility.filter(authorsList, visibility, processAuthor);
```

### 2.4 access 字段 - 细粒度控制

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

## 3. 服务端渲染期会员凭证校验

### 3.1 整体请求处理流程

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

### 3.2 会员会话加载中间件

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

### 3.3 MembersSSR 核心实现

位置：`ghost/core/core/server/services/members/members-ssr.js`

#### 3.3.1 会话 Cookie 配置

```javascript
this.sessionCookieOptions = {
    signed: true,           // 使用签名防篡改
    httpOnly: true,         // 禁止 JS 访问
    sameSite: 'lax',        // 跨站保护
    maxAge: 60 * 60 * 24 * 184,  // 6个月
    path: cookiePath
};
```

#### 3.3.2 会话验证流程

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

#### 3.3.3 Cookie 签名验证

```javascript
_getSessionCookies(req, res) {
    const cookies = this._getCookies(req, res);
    const value = cookies.get(this.sessionCookieName, {signed: true});
    if (!value) {
        // 清理孤立的未签名 cookie
        const unsignedValue = cookies.get(this.sessionCookieName, {signed: false});
        if (unsignedValue) {
            cookies.set(this.sessionCookieName, null, {...this.sessionCookieOptions, signed: false});
        }
        throw new BadRequestError({...});
    }
    return value;
}
```

### 3.4 会员登录流程（Magic Link）

位置：`middleware.js` 第356-456行

#### 3.4.1 Magic Link 验证与会话创建

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

#### 3.4.2 访问缓存 Cookie（用于 CDN 缓存）

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

### 3.5 UUID 认证方式（邮件链接）

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

### 3.6 请求上下文传递

会员数据通过以下方式传递到渲染层：

1. **`req.member`** - Express 请求对象
2. **`res.locals.member`** - Handlebars 模板上下文
3. **`frame.original.context.member`** - API 序列化器上下文（用于 post-gating）

---

## 4. 主题升级兼容兜底策略

### 4.1 主题验证系统

位置：`ghost/core/core/server/services/themes/validate.js`

Ghost 使用 **gscan** 工具进行主题兼容性验证。

#### 4.1.1 验证流程

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

#### 4.1.2 错误等级

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

### 4.2 启动时的容错机制

位置：`ghost/core/core/server/services/themes/activate.js` 第17-44行

#### 4.2.1 启动时激活流程

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

### 4.3 API 激活时的严格检查

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

### 4.4 版本兼容性检查

#### 4.4.1 主题版本检查

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

#### 4.4.2 API 版本匹配

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

### 4.5 主题激活桥接

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

### 4.6 兼容性兜底策略总结

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

