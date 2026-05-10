# Ghost Portal 脚本嵌入与设置同步机制分析

## 1. 概述

Portal 是 Ghost 的成员系统前端应用，提供登录、注册、订阅管理等功能。它通过 UMD  bundle 形式动态注入到站点页面中，与后台设置保持实时同步。

## 2. 脚本注入机制

### 2.1 注入入口：{{ghost_head}}

Portal 脚本通过 Ghost 主题中的 `{{ghost_head}}` 辅助函数注入到页面头部。

**核心文件**：`ghost/core/core/frontend/helpers/ghost_head.js`

```javascript
// ghost_head.js:51-81
function getMembersHelper(data, frontendKey, excludeList) {
    // 检查是否需要加载 Portal
    if (!settingsCache.get('members_enabled') && 
        !settingsCache.get('donations_enabled') && 
        !settingsCache.get('recommendations_enabled')) {
        return '';
    }
    
    if (!excludeList.has('portal')) {
        const {scriptUrl} = getFrontendAppConfig('portal');
        
        // 构建数据属性
        const attributes = {
            i18n: true,
            ghost: urlUtils.getSiteUrl(),
            key: frontendKey,
            api: urlUtils.urlFor('api', {type: 'content'}, true),
            locale: settingsCache.get('locale') || 'en'
        };
        
        if (colorString) {
            attributes['accent-color'] = colorString;
        }
        
        const dataAttributes = getDataAttributes(attributes);
        membersHelper += `<script defer src="${scriptUrl}" ${dataAttributes} crossorigin="anonymous"></script>`;
    }
    // ...
}
```

### 2.2 脚本 URL 配置

Portal 脚本默认从 JSDelivr CDN 加载，支持版本控制。

**核心文件**：`ghost/core/core/shared/config/defaults.json:242-245`

```json
{
    "portal": {
        "url": "https://cdn.jsdelivr.net/ghost/portal@~{version}/umd/portal.min.js",
        "version": "2.68"
    }
}
```

**配置获取**：`ghost/core/core/frontend/utils/frontend-apps.js:3-18`

```javascript
function getFrontendAppConfig(app) {
    const appVersion = config.get(`${app}:version`);
    let scriptUrl = config.get(`${app}:url`);
    if (typeof scriptUrl === 'string' && scriptUrl.includes('{version}')) {
        scriptUrl = scriptUrl.replace('{version}', appVersion);
    }
    return {scriptUrl, stylesUrl, appVersion};
}
```

### 2.3 数据属性传递

Portal 脚本通过 `data-*` 属性接收配置：

| 属性名 | 用途 | 来源 |
|--------|------|------|
| `data-ghost` | Ghost 站点 URL | `urlUtils.getSiteUrl()` |
| `data-key` | Content API Key | `getFrontendKey()` |
| `data-api` | Content API 地址 | `urlUtils.urlFor('api', {type: 'content'}, true)` |
| `data-locale` | 语言设置 | `settingsCache.get('locale')` |
| `data-accent-color` | 强调色（可选） | `data.site.accent_color` |
| `data-i18n` | 是否启用国际化 | 固定为 `true` |

## 3. Portal 初始化流程

### 3.1 入口脚本

**核心文件**：`apps/portal/src/index.js`

```javascript
// 从 script 标签读取配置
function getSiteData() {
    const scriptTag = document.querySelector('script[data-ghost]');
    if (scriptTag) {
        return {
            siteUrl: scriptTag.dataset.ghost,
            apiKey: scriptTag.dataset.key,
            apiUrl: scriptTag.dataset.api,
            siteI18nEnabled: scriptTag.dataset.i18n === 'true',
            locale: scriptTag.dataset.locale
        };
    }
    return {};
}

// 创建根元素并初始化
function init() {
    const {siteUrl, apiKey, apiUrl, siteI18nEnabled, locale} = getSiteData();
    const siteUrlFinal = siteUrl || window.location.origin;
    
    addRootDiv(); // 创建 <div id="ghost-portal-root">
    handleTokenUrl(); // 清理 URL 中的 token 参数
    
    ReactDOM.render(
        <App siteUrl={siteUrlFinal} 
             customSiteUrl={siteUrl} 
             apiKey={apiKey} 
             apiUrl={apiUrl} 
             siteI18nEnabled={siteI18nEnabled} 
             locale={locale} />,
        document.getElementById('ghost-portal-root')
    );
}
```

### 3.2 数据初始化

**核心文件**：`apps/portal/src/app.js:264-323`

```javascript
async initSetup() {
    // 从多个源获取数据
    const {site, member, offers, page, showPopup, ...rest} = await this.fetchData();
    
    // 设置国际化
    const i18nLanguage = this.props.siteI18nEnabled 
        ? (this.props.locale || site.locale || 'en') 
        : 'en';
    i18n.changeLanguage(i18nLanguage);
    
    this.setState({
        site, member, offers, page, showPopup,
        action: 'init:success',
        initStatus: 'success'
    });
    
    // 监听 hash 变化（用于预览模式）
    window.addEventListener('hashchange', this.hashHandler, false);
}
```

### 3.3 API 数据获取

**核心文件**：`apps/portal/src/utils/api.js:856-895`

```javascript
api.init = async () => {
    // 并行获取成员会话数据
    let [member] = await Promise.all([
        api.member.sessionData()
    ]);
    
    let site = {};
    let newsletters = [];
    let tiers = [];
    let settings = {};
    
    try {
        // 并行获取站点设置、会员等级、新闻通讯
        [{settings}, {tiers}, {newsletters}] = await Promise.all([
            api.site.settings(),      // Content API: /settings/
            api.site.tiers(),         // Content API: /tiers/
            api.site.newsletters()    // Content API: /newsletters/
        ]);
        
        site = {
            ...settings,
            newsletters,
            tiers: transformApiTiersData({tiers})
        };
    } catch (e) {
        // 忽略错误
    }
    
    // 付费会员获取优惠信息
    if (member && member.paid) {
        try {
            const offersData = await api.member.offers();
            offers = offersData.offers || [];
        } catch (e) {
            console.warn('[Portal] Failed to load member offers:', e);
        }
    }
    
    site = transformApiSiteData({site});
    return {site, member, offers};
};
```

## 4. 跨域加载与 CSP 限制

### 4.1 Ghost 的 CSP 策略现状

Ghost 核心代码库**未主动设置 Content-Security-Policy 响应头**。

**证据**：
- 未找到 `helmet`、`content-security-policy` 或 CSP 配置相关中间件代码
- 仅在 Mailgun 邮件分析 `tracking-open.png` 打开/链接点击追踪和测试文件中有 CSP 引用（邮件内容安全策略）
- 服务器端依赖 Helmet，但 Helmet v4+ 默认**禁用 CSP** 以防止意外破坏站点

这意味着 Portal 运行时依赖浏览器的**默认同源策略**，不依赖 CSP 强制执行。

### 4.2 关键外部资源与受限风险分析

尽管 Ghost 不强制 CSP，但部署反向代理或 CDN 可能添加 CSP 头。此时以下外部资源需要被允许：

#### 4.2.1 脚本资源 (`script-src`)

| 来源 | URL 示例 | 用途 | 缺失影响 |
|------|-----------|------|-----------|
| **Portal UMD** | `https://cdn.jsdelivr.net/ghost/portal@*/umd/portal.min.js | 核心功能脚本 | Portal 完全无法加载，成员功能全部失效 |
| **Stripe SDK** | `https://js.stripe.com/v3/` | 付费会员支付 | 付费订阅/结账完全失效 |
| **FirstPromoter** | `https://cdn.firstpromoter.com/fprom.js` | 联盟营销追踪 | 联盟追踪失效，但不影响核心功能 |

#### 4.2.2 接口资源 (`connect-src` / `fetch-src`)

| 接口类型 | URL 模式 | 用途 | 缺失影响 |
|-----------|----------|------|-----------|
| **Content API** | `{siteUrl}/ghost/api/content/*` | 获取站点设置、价格、新闻通讯 | Portal UI 无法渲染，按钮样式、价格显示失效 |
| **Members API** | `{siteUrl}/members/api/*` | 成员登录、会话管理、编辑资料 | 登录/注册/订阅全部失效 |
| **Admin API** | `{siteUrl}/ghost/api/admin/*` | 评论功能（如有） | 评论无法加载 |
| **Stripe API** | `https://api.stripe.com/*` | 支付处理 | 付费订阅/结账失效 |
| **Sentry** | `{dsn}.ingest.sentry.io` | 错误监控（可选，通过 `portal_sentry.dsn` 配置） | 错误上报失效，不影响功能 |

#### 4.2.3 嵌入式框架 (`frame-src`)

| 来源 | URL | 用途 | 缺失影响 |
|------|-----|------|-----------|
| **Stripe Checkout** | `https://checkout.stripe.com` | Stripe 结账弹窗 | 付费结账无法弹窗 |

#### 4.2.4 样式资源 (`style-src`)

Portal 使用内联样式（`srcDoc` iframe + React 内联样式），因此需要：
- `'unsafe-inline'` 用于 `style-src`
- `'unsafe-inline'` 或 `data:` 用于 SVG 数据 URL（背景图等）

#### 4.2.5 图片资源 (`img-src`)

| 来源 | 用途 |
|------|------|
| `{siteUrl}/content/images/*` | 站点图标、文章图片 |
| `https://*.stripe.com` | Stripe 支付相关图片 |
| `data:` | 内联 SVG 图标 |

### 4.3 反向代理/CDN 强制 CSP 时的推荐策略

如果基础设施强制设置 CSP，需要在 CSP 策略中添加以下白名单：

```http
Content-Security-Policy:
  default-src 'self';
  
  # 脚本源：核心功能
  script-src 'self' 
    https://cdn.jsdelivr.net 
    https://js.stripe.com 
    https://cdn.firstpromoter.com
    'unsafe-inline';  # 用于 Stripe 内联脚本
  
  # 连接源：所有 API
  connect-src 'self' 
    https://api.stripe.com
    https://*.ingest.sentry.io;  # 可选，用于 Sentry
  
  # 框架源：Stripe Checkout
  frame-src 'self' 
    https://checkout.stripe.com 
    https://js.stripe.com;
  
  # 样式源：Portal 使用内联样式
  style-src 'self' 'unsafe-inline';
  
  # 图片源
  img-src 'self' data: https://*.stripe.com;
```

### 4.4 脚本加载方式

Portal 使用 `crossorigin="anonymous"` 属性加载脚本：

```html
<script defer src="..." crossorigin="anonymous"></script>
```

这允许：
- 从 CDN 加载脚本时不发送用户凭证（cookies）
- 错误信息可以被捕获（Sentry 等）

### 4.5 CORS 配置

Ghost 为 Members API 配置了专门的 CORS 中间件。

**核心文件**：`ghost/core/core/server/web/members/middleware/cors.js`

```javascript
const ENABLE_CORS = {origin: true, maxAge, credentials: true};
const WILDCARD_CORS = {origin: '*', maxAge};

function isAllowedOrigin(origin) {
    if (!origin || origin === 'null') {
        return false;
    }
    
    const originUrl = new URL(origin);
    
    // 允许 localhost 请求
    if (originUrl.hostname === 'localhost' || originUrl.hostname === '127.0.0.1') {
        return true;
    }
    
    // 允许站点域名
    const siteUrl = new URL(config.get('url'));
    if (originUrl.host === siteUrl.host) {
        return true;
    }
    
    // 允许 admin 域名
    if (config.get('admin:url')) {
        const adminUrl = new URL(config.get('admin:url'));
        if (originUrl.host === adminUrl.host) {
            return true;
        }
    }
    
    return false;
}

function corsOptionsDelegate(req, callback) {
    const origin = req.get('origin');
    if (isAllowedOrigin(origin)) {
        return callback(null, ENABLE_CORS);  // 带 credentials
    } else {
        return callback(null, WILDCARD_CORS); // 不带 credentials
    }
}
```

**CORS 预检请求限制**：

| 请求源 | Origin 匹配 | CORS 响应 | 凭证传递 |
|--------|------------|------------|----------|
| 站点域名（前端页面） | `isAllowedOrigin()` 返回 `true` | `Access-Control-Allow-Origin: {origin}` + `Access-Control-Allow-Credentials: true` | ✓ 允许 `credentials: 'same-origin'` |
| Admin 后台域名 | `admin:url` 匹配 | 同上 | ✓ 允许 |
| localhost | 开发环境 | 同上 | ✓ 允许 |
| 其他任意来源 | 不匹配 | `Access-Control-Allow-Origin: *` | ✗ 不发送 Cookie，仅公共数据可访问 |

### 4.6 凭证传递机制

Portal API 请求使用 `credentials: 'same-origin'`：

**核心文件**：`apps/portal/src/utils/api.js:24-32`

```javascript
function makeRequest({url, method = 'GET', headers = {}, credentials = undefined, body = undefined}) {
    const options = {method, headers, credentials, body};
    return fetch(url, options);
}

// 成员会话数据请求
api.member.sessionData() {
    const url = endpointFor({type: 'members', resource: 'member'});
    return makeRequest({
        url,
        credentials: 'same-origin'  // 传递 cookies
    });
}
```

**凭证传递边界**：
- `credentials: 'same-origin'` 仅向**同源**请求发送 Cookie
- 跨域到 CDN（jsdelivr、stripe）**不发送** Cookie
- Members API 必须与前端**同源**或匹配 CORS 白名单

### 4.7 Stripe 第三方脚本注入条件

付费会员功能需要加载 Stripe 脚本：

**核心文件**：`ghost/core/core/frontend/helpers/ghost_head.js:77-79`

```javascript
if (settingsCache.get('paid_members_enabled')) {
    membersHelper += `<script async src="https://js.stripe.com/v3/"></script>`;
}
```

**注入条件**：`settingsCache.get('paid_members_enabled')` 为 `true`

## 5. iframe 与父页通信

### 5.1 双重 iframe 架构

Portal 使用**两层 iframe**：

1. **外层 iframe（后台预览时）**：Admin 后台通过 `<iframe>` 嵌入站点页面
2. **内层 iframe（弹窗时）**：Portal 弹窗使用 `srcDoc` 创建同源 iframe

#### 5.1.1 内层 iframe：Portal 弹窗 iframe

Portal 弹窗使用 `srcDoc` 创建**同源** iframe，避免跨域问题：

**核心文件**：`apps/portal/src/components/frame.js:27-43`

```javascript
render() {
    const {children, head, title = '', ...rest} = this.props;
    return (
        <iframe
            srcDoc={`<!DOCTYPE html>`}  // 关键：srcDoc 创建同源文档
            ref={node => (this.node = node)}
            title={title}
            style={style} 
            frameBorder="0"
            {...rest}
        >
            {this.iframeHead && createPortal(head, this.iframeHead)}
            {this.iframeRoot && createPortal(children, this.iframeRoot)}
        </iframe>
    );
}
```

**核心文件**：`apps/portal/src/components/popup-modal.js:307-338`

```javascript
renderFrameContainer() {
    return (
        <div style={Styles.modalContainer}>
            <Frame 
                style={frameStyle} 
                title="portal-popup" 
                head={this.renderFrameStyles()}
                dataTestId='portal-popup-frame'
                dataDir={this.context.dir}
            >
                <div className={className} onClick={...}></div>
                <PopupContent isMobile={isMobile} />
            </Frame>
        </div>
    );
}
```

**同源优势**：`srcDoc` 创建的 iframe 与父页**共享 origin**，可直接访问 `window.parent`，无需 CORS 限制。

#### 5.1.2 外层 iframe：后台预览 iframe

Admin 后台预览时，站点页面被嵌入在 Admin 后台的 iframe 中：

**核心文件**：`apps/admin-x-settings/src/components/settings/membership/portal/portal-frame.tsx:66-77`

```tsx
<iframe
    ref={iframeRef}
    src={href}  // 加载完整站点 URL
    title="Portal Preview"
    width="100%"
    height="100%"
/>
```

此时形成嵌套结构：

```
Admin 页面 (https://admin.site.com)
    └── iframe (外层，跨域)
        └── 站点页面 (https://www.site.com)
            └── Portal 挂载
                └── iframe (内层，srcDoc，同源)
```

### 5.2 postMessage 消息事件类型与职责

Portal 使用 `window.postMessage` 进行**跨窗口通知**，但**不传递登录状态或敏感数据**。

#### 5.2.1 事件列表

| 事件类型 | 发送方 | 接收方 | 触发时机 | 载荷 | 实际用途 |
|---------|--------|--------|---------|------|----------|
| `portal-ready` | Portal（内层） | 外层 iframe 父页（Admin） | Portal 初始化完成 | `{}` | 通知 Admin 外层 iframe 宿主页面**Portal 已加载完成** |
| `portal-preview-ready` | Portal（内层） | Admin 外层 iframe 父页 | 预览模式初始化完成 | `{}` | 通知 Admin 外层 iframe**预览 iframe 宿主页面**Portal 预览已渲染，可显示 |
| `portal-preview-updated` | Portal popup（内层） | Admin 外层 iframe 父页 | 预览模式弹窗高度变化 | `{height: number}` | 让 Admin 外层 iframe 宿主**动态调整 iframe 高度** |

#### 5.2.2 事件代码

**核心文件**：`apps/portal/src/app.js:149-156`

```javascript
sendPortalReadyEvent() {
    if (window.self !== window.parent) {
        window.parent.postMessage({
            type: 'portal-ready',
            payload: {}
        }, '*');  // 目标 origin: *（通知无敏感数据）
    }
}
```

**核心文件**：`apps/portal/src/components/popup-modal.js:79-92`

```javascript
sendContainerHeightChangeEvent() {
    if (this.node && hasMode(['preview'])) {
        if (this.node?.clientHeight !== this.lastContainerHeight) {
            this.lastContainerHeight = this.node?.clientHeight;
            window.parent.postMessage({
                type: 'portal-preview-updated',
                payload: {
                    height: this.lastContainerHeight
                }
            }, '*');
        }
    }
}
```

**核心文件**：`apps/portal/src/components/popup-modal.js:139-146`

```javascript
sendPortalPreviewReadyEvent() {
    if (window.self !== window.parent) {
        window.parent.postMessage({
            type: 'portal-preview-ready',
            payload: {}
        }, '*');
    }
}
```

#### 5.2.3 Admin 侧消息监听

**核心文件**：`apps/admin-x-settings/src/components/settings/membership/portal/portal-frame.tsx:27-47`

```tsx
useEffect(() => {
    const messageListener = (event: MessageEvent) => {
        if (!href) {
            return;
        }
        const originURL = new URL(event.origin);

        if (originURL.origin === new URL(href).origin) {
            // 仅信任来自站点 URL 的消息
            if (event?.data?.type === 'portal-preview-ready') {
                makeVisible();  // 隐藏加载动画，显示 iframe
            }
        }
    };

    window.addEventListener('message', messageListener, true);
    return () => {
        window.removeEventListener('message', messageListener, true);
    };
}, [href, ...]);
```

**安全性**：Admin 监听时会**验证消息的 `event.origin` 必须匹配 iframe `href.origin`，防止 XSS 消息伪造。

### 5.3 登录与订阅状态边界：postMessage vs Cookie vs Members API

#### 5.3.1 职责边界表

| 机制 | 数据类型 | 安全性 | 传递方向 | 典型场景 | 能否传递登录状态 |
|------|---------|--------|---------|---------|----------------|
| **postMessage** | UI 通知、高度、就绪信号 | ❌ 无加密，目标 origin 校验 | 子→父（单向） | iframe 高度调整、加载状态通知 | ❌ **不能** |
| **Cookie** | 会话标识 transient_id | ✓ HttpOnly + Signed + SameSite | 浏览器自动附加到请求 | 所有 `/members/api/*` 请求 | ✓ **唯一真实状态存储 |
| **Members API** | 成员数据（JSON） | ✓ 基于 Cookie 会话认证 | Portal ↔ 服务器 | 获取成员资料、编辑信息、登出 | ✓ 通过 Cookie 认证后返回 |

#### 5.3.2 postMessage 不能传递登录状态的原因

1. **postMessage 消息体可见**：消息内容对页面上任何 `window.addEventListener('message')` 监听器都可见
2. **postMessage 无加密**：消息明文发送，不适合传递敏感数据
3. **XSS 风险**：如果父页存在 XSS，可通过 message 事件窃取数据

#### 5.3.3 Cookie 机制详解

**核心文件**：`ghost/core/core/server/services/members/members-ssr.js:67-90`

```javascript
this.sessionCookieName = 'members-ssr';

this.sessionCookieOptions = {
    signed: true,      // 使用 cookieKeys 签名
    httpOnly: true,   // ❌ JS 不可访问，防止 XSS 窃取
    sameSite: 'lax', // 限制第三方嵌入
    maxAge: 1000 * 60 * 60 * 24 * 184,  // 6 个月
    path: cookiePath
};

this.cookiesOptions = {
    keys: Array.isArray(cookieKeys) ? cookieKeys : [cookieKeys],
    secure: cookieSecure  // HTTPS 生产环境强制
};
```

**Cookie 存储内容**：

| Cookie 名称 | 值 | 说明 |
|--------------|---|------|
| `members-ssr` | `transient_id` (JWT-like signed value | 会话标识，用于查询成员身份 |
| `ghost-access` | `{tierId}:{timestamp}` | 可选，用于内容缓存分层（启用 `cacheMembersContent` |
| `ghost-access-hmac` | HMAC of `ghost-access` | 验证 `ghost-access` 完整性 |

**登录流程**（Cookie 机制）：

```
1. 用户点击 magic link
        │
        ▼
2. GET /members/ssr?token=xxx
        │
        ▼
3. Server: exchangeTokenForSession()
   ├── 验证 token
   ├── 查找 member
   └── Set-Cookie: members-ssr={transient_id}
        │ (HttpOnly, Signed, SameSite=lax)
        ▼
4. 后续请求（自动携带 Cookie）
   GET /members/api/member/
   (credentials: 'same-origin')
        │
        ▼
5. Server: getMemberDataFromSession()
   ├── 读取 members-ssr Cookie
   ├── 验证签名
   └── 通过 transient_id 查询 member
   └── 返回成员数据 JSON
```

**核心文件**：`ghost/core/core/server/services/members/members-ssr.js:305-309`

```javascript
async getMemberDataFromSession(req, res) {
    const transientId = this._getSessionCookies(req, res);
    const member = await this._getMemberIdentityDataFromTransientId(transientId);
    return member;
}
```

#### 5.3.4 Members API 职责

Members API 是 Portal 获取和修改成员状态的**唯一可信通道**：

| API 端点 | 方法 | 认证方式 | 功能 | 源码依据 |
|---------|------|---------|------|---------|
| `/members/api/member/` | GET | Cookie (`members-ssr`) | 获取当前登录成员 | `ghost/core/core/server/web/members/app.js:59` |
| `/members/api/member/` | PUT | Cookie | 更新成员资料 | `ghost/core/core/server/web/members/app.js:62` |
| `/members/api/session/` | DELETE | Cookie | 清除会话 Cookie | `ghost/core/core/server/web/members/app.js:75` |
| `/members/api/session/` | GET | Cookie | 获取身份 JWT | `ghost/core/core/server/web/members/app.js:74` |
| `/members/api/send-magic-link/` | POST | Integrity Token + CSRF | 发送登录 magic link | `ghost/core/core/server/web/members/app.js:80-92` |
| `/members/api/integrity-token/` | GET | 无 | 获取请求完整性 Token | `ghost/core/core/server/web/members/app.js:78` |

**核心文件**：`ghost/core/core/server/services/members/middleware.js:243-255`

```javascript
const getMemberData = async function getMemberData(req, res) {
    try {
        const member = await membersService.ssr.getMemberDataFromSession(req, res);
        if (member) {
            res.json(formattedMemberResponse(member));
        } else {
            res.json(null);  // 未登录
        }
    } catch (err) {
        res.writeHead(204);
        res.end();
    }
};
```

#### 5.3.5 登出流程

**核心文件**：`ghost/core/core/server/services/members/middleware.js:227-241`

```javascript
const deleteSession = async function deleteSession(req, res) {
    try {
        await membersService.ssr.deleteSession(req, res);
        // deleteSession 内部：
        // ├── if(req.body.all) { cycleTransientId(memberId) }
        // └── _removeSessionCookie() → Set-Cookie: members-ssr=null; Max-Age=0
        res.writeHead(204);
        res.end();
    } catch (err) {
        // ...
    }
};
```

登出后：
1. 清除 `members-ssr` Cookie
2. 可选：轮换 `transient_id`（`all=true` 使所有设备登出
3. 后续 `/members/api/member/` 返回 `null`

### 5.4 状态同步边界总结

```
┌───────────────────────────────────────────────────────────────────────┐
│                     状态同步机制分层                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  postMessage 层 (UI 通知层                              │     │
│  │                                                       │     │
│  │  'portal-ready'          → 通知父页"已就绪（无状态）      │     │
│  │  'portal-preview-ready'  → 通知父页"预览已渲染          │     │
│  │  'portal-preview-updated' → 通知父页"高度变化        │     │
│  │                                                       │     │
│  │  ❌ 不传递任何登录/订阅状态                         │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                              ▲                                    │
│                              │ iframe 通信                           │
┌───────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Cookie 层 (会话标识层)                              │     │
│  │                                                       │     │
│  │  members-ssr Cookie                                       │     │
│  │  ┌─────────┐    ┌──────────────┐    ┌────────┐   │     │
│  │  │ HttpOnly  │    │ Signed      │    │ SameSite │   │     │
│  │  │ JS 不可读 │───►│ 防篡改     │───►│ lax      │   │     │
│  │  │ 防 XSS  │    │              │    │ 防 CSRF │   │     │
│  │  └─────────┘    └──────────────┘    └────────┘   │     │
│  │                                                       │     │
│  │  值: transient_id (会话临时标识)                     │     │
│  │  有效期: 6 个月                                        │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                              ▲                                    │
│                              │ 浏览器自动附加到请求                   │
│                              │ credentials: 'same-origin'            │
┌───────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Members API 层 (状态操作层)                            │     │
│  │                                                       │     │
│  │  GET    /members/api/member/    ← 查身份（验证 Cookie） │     │
│  │  PUT    /members/api/member/    ← 编辑资料              │     │
│  │  DELETE /members/api/session/   ← 清除会话 Cookie      │     │
│  │  GET    /members/api/session/   ← 获取身份 JWT         │     │
│  │                                                       │     │
│  │  ✓ 是状态唯一可信来源                                 │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                     │
└───────────────────────────────────────────────────────────────────────┘
```

## 6. 后台设置同步与缓存刷新

### 6.1 服务端缓存机制

**核心文件**：`ghost/core/core/shared/settings-cache/cache-manager.js`

```javascript
class CacheManager {
    init(events, settingsCollection, calculatedFields, cacheStore, settingsOverrides) {
        this.settingsCache = cacheStore;
        this.reset(events);
        
        // 填充初始缓存
        if (settingsCollection && settingsCollection.models) {
            _.each(settingsCollection.models, this._updateSettingFromModel);
        }
        
        // 监听设置变更事件，自动更新缓存
        events.on('settings.edited', this._updateSettingFromModel);
        events.on('settings.added', this._updateSettingFromModel);
        events.on('settings.deleted', this._updateSettingFromModel);
        
        // 监听计算字段依赖
        this.calculatedFields.forEach((field) => {
            field.dependents.forEach((dependent) => {
                events.on(`settings.${dependent}.edited', this._updateCalculatedField(field));
            });
        });
    }
    
    _updateSettingFromModel(settingModel) {
        debug('Auto updating', settingModel.get('key'));
        this.set(settingModel.get('key'), settingModel.toJSON());
    }
}
```

### 6.2 事件触发机制

设置变更时触发事件，自动更新缓存。

**核心文件**：`ghost/core/core/server/models/settings.js:106-130`

```javascript
emitChange: function emitChange(event, options) {
    const eventToTrigger = 'settings' + '.' + event;
    ghostBookshelf.Model.prototype.emitChange.bind(this)(this, eventToTrigger, options);
},

onUpdated: function onUpdated(model, options) {
    ghostBookshelf.Model.prototype.onUpdated.apply(this, arguments);
    
    // 触发两个事件：
    // 1. 通用事件：settings.edited
    model.emitChange('edited', options);
    // 2. 特定事件：settings.{key}.edited
    model.emitChange(model.attributes.key + '.' + 'edited', options);
}
```

### 6.3 公开设置白名单

Portal 可访问的设置受白名单控制。

**核心文件**：`ghost/core/core/shared/settings-cache/public.js`

```javascript
module.exports = {
    title: 'title',
    description: 'description',
    accent_color: 'accent_color',
    locale: 'locale',
    
    // Portal 相关设置
    members_enabled: 'members_enabled',
    donations_enabled: 'donations_enabled',
    recommendations_enabled: 'recommendations_enabled',
    allow_self_signup: 'allow_self_signup',
    members_signup_access: 'members_signup_access',
    paid_members_enabled: 'paid_members_enabled',
    
    // Portal 外观设置
    portal_button_style: 'portal_button_style',
    portal_button_signup_text: 'portal_button_signup_text',
    portal_button_icon: 'portal_button_icon',
    portal_signup_terms_html: 'portal_signup_terms_html',
    portal_signup_checkbox_required: 'portal_signup_checkbox_required',
    portal_plans: 'portal_plans',
    portal_default_plan: 'portal_default_plan',
    portal_name: 'portal_name',
    portal_button: 'portal_button',
    
    // 其他公开设置...
};
```

### 6.4 实时同步层级

#### 层级 1：服务端内存缓存（毫秒级）

```
后台修改设置 → 数据库更新 → 触发 settings.edited 事件 → CacheManager 更新内存缓存
```

**时间**：几乎实时

**影响范围**：后续所有服务端渲染

#### 层级 2：页面刷新（用户触发）

用户刷新页面后：
1. `{{ghost_head}}` 使用最新的 `settingsCache` 生成脚本标签
2. 可能包含新的 `data-accent-color`、`data-locale` 等
3. Portal 重新初始化，调用 API 获取最新设置

#### 层级 3：API 实时获取（每次请求）

Portal 运行时每次 `initSetup()` 都会：
1. 调用 `api.site.settings()` 获取最新设置
2. 调用 `api.site.tiers()` 获取最新会员等级
3. 调用 `api.site.newsletters()` 获取最新新闻通讯

### 6.5 已打开页面的设置变更：触发条件与时序

#### 6.5.1 生产环境（站点页面已打开）

**问题**：管理员在后台修改设置后，访客浏览器中已打开的页面何时能看到新设置？

**答案**：取决于**数据来源渠道**：

| 数据来源 | 修改后已打开页面是否实时？ | 触发条件 |
|---------|----------------------|-----------|
| `data-*` 属性（如 accent_color | ❌ 否 | 刷新页面 |
| Content API `/settings/` 返回值 | ❌ 否 | 打开 Portal 弹窗/切换页面/刷新 |
| Members API 成员状态 | ✓ 是（Cookie 认证，非设置） | 每次 API 调用 |

#### 6.5.2 生产环境详细时序

```
T0: 用户访问站点页面
    │
    ▼
    Portal 初始化
    ├── getSiteData() 从 script 读取 data-* 属性（静态快照）
    └── fetchApiData() 调用 Content API（此时获取最新设置
    │
    └── 渲染 UI（基于当前设置快照）

T1: 管理员后台修改 portal_button_style = 'icon-and-text'
    │
    ▼
    服务器端：
    ├── SettingsModel.edit()
    ├── 数据库更新
    ├── 触发 settings.edited 事件
    └── CacheManager._updateSettingFromModel() 更新内存缓存 ✓
    │
    └── 新请求：
        ✓ 新页面渲染 {{ghost_head}} 用新值
        ✓ /content/settings/ 返回新值
    │
    ▼
    已打开页面（访客浏览器）：
    ├── ❌ 已读取的 data-* 属性保持旧值
    ├── ❌ Portal React state 保持旧值
    └── ❌ UI 无变化

T2: 用户操作触发：
    场景 A: 用户刷新页面
        │
        ▼
        重新渲染 {{ghost_head}} → 新 data-* 属性
        Portal 重新初始化 → fetchApiData() 获取新设置
        ✓ UI 更新

    场景 B: 用户点击 Portal 按钮 → 打开弹窗
        │
        ▼
        Portal 不重新初始化（同一 React 实例）
        ❌ 用旧 state 渲染弹窗

    场景 C: 用户导航到新页面（SPA 未刷新）
        │
        ▼
        Portal 仍为同一 React 实例
        ❌ data-* 未重新读取
        ❌ initSetup() 不再执行
        └── 仅在某些主题有 SPA 路由？不刷新页面
```

#### 6.5.3 触发已打开页面看到新设置需要满足的条件

**场景 1：** 刷新页面 ✓

```javascript
// 条件：window.location.reload()
结果：
  ├── 重新请求 HTML → {{ghost_head}} 用最新 settingsCache
├── Portal 脚本重新执行 init()
  └── initSetup() → fetchApiData() → 读取最新 /settings/
  └── ✓ 显示新设置
```

**场景 2：** 打开新 Portal 弹窗（部分场景更新？不保证）

```javascript
// 取决于设置来源：
├── accent_color 来自 data-accent-color (script tag)
│   不重新读取 script → ❌ 不更新
│
├── portal_button_style 来自 /content/settings/
│   仅 initSetup() 时调用一次
│   弹窗打开不重新调用 → ❌ 不更新
│
└── member 状态来自 /members/api/member/
    每次 API 调用 → ✓ 实时（非设置）
```

**场景 3：** 后台保存设置时手动强制刷新（后台预览

```javascript
// gh-site-iframe.js 中使用 `resetSrcAttribute() 强制 iframe 重新加载：

// ghost/admin/app/components/gh-site-iframe.js:32-56
resetSrcAttribute(iframe) {
    if (this.args.guid !== this._lastGuid) {
        try {
            if (iframe.contentWindow.location.reload();
        } catch (e) {
            if (e.name === 'SecurityError') {
                iframe.src = this.srcUrl;
            }
        }
    }
}
```

#### 6.5.4 生产环境没有实时同步机制对比

| 机制 | 已打开页面实时？ | 触发条件 | 延迟 |
|------|-------------|---------|------|
| 服务端内存缓存 | N/A | settingsCache 更新 | 毫秒级 |
| 新页面渲染 | N/A | 新请求 | 下一次请求 |
| 已打开页面 data-* | ❌ 否 | 刷新页面 | 直到用户刷新 |
| 已打开页面 React state | ❌ 否 | 刷新页面 / 重新加载 Portal | 直到刷新 |
| Content API /settings/ | ❌ 否 | Portal 重新 initSetup() | 直到刷新 |

**结论**：生产环境已打开的页面**没有实时推送机制**。管理员修改设置后，已打开的浏览器标签页的 Portal 保持旧设置，直到用户刷新页面或打开新页面。

### 6.6 预览模式实时同步：URL hash 机制

后台预览时，设置通过 URL hash 实时传递。这是**仅预览才有的实时同步机制，生产环境不使用。

#### 6.6.1 后台构建预览 URL

**核心文件**：`apps/admin-x-settings/src/utils/get-portal-preview-url.ts:14-75`

```typescript
export const getPortalPreviewUrl = ({settings, config, tiers, siteData, selectedTab}: portalPreviewUrlTypes): string | null => {
    const baseUrl = siteData.url.replace(/\/$/, '');
    const portalBase = '/?v=modal-portal-settings#/portal/preview';
    //                              ▲
    //                    标记预览模式
    
    const settingsParam = new URLSearchParams();
    
    // 将当前表单值编码到 URL 参数中
    settingsParam.append('button', getSettingValue(settings, 'portal_button') ? 'true' : 'false');
    settingsParam.append('name', getSettingValue(settings, 'portal_name') ? 'true' : 'false');
    settingsParam.append('accentColor', encodeURIComponent(accentColor));
    settingsParam.append('buttonStyle', encodeURIComponent(portalButtonStyle));
    settingsParam.append('signupButtonText', encodeURIComponent(signupButtonText));
    settingsParam.append('membersSignupAccess', getSettingValue(settings, 'members_signup_access'));
    // ... 更多设置
    
    return `${baseUrl}${portalBase}?${settingsParam.toString()}`;
};
```

生成的 URL 示例：

```
https://site.com/?v=modal-portal-settings
  #/portal/preview
  ?button=true
  &name=true
  &accentColor=%23FF5733
  &buttonStyle=icon-and-text
  &signupButtonText=Subscribe
```

#### 6.6.2 Portal 解析预览参数

**核心文件**：`apps/portal/src/utils/check-mode.js:1-13`

```javascript
export const isPreviewMode = function () {
    return isNormalPreviewMode() || isOfferPreviewMode();
};

export const isNormalPreviewMode = function () {
    const [path] = window.location.hash.substr(1).split('?');
    return (path === '/portal/preview');  // 检查 hash 路径
};

export const isOfferPreviewMode = function () {
    const [path] = window.location.hash.substr(1).split('?');
    return (path === '/portal/preview/offer');
};
```

**核心文件**：`apps/portal/src/app.js:760-776`

```javascript
fetchPreviewData() {
    const [, qs] = window.location.hash.substr(1).split('?');
    if (hasMode(['preview'])) {
        let data = {};
        if (hasMode(['offerPreview'])) {
            data = this.fetchOfferQueryStrData(qs);
        } else {
            data = this.fetchQueryStrData(qs);
        }
        return {
            ...data,
            showPopup: true
        };
    }
    return {};
}
```

#### 6.6.3 fetchQueryStrData 解析参数

**核心文件**：`apps/portal/src/app.js:432-532`

```javascript
fetchQueryStrData(qs = '') {
    const qsParams = new URLSearchParams(qs);
    const data = {
        site: { plans: {} }
    };
    
    for (let pair of qsParams.entries()) {
        const key = pair[0];
        const value = decodeURIComponent(pair[1]);
        
        // 解析各个预览参数
        if (key === 'button') {
            data.site.portal_button = JSON.parse(value);
        } else if (key === 'accentColor') {
            data.site.accent_color = value;
        } else if (key === 'buttonStyle') {
            data.site.portal_button_style = value;
        } else if (key === 'signupButtonText') {
            data.site.portal_button_signup_text = value || '';
        }
        // ... 更多设置值
    }
    return data;
}
```

#### 6.6.4 预览模式实时同步

**核心文件**：`apps/portal/src/app.js:926-951`

```javascript
async updateStateForPreviewLinks() {
    const {site: previewSite, ...restPreviewData} = this.fetchPreviewData();
    const linkData = await this.fetchLinkData(this.state.site, this.state.member);
    
    const updatedState = {
        site: {
            ...this.state.site,
            ...(previewSite || {}),
            plans: {
                ...(this.state.site && this.state.site.plans),
                ...(previewSite || {}).plans
            }
        },
        ...restLinkData,
        ...restPreviewData
    };
    this.setState(updatedState);
}
```

**核心文件**：`apps/portal/src/app.js:294-297`

```javascript
// 监听 hash 变化（预览模式）
this.hashHandler = () => {
    this.updateStateForPreviewLinks();
};
window.addEventListener('hashchange', this.hashHandler, false);
```

#### 6.6.5 后台预览实时同步时序

```
后台预览 iframe 实时同步流程：

T0: Admin 后台加载 Portal 设置页面
    │
    ▼
    PortalModal 组件渲染
    ├── getPortalPreviewUrl({settings: localSettings, ...})
    │   用当前本地表单值构建预览 URL
    │
    └── PortalFrame <iframe src={href}/> 加载

T1: 用户在后台表单修改 accent_color
    │
    ▼
    localSettings (React state) 更新
    │
    └── 组件重新渲染
        │
        └── getPortalPreviewUrl() 重新计算
            │
            └── 新 href = "...&accentColor=%23FF5733"
            │
            └── PortalFrame src 变更
            │
            └── iframe 重新加载整个页面

T2: iframe 重新加载（完整页面重载）
    │
    ▼
    站点页面重新加载（完整 HTML + Portal 重新初始化）
    │
    └── Portal init()
        ├── isNormalPreviewMode() → true（hash 路径为 #/portal/preview）
        │
        ├── fetchPreviewData() 解析 hash query 中的 accentColor
        │
        └── updateStateForPreviewLinks() → setState
        │
        └── ✓ 显示新颜色（iframe 已重载）
```

**预览模式与生产模式对比**：

| 维度 | 预览模式（后台） | 生产模式（访客） |
|------|-----------------|-----------------|
| 设置传递方式 | URL hash 参数 | data-* 属性 + Content API |
| 已打开页面实时同步？ | ✓ 是（iframe 重载） | ❌ 否 |
| 数据来源优先级 | hash 参数 > API | API + data-* |
| 触发实时 | 修改即生效 | 需刷新页面 |
| 持久化到数据库？ | ❌ 否（本地表单 state） | ✓ 是（保存时） |

### 6.7 缓存配置

**核心文件**：`ghost/core/core/shared/config/defaults.json:148-197`

```json
{
    "caching": {
        "frontend": {
            "maxAge": 0  // 前端页面不缓存
        },
        "contentAPI": {
            "maxAge": 0  // Content API 不缓存
        },
        "publicAssets": {
            "maxAge": 31536000  // 静态资源长期缓存
        },
        "cors": {
            "maxAge": 86400  // CORS 预检缓存 1 天
        }
    }
}
```

**缓存策略解释：

| 缓存类型 | maxAge | 说明 |
|---------|--------|------|
| `frontend` | 0 | 不缓存 HTML 页面，每次请求重新渲染 |
| `contentAPI` | 0 | 不缓存 API 响应，每次返回最新值 |
| `publicAssets` | 1 年 | 静态资源（图片、CSS、JS）长期缓存 |
| `cors` | 1 天 | CORS 预检请求缓存，减少 OPTIONS 请求 |

## 7. 数据属性交互

Portal 支持通过 HTML 数据属性与页面元素交互。

**核心文件**：`apps/portal/src/data-attributes.js`

### 7.1 支持的属性

| 属性名 | 用途 | 示例 |
|--------|------|------|
| `data-members-form` | 登录/注册表单 | `<form data-members-form="signup">` |
| `data-members-email` | 邮箱输入框 | `<input data-members-email>` |
| `data-members-name` | 姓名输入框 | `<input data-members-name>` |
| `data-members-plan` | 价格按钮 | `<button data-members-plan="monthly">` |
| `data-members-signout` | 退出按钮 | `<a data-members-signout>` |
| `data-members-edit-billing` | 编辑账单 | `<a data-members-edit-billing>` |
| `data-members-manage-billing` | 管理账单 | `<a data-members-manage-billing>` |
| `data-members-cancel-subscription` | 取消订阅 | `<a data-members-cancel-subscription="sub_xxx">` |
| `data-portal` | 自定义触发器 | `<button data-portal="signup">` |
| `data-recommendation` | 推荐按钮 | `<a data-recommendation="rec_xxx">` |

### 7.2 处理逻辑

**核心文件**：`apps/portal/src/data-attributes.js:208-228`

```javascript
export function handleDataAttributes({siteUrl, site = {}, member, offers = [], doAction, captureException} = {}) {
    // 处理登录/注册表单
    Array.prototype.forEach.call(document.querySelectorAll('form[data-members-form]'), function (form) {
        function submitHandler(event) {
            formSubmitHandler({event, errorEl, form, siteUrl, submitHandler, doAction, captureException});
        }
        form.addEventListener('submit', submitHandler);
    });
    
    // 处理价格按钮
    Array.prototype.forEach.call(document.querySelectorAll('[data-members-plan]'), function (el) {
        function clickHandler(event) {
            planClickHandler({el, event, errorEl, member, site, siteUrl, clickHandler});
        }
        el.addEventListener('click', clickHandler);
    });
    
    // ... 其他属性处理
}
```

## 8. 架构图总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                        站点页面 (Theme)                        │
│                                                                 │
│  <head>                                                         │
│    {{ghost_head}}                                               │
│      │                                                          │
│      ├─> <script src="CDN/portal.min.js"                        │
│      │            data-ghost="https://site.com"                 │
│      │            data-key="content_api_key"                    │
│      │            data-api="https://site.com/ghost/api/content/"│
│      │            data-locale="zh-CN"                           │
│      │            crossorigin="anonymous">                          │
│      │                                                          │
│      └─> <script src="https://js.stripe.com/v3/"> (付费时)│
│                                                                 │
│  <body>                                                         │
│    <div id="ghost-portal-root"></div>  <-- Portal 挂载点   │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼ 加载 Portal UMD bundle
┌─────────────────────────────────────────────────────────────────────┐
│                      Portal React 应用                               │
│                                                                 │
│  init()                                                         │
│    │                                                            │
│    ├─> getSiteData()  -- 从 script 标签读取 data-* 属性  │
│    │                                                            │
│    └─> <App> 初始化                                           │
│         │                                                       │
│         └─> initSetup()                                          │
│               │                                                  │
│               ├─> fetchData()                                     │
│               │    │                                             │
│               │    ├─> fetchApiData()  <─── 实时 API 调用      │
│               │    │    │                                        │
│               │    │    ├─> GET /members/api/member/            │
│               │    │    │     (credentials: same-origin)             │
│               │    │    │                                    │
│               │    │    ├─> GET /ghost/api/content/settings/   │
│               │    │    │                                    │
│               │    │    ├─> GET /ghost/api/content/tiers/    │
│               │    │    │                                    │
│               │    │    └─> GET /ghost/api/content/newsletters/  │
│               │    │                                             │
│               │    ├─> fetchPreviewData()  <── URL hash (预览模式)│
│               │    │                                             │
│               │    └─> fetchLinkData()  <── #/portal/... 链接 │
│               │                                                  │
│               └─> 监听 hashchange 事件 (预览模式实时更新)    │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼ 弹出弹窗时创建 iframe
┌─────────────────────────────────────────────────────────────────────┐
│                   Portal iframe (srcDoc)                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐            │
│  │  <Frame>                                            │            │
│  │    │                                                │            │
│  │    ├─> 与父页通信 (postMessage)                      │            │
│  │    │    ├─> 'portal-ready'                          │            │
│  │    │    ├─> 'portal-preview-ready'                 │            │
│  │    │    └─> 'portal-preview-updated' (高度变更)    │            │
│  │    │                                                │            │
│  │    └─> 渲染页面组件 (Signup, Signin, Account...)     │            │
│  └─────────────────────────────────────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                        Ghost 后端服务                               │
│                                                                 │
│  ┌──────────────────┐    ┌──────────────────┐                       │
│  │  数据库 (MySQL)   │    │  settingsCache   │                       │
│  │                  │    │  (内存缓存)       │                       │
│  │  settings 表      │◄───┤                  │                       │
│  │  - key            │    │  get(key)        │                       │
│  │  - value          │    │  getPublic()     │                       │
│  │  - type           │    │                  │                       │
│  └────────┬─────────┘    └────────▲─────────┘                       │
│           │                       │                                  │
│           ▼                       │                                  │
│  ┌──────────────────┐             │                                  │
│  │  Settings Model  │             │                                  │
│  │                  │             │                                  │
│  │  edit()          │───事件───►  │  settings.edited                │
│  │  onUpdated()     │             │  _updateSettingFromModel()       │
│  └──────────────────┘             │                                  │
│                                   │                                  │
│  ┌──────────────────┐             │                                  │
│  │  {{ghost_head}}  │◄────────────┘                                  │
│  │  辅助函数         │                                                │
│  │                  │                                                │
│  │  读取 settingsCache                                              │
│  │  生成脚本注入 HTML                                               │
│  └──────────────────┘                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      后台设置变更流程                                 │
│                                                                 │
│  管理员在后台修改设置                                                 │
│         │                                                            │
│         ▼                                                            │
│  SettingsModel.edit()                                                │
│         │                                                            │
│         ▼                                                            │
│  数据库更新 settings 表                                              │
│         │                                                            │
│         ▼                                                            │
│  触发事件:                                                            │
│    - settings.edited                                                 │
│    - settings.{key}.edited (如: settings.portal_button.edited)       │
│         │                                                            │
│         ▼                                                            │
│  CacheManager._updateSettingFromModel()  ──►  更新内存缓存           │
│         │                                                            │
│         ▼                                                            │
│  后续请求:                                                            │
│    ├─> 新页面渲染: {{ghost_head}} 使用新值                            │
│    └─> API 调用: /content/settings/ 返回新值                          │
│                                                                 │
│  已打开的页面: 需要刷新或重新初始化 Portal                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                    预览模式实时同步（后台 iframe）                         │
│                                                                 │
│  Admin: PortalModal localSettings (React state)            │
│         │                                                            │
│         ▼ 每次输入变化                                               │
│  getPortalPreviewUrl(localSettings)                           │
│         │                                                            │
│         ▼ 构建 URL                                                     │
│  /?v=modal-portal-settings#/portal/preview?button=true&accentColor=... │
│         │                                                            │
│         ▼ iframe src 变化                                              │
│  站点页面重新加载 / hashchange 事件                              │
│         │                                                            │
│         ▼                                                            │
│  Portal:                                                           │
│    ├── isPreviewMode() = true                                         │
│    ├── fetchPreviewData() 解析 hash 参数                            │
│    └── updateStateForPreviewLinks() setState                          │
│                                                                 │
│  ✓ 实时预览，无需保存                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## 9. 关键文件索引

| 功能模块 | 文件路径 |
|----------|----------|
| 脚本注入 | `ghost/core/core/frontend/helpers/ghost_head.js` |
| 前端应用配置 | `ghost/core/core/frontend/utils/frontend-apps.js` |
| Portal 入口 | `apps/portal/src/index.js` |
| Portal 主应用 | `apps/portal/src/app.js` |
| Portal API | `apps/portal/src/utils/api.js` |
| 数据属性处理 | `apps/portal/src/data-attributes.js` |
| Iframe 组件 | `apps/portal/src/components/frame.js` |
| 弹窗组件 | `apps/portal/src/components/popup-modal.js` |
| 预览模式检测 | `apps/portal/src/utils/check-mode.js` |
| 公开设置白名单 | `ghost/core/core/shared/settings-cache/public.js` |
| 缓存管理器 | `ghost/core/core/shared/settings-cache/cache-manager.js` |
| Members 会话 Cookie | `ghost/core/core/server/services/members/members-ssr.js` |
| Members API 中间件 | `ghost/core/core/server/services/members/middleware.js` |
| Members CORS | `ghost/core/core/server/web/members/middleware/cors.js` |
| 设置模型 | `ghost/core/core/server/models/settings.js` |
| 默认配置 | `ghost/core/core/shared/config/defaults.json` |
| 预览 URL 生成 | `apps/admin-x-settings/src/utils/get-portal-preview-url.ts` |
| 预览 iframe 包装 | `apps/admin-x-settings/src/components/settings/membership/portal/portal-frame.tsx` |
| Portal 设置弹窗 | `apps/admin-x-settings/src/components/settings/membership/portal/portal-modal.tsx` |
| Admin 站点 iframe | `ghost/admin/app/components/gh-site-iframe.js` |
