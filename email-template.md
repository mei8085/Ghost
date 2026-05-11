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
│  ghost/core/core/server/services/email-service/              │
│  batch-sending-service.js                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      SendingService                          │
│  (发送协调：缓存渲染结果、构建收件人列表、调用 Provider)     │
│  ghost/core/core/server/services/email-service/              │
│  sending-service.js                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      EmailRenderer                           │
│  (核心渲染：模板解析、变量替换、CSS 内联、客户端兼容处理)     │
│  ghost/core/core/server/services/email-service/              │
│  email-renderer.js                                           │
└─────────────────────────────────────────────────────────────┘
```

### 依赖注入初始化

**文件**：`ghost/core/core/server/services/email-service/email-service-wrapper.js:13-110`

```javascript
init() {
    // i18n 初始化
    const i18nLanguage = settingsCache.get('locale') || 'en';  // email-service-wrapper.js:62
    const i18n = i18nLib(i18nLanguage, 'ghost');              // email-service-wrapper.js:63

    // 监听 locale 变化
    events.on('settings.locale.edited', (model) => {
        i18n.changeLanguage(model.get('value'));             // email-service-wrapper.js:69
    });

    // EmailRenderer 初始化（包含 i18n 注入）
    const emailRenderer = new EmailRenderer({
        settingsCache,
        // ...
        t: i18n.t,                                           // email-service-wrapper.js:94
        dir: i18n.dir.bind(i18n)                              // email-service-wrapper.js:95
    });

    // SendingService 初始化
    const sendingService = new SendingService({
        emailProvider: mailgunEmailProvider,                   // email-service-wrapper.js:105
        emailRenderer,
        emailAddressService
    });

    // BatchSendingService 初始化
    const batchSendingService = new BatchSendingService({
        emailRenderer,
        sendingService,
        jobsService,
        emailSegmenter,
        domainWarmingService,
        models,
        db,
        sentry
    });
}
```

### 渲染流程总览

| 步骤 | 函数 | 文件 | 行号 |
|------|------|------|------|
| 1. 触发发送 | `BatchSendingService.emailJob()` | `batch-sending-service.js` | 133 |
| 2. 创建批次 | `BatchSendingService.createBatches()` | `batch-sending-service.js` | 237 |
| 3. 渲染邮件体 | `EmailRenderer.renderBody()` | `email-renderer.js` | 393 |
| 4. 构建收件人 | `SendingService.buildRecipients()` | `sending-service.js` | 162 |
| 5. 发送 | `SendingService.send()` → Provider | `sending-service.js` | 110 |

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
        background_color: 'light',           // newsletter.js:30 - 语义值
        header_background_color: 'transparent',  // newsletter.js:32 - 语义值
        button_color: 'accent',              // newsletter.js:28 - 语义值
        link_color: 'accent',                // newsletter.js:29 - 语义值
        
        // 枚举值
        button_corners: 'rounded',           // newsletter.js:36 - square | rounded | pill
        button_style: 'fill',                // newsletter.js:34 - fill | outline
        title_font_weight: 'bold',           // newsletter.js:22 - normal | medium | semibold | bold
        link_style: 'underline',             // newsletter.js:31 - underline | plain
        image_corners: 'square',             // newsletter.js:37 - square | rounded
        
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

const DEFAULT_ACCENT_COLOR = '#15212A';     // email-design.js:29
const DEFAULT_DIVIDER_COLOR = '#e0e7eb';    // email-design.js:30
const VALID_HEX_REGEX = /^#([0-9a-f]{3}){1,2}$/i;  // email-design.js:27

const getHexColor = (value, fallback) => (isValidHexColor(value) ? value : fallback);
// isValidHexColor 只接受 #xxx 格式，'light'、'transparent' 都返回 false

exports.getEmailDesign = (settings) => {
    // accentColor: 无效值回退到 DEFAULT_ACCENT_COLOR (#15212A)
    const accentColor = getHexColor(settings.accentColor, DEFAULT_ACCENT_COLOR);  // email-design.js:76
    
    // backgroundColor: 无效值回退到 #ffffff（'light' 被视为无效）
    const backgroundColor = getHexColor(settings.backgroundColor, '#ffffff');      // email-design.js:79
    
    // buttonColor: 'accent' 被特殊处理，其他无效值回退到 accentColor
    let buttonColor;
    switch (settings.buttonColor) {
    case 'accent':
        buttonColor = accentColor;                                    // email-design.js:84
        break;
    case null:
        buttonColor = textColorForBackgroundColor(backgroundColor).hex();  // email-design.js:87
        break;
    default:
        buttonColor = getHexColor(settings.buttonColor, accentColor);     // email-design.js:91
        break;
    }
    
    // dividerColor: 'accent' 被特殊处理
    let dividerColor;
    if (settings.dividerColor === 'accent') {
        dividerColor = accentColor;                                 // email-design.js:96
    } else {
        dividerColor = getHexColor(settings.dividerColor, DEFAULT_DIVIDER_COLOR);  // email-design.js:99
    }
    
    // headerBackgroundColor: 'accent' 被特殊处理，其他无效值 → null
    let headerBackgroundColor;
    if (settings.headerBackgroundColor === 'accent') {
        headerBackgroundColor = accentColor;                        // email-design.js:104
    } else if (isValidHexColor(settings.headerBackgroundColor)) {
        headerBackgroundColor = settings.headerBackgroundColor;     // email-design.js:107
    } else {
        headerBackgroundColor = null;  // 'transparent' → null      // email-design.js:110
    }
    
    // linkColor: 'accent' 被特殊处理
    let linkColor;
    switch (settings.linkColor) {
    case 'accent':
        linkColor = accentColor;                                    // email-design.js:116
        break;
    case null:
        linkColor = textColorForBackgroundColor(backgroundColor).hex();  // email-design.js:119
        break;
    default:
        linkColor = getHexColor(settings.linkColor, accentColor);       // email-design.js:123
        break;
    }
    
    // titleWeight: 枚举值映射
    let titleWeight;
    switch (settings.titleFontWeight) {
    case 'normal': titleWeight = 400; break;
    case 'medium': titleWeight = 500; break;
    case 'semibold': titleWeight = 600; break;
    default: titleWeight = 700; break;                                 // 'bold'
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
    'accent',       // email-design.test.js:18 - 被视为无效（除非特殊处理）
    'light',        // email-design.test.js:19 - 被视为无效！
    'dark',         // email-design.test.js:20 - 被视为无效！
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

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:1025-1058`

```javascript
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const emailDesign = this.#getEmailDesign(newsletter);              // email-renderer.js:1026
    // ...
    const data = {
        // ...
        // CSS 变量：展开 emailDesign
        ...emailDesign,                                               // email-renderer.js:1047
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

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:813-860`

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

    if (member.status === 'paid') {
        let activeSubscription = member.subscriptions.find((subscription) => {
            return subscription.status === 'trialing' || subscription.status === 'active';
        });
        if (!activeSubscription) {
            return '';
        }
        
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

    if (member.status === 'comped') {
        const expires = member.tiers[0]?.expiry_at ?? null;
        if (expires) {
            const timezone = this.#settingsCache.get('timezone');
            const date = formatDateLong(expires, timezone, locale);
            return t(messages.subscriptionStatus.complimentaryExpires, {date});
        }
        return t(messages.subscriptionStatus.complimentaryInfinite);
    }

    return '';
}
```

### 构建时扫描 vs 发送时绑定

**步骤 1 - 构建时**：`EmailRenderer.buildReplacementDefinitions()` 扫描 HTML，只构建实际使用的变量定义

**步骤 2 - 发送时**：`SendingService.buildRecipients()` 为每个会员计算实际值

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:162-182`

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
        // Remove invalid recipient email addresses
        const isValidRecipient = validator.isEmail(recipient.email, {legacy: false});
        if (!isValidRecipient) {
            logging.warn(`Removed recipient ${recipient.email} from list because it is not a valid email address`);
        }
        return isValidRecipient;
    });
}
```

---

## 批量发送与跨邮箱客户端兼容

### 分批策略

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:237-340`

```javascript
async createBatches({email, post, newsletter}) {
    logging.info(`Creating batches for email ${email.id}`);

    // 域名预热限制
    let domainWarmupLimit = Infinity;
    if (this.#domainWarmingService.isEnabled()) {
        domainWarmupLimit = Number.isInteger(email.get('csd_email_count')) ? email.get('csd_email_count') : Infinity;
    }

    const segments = await this.#emailRenderer.getSegments(post);
    const batches = [];
    const BATCH_SIZE = this.#sendingService.getMaximumRecipients();  // batch-sending-service.js:248
    let totalCount = 0;

    for (const segment of segments) {
        const segmentFilter = this.#emailSegmenter.getMemberFilterForSegment(
            newsletter, 
            email.get('recipient_filter'), 
            segment
        );  // batch-sending-service.js:254

        // 使用 ObjectId 游标分页，确保只包含邮件创建时已存在的会员
        // 注：使用 id 而非 created_at，因为导入会员可能设置 created_at 为过去/未来值
        let lastId = email.id;  // batch-sending-service.js:261

        while (!members || lastId) {
            // 按 id < email.id 过滤
            const filter = segmentFilter + `+id:<'${lastId}'`;  // batch-sending-service.js:266

            members = await this.#models.Member.getFilteredCollectionQuery({filter})
                .orderByRaw('id DESC')
                .select('members.id', 'members.uuid', 'members.email', 'members.name')
                .limit(BATCH_SIZE + 1);  // batch-sending-service.js:269-271

            if (members.length > 0) {
                const remainingCustomDomainCapacity = domainWarmupLimit - totalCount;
                const membersToProcess = Math.min(members.length, BATCH_SIZE);

                const shouldSplitBatch = remainingCustomDomainCapacity > 0 && remainingCustomDomainCapacity < membersToProcess;
                if (shouldSplitBatch) {
                    // 拆分批次：部分走自定义域名，部分走 fallback
                    totalCount += await this.#createBatchWithRetry({
                        email,
                        segment,
                        members: members.slice(0, remainingCustomDomainCapacity),
                        useFallbackDomain: false,
                        batches
                    });
                    totalCount += await this.#createBatchWithRetry({
                        email,
                        segment,
                        members: members.slice(remainingCustomDomainCapacity, membersToProcess),
                        useFallbackDomain: true,
                        batches
                    });
                } else {
                    // 单一批次
                    totalCount += await this.#createBatchWithRetry({
                        email,
                        segment,
                        members: members.slice(0, membersToProcess),
                        useFallbackDomain: totalCount >= domainWarmupLimit,
                        batches
                    });
                }
            }

            // 更新游标：如果有 BATCH_SIZE + 1 条，说明还有更多
            if (members.length > BATCH_SIZE) {
                lastId = members[members.length - 2].id;  // batch-sending-service.js:308
            } else {
                break;
            }
        }
    }

    // 校验并更新 email_count
    if (email.get('email_count') !== totalCount) {
        await email.save({
            email_count: totalCount,
            ...(this.#domainWarmingService.isEnabled() 
                ? {csd_email_count: Math.min(totalCount, domainWarmupLimit)} 
                : {})
        }, {patch: true, require: false, autoRefresh: false});
    }

    return batches;
}
```

### 创建 EmailBatch + EmailRecipient

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:382-426`

```javascript
async createBatch(email, segment, members, options) {
    if (!options || !options.transacting) {
        return this.#models.EmailBatch.transaction(async (transacting) => {
            return this.createBatch(email, segment, members, {transacting, ...options});
        });
    }

    const batch = await this.#models.EmailBatch.add({
        email_id: email.id,
        member_segment: segment,
        status: 'pending',
        fallback_sending_domain: Boolean(options.useFallbackDomain)
    }, options);  // batch-sending-service.js:391-396

    // 构建 EmailRecipient 数据
    const recipientData = [];
    members.forEach((memberRow) => {
        if (!memberRow.id || !memberRow.uuid || !memberRow.email) {
            logging.warn(`Member row not included as email recipient due to missing data`);
            return;
        }

        recipientData.push({
            id: ObjectID().toHexString(),
            email_id: email.id,
            member_id: memberRow.id,
            batch_id: batch.id,
            member_uuid: memberRow.uuid,
            member_email: memberRow.email,
            member_name: memberRow.name
        });  // batch-sending-service.js:406-414
    });

    // 批量插入
    const insertQuery = this.#db.knex('email_recipients').insert(recipientData);
    if (options.transacting) {
        insertQuery.transacting(options.transacting);
    }
    await insertQuery;  // batch-sending-service.js:417-424

    return batch;
}
```

### 分段渲染（免费/付费会员不同内容）

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:335-361`

```javascript
async getSegments(post) {
    const allowedSegments = ['status:free', 'status:-free'];
    const html = await this.renderPostBaseHtml(post);

    // 1. 有 paywall card (<!--members-only-->) → 必须分两段
    if (html.indexOf('<!--members-only-->') !== -1) {
        // 免费和付费会员内容不同
        return allowedSegments;  // email-renderer.js:344
    }

    const $ = cheerioLoad(html);

    // 2. 检查 data-gh-segment 属性
    let allSegments = $('[data-gh-segment]')
        .get()
        .map(el => el.attribs['data-gh-segment']);

    const segments = [...new Set(allSegments)].filter(segment => allowedSegments.includes(segment));
    if (segments.length === 0) {
        // 无差异 → 单一段 [null]
        return [null];  // email-renderer.js:356
    }

    // 有差异 → 分两段
    return allowedSegments;
}
```

### 应用分段过滤

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:393-585`

```javascript
async renderBody(post, newsletter, segment, options) {
    let html = await this.renderPostBaseHtml(post, newsletter);

    // Paywall 和会员专属内容处理
    const isPaidPost = post.get('visibility') === 'paid' || post.get('visibility') === 'tiers';
    const membersOnlyIndex = html.indexOf('<!--members-only-->');
    const hasMembersOnlyContent = membersOnlyIndex !== -1;
    let addPaywall = false;

    if (isPaidPost && hasMembersOnlyContent) {
        if (segment === 'status:free') {
            // 免费会员：截断内容 + 添加 paywall
            addPaywall = true;
            html = html.slice(0, membersOnlyIndex);  // 截断  // email-renderer.js:408
        }
    }

    let $ = cheerioLoad(html);

    // 移除不属于当前分段的内容（在模板渲染前执行，因为 preheader 可能用到）
    $('[data-gh-segment]').get().forEach((node) => {
        if (node.attribs['data-gh-segment'] !== segment) {
            $(node).remove();  // email-renderer.js:421
        } else {
            $(node).removeAttr('data-gh-segment');  // 清理属性
        }
    });  // email-renderer.js:418-426

    html = $.html();

    const templateData = await this.getTemplateData({
        post,
        newsletter,
        html,
        addPaywall,
        segment
    });
    html = await this.renderTemplate(templateData);

    // ... 后续：链接跟踪、Juice、兼容处理等
}
```

### 并发控制与重试

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:10, 37-39, 466-478, 675-728`

```javascript
const MAX_SENDING_CONCURRENCY = 2;  // 最多 2 个批次并发发送  // batch-sending-service.js:10

// 重试配置
#BEFORE_RETRY_CONFIG = {maxRetries: 10, maxTime: 10 * 60 * 1000, sleep: 2000};     // batch-sending-service.js:37
#AFTER_RETRY_CONFIG = {maxRetries: 20, maxTime: 30 * 60 * 1000, sleep: 2000};       // batch-sending-service.js:38
#MAILGUN_API_RETRY_CONFIG = {sleep: 10 * 1000, maxRetries: 6};                        // batch-sending-service.js:39

// 并发控制实现
async sendBatches({email, batches, post, newsletter}) {
    const queue = batches.slice();

    // 递归 worker
    let runNext;
    runNext = async () => {
        const batch = queue.shift();
        if (batch) {
            if (await this.sendBatch({...})) {
                succeededCount += 1;
            }
            await runNext();  // 递归调用
        }
    };

    // 启动 MAX_SENDING_CONCURRENCY 个并发 worker
    await Promise.all(
        new Array(MAX_SENDING_CONCURRENCY).fill(0).map(() => runNext())
    );  // batch-sending-service.js:467

    if (succeededCount < batches.length) {
        throw new errors.EmailError({...});
    }
}

// 重试实现
async retryDb(func, options) {
    const retryCount = (options.retryCount ?? 0);

    try {
        const response = await func();
        return response;
    } catch (e) {
        const sleep = (options.sleep ?? 0);
        
        // 超过最大重试次数或截止时间 → 抛出
        if (retryCount >= options.maxRetries 
            || (options.stopAfterDate && (new Date(Date.now() + sleep)) > options.stopAfterDate)) {
            throw e;
        }

        // 等待 → 指数退避（sleep * 2）
        if (sleep) {
            await new Promise((resolve) => setTimeout(resolve, sleep));
        }
        return await this.retryDb(
            func, 
            {...options, retryCount: retryCount + 1, sleep: sleep * 2}  // 指数退避
        );  // batch-sending-service.js:726
    }
}
```

### 缓存渲染结果

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:110-154`

```javascript
async send({post, newsletter, segment, members, emailId}, options) {
    const cacheId = emailId + '-' + (segment ?? 'null');  // sending-service.js:111
    const isTestEmail = options.isTestEmail ?? false;

    let emailBody;

    // 尝试从缓存获取
    if (options.emailBodyCache) {
        emailBody = options.emailBodyCache.get(cacheId);  // sending-service.js:120
    }

    // 缓存未命中 → 渲染
    if (!emailBody) {
        emailBody = await this.#emailRenderer.renderBody(
            post,
            newsletter,
            segment,
            {
                clickTrackingEnabled: !!options.clickTrackingEnabled
            }
        );  // sending-service.js:124-131
        
        // 写入缓存
        if (options.emailBodyCache) {
            options.emailBodyCache.set(cacheId, emailBody);  // sending-service.js:133
        }
    }

    // 构建收件人 → 调用 Provider
    const recipients = this.buildRecipients(members, emailBody.replacements);
    return await this.#emailProvider.send({
        subject: this.#emailRenderer.getSubject(post, isTestEmail),
        from: this.#emailRenderer.getFromAddress(post, newsletter, !!options.useFallbackAddress),
        replyTo: this.#emailRenderer.getReplyToAddress(post, newsletter, !!options.useFallbackAddress) ?? undefined,
        html: emailBody.html,
        plaintext: emailBody.plaintext,
        recipients,
        emailId: emailId,
        replacementDefinitions: emailBody.replacements,
        domainOverride: options.useFallbackAddress ? this.#emailAddressService.fallbackDomain : undefined
    }, {...});
}
```

### 跨邮箱客户端兼容性处理

**核心处理流程**：`ghost/core/core/server/services/email-renderer.js:506-579`

#### 1. Juice 内联 CSS

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:523-525`

```javascript
// Juice HTML (inline CSS)
const juice = require('juice');
html = juice(html, {inlinePseudoElements: true, removeStyleTags: true});
```

**目的**：将 `<style>` 中的样式内联到 `style` 属性，因为 Gmail、Outlook 等客户端会移除 `<style>` 标签。

#### 2. 图片尺寸修复（Outlook 不支持 width: auto）

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:506-517, 530-543`

```javascript
// 在 Juice 内联前记录原始 width/height 属性
// 如果 CSS 设置了 width: auto 或 height: auto，Juice 会显式设置到属性上
// 这是 Outlook 不支持的，需要恢复原始值
const originalImageSizes = $('img').get().map((image) => {
    const src = image.attribs.src;
    const width = image.attribs.width;
    const height = image.attribs.height;
    return {src, width, height};
});  // email-renderer.js:512-517

// ... Juice 内联后 ...

// 重置任何 height="auto" 或 width="auto" 到原始值
const imageTags = $('img').get();
for (let i = 0; i < imageTags.length; i += 1) {
    if (imageTags[i].attribs.src === originalImageSizes[i].src) {
        if (imageTags[i].attribs.width === 'auto' && originalImageSizes[i].width) {
            imageTags[i].attribs.width = originalImageSizes[i].width;
        }
        if (imageTags[i].attribs.height === 'auto' && originalImageSizes[i].height) {
            imageTags[i].attribs.height = originalImageSizes[i].height;
        }
    }
}  // email-renderer.js:531-543
```

#### 3. 强制所有链接在新窗口打开

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:545-546`

```javascript
$('a').attr('target', '_blank');
```

#### 4. 语义化标签转 `<div>`

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:548-549`

```javascript
// convert figure and figcaption to div so that Outlook applies margins
$('figure, figcaption').each((i, elem) => !!(elem.tagName = 'div'));
```

**原因**：Outlook、Yahoo 等客户端对 `<figure>` 和 `<figcaption>` 的 margin 支持有问题。

#### 5. 暗色/亮色模式图片切换

**文件**：`ghost/core/core/server/services/email-renderer.js:551-560`

```javascript
// Remove duplicate black/white images (CSS based solution not working in Outlook)
if (templateData.backgroundIsDark) {
    $('img.is-light-background').each((i, elem) => {
        $(elem).remove();
    });
} else {
    $('img.is-dark-background').each((i, elem) => {
        $(elem).remove();
    });
}
```

**原因**：CSS 方案在 Outlook 中不生效，必须在 HTML 层面移除。

#### 6. 特殊字符转义（Outlook 兼容）

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:573-578`

```javascript
// Fix any unsupported chars in Outlook
html = html.replace(/&apos;/g, '&#39;');     // 单引号
html = html.replace(/→/g, '&rarr;');          // 右箭头
html = html.replace(/–/g, '&ndash;');         // 短破折号
html = html.replace(/“/g, '&ldquo;');         // 左双引号
html = html.replace(/”/g, '&rdquo;');         // 右双引号
```

#### 7. Outlook 条件注释

**文件**：`ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs`

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

<!-- 实际内容 -->
<tr>
  <td class="wrapper" ...>
    <table class="body" align="center" style="max-width:600px;">
      ...
    </table>
  </td>
</tr>

<!-- Outlook 闭合标签 -->
<!--[if mso]>
      </table>
    </center>
  </td>
</tr>
<![endif]-->
```

#### 8. 响应式样式

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

**文件**：`ghost/core/core/server/services/email-service/email-service-wrapper.js:62-95`

```javascript
// 从 settings 读取 locale（可能是 'en' 简写）
const i18nLanguage = settingsCache.get('locale') || 'en';  // email-service-wrapper.js:62

// 创建 i18n 实例
const i18n = i18nLib(i18nLanguage, 'ghost');  // email-service-wrapper.js:63

// 监听 locale 变化
events.on('settings.locale.edited', (model) => {
    debug('locale changed, updating i18n to', model.get('value'));
    i18n.changeLanguage(model.get('value'));  // email-service-wrapper.js:69
});

// 注入 EmailRenderer
const emailRenderer = new EmailRenderer({
    settingsCache,
    settingsHelpers,
    // ...
    t: i18n.t,           // email-service-wrapper.js:94 - 真实的 i18n.t 函数
    dir: i18n.dir.bind(i18n)  // email-service-wrapper.js:95 - i18n.dir 函数
});
```

### 占位函数 vs 真实翻译函数

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:26-48`

```javascript
/**
 * Wrapper function so that i18next-parser can find these strings.
 * 仅用于静态分析提取翻译 key，不是实际运行时使用的函数！
 *
 * @template T
 * @param {T} x
 * @returns {T}
 */
const t = (x) => {
    return x;
};  // email-renderer.js:33

// 用占位函数提取翻译 key
const messages = {
    subscriptionStatus: {
        free: '',
        expired: t('Your subscription has expired.'),
        canceled: t('Your subscription has been canceled and will expire on {date}...'),
        active: t('Your subscription will renew on {date}.'),
        trial: t('Your free trial ends on {date}...'),
        complimentaryExpires: t('Your subscription will expire on {date}.'),
        complimentaryInfinite: '',
        giftExpires: t('Your subscription will expire on {date}.')
    }
};  // email-renderer.js:36-48
```

**关键区别**：

| 位置 | 函数 | 用途 |
|------|------|------|
| 文件顶部 `const t = (x) => x` | 占位函数 | i18next-parser 静态分析提取 key |
| 构造函数注入的 `this.#t` | `i18n.t` | 运行时实际执行翻译 |

### locale 验证与降级

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:66-74, 271-284`

```javascript
const DEFAULT_LOCALE = 'en-gb';  // email-renderer.js:66

function isValidLocale(locale) {
    try {
        // 使用 Intl.DateTimeFormat 验证
        new Intl.DateTimeFormat(locale);
        return true;
    } catch (e) {
        return false;
    }
}  // email-renderer.js:68-74

#getValidLocale() {
    let locale = this.#settingsCache.get('locale') || DEFAULT_LOCALE;
    // 可能得到 'en'、'zh' 等简写
    
    locale = locale.trim();
    
    // 'en' 简写不被 Intl 完全支持，或 locale 无效 → 降级到 'en-gb'
    if (locale === 'en' || !isValidLocale(locale)) {
        locale = DEFAULT_LOCALE;  // email-renderer.js:280
    }
    
    return locale;
}  // email-renderer.js:272-284
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

**使用示例**：`email-renderer.js:1031-1040`

```javascript
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const timezone = this.#settingsCache.get('timezone');
    const locale = this.#getValidLocale();
    
    const publishedAt = (post.get('published_at') 
        ? DateTime.fromJSDate(post.get('published_at')) 
        : DateTime.local())
        .setZone(timezone)
        .setLocale(locale)
        .toLocaleString({
            year: 'numeric',
            month: 'short',
            day: 'numeric'
        });  // email-renderer.js:1031-1040
}
```

### RTL 方向支持

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:201, 237-238, 1023-1024, 1124`

```javascript
// 构造函数 JSDoc 说明
/**
 * @param {(locale: string) => 'rtl' | 'ltr'} dependencies.dir 
 *        Returns 'rtl' or 'ltr' for a given locale (i18next's `i18n.dir`)
 */

// 构造函数注入
constructor({
    settingsCache,
    settingsHelpers,
    // ...
    t,
    dir  // email-renderer.js:237 - 注入
}) {
    this.#t = t;
    this.#dir = dir;  // email-renderer.js:238 - 赋值
}

// 使用
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const locale = this.#getValidLocale();
    const direction = this.#dir(locale);  // email-renderer.js:1024 - 调用 i18n.dir()
    
    const data = {
        site: {
            locale,
            direction  // 'rtl' 或 'ltr'
        },
        // ...
    };
}
```

**模板应用**：`ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs`

```handlebars
<html lang="{{site.locale}}" dir="{{site.direction}}">
```

**样式中的 RTL 调整**：`ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs`

```handlebars
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
        let activeSubscription = member.subscriptions.find((subscription) => {
            return subscription.status === 'trialing' || subscription.status === 'active';
        });
        if (!activeSubscription) {
            return '';
        }
        
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

    if (member.status === 'comped') {
        const expires = member.tiers[0]?.expiry_at ?? null;
        if (expires) {
            const timezone = this.#settingsCache.get('timezone');
            const date = formatDateLong(expires, timezone, locale);
            return t(messages.subscriptionStatus.complimentaryExpires, {date});
        }
        return t(messages.subscriptionStatus.complimentaryInfinite);
    }

    return '';
}
```

### 状态变量翻译

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:719, 776-786`

```javascript
buildReplacementDefinitions({html, newsletterUuid}) {
    const t = this.#t;           // email-renderer.js:720 - 真实的 i18n.t
    const locale = this.#getValidLocale();

    const baseDefinitions = [
        // ...
        {
            id: 'status',
            getValue: (member) => {
                if (member.status === 'comped') {
                    return t('complimentary');  // email-renderer.js:778
                }
                if (this.isMemberTrialing(member)) {
                    return t('trialing');  // email-renderer.js:781
                }
                // other possible statuses: t('free'), t('paid')
                return t(member.status);  // email-renderer.js:785
            }
        }
    ];
}
```

### 多语言兜底总结

| 场景 | 处理方式 | 文件 | 行号 | 示例 |
|------|----------|------|------|------|
| settings 存 `'en'` 简写 | `#getValidLocale()` 降级 | `email-renderer.js` | 278-280 | `'en'` → `'en-gb'` |
| locale 无效 | `#getValidLocale()` 降级 | `email-renderer.js` | 278-280 | `'invalid'` → `'en-gb'` |
| 文本翻译 | `i18n.t()` 执行 | `email-renderer.js` | 720, 778, 781 | `t('complimentary')` |
| 日期格式 | Luxon `setLocale()` | `email-renderer.js` | 82-88 | `setLocale('zh-cn')` |
| 文本方向 | `i18n.dir()` | `email-renderer.js` | 1024 | `'ar'` → `'rtl'` |

---

## 关键文件索引

### 核心服务

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-service/email-service-wrapper.js` | 依赖注入、i18n 初始化、事件监听 |
| `ghost/core/core/server/services/email-service/email-renderer.js` | 邮件渲染核心（模板、变量、兼容处理） |
| `ghost/core/core/server/services/email-service/sending-service.js` | 发送协调（缓存、收件人构建） |
| `ghost/core/core/server/services/email-service/batch-sending-service.js` | 批量调度（分批、并发、重试、域名预热） |

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
