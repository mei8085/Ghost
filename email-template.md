# Ghost 自定义 Newsletter 模板渲染过程分析

## 目录

1. [整体架构](#整体架构)
2. [后台 Email Designer 与主题模板共享版式](#后台-email-designer-与主题模板共享版式)
3. [个性化变量绑定到会员字段](#个性化变量绑定到会员字段)
4. [批量发送与跨邮箱客户端兼容](#批量发送与跨邮箱客户端兼容)
5. [多语言兜底机制](#多语言兜底机制)
6. [关键文件索引](#关键文件索引)

---

## 整体架构

### 三层服务架构

Ghost 的 newsletter 邮件渲染系统采用三层服务架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    BatchSendingService                       │
│  (批量调度：分批、并发控制、重试机制、域名预热)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      SendingService                          │
│  (发送协调：缓存渲染结果、构建收件人列表、调用 Provider)     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      EmailRenderer                           │
│  (核心渲染：模板解析、变量替换、CSS 内联、客户端兼容处理)     │
└─────────────────────────────────────────────────────────────┘
```

### 依赖注入初始化

**文件**：`ghost/core/core/server/services/email-service/email-service-wrapper.js:13`

```javascript
init() {
    const i18nLanguage = settingsCache.get('locale') || 'en';  // 从 settings 读 locale
    const i18n = i18nLib(i18nLanguage, 'ghost');              // 创建 i18n 实例

    const emailRenderer = new EmailRenderer({
        settingsCache,
        // ...
        t: i18n.t,           // 注入真实的 i18n.t 翻译函数
        dir: i18n.dir.bind(i18n)  // 注入 i18n.dir 方向判断
    });

    const sendingService = new SendingService({
        emailProvider: mailgunEmailProvider,
        emailRenderer,
        // ...
    });

    const batchSendingService = new BatchSendingService({
        sendingService,
        emailRenderer,
        // ...
    });
}
```

### 渲染流程总览

1. **触发发送**：`BatchSendingService.emailJob()` 被 JobsService 调度执行
2. **创建批次**：按 1000 人/批创建 `EmailBatch`，关联 `EmailRecipient`
3. **渲染邮件体**：`EmailRenderer.renderBody()` 执行完整渲染流水线
4. **构建收件人**：`SendingService.buildRecipients()` 绑定个性化变量
5. **发送**：通过 Mailgun 等 Provider 批量发送

---

## 后台 Email Designer 与主题模板共享版式

### 核心设计：语义值存储 + 独立解析

**关键发现**：前后端不共享计算函数，而是各自独立实现相同的解析逻辑。

#### 数据流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Newsletters 表（数据库存储层）                        │
│  background_color: 'light'  (默认值，语义值，非 HEX)                     │
│  header_background_color: 'transparent'  (语义值)                        │
│  button_color: 'accent'  (语义值)                                        │
│  link_color: 'accent'  (语义值)                                          │
│  title_font_weight: 'bold'  (枚举值)                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
    ┌───────────────┐      ┌──────────────────┐    ┌──────────────────┐
    │  管理端预览    │      │   邮件渲染端      │    │   HEX 颜色场景   │
    │               │      │                  │    │                  │
    │ Newsletter-   │      │ getEmailDesign() │    │ 直接存储 #ffffff │
    │ Preview       │      │ (只处理 HEX)      │    │                  │
    │ (newsletter-  │      │                  │    │                  │
    │  preview.tsx) │      │                  │    │                  │
    └───────────────┘      └──────────────────┘    └──────────────────┘
```

#### Newsletter 模型默认值

**文件**：`ghost/core/core/server/models/newsletter.js:9-41`

```javascript
defaults: function defaults() {
    return {
        // 语义值（非 HEX）
        background_color: 'light',           // 语义值
        header_background_color: 'transparent',  // 语义值
        button_color: 'accent',              // 语义值
        link_color: 'accent',                // 语义值
        
        // 枚举值
        button_corners: 'rounded',           // square | rounded | pill
        button_style: 'fill',                // fill | outline
        title_font_weight: 'bold',           // normal | medium | semibold | bold
        link_style: 'underline',             // underline | plain
        image_corners: 'square',             // square | rounded
        
        // 布尔值
        show_badge: true,
        show_header_icon: true,
        show_header_title: true,
        show_feature_image: true,
        show_comment_cta: true,
        feedback_enabled: false
    };
}
```

#### 管理端预览解析

**文件**：`apps/admin-x-settings/src/components/settings/email/newsletters/newsletter-preview.tsx:26-52`

```typescript
// 管理端独立实现语义值 → HEX 的转换
const backgroundColor = () => {
    const value = newsletter.background_color;
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    
    if (validHex.test(value)) {
        return value;           // HEX 直接返回
    }
    return '#ffffff';           // 'light' 或其他无效值 → 白色
};

const headerBackgroundColor = () => {
    const value = newsletter.header_background_color;
    
    if (!value || value === 'transparent') {
        return 'transparent';   // 语义值 → 透明
    }
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    if (validHex.test(value)) {
        return value;
    }
    return 'transparent';       // 其他无效值 → 透明
};

const buttonColor = () => {
    const value = newsletter.button_color;
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    
    if (validHex.test(value || '')) {
        return value;           // HEX 直接返回
    }
    
    if (value === null) {
        const bg = backgroundColor();
        return textColorForBackgroundColor(bg).hex();  // Auto → 背景对比色
    }
    
    return siteData.accent_color;  // 'accent' 或其他无效值 → 站点主色
};

const linkColor = () => {
    const value = newsletter.link_color === undefined ? 'accent' : newsletter.link_color;
    
    if (value === 'accent') {
        return siteData.accent_color;  // 语义值 → 站点主色
    }
    
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    if (validHex.test(value || '')) {
        return value;
    }
    
    // 其他无效值 → 背景对比色
    return textColorForBackgroundColor(backgroundColor()).hex();
};
```

#### 管理端设计工具函数

**文件**：`apps/admin-x-settings/src/components/settings/email-design/design-utils.ts:18-128`

```typescript
const VALID_HEX = /^#(?:[0-9a-f]{3}){1,2}$/i;

function resolveBackgroundColor(value: string): string {
    if (VALID_HEX.test(value)) return value;
    return '#ffffff';           // 'light' 等语义值 → 白色
}

function resolveHeaderBackgroundColor(value: string, accentColor: string): string {
    if (!value || value === 'transparent') return 'transparent';
    if (value === 'accent') return accentColor;
    if (VALID_HEX.test(value)) return value;
    return 'transparent';
}

function resolveButtonColor(value: string | null, accentColor: string, bgColor: string): string {
    if (VALID_HEX.test(value || '')) return value!;
    if (value === null) return textColorForBackgroundColor(bgColor).hex();
    return accentColor;         // 'accent' 等 → 站点主色
}

export function resolveAllColors(settings: EmailDesignSettings, accentColor: string): ResolvedEmailColors {
    const bgColor = resolveBackgroundColor(settings.background_color);
    const headerBgColor = resolveHeaderBackgroundColor(settings.header_background_color, accentColor);
    const buttonColor = resolveButtonColor(settings.button_color, accentColor, bgColor);
    // ...
}

export function resolveButtonCorners(corners: string | undefined): string {
    switch (corners) {
    case 'square': return 'rounded-none';
    case 'pill': return 'rounded-full';
    case 'rounded':
    default: return 'rounded-[6px]';
    }
}
```

#### 邮件渲染端 getEmailDesign

**文件**：`ghost/core/core/server/services/email-rendering/email-design.js:71-198`

```javascript
// email-design.js 只处理 HEX 颜色，不识别 'light'、'transparent' 等语义值

const DEFAULT_ACCENT_COLOR = '#15212A';
const DEFAULT_DIVIDER_COLOR = '#e0e7eb';
const VALID_HEX_REGEX = /^#([0-9a-f]{3}){1,2}$/i;

const getHexColor = (value, fallback) => (isValidHexColor(value) ? value : fallback);

exports.getEmailDesign = (settings) => {
    // accentColor: 无效值回退到 DEFAULT_ACCENT_COLOR (#15212A)
    const accentColor = getHexColor(settings.accentColor, DEFAULT_ACCENT_COLOR);
    
    // backgroundColor: 无效值回退到 #ffffff（'light' 被视为无效）
    const backgroundColor = getHexColor(settings.backgroundColor, '#ffffff');
    
    // buttonColor: 'accent' 被特殊处理，其他无效值回退到 accentColor
    let buttonColor;
    switch (settings.buttonColor) {
    case 'accent':
        buttonColor = accentColor;
        break;
    case null:
        buttonColor = textColorForBackgroundColor(backgroundColor).hex();
        break;
    default:
        buttonColor = getHexColor(settings.buttonColor, accentColor);
        break;
    }
    
    // dividerColor: 'accent' 被特殊处理
    let dividerColor;
    if (settings.dividerColor === 'accent') {
        dividerColor = accentColor;
    } else {
        dividerColor = getHexColor(settings.dividerColor, DEFAULT_DIVIDER_COLOR);
    }
    
    // headerBackgroundColor: 'accent' 被特殊处理，其他无效值 → null
    let headerBackgroundColor;
    if (settings.headerBackgroundColor === 'accent') {
        headerBackgroundColor = accentColor;
    } else if (isValidHexColor(settings.headerBackgroundColor)) {
        headerBackgroundColor = settings.headerBackgroundColor;
    } else {
        headerBackgroundColor = null;  // 'transparent' → null
    }
    
    // linkColor: 'accent' 被特殊处理
    let linkColor;
    switch (settings.linkColor) {
    case 'accent':
        linkColor = accentColor;
        break;
    case null:
        linkColor = textColorForBackgroundColor(backgroundColor).hex();
        break;
    default:
        linkColor = getHexColor(settings.linkColor, accentColor);
        break;
    }
    
    // titleWeight: 枚举值映射
    let titleWeight;
    switch (settings.titleFontWeight) {
    case 'normal': titleWeight = 400; break;
    case 'medium': titleWeight = 500; break;
    case 'semibold': titleWeight = 600; break;
    default: titleWeight = 700; break;
    }
    
    return {
        accentColor,
        backgroundColor,
        backgroundIsDark: isDark(backgroundColor),
        buttonColor,
        buttonTextColor: textColorForBackgroundColor(buttonColor).hex(),
        headerBackgroundColor,
        linkColor,
        titleWeight: String(titleWeight),
        // ...
    };
};
```

#### 单元测试验证语义值处理

**文件**：`ghost/core/test/unit/server/services/email-rendering/email-design.test.js:11-32`

```javascript
const INVALID_HEX_COLORS = [
    null,
    undefined,
    0xabcdef,
    ['#ff9900'],
    '',
    'invalid',
    'accent',       // 被视为无效（除非特殊处理）
    'light',        // 被视为无效！
    'dark',         // 被视为无效！
    '#',
    '#F',
    // ...
];

it('returns the default background color when input is invalid', function () {
    for (const backgroundColor of INVALID_HEX_COLORS) {
        const result = getEmailDesign({...baseSettings, backgroundColor});
        assertColorsEqual(result.backgroundColor, '#ffffff');  // 都回退到白色
    }
});
```

#### EmailRenderer 调用 getEmailDesign

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:1025`

```javascript
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const emailDesign = this.#getEmailDesign(newsletter);
    // ...
    const data = {
        // ...
        // CSS 变量：展开 emailDesign
        ...emailDesign,
        // ...
    };
}

#getEmailDesign(newsletter) {
    return getEmailDesign({
        accentColor: this.#settingsCache?.get('accent_color'),
        backgroundColor: newsletter?.get('background_color'),         // 'light'
        buttonColor: newsletter?.get('button_color'),                 // 'accent'
        buttonCorners: newsletter?.get('button_corners'),             // 'rounded'
        buttonStyle: newsletter?.get('button_style'),                 // 'fill'
        headerBackgroundColor: newsletter?.get('header_background_color'),  // 'transparent'
        linkColor: newsletter?.get('link_color'),                     // 'accent'
        linkStyle: newsletter?.get('link_style'),                     // 'underline'
        titleFontWeight: newsletter?.get('title_font_weight')         // 'bold'
    });
}
```

#### 语义值 → HEX 转换路径

| 字段 | 数据库值 | email-design.js 处理 | 最终 HEX |
|------|----------|---------------------|----------|
| `background_color` | `'light'` | `getHexColor('light', '#ffffff')` → 无效 | `#ffffff` |
| `header_background_color` | `'transparent'` | `isValidHexColor('transparent')` → 否 | `null` |
| `button_color` | `'accent'` | `case 'accent'` → `accentColor` | 站点 `accent_color` |
| `link_color` | `'accent'` | `case 'accent'` → `accentColor` | 站点 `accent_color` |

#### Handlebars 模板注入样式

**文件**：`ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs`

```handlebars
/* emailDesign 展开后注入 */
.header {
    {{#if headerBackgroundColor}}
    background-color: {{headerBackgroundColor}};
    {{else}}
    background-color: {{backgroundColor}};
    {{/if}}
}

.post-title {
    color: {{postTitleColor}};
    font-weight: {{titleWeight}};
}

a {
    color: {{linkColor}};
    {{#if (eq linkStyle "underline")}}
    text-decoration: underline;
    {{/if}}
}
```

---

## 个性化变量绑定到会员字段

### 变量语法

邮件模板使用 `%%{variable}%%` 语法标识个性化变量：

**文件**：`ghost/core/core/server/services/email-service/email-templates/template.hbs`

```handlebars
<a href="%%{unsubscribe_url}%%">{{t 'Unsubscribe'}}</a>
<p class="%%{name_class}%%">{{t 'Name'}}: %%{name, "not provided"}%%</p>
<p>{{t 'Email'}}: <a href="#">%%{email}%%</a></p>
```

### 支持的变量定义

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:719-810`

```javascript
buildReplacementDefinitions({html, newsletterUuid}) {
    const t = this.#t;           // 真实的 i18n.t 实例（不是占位函数）
    const locale = this.#getValidLocale();

    const baseDefinitions = [
        {
            id: 'unsubscribe_url',
            getValue: (member) => {
                return this.createUnsubscribeUrl(member.uuid, {newsletterUuid});
            }
        },
        {
            id: 'manage_account_url',
            getValue: () => {
                return this.createManageAccountUrl();
            }
        },
        {
            id: 'uuid',
            getValue: (member) => {
                return member.uuid;
            }
        },
        {
            id: 'key',  // HMAC 签名，用于验证链接
            getValue: (member) => {
                return crypto.createHmac('sha256', this.#settingsHelpers.getMembersValidationKey())
                    .update(member.uuid).digest('hex');
            }
        },
        {
            id: 'first_name',
            getValue: (member) => {
                return member.name?.split(' ')[0];
            }
        },
        {
            id: 'name',
            getValue: (member) => {
                return member.name;
            }
        },
        {
            id: 'name_class',  // 用于条件隐藏（name 为空时返回 'hidden'）
            getValue: (member) => {
                return member.name ? '' : 'hidden';
            }
        },
        {
            id: 'email',
            getValue: (member) => {
                return member.email;
            }
        },
        {
            id: 'created_at',
            getValue: (member) => {
                const timezone = this.#settingsCache.get('timezone');
                return member.createdAt ? formatDateLong(member.createdAt, timezone, locale) : '';
            }
        },
        {
            id: 'status',
            getValue: (member) => {
                if (member.status === 'comped') {
                    return t('complimentary');
                }
                if (this.isMemberTrialing(member)) {
                    return t('trialing');
                }
                // 其他可能值：t('free'), t('paid')
                return t(member.status);
            }
        },
        {
            id: 'status_text',  // 完整订阅状态描述（含日期）
            getValue: (member) => {
                return this.getMemberStatusText(member);
            }
        },
        {
            id: 'list_unsubscribe',  // 邮件头 List-Unsubscribe
            getValue: (member) => {
                return this.createUnsubscribeUrl(member.uuid, {newsletterUuid});
            },
            required: true  // 即使模板未使用也强制添加
        },
        {
            id: 'uniqueid',  // 绕过 ESP 图片代理缓存
            getValue: () => {
                return crypto.randomUUID();
            }
        }
    ];
```

### Fallback 机制

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:813-842`

```javascript
const EMAIL_REPLACEMENT_REGEX = /%%\{(.*?)\}%%/g;
const REPLACEMENT_STRING_REGEX = /^(?<recipientProperty>\w+?)(?:,? *(?:"|&quot;)(?<fallback>.*?)(?:"|&quot;))?$/;

const replacements = [];

// 扫描 HTML 中实际使用的变量
let result;
while ((result = EMAIL_REPLACEMENT_REGEX.exec(html)) !== null) {
    const [replacementMatch, replacementStr] = result;
    
    // 去重
    if (replacements.find(r => r.id === replacementStr)) {
        continue;
    }
    
    // 解析：%%{name, "not provided"}%%
    const match = replacementStr.match(REPLACEMENT_STRING_REGEX);
    if (match) {
        const {recipientProperty, fallback} = match.groups;
        // recipientProperty = 'name'
        // fallback = 'not provided'
        
        const definition = baseDefinitions.find(d => d.id === recipientProperty);
        if (definition) {
            replacements.push({
                id: replacementStr,
                originalId: recipientProperty,
                token: new RegExp(...),
                // 有 fallback 时：值为空则返回 fallback
                getValue: fallback ? (member => definition.getValue(member) || fallback) : definition.getValue
            });
        }
    }
}
```

### 订阅状态文本

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:642-709`

```javascript
getMemberStatusText(member) {
    const t = this.#t;      // i18n.t 实例
    const locale = this.#getValidLocale();

    if (member.status === 'free') {
        return t(messages.subscriptionStatus.free);
    }

    if (member.status === 'gift') {
        const expires = member.tiers[0]?.expiry_at ?? null;
        if (expires) {
            const timezone = this.#settingsCache.get('timezone');
            const date = formatDateLong(expires, timezone, locale);
            return t(messages.subscriptionStatus.giftExpires, {date});
        }
        return '';
    }

    // ... 处理 paid、comped 等状态
}
```

### 构建时扫描 vs 发送时绑定

**步骤 1 - 构建时**：`EmailRenderer.buildReplacementDefinitions()` 扫描 HTML，只构建实际使用的变量定义

**步骤 2 - 发送时**：`SendingService.buildRecipients()` 为每个会员计算实际值

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:162`

```javascript
buildRecipients(members, replacementDefinitions) {
    return members.map((member) => {
        return {
            email: member.email?.trim(),
            replacements: replacementDefinitions.map((def) => {
                return {
                    id: def.id,
                    token: def.token,
                    value: def.getValue(member) || ''  // 每个会员独立计算
                };
            })
        };
    }).filter((recipient) => {
        return validator.isEmail(recipient.email, {legacy: false});
    });
}
```

---

## 批量发送与跨邮箱客户端兼容

### 分批策略

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:237`

```javascript
async createBatches({email, post, newsletter}) {
    const segments = await this.#emailRenderer.getSegments(post);
    const BATCH_SIZE = this.#sendingService.getMaximumRecipients();  // 1000
    
    for (const segment of segments) {
        let lastId = email.id;  // ObjectId 用于游标分页
        
        while (!members || lastId) {
            // 按 id < email.id 确保只包含邮件创建时已存在的会员
            const filter = segmentFilter + `+id:<'${lastId}'`;
            members = await this.#models.Member.getFilteredCollectionQuery({filter})
                .orderByRaw('id DESC')
                .select('members.id', 'members.uuid', 'members.email', 'members.name')
                .limit(BATCH_SIZE + 1);
            
            if (members.length > 0) {
                // 创建 EmailBatch + EmailRecipient 记录
                await this.createBatch(email, segment, members.slice(0, BATCH_SIZE), options);
            }
            
            lastId = members.length > BATCH_SIZE ? members[BATCH_SIZE - 1].id : null;
        }
    }
}
```

### 分段渲染（免费/付费会员不同内容）

**文件**：`ghost/core/core/server/services/email-renderer.js:335`

```javascript
async getSegments(post) {
    const allowedSegments = ['status:free', 'status:-free'];
    const html = await this.renderPostBaseHtml(post);
    
    // 1. 有 paywall card → 必须分两段
    if (html.indexOf('<!--members-only-->') !== -1) {
        return allowedSegments;
    }
    
    // 2. 检查 data-gh-segment 属性
    const $ = cheerioLoad(html);
    const segments = $('[data-gh-segment]')
        .get()
        .map(el => el.attribs['data-gh-segment']);
    
    return [...new Set(segments)].filter(segment => allowedSegments.includes(segment));
}
```

### 应用分段过滤

**文件**：`ghost/core/core/server/services/email-renderer.js:393`

```javascript
async renderBody(post, newsletter, segment, options) {
    let html = await this.renderPostBaseHtml(post, newsletter);
    
    // 移除不属于当前分段的内容
    $('[data-gh-segment]').get().forEach((node) => {
        if (node.attribs['data-gh-segment'] !== segment) {
            $(node).remove();
        } else {
            $(node).removeAttr('data-gh-segment');
        }
    });
    
    // 付费文章免费会员截断 + 添加 paywall
    const isPaidPost = post.get('visibility') === 'paid' || post.get('visibility') === 'tiers';
    const membersOnlyIndex = html.indexOf('<!--members-only-->');
    if (isPaidPost && membersOnlyIndex !== -1 && segment === 'status:free') {
        html = html.slice(0, membersOnlyIndex);  // 截断
        addPaywall = true;
    }
}
```

### 并发控制与重试

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:466`

```javascript
const MAX_SENDING_CONCURRENCY = 2;  // 最多 2 个批次并发发送

// 数据库操作重试（发送前）
#BEFORE_RETRY_CONFIG = {maxRetries: 10, maxTime: 10 * 60 * 1000, sleep: 2000};

// 数据库操作重试（发送后）
#AFTER_RETRY_CONFIG = {maxRetries: 20, maxTime: 30 * 60 * 1000, sleep: 2000};

// Mailgun API 重试
#MAILGUN_API_RETRY_CONFIG = {sleep: 10 * 1000, maxRetries: 6};
```

### 缓存渲染结果

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:119`

```javascript
const cacheId = emailId + '-' + (segment ?? 'null');

if (options.emailBodyCache) {
    emailBody = options.emailBodyCache.get(cacheId);
}

if (!emailBody) {
    emailBody = await this.#emailRenderer.renderBody(post, newsletter, segment, options);
    if (options.emailBodyCache) {
        options.emailBodyCache.set(cacheId, emailBody);
    }
}
```

### 跨邮箱客户端兼容性处理

**核心处理流程**：`ghost/core/core/server/services/email-renderer.js:506-579`

#### 1. Juice 内联 CSS

```javascript
const juice = require('juice');
html = juice(html, {inlinePseudoElements: true, removeStyleTags: true});
```

将 `<style>` 中的样式内联到 `style` 属性，因为 Gmail、Outlook 等客户端会移除 `<style>` 标签。

#### 2. 图片尺寸修复（Outlook 不支持 width: auto）

```javascript
// 记录原始 width/height 属性
const originalImageSizes = $('img').get().map((image) => {
    return {src: image.attribs.src, width: image.attribs.width, height: image.attribs.height};
});

// Juice 内联后可能把 width/height 设为 'auto'
// Outlook 不支持，需要恢复原始值
for (let i = 0; i < imageTags.length; i += 1) {
    if (imageTags[i].attribs.width === 'auto' && originalImageSizes[i].width) {
        imageTags[i].attribs.width = originalImageSizes[i].width;
    }
}
```

#### 3. 语义化标签转 `<div>`

```javascript
// Outlook、Yahoo 不支持 <figure> 和 <figcaption>
$('figure, figcaption').each((i, elem) => !!(elem.tagName = 'div'));
```

#### 4. 特殊字符转义（Outlook 兼容）

```javascript
html = html.replace(/&apos;/g, '&#39;');     // 单引号
html = html.replace(/→/g, '&rarr;');          // 右箭头
html = html.replace(/–/g, '&ndash;');         // 短破折号
html = html.replace(/“/g, '&ldquo;');         // 左双引号
html = html.replace(/”/g, '&rdquo;');         // 右双引号
```

#### 5. Outlook 条件注释

**模板**：`ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs`

```handlebars
<!--[if mso]>
<xml>
  <o:OfficeDocumentSettings>
    <o:PixelsPerInch>96</o:PixelsPerInch>
    <o:AllowPNG/>
  </o:OfficeDocumentSettings>
</xml>
<![endif]-->

<!-- Outlook 不尊重 max-width，需要额外的居中 table -->
<!--[if mso]>
<tr>
  <td>
    <center>
      <table border="0" cellpadding="0" cellspacing="0" width="600">
<![endif]-->
```

#### 6. 暗色/亮色模式图片切换

```javascript
// 根据背景色移除不匹配的图片
if (templateData.backgroundIsDark) {
    $('img.is-light-background').each((i, elem) => $(elem).remove());
} else {
    $('img.is-dark-background').each((i, elem) => $(elem).remove());
}
```

#### 7. 响应式样式

**样式文件**：`ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs`

```css
@media only screen and (max-width: 620px) {
    table.body p, table.body ul, table.body ol, table.body td {
        font-size: 16px;
    }
    table.header .post-title a {
        font-size: 26px !important;
        line-height: 1.1 !important;
    }
    .hide-mobile { display: none; }
    .mobile-only { display: initial !important; }
}
```

---

## 多语言兜底机制

### 完整 locale 处理链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                    settings_cache 表（数据库）                       │
│  key: 'locale', value: 'en'  (用户配置，简写)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  email-service  │    │   EmailRenderer │    │   日期格式化     │
    │  -wrapper.js    │    │                 │    │                 │
    │                 │    │ #getValidLocale │    │ formatDateLong() │
    │ i18nLib('en')   │    │ 'en' → 'en-gb'  │    │ Luxon setLocale │
    │ (i18n 实例)     │    │                 │    │                 │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### i18n 实例初始化

**文件**：`ghost/core/core/server/services/email-service/email-service-wrapper.js:62-68`

```javascript
const i18nLanguage = settingsCache.get('locale') || 'en';
const i18n = i18nLib(i18nLanguage, 'ghost');  // 使用 @tryghost/i18n

// 监听 locale 变化
events.on('settings.locale.edited', (model) => {
    debug('locale changed, updating i18n to', model.get('value'));
    i18n.changeLanguage(model.get('value'));
});

// 注入 EmailRenderer
const emailRenderer = new EmailRenderer({
    // ...
    t: i18n.t,           // 真实的 i18n.t 函数
    dir: i18n.dir.bind(i18n)  // i18n.dir 函数
});
```

### 占位函数 vs 真实翻译函数

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:26-34`

```javascript
/**
 * Wrapper function so that i18next-parser can find these strings.
 * （仅用于静态分析提取翻译 key，不是实际运行时使用的函数）
 *
 * @template T
 * @param {T} x
 * @returns {T}
 */
const t = (x) => {
    return x;
};

const messages = {
    subscriptionStatus: {
        free: '',
        expired: t('Your subscription has expired.'),
        canceled: t('Your subscription has been canceled and will expire on {date}...'),
        active: t('Your subscription will renew on {date}.'),
        trial: t('Your free trial ends on {date}...'),
        // ...
    }
};
```

**关键区别**：

| 位置 | 函数 | 用途 |
|------|------|------|
| 文件顶部 `const t = (x) => x` | 占位函数 | i18next-parser 静态分析提取 key |
| 构造函数注入的 `this.#t` | `i18n.t` | 运行时实际执行翻译 |

### locale 验证与降级

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:66-74, 271-284`

```javascript
const DEFAULT_LOCALE = 'en-gb';

function isValidLocale(locale) {
    try {
        // 使用 Intl.DateTimeFormat 验证
        new Intl.DateTimeFormat(locale);
        return true;
    } catch (e) {
        return false;
    }
}

#getValidLocale() {
    let locale = this.#settingsCache.get('locale') || DEFAULT_LOCALE;
    // 可能得到 'en'、'zh' 等简写
    
    locale = locale.trim();
    
    // 'en' 简写不被 Intl 完全支持，或 locale 无效 → 降级到 'en-gb'
    if (locale === 'en' || !isValidLocale(locale)) {
        locale = DEFAULT_LOCALE;
    }
    
    return locale;
}
```

### 日期本地化

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:82-88`

```javascript
function formatDateLong(date, timezone, locale = DEFAULT_LOCALE) {
    return DateTime.fromJSDate(date)
        .setZone(timezone)           // 先设置时区
        .setLocale(locale)           // 再设置 locale
        .toLocaleString({            // 本地化输出
            year: 'numeric',
            month: 'long',
            day: 'numeric'
        });
}
```

**使用示例**：`email-renderer.js:1031-1036`

```javascript
const timezone = this.#settingsCache.get('timezone');
const locale = this.#getValidLocale();
const publishedAt = (post.get('published_at') ? DateTime.fromJSDate(post.get('published_at')) : DateTime.local())
    .setZone(timezone)
    .setLocale(locale)
    .toLocaleString({
        year: 'numeric',
        month: 'short',
        day: 'numeric'
    });
```

### RTL 方向支持

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:1124, 201, 238`

```javascript
// 构造函数注入
/**
 * @param {(locale: string) => 'rtl' | 'ltr'} dependencies.dir 
 *        Returns 'rtl' or 'ltr' for a given locale (i18next's `i18n.dir`)
 */
constructor({..., t, dir}) {
    this.#t = t;
    this.#dir = dir;
}

// 使用
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const locale = this.#getValidLocale();
    const direction = this.#dir(locale);  // 调用 i18n.dir()
    
    const data = {
        site: {
            locale,
            direction  // 'rtl' 或 'ltr'
        },
        // ...
    };
}
```

**模板应用**：`email-rendering/partials/email-wrapper.hbs`

```handlebars
<html lang="{{site.locale}}" dir="{{site.direction}}">
```

**样式中的 RTL 调整**：`email-templates/partials/styles.hbs`

```css
.manage-subscription {
    text-align: {{#if (eq site.direction "rtl")}}left{{else}}right{{/if}};
}
```

### 翻译文本实际使用

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:642-709`

```javascript
getMemberStatusText(member) {
    const t = this.#t;      // 真实的 i18n.t
    const locale = this.#getValidLocale();

    if (member.status === 'gift') {
        const expires = member.tiers[0]?.expiry_at ?? null;
        if (expires) {
            const timezone = this.#settingsCache.get('timezone');
            const date = formatDateLong(expires, timezone, locale);
            return t(messages.subscriptionStatus.giftExpires, {date});
            // 等价于: t('Your subscription will expire on {date}.', {date})
        }
        return '';
    }

    if (member.status === 'paid') {
        // ...
        if (activeSubscription.trial_end_at && activeSubscription.trial_end_at > new Date()) {
            const date = formatDateLong(activeSubscription.trial_end_at, timezone, locale);
            return t(messages.subscriptionStatus.trial, {date});
        }

        const date = formatDateLong(activeSubscription.current_period_end, timezone, locale);
        if (activeSubscription.cancel_at_period_end) {
            return t(messages.subscriptionStatus.canceled, {date});
        }
        return t(messages.subscriptionStatus.active, {date});
    }
}
```

### 状态变量翻译

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:776-786`

```javascript
{
    id: 'status',
    getValue: (member) => {
        if (member.status === 'comped') {
            return t('complimentary');
        }
        if (this.isMemberTrialing(member)) {
            return t('trialing');
        }
        // other possible statuses: t('free'), t('paid')
        return t(member.status);
    }
}
```

### 多语言兜底总结

| 场景 | 处理方式 | 示例 |
|------|----------|------|
| settings 存 `'en'` 简写 | `#getValidLocale()` 降级 | `'en'` → `'en-gb'` |
| locale 无效 | `#getValidLocale()` 降级 | `'invalid'` → `'en-gb'` |
| 文本翻译 | `i18n.t()` 执行 | `t('complimentary')` |
| 日期格式 | Luxon `setLocale()` | `setLocale('zh-cn')` |
| 文本方向 | `i18n.dir()` | `'ar'` → `'rtl'` |

---

## 关键文件索引

### 核心服务

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-service/email-service-wrapper.js` | 依赖注入、i18n 初始化 |
| `ghost/core/core/server/services/email-service/email-renderer.js` | 邮件渲染核心（模板、变量、兼容处理） |
| `ghost/core/core/server/services/email-service/sending-service.js` | 发送协调（缓存、收件人构建） |
| `ghost/core/core/server/services/email-service/batch-sending-service.js` | 批量调度（分批、并发、重试） |

### 设计计算

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-rendering/email-design.js` | 后端 HEX 颜色计算（不处理语义值） |
| `ghost/core/core/server/models/newsletter.js` | Newsletter 模型 + 语义值默认值 |
| `apps/admin-x-settings/src/components/settings/email/newsletters/newsletter-preview.tsx` | 管理端预览（语义值 → HEX 转换） |
| `apps/admin-x-settings/src/components/settings/email-design/design-utils.ts` | 管理端设计工具函数（语义值 → HEX 转换） |

### 模板文件

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs` | HTML 骨架 + Outlook 条件注释 |
| `ghost/core/core/server/services/email-service/email-templates/template.hbs` | 内容区域 + 页脚 |
| `ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs` | 动态样式注入 |

### 测试文件

| 文件 | 职责 |
|------|------|
| `ghost/core/test/unit/server/services/email-rendering/email-design.test.js` | 验证语义值被视为无效 HEX |
| `ghost/core/test/unit/server/services/email-service/batch-sending-service.test.js` | 批量发送单元测试 |
| `ghost/core/test/integration/services/email-service/batch-sending.test.js` | 批量发送集成测试 |

---

## 关键偏差修正说明

### 偏差 1：语义值 vs HEX 颜色

**原描述**：Newsletter 模型默认值 `background_color: 'light'` 直接传递给 `getEmailDesign()`

**修正**：
- Newsletter 模型存储语义值 `'light'`、`'transparent'`、`'accent'`
- `email-design.js` 的 `getHexColor()` 只接受 `#xxx` 格式，`'light'` 被视为无效值
- 实际转换：管理端预览组件独立实现语义值 → HEX 的映射
- 邮件渲染端：`'light'` 因验证失败回退到 `#ffffff`

**证据**：
- `email-design.test.js` 中 `'light'` 被列在 `INVALID_HEX_COLORS`
- `newsletter.js:30` 定义 `background_color: 'light'`
- `email-design.js:76` 调用 `getHexColor(settings.backgroundColor, '#ffffff')`

### 偏差 2：`#t` 函数来源

**原描述**：`#t` 是文件顶部的占位函数

**修正**：
- 文件顶部的 `const t = (x) => x` 是占位函数，仅用于 i18next-parser 静态分析提取翻译 key
- 实际运行时使用的是构造函数注入的 `this.#t = i18n.t`（`@tryghost/i18n` 实例）

**证据**：
- `email-service-wrapper.js:94` 注入 `t: i18n.t`
- `email-renderer.js:237` 赋值 `this.#t = t`
- `email-renderer.js:32` 注释明确说明是 wrapper for i18next-parser

### 偏差 3：版式共享机制

**原描述**：前后端共享同一 `getEmailDesign` 函数

**修正**：
- 前后端不共享计算函数
- 后端：`email-design.js` 的 `getEmailDesign()` 只处理 HEX 颜色
- 前端：`newsletter-preview.tsx` 独立实现语义值 → HEX 转换
- 共享的是**规则**（`'accent'` → 站点主色、`null` → 背景对比色），不是代码

**证据**：
- 后端 `getEmailDesign()` 不导入任何前端文件
- 前端 `newsletter-preview.tsx` 自己实现 `backgroundColor()`、`buttonColor()` 等函数
- 单元测试验证语义值在后端被视为无效

### 偏差 4：locale 处理

**原描述**：settingsCache 直接存完整 locale

**修正**：
- settingsCache 可能存 `'en'` 简写（用户配置）
- `#getValidLocale()` 验证后降级到 `'en-gb'`
- 两层 locale：settings 层（用户输入）和渲染层（验证后）

**证据**：
- `email-service-wrapper.js:62` 使用 `settingsCache.get('locale') || 'en'`
- `email-renderer.js:278-280` 检查 `if (locale === 'en' || !isValidLocale(locale))`
