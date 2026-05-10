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

### 4.1 脚本加载方式

Portal 使用 `crossorigin="anonymous"` 属性加载脚本：

```html
<script defer src="..." crossorigin="anonymous"></script>
```

这允许：
- 从 CDN 加载脚本时不发送用户凭证（cookies）
- 错误信息可以被捕获（Sentry 等）

### 4.2 CORS 配置

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

### 4.3 凭证传递

Portal API 请求使用 `credentials: 'same-origin'` 或 `credentials: 'include'`：

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

### 4.4 Stripe 第三方脚本

付费会员功能需要加载 Stripe 脚本：

**核心文件**：`ghost/core/core/frontend/helpers/ghost_head.js:77-79`

```javascript
if (settingsCache.get('paid_members_enabled')) {
    membersHelper += `<script async src="https://js.stripe.com/v3/"></script>`;
}
```

## 5. iframe 与父页通信

### 5.1 iframe 结构

Portal 使用 `srcDoc` 创建同源 iframe，避免跨域问题：

**核心文件**：`apps/portal/src/components/frame.js:27-43`

```javascript
render() {
    const {children, head, title = '', ...rest} = this.props;
    return (
        <iframe
            srcDoc={`<!DOCTYPE html>`}
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

### 5.2 消息事件类型

Portal 使用 `window.postMessage` 进行跨窗口通信：

**核心文件**：`apps/portal/src/app.js:149-156`

```javascript
sendPortalReadyEvent() {
    if (window.self !== window.parent) {
        window.parent.postMessage({
            type: 'portal-ready',
            payload: {}
        }, '*');
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

### 5.3 登录与订阅状态传递

登录和订阅状态通过 **HTTP-only Cookies** 维护，不直接通过 postMessage 传递：

1. **登录流程**：
   - Portal 发送登录请求到 `/members/api/send-magic-link/`
   - 用户点击邮件中的 magic link
   - 服务器设置 `ghost-members-ssr` Cookie
   - 后续请求通过 `credentials: 'same-origin'` 携带 Cookie

2. **状态检查**：
   - Portal 初始化时调用 `api.member.sessionData()`
   - 请求自动携带 Cookie
   - 返回成员信息（如果已登录）

**核心文件**：`apps/portal/src/utils/api.js:240-251`

```javascript
sessionData() {
    const url = endpointFor({type: 'members', resource: 'member'});
    return makeRequest({
        url,
        credentials: 'same-origin'  // 自动携带 Cookie
    }).then(function (res) {
        if (!res.ok || res.status === 204) {
            return null;  // 未登录
        }
        return res.json();  // 已登录，返回成员信息
    });
}
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
                events.on(`settings.${dependent}.edited`, this._updateCalculatedField(field));
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

#### 层级 1：服务端缓存（毫秒级）

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

### 6.5 预览模式实时同步

后台预览时，设置通过 URL hash 实时传递：

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
        ...restPreviewData
    };
    this.setState(updatedState);
}
```

### 6.6 缓存配置

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
│                        站点页面 (Theme)                              │
│                                                                      │
│  <head>                                                             │
│    {{ghost_head}}                                                   │
│      │                                                              │
│      ├─> <script src="CDN/portal.min.js"                            │
│      │            data-ghost="https://site.com"                     │
│      │            data-key="content_api_key"                        │
│      │            data-api="https://site.com/ghost/api/content/"    │
│      │            data-locale="zh-CN"                               │
│      │            crossorigin="anonymous">                          │
│      │                                                              │
│      └─> <script src="https://js.stripe.com/v3/"> (付费时)          │
│                                                                      │
│  <body>                                                             │
│    <div id="ghost-portal-root"></div>  <-- Portal 挂载点             │
│                                                                      │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼ 加载 Portal UMD bundle
┌─────────────────────────────────────────────────────────────────────┐
│                      Portal React 应用                               │
│                                                                      │
│  init()                                                             │
│    │                                                                │
│    ├─> getSiteData()  -- 从 script 标签读取 data-* 属性              │
│    │                                                                │
│    └─> <App> 初始化                                                   │
│         │                                                           │
│         └─> initSetup()                                              │
│               │                                                      │
│               ├─> fetchData()                                         │
│               │    │                                                 │
│               │    ├─> fetchApiData()  <─── 实时 API 调用            │
│               │    │    │                                            │
│               │    │    ├─> GET /members/api/member/                 │
│               │    │    │     (credentials: same-origin)             │
│               │    │    │                                            │
│               │    │    ├─> GET /ghost/api/content/settings/         │
│               │    │    │                                            │
│               │    │    ├─> GET /ghost/api/content/tiers/            │
│               │    │    │                                            │
│               │    │    └─> GET /ghost/api/content/newsletters/      │
│               │    │                                                 │
│               │    ├─> fetchPreviewData()  <── URL hash (预览模式)   │
│               │    │                                                 │
│               │    └─> fetchLinkData()  <── #/portal/... 链接       │
│               │                                                      │
│               └─> 监听 hashchange 事件 (预览模式实时更新)            │
│                                                                      │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼ 弹出弹窗时创建 iframe
┌─────────────────────────────────────────────────────────────────────┐
│                   Portal iframe (srcDoc)                            │
│                                                                      │
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
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                        Ghost 后端服务                               │
│                                                                      │
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
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      后台设置变更流程                                 │
│                                                                      │
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
│                                                                      │
│  已打开的页面: 需要刷新或重新初始化 Portal                           │
│                                                                      │
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
| 公开设置白名单 | `ghost/core/core/shared/settings-cache/public.js` |
| 缓存管理器 | `ghost/core/core/shared/settings-cache/cache-manager.js` |
| Members CORS | `ghost/core/core/server/web/members/middleware/cors.js` |
| 设置模型 | `ghost/core/core/server/models/settings.js` |
| 默认配置 | `ghost/core/core/shared/config/defaults.json` |
