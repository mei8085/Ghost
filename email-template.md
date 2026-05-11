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
        settingsHelpers,
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

| 步骤 | 函数 | 功能 | 文件 | 精确行号 |
|------|------|------|------|----------|
| 1. 触发发送 | `emailJob({emailId})` | 状态锁 + 调用 sendEmail | `batch-sending-service.js` | 133-190 |
| 2. 创建批次 | `createBatches({email, newsletter, post})` | 分段、分批、域名预热 | `batch-sending-service.js` | 237-340 |
| 3. 渲染邮件体 | `renderBody(post, newsletter, segment, options)` | 完整渲染流水线 | `email-renderer.js` | 393-585 |
| 4. 构建收件人 | `buildRecipients(members, replacementDefinitions)` | 会员字段绑定到变量 | `sending-service.js` | 162-182 |
| 5. 发送 | `send({post, newsletter, segment, members, emailId}, options)` | 缓存 + 调用 Provider | `sending-service.js` | 110-154 |

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
        background_color: 'light',           // newsletter.js:30
        header_background_color: 'transparent',  // newsletter.js:32
        button_color: 'accent',              // newsletter.js:28
        link_color: 'accent',                // newsletter.js:29
        
        // 枚举值
        button_corners: 'rounded',           // newsletter.js:36
        button_style: 'fill',                // newsletter.js:34
        title_font_weight: 'bold',           // newsletter.js:22
        link_style: 'underline',             // newsletter.js:31
        image_corners: 'square',             // newsletter.js:37
        
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
        return value;
    }
    return '#ffffff';           // 'light' 或其他无效值 → 白色
};

const headerBackgroundColor = () => {
    const value = newsletter.header_background_color;
    
    if (!value || value === 'transparent') {
        return 'transparent';
    }
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    if (validHex.test(value)) {
        return value;
    }
    return 'transparent';
};

const buttonColor = () => {
    const value = newsletter.button_color;
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    
    if (validHex.test(value || '')) {
        return value;
    }
    
    if (value === null) {
        const bg = backgroundColor();
        return textColorForBackgroundColor(bg).hex();
    }
    
    return siteData.accent_color;  // 'accent' 或其他无效值 → 站点主色
};

const linkColor = () => {
    const value = newsletter.link_color === undefined ? 'accent' : newsletter.link_color;
    
    if (value === 'accent') {
        return siteData.accent_color;
    }
    
    const validHex = /#([0-9a-f]{3}){1,2}$/i;
    if (validHex.test(value || '')) {
        return value;
    }
    
    return textColorForBackgroundColor(backgroundColor()).hex();
};
```

#### 管理端设计工具函数

**文件**：`apps/admin-x-settings/src/components/settings/email-design/design-utils.ts:18-128`

```typescript
const VALID_HEX = /^#(?:[0-9a-f]{3}){1,2}$/i;

function resolveBackgroundColor(value: string): string {
    if (VALID_HEX.test(value)) return value;
    return '#ffffff';
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
    return accentColor;
}

export function resolveAllColors(settings: EmailDesignSettings, accentColor: string): ResolvedEmailColors {
    const bgColor = resolveBackgroundColor(settings.background_color);
    const headerBgColor = resolveHeaderBackgroundColor(settings.header_background_color, accentColor);
    const buttonColor = resolveButtonColor(settings.button_color, accentColor, bgColor);
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
        titleWeight: String(titleWeight)
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
    'accent',       // email-design.test.js:18
    'light',        // email-design.test.js:19
    'dark',         // email-design.test.js:20
    '#',
    '#F'
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
    
    const data = {
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

**文件**：`ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs:9-16`

```handlebars
/* emailDesign 展开后注入 */
.header {
    {{#if headerBackgroundColor}}
    background-color: {{headerBackgroundColor}};
    {{else}}
    background-color: {{backgroundColor}};
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
    const t = this.#t;           // 真实的 i18n.t 实例
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
            getValue: () => this.createManageAccountUrl()
        },
        {
            id: 'uuid',
            getValue: (member) => member.uuid
        },
        {
            id: 'key',  // HMAC 签名
            getValue: (member) => {
                return crypto.createHmac('sha256', this.#settingsHelpers.getMembersValidationKey())
                    .update(member.uuid).digest('hex');
            }
        },
        {
            id: 'first_name',
            getValue: (member) => member.name?.split(' ')[0]
        },
        {
            id: 'name',
            getValue: (member) => member.name
        },
        {
            id: 'name_class',  // name 为空时返回 'hidden'
            getValue: (member) => member.name ? '' : 'hidden'
        },
        {
            id: 'email',
            getValue: (member) => member.email
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
                return t(member.status);
            }
        },
        {
            id: 'status_text',
            getValue: (member) => this.getMemberStatusText(member)
        },
        {
            id: 'list_unsubscribe',  // 邮件头 List-Unsubscribe
            getValue: (member) => {
                return this.createUnsubscribeUrl(member.uuid, {newsletterUuid});
            },
            required: true
        },
        {
            id: 'uniqueid',  // 绕过 ESP 图片代理缓存
            getValue: () => crypto.randomUUID()
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
    const t = this.#t;
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

### 完整发送链路

```
emailJob({emailId})                  batch-sending-service.js:133-190
  │
  ├── updateStatusLock('submitting')           140-145
  └── sendEmail(email)                         161
       │
       ├── getLazyRelation('newsletter')       201-203
       ├── getLazyRelation('post')             205-207
       ├── getBatches(email)                   209-211
       │     └── EmailBatch.findAll()          228
       │
       ├── createBatches(...)                  214 (如果没有批次)
       │     └── (见下文)
       │
       └── sendBatches(...)                    216
            └── (见下文)
```

### 分批策略

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:237-340`

```javascript
async createBatches({email, post, newsletter}) {
    logging.info(`Creating batches for email ${email.id}`);

    // 1. 域名预热限制
    let domainWarmupLimit = Infinity;
    if (this.#domainWarmingService.isEnabled()) {
        domainWarmupLimit = Number.isInteger(email.get('csd_email_count')) 
            ? email.get('csd_email_count') 
            : Infinity;  // batch-sending-service.js:241-244
    }

    // 2. 获取分段（可能是 [null] 或 ['status:free', 'status:-free']）
    const segments = await this.#emailRenderer.getSegments(post);  // 246
    const batches = [];
    const BATCH_SIZE = this.#sendingService.getMaximumRecipients();  // 248
    let totalCount = 0;

    for (const segment of segments) {
        logging.info(`Creating batches for email ${email.id} segment ${segment}`);

        // 3. 构建 NQL filter（newsletter + recipient_filter + segment）
        const segmentFilter = this.#emailSegmenter.getMemberFilterForSegment(
            newsletter, 
            email.get('recipient_filter'), 
            segment
        );  // 254

        // 4. ObjectId 游标分页
        // 注：使用 id 而非 created_at，因为导入会员可能设置 created_at 为过去/未来值
        let lastId = email.id;  // 261

        while (!members || lastId) {
            logging.info(`Fetching members batch for email ${email.id} segment ${segment}, lastId: ${lastId}`);

            // 5. 查询：id < email.id + 倒序 + 分页
            const filter = segmentFilter + `+id:<'${lastId}'`;  // 266

            members = await this.#models.Member.getFilteredCollectionQuery({filter})
                .orderByRaw('id DESC')
                .select('members.id', 'members.uuid', 'members.email', 'members.name')
                .limit(BATCH_SIZE + 1);  // 269-271

            if (members.length > 0) {
                // 6. 域名预热：决定是否需要拆分批次
                const remainingCustomDomainCapacity = domainWarmupLimit - totalCount;
                const membersToProcess = Math.min(members.length, BATCH_SIZE);

                const shouldSplitBatch = remainingCustomDomainCapacity > 0 
                    && remainingCustomDomainCapacity < membersToProcess;  // 278

                if (shouldSplitBatch) {
                    // 拆分批次：部分走自定义域名，部分走 fallback
                    totalCount += await this.#createBatchWithRetry({
                        email, segment,
                        members: members.slice(0, remainingCustomDomainCapacity),
                        useFallbackDomain: false,
                        batches
                    });  // 281-287

                    totalCount += await this.#createBatchWithRetry({
                        email, segment,
                        members: members.slice(remainingCustomDomainCapacity, membersToProcess),
                        useFallbackDomain: true,
                        batches
                    });  // 288-294
                } else {
                    // 单一批次
                    totalCount += await this.#createBatchWithRetry({
                        email, segment,
                        members: members.slice(0, membersToProcess),
                        useFallbackDomain: totalCount >= domainWarmupLimit,
                        batches
                    });  // 297-303
                }
            }

            // 7. 游标更新：如果有 BATCH_SIZE + 1 条，说明还有更多
            if (members.length > BATCH_SIZE) {
                lastId = members[members.length - 2].id;  // 308
            } else {
                break;  // 310
            }
        }
    }

    logging.info(`Created ${batches.length} batches for email ${email.id} with ${totalCount} recipients`);

    // 8. 校验并更新 email_count（与创建邮件时的估算值对比）
    if (email.get('email_count') !== totalCount) {
        logging.error(`Email ${email.id} has wrong stored email_count ${email.get('email_count')}, did expect ${totalCount}. Updating the model.`);

        // 误差率 > 1% 时记录到 Sentry
        const errorRate = Math.abs((totalCount - email.get('email_count')) / email.get('email_count'));
        if (this.#sentry && errorRate >= 0.01) {
            this.#sentry.captureMessage(`Email ${email.id} has wrong stored email_count...`);
        }

        const newEmailUpdate = {
            email_count: totalCount
        };
        if (this.#domainWarmingService.isEnabled()) {
            newEmailUpdate.csd_email_count = Math.min(totalCount, domainWarmupLimit);
        }

        await email.save(newEmailUpdate, {patch: true, require: false, autoRefresh: false});  // 337
    }

    return batches;  // 339
}
```

### #createBatchWithRetry 包装

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:352-370`

```javascript
async #createBatchWithRetry({email, segment, members, useFallbackDomain, batches}) {
    if (members.length === 0) {
        return 0;  // 354
    }

    const batch = await this.retryDb(
        async () => {
            return await this.createBatch(email, segment, members, {
                useFallbackDomain
            });  // 359
        },
        {
            ...this.#getBeforeRetryConfig(email),
            description: `createBatch email ${email.id} segment ${segment}${useFallbackDomain ? ' (fallback domain)' : ' (custom domain)'}`
        }
    );  // 357-367

    batches.push(batch);  // 368
    return members.length;  // 369
}
```

### 创建 EmailBatch + EmailRecipient

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:382-426`

```javascript
async createBatch(email, segment, members, options) {
    // 1. 如果没有事务，创建新事务
    if (!options || !options.transacting) {
        return this.#models.EmailBatch.transaction(async (transacting) => {
            return this.createBatch(email, segment, members, {transacting, ...options});
        });  // 383-386
    }

    logging.info(`Creating batch for email ${email.id} segment ${segment} with ${members.length} members`);

    // 2. 创建 EmailBatch
    const batch = await this.#models.EmailBatch.add({
        email_id: email.id,
        member_segment: segment,
        status: 'pending',
        fallback_sending_domain: Boolean(options.useFallbackDomain)
    }, options);  // 391-396

    // 3. 构建 EmailRecipient 数据
    const recipientData = [];

    members.forEach((memberRow) => {
        if (!memberRow.id || !memberRow.uuid || !memberRow.email) {
            logging.warn(`Member row not included as email recipient due to missing data - id: ${memberRow.id}, uuid: ${memberRow.uuid}, email: ${memberRow.email}`);
            return;  // 401-404
        }

        recipientData.push({
            id: ObjectID().toHexString(),
            email_id: email.id,
            member_id: memberRow.id,
            batch_id: batch.id,
            member_uuid: memberRow.uuid,
            member_email: memberRow.email,
            member_name: memberRow.name
        });  // 406-414
    });

    // 4. 批量插入 EmailRecipient
    const insertQuery = this.#db.knex('email_recipients').insert(recipientData);  // 417

    if (options.transacting) {
        insertQuery.transacting(options.transacting);  // 419-420
    }

    logging.info(`Inserting ${recipientData.length} recipients for email ${email.id} batch ${batch.id}`);
    await insertQuery;  // 424

    return batch;  // 425
}
```

### 分段渲染检测

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:335-361`

```javascript
async getSegments(post) {
    const allowedSegments = ['status:free', 'status:-free'];
    const html = await this.renderPostBaseHtml(post);

    // 1. 有 paywall card (<!--members-only-->) → 必须分两段
    if (html.indexOf('<!--members-only-->') !== -1) {
        return allowedSegments;  // 344
    }

    const $ = cheerioLoad(html);

    // 2. 检查 data-gh-segment 属性
    let allSegments = $('[data-gh-segment]')
        .get()
        .map(el => el.attribs['data-gh-segment']);

    const segments = [...new Set(allSegments)].filter(segment => allowedSegments.includes(segment));
    
    if (segments.length === 0) {
        // 无差异 → 单一段 [null]
        return [null];  // 356
    }

    // 有差异 → 分两段
    return allowedSegments;  // 375
}
```

### 应用分段过滤 + 完整 renderBody 流程

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:393-585`

```javascript
async renderBody(post, newsletter, segment, options) {
    let html = await this.renderPostBaseHtml(post, newsletter);  // 394

    // ── 步骤 1：Paywall 截断 ──────────────────────────────────
    const isPaidPost = post.get('visibility') === 'paid' || post.get('visibility') === 'tiers';  // 397
    const membersOnlyIndex = html.indexOf('<!--members-only-->');  // 398
    const hasMembersOnlyContent = membersOnlyIndex !== -1;  // 399
    let addPaywall = false;  // 400

    if (isPaidPost && hasMembersOnlyContent) {
        if (segment === 'status:free') {
            // 免费会员：截断内容 + 添加 paywall
            addPaywall = true;  // 405
            html = html.slice(0, membersOnlyIndex);  // 408
        }
    }

    let $ = cheerioLoad(html);  // 412

    // ── 步骤 2：data-gh-segment 过滤 ──────────────────────────
    // 必须在模板渲染前执行，因为 preheader 可能用到 HTML
    $('[data-gh-segment]').get().forEach((node) => {
        if (node.attribs['data-gh-segment'] !== segment) {
            $(node).remove();  // 421
        } else {
            $(node).removeAttr('data-gh-segment');  // 424
        }
    });  // 418-426

    html = $.html();  // 428

    // ── 步骤 3：模板渲染 ──────────────────────────────────────
    const templateData = await this.getTemplateData({
        post, newsletter, html, addPaywall, segment
    });  // 430-436
    html = await this.renderTemplate(templateData);  // 437

    // ── 步骤 4：链接跟踪 ──────────────────────────────────────
    const base = templateData.post.url;  // 440

    if (options.clickTrackingEnabled) {
        // 有跟踪：替换为追踪链接 + 添加 newsletter ref + post attribution
        html = await this.#linkReplacer.replace(html, async (url, originalPath) => {
            if (originalPath.startsWith('%%{') && originalPath.endsWith('}%%')) {
                return originalPath;  // 446-448
            }
            if (originalPath === '#') {
                return originalPath;  // 451-452
            }
            // 跳过包含其他 replacement patterns 的链接（%{key}%% 需要 Mailgun 直接替换）
            const urlStr = url.toString();
            const withoutUuid = urlStr.replace(/%%\{uuid\}%%/g, '');
            if (/%%\{[^}]+\}%%/.test(withoutUuid)) {
                return urlStr;  // 459-462
            }
            // 添加 newsletter source attribution
            const isSite = this.#urlUtils.isSiteUrl(url);
            if (isSite) {
                url = this.#outboundLinkTagger.addToUrl(url, newsletter);
                url = this.#memberAttributionService.addPostAttributionTracking(url, post);
            } else {
                url = this.#outboundLinkTagger.addToUrl(url);
            }  // 465-476
            // 跳过 Powered by Ghost badge
            if (url.hostname === 'ghost.org' && url.pathname === '/' && url.searchParams.get('via') === 'pbg-newsletter') {
                return url.toString();  // 479-481
            }
            // 添加点击跟踪
            url = await this.#linkTracking.service.addTrackingToUrl(url, post, '--uuid--');
            const str = url.toString().replace(/--uuid--/g, '%%{uuid}%%');
            return str;  // 484-488
        }, {base});
    } else {
        // 无跟踪：仅替换相对链接为绝对链接
        html = await this.#linkReplacer.replace(html, (url, originalPath) => {
            if (originalPath.startsWith('%%{') && originalPath.endsWith('}%%')) {
                return originalPath;  // 493-495
            }
            if (originalPath === '#') {
                return originalPath;  // 499-500
            }
            return url;  // 502
        }, {base});  // 490-504
    }

    // ── 步骤 5：记录图片原始尺寸（Juice 前） ──────────────────
    // 如果 CSS 设置了 width: auto 或 height: auto，Juice 会显式设置到属性上
    // 这是 Outlook 不支持的，需要恢复原始值
    $ = cheerioLoad(html);  // 511
    const originalImageSizes = $('img').get().map((image) => {
        const src = image.attribs.src;
        const width = image.attribs.width;
        const height = image.attribs.height;
        return {src, width, height};
    });  // 512-517

    // ── 步骤 6：给 figcaption 加类（Juice 前，CSS 需要） ─────
    $('figcaption').each((i, elem) => !!($(elem).addClass('kg-card-figcaption')));  // 519-520
    html = $.html();  // 521

    // ── 步骤 7：Juice 内联 CSS ──────────────────────────────
    const juice = require('juice');
    html = juice(html, {inlinePseudoElements: true, removeStyleTags: true});  // 523-525

    // ── 步骤 8：Juice 后处理 ─────────────────────────────────
    $ = cheerioLoad(html);  // 528

    // 8a. 恢复图片 width/height = 'auto' → 原始值
    const imageTags = $('img').get();
    for (let i = 0; i < imageTags.length; i += 1) {
        if (imageTags[i].attribs.src === originalImageSizes[i].src) {
            if (imageTags[i].attribs.width === 'auto' && originalImageSizes[i].width) {
                imageTags[i].attribs.width = originalImageSizes[i].width;  // 536-537
            }
            if (imageTags[i].attribs.height === 'auto' && originalImageSizes[i].height) {
                imageTags[i].attribs.height = originalImageSizes[i].height;  // 539-540
            }
        }
    }  // 531-543

    // 8b. 强制所有链接在新窗口打开
    $('a').attr('target', '_blank');  // 546

    // 8c. 语义化标签转 div（Outlook margin 支持问题）
    $('figure, figcaption').each((i, elem) => !!(elem.tagName = 'div'));  // 549

    // 8d. 暗色/亮色模式图片切换（CSS 方案在 Outlook 不生效）
    if (templateData.backgroundIsDark) {
        $('img.is-light-background').each((i, elem) => $(elem).remove());  // 553-555
    } else {
        $('img.is-dark-background').each((i, elem) => $(elem).remove());  // 557-559
    }  // 552-560

    // ── 步骤 9：HTML → 变量定义 ──────────────────────────────
    html = $.html();  // 563
    const replacementDefinitions = this.buildReplacementDefinitions({
        html, 
        newsletterUuid: newsletter.get('uuid')
    });  // 566

    // ── 步骤 10：HTML → 纯文本 ──────────────────────────────
    const plaintext = htmlToPlaintext.email(html);  // 571

    // ── 步骤 11：特殊字符转义（Outlook 兼容） ────────────────
    html = html.replace(/&apos;/g, '&#39;');     // 单引号
    html = html.replace(/→/g, '&rarr;');          // 右箭头
    html = html.replace(/–/g, '&ndash;');         // 短破折号
    html = html.replace(/“/g, '&ldquo;');         // 左双引号
    html = html.replace(/”/g, '&rdquo;');         // 右双引号  // 574-578

    return {
        html,
        plaintext,
        replacements: replacementDefinitions
    };  // 580-584
}
```

### sendBatches 并发控制

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:428-479`

```javascript
async sendBatches({email, batches, post, newsletter}) {
    logging.info(`Sending ${batches.length} batches for email ${email.id}`);
    const deadline = this.getDeliveryDeadline(email);  // 430

    if (deadline) {
        logging.info(`Delivery deadline for email ${email.id} is ${deadline}`);
    }

    // 1. 渲染结果缓存（按 segment 缓存，同一 segment 的批次共享）
    /** @type {Map<string, import('./email-renderer').EmailBody>} */
    const emailBodyCache = new Map();  // 437

    // 2. 计算投递时间（如果有 deadline）
    const deliveryTimes = this.calculateDeliveryTimes(email, batches.length);  // 440

    // 3. 队列 + 并发控制
    let succeededCount = 0;  // 443
    const queue = batches.slice();  // 444

    // 递归 worker
    let runNext;
    runNext = async () => {
        const batch = queue.shift();  // 449
        if (batch) {
            const batchData = {email, batch, post, newsletter, emailBodyCache, deliveryTime: undefined};
            // 只有在有 deadline 且未过期时才设置投递时间
            if (deadline && deadline.getTime() > Date.now()) {
                const deliveryTime = deliveryTimes.shift();
                if (deliveryTime && deliveryTime >= Date.now()) {
                    batchData.deliveryTime = deliveryTime;  // 456
                }
            }
            if (await this.sendBatch(batchData)) {
                succeededCount += 1;  // 460
            }
            await runNext();  // 462 - 递归调用
        }
    };  // 447-464

    // 启动 MAX_SENDING_CONCURRENCY 个并发 worker
    await Promise.all(
        new Array(MAX_SENDING_CONCURRENCY).fill(0).map(() => runNext())
    );  // 467

    if (succeededCount < batches.length) {
        if (succeededCount > 0) {
            throw new errors.EmailError({
                message: tpl(messages.emailErrorPartialFailure)
            });
        }
        throw new errors.EmailError({
            message: tpl(messages.emailError)
        });  // 469-478
    }
}
```

### sendBatch 单批次发送

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:486-601`

```javascript
async sendBatch({email, batch: originalBatch, post, newsletter, emailBodyCache, deliveryTime}) {
    logging.info(`Sending batch ${originalBatch.id} for email ${email.id}`);

    // ── 步骤 1：状态锁 ─────────────────────────────────────
    const batch = await this.retryDb(
        async () => {
            return await this.updateStatusLock(
                this.#models.EmailBatch, 
                originalBatch.id, 
                'submitting', 
                ['pending', 'failed']
            );  // 493
        },
        {...this.#getBeforeRetryConfig(email), description: `updateStatusLock batch ${originalBatch.id} -> submitting`}
    );  // 491-496

    if (!batch) {
        logging.error(`Tried sending email batch that is not pending or failed ${originalBatch.id}`);
        return true;  // 499
    }

    let succeeded = false;  // 502

    try {
        // ── 步骤 2：获取批次会员 ──────────────────────────────
        let members = await this.retryDb(
            async () => {
                const m = await this.getBatchMembers(batch.id);  // 507

                // 0 条记录 → 可能是主从复制延迟，抛出重试
                if (m.length === 0) {
                    throw new errors.EmailError({
                        message: `No members found for batch ${batch.id}, possible replication lag`
                    });  // 513
                }

                return m;  // 517
            },
            {...this.#getBeforeRetryConfig(email), description: `getBatchMembers batch ${originalBatch.id}`}
        );  // 505-520

        // ── 步骤 3：调用 SendingService.send ─────────────────
        const response = await this.retryDb(
            async () => {
                return await this.#sendingService.send({
                    emailId: email.id,
                    post,
                    newsletter,
                    segment: batch.get('member_segment'),
                    members
                }, {
                    openTrackingEnabled: !!email.get('track_opens'),
                    clickTrackingEnabled: !!email.get('track_clicks'),
                    useFallbackAddress: batch.get('fallback_sending_domain'),
                    deliveryTime,
                    emailBodyCache
                });  // 523-535
            }, 
            {...this.#MAILGUN_API_RETRY_CONFIG, description: `Sending email batch ${originalBatch.id} ...`}
        );  // 522-536

        succeeded = true;  // 537

        // ── 步骤 4：更新状态为 submitted ─────────────────────
        await this.retryDb(
            async () => {
                await batch.save({
                    status: 'submitted',
                    provider_id: response.id,
                    error_status_code: null,
                    error_message: null,
                    error_data: null
                }, {patch: true, require: false, autoRefresh: false});  // 541-548
            },
            {...this.#AFTER_RETRY_CONFIG, description: `save batch ${originalBatch.id} -> submitted`}
        );  // 539-551

    } catch (err) {
        // ── 错误处理 ────────────────────────────────────────
        if (err.code && err.code === 'BULK_EMAIL_SEND_FAILED') {
            logging.error(err);
            if (this.#sentry) {
                this.#sentry.captureException(err);
            }  // 553-558
        } else {
            const ghostError = new errors.EmailError({
                err,
                code: 'BULK_EMAIL_SEND_FAILED',
                message: `Error sending email batch ${batch.id}`,
                context: err.message
            });
            logging.error(ghostError);
            if (this.#sentry) {
                this.#sentry.captureException(err);
            }  // 560-571
        }

        if (!succeeded) {
            // 只有在确实未发送时才标记为 failed
            // （罕见边界：已发送但设置状态失败时不重发）
            await this.retryDb(
                async () => {
                    await batch.save({
                        status: 'failed',
                        error_status_code: err.statusCode ?? null,
                        error_message: err.message,
                        error_data: err.errorDetails ?? null
                    }, {patch: true, require: false, autoRefresh: false});  // 578-583
                },
                {...this.#AFTER_RETRY_CONFIG, description: `save batch ${originalBatch.id} -> failed`}
            );  // 576-586
        }
    }

    // ── 步骤 5：标记收件人为 processed ──────────────────────
    await this.retryDb(
        async () => {
            await this.#models.EmailRecipient
                .where({batch_id: batch.id})
                .save({processed_at: new Date()}, {patch: true, require: false, autoRefresh: false});  // 593-595
        },
        {...this.#AFTER_RETRY_CONFIG, description: `save EmailRecipients ${originalBatch.id} processed_at`}
    );  // 591-598

    return succeeded;  // 600
}
```

### getBatchMembers：EmailRecipient → MemberLike 转换

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:609-635`

```javascript
async getBatchMembers(batchId) {
    // 1. 查询 EmailRecipient + 关联 member、stripeSubscriptions、products
    let models = await this.#models.EmailRecipient.findAll({
        filter: `batch_id:'${batchId}'`, 
        withRelated: ['member', 'member.stripeSubscriptions', 'member.products']
    });  // 610

    // 2. 校验批次大小
    const BATCH_SIZE = this.#sendingService.getMaximumRecipients();
    if (models.length > BATCH_SIZE) {
        throw new errors.EmailError({
            message: `Email batch ${batchId} has ${models.length} members, which exceeds the maximum of ${BATCH_SIZE} members per batch.`
        });  // 614-616
    }

    // 3. 转换为 MemberLike 接口（隔离实现细节）
    return models.map((model) => {
        const subscriptions = model.related('member').related('stripeSubscriptions').toJSON();
        const tiers = model.related('member').related('products').toJSON();

        return {
            id: model.get('member_id'),
            uuid: model.get('member_uuid'),
            email: model.get('member_email'),
            name: model.get('member_name'),
            createdAt: model.related('member')?.get('created_at') ?? null,
            status: model.related('member')?.get('status') ?? 'free',
            subscriptions,
            tiers
        };  // 624-633
    });
}
```

### updateStatusLock：行级锁状态检查

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:646-659`

```javascript
async updateStatusLock(Model, id, status, allowedStatuses) {
    let model;
    await Model.transaction(async (transacting) => {
        // 1. FOR UPDATE 行级锁
        model = await Model.findOne({id}, {require: true, transacting, forUpdate: true});  // 649

        // 2. 检查当前状态是否允许变更
        if (!allowedStatuses.includes(model.get('status'))) {
            model = undefined;
            return;  // 651-653
        }

        // 3. 更新状态
        await model.save({status}, {patch: true, transacting, autoRefresh: false});  // 654-656
    });

    return model;  // 658
}
```

### retryDb：指数退避重试

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:675-728`

```javascript
async retryDb(func, options) {
    // 1. 计算截止时间
    if (options.maxTime !== undefined) {
        const stopAfterDate = new Date(Date.now() + options.maxTime);
        if (!options.stopAfterDate || stopAfterDate < options.stopAfterDate) {
            options = {...options, stopAfterDate};  // 677-680
        }
    }

    const retryCount = (options.retryCount ?? 0);  // 682

    try {
        if (retryCount > 0) {
            logging.info(`[BULK_EMAIL_DB_RETRY] ${options.description} - Retrying ${retryCount + 1}th try`);
        } else {
            logging.info(`[BULK_EMAIL_DB_RETRY] ${options.description} - Started (1st try)`);
        }  // 685-689

        const response = await func();  // 691

        logging.info(`[BULK_EMAIL_DB_RETRY] ${options.description} - Finished (after ${retryCount + 1}${retryCount === 0 ? 'st try' : ' tries'})`);
        return response;  // 693-695

    } catch (e) {
        const sleep = (options.sleep ?? 0);  // 697

        // 2. 检查是否需要继续重试
        if (retryCount >= options.maxRetries 
            || (options.stopAfterDate && (new Date(Date.now() + sleep)) > options.stopAfterDate)) {
            if (retryCount > 0) {
                const ghostError = new errors.EmailError({
                    err: e,
                    code: 'BULK_EMAIL_DB_RETRY',
                    message: `[BULK_EMAIL_DB_RETRY] ${options.description} - Failed and stopped retrying: ${retryCount >= options.maxRetries ? 'max retries reached' : 'max time reached'}`,
                    context: e.message
                });
                logging.error(ghostError);
            }
            throw e;  // 698-710
        }

        // 3. 记录错误日志
        const ghostError = new errors.EmailError({
            err,
            code: 'BULK_EMAIL_DB_RETRY',
            message: `[BULK_EMAIL_DB_RETRY] ${options.description} - Failed (${retryCount + 1}${retryCount === 0 ? 'st' : 'th'} try)`,
            context: e.message
        });
        logging.error(ghostError);  // 712-718

        // 4. 等待 + 指数退避（sleep * 2）
        if (sleep) {
            await new Promise((resolve) => setTimeout(resolve, sleep));
        }  // 721-724

        return await this.retryDb(
            func, 
            {...options, retryCount: retryCount + 1, sleep: sleep * 2}  // 指数退避
        );  // 726
    }
}
```

### #BEFORE/#AFTER/#MAILGUN 重试配置

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:37-39`

```javascript
// 发送前的数据库操作（状态检查、查询会员等）
#BEFORE_RETRY_CONFIG = {maxRetries: 10, maxTime: 10 * 60 * 1000, sleep: 2000};     // 10次重试，最多10分钟

// 发送后的数据库操作（更新状态等）
#AFTER_RETRY_CONFIG = {maxRetries: 20, maxTime: 30 * 60 * 1000, sleep: 2000};       // 20次重试，最多30分钟

// Mailgun API 调用
#MAILGUN_API_RETRY_CONFIG = {sleep: 10 * 1000, maxRetries: 6};                        // 6次重试，10秒间隔
```

### #getBeforeRetryConfig：动态截止时间

**文件**：`ghost/core/core/server/services/email-service/batch-sending-service.js:151-158`

```javascript
// 在 emailJob 中计算严格截止时间
const expectedBatchCount = Math.ceil(email.get('email_count') / 1000);  // 152
const minimumSecondsPerBatch = 26;  // 153
const stopAfter = Math.max(
    expectedBatchCount * minimumSecondsPerBatch * 1000, 
    this.#BEFORE_RETRY_CONFIG.maxTime
);  // 154
const retryCutOffTime = new Date(startTime + stopAfter);  // 155

email._retryCutOffTime = retryCutOffTime;  // 158
```

### SendingService.send：缓存 + 调用 Provider

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:110-154`

```javascript
async send({post, newsletter, segment, members, emailId}, options) {
    const cacheId = emailId + '-' + (segment ?? 'null');  // 111
    const isTestEmail = options.isTestEmail ?? false;  // 112

    /** @type {EmailBody | undefined} */
    let emailBody;

    // 1. 尝试从缓存获取
    if (options.emailBodyCache) {
        emailBody = options.emailBodyCache.get(cacheId);  // 120
    }

    // 2. 缓存未命中 → 渲染
    if (!emailBody) {
        emailBody = await this.#emailRenderer.renderBody(
            post,
            newsletter,
            segment,
            {
                clickTrackingEnabled: !!options.clickTrackingEnabled
            }
        );  // 124-131
        
        if (options.emailBodyCache) {
            options.emailBodyCache.set(cacheId, emailBody);  // 133
        }
    }

    // 3. 构建收件人
    const recipients = this.buildRecipients(members, emailBody.replacements);  // 137

    // 4. 调用 Provider
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
    }, {
        clickTrackingEnabled: !!options.clickTrackingEnabled,
        openTrackingEnabled: !!options.openTrackingEnabled,
        useFallbackAddress: !!options.useFallbackAddress,
        ...(options.deliveryTime && {deliveryTime: options.deliveryTime})
    });  // 138-153
}
```

### 跨邮箱客户端兼容性处理汇总

| 步骤 | 功能 | 文件 | 精确行号 | 证据 |
|------|------|------|----------|------|
| 0. CSS 方案（被 HTML 方案覆盖） | 暗色图片隐藏 | `styles.hbs` | 4-7 | `display: none !important; mso-hide: all !important;` 但 Outlook 不生效 |
| 1. figcaption 加类 | 为 CSS 样式准备 | `email-renderer.js` | 519-520 | `addClass('kg-card-figcaption')` |
| 2. Juice 内联 CSS | style 标签 → 内联属性 | `email-renderer.js` | 523-525 | `juice(html, {inlinePseudoElements: true, removeStyleTags: true})` |
| 3. 图片 auto 修复 | Outlook 不支持 width: auto | `email-renderer.js` | 531-543 | `if (imageTags[i].attribs.width === 'auto') → 恢复 original` |
| 4. 强制新窗口 | 链接 target | `email-renderer.js` | 546 | `$('a').attr('target', '_blank')` |
| 5. 语义化标签转 div | Outlook margin 问题 | `email-renderer.js` | 549 | `$('figure, figcaption').each(... tagName = 'div')` |
| 6. 暗色图片切换 | 替代 CSS 方案 | `email-renderer.js` | 552-560 | `$('img.is-light-background').remove()` 等 |
| 7. 特殊字符转义 | Outlook 显示问题 | `email-renderer.js` | 574-578 | `replace(/&apos;/g, '&#39;')` 等 5 个替换 |
| 8. PixelsPerInch | Outlook DPI 缩放 | `email-wrapper.hbs` | 6 | `<!--[if mso]><o:PixelsPerInch>96</o:PixelsPerInch>` |
| 9. Header 居中 | Outlook 不尊重 max-width | `email-wrapper.hbs` | 21-26 | `<!--[if mso]><center><table width="600">` |
| 10. Body 居中 | Outlook 不尊重 max-width | `email-wrapper.hbs` | 168-173 | `<!--[if mso]><center><table width="600">` |

### Outlook 条件注释完整位置

**文件**：`ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs:6, 20-26, 167-173`

```handlebars
<!-- 位置 1：DPI 缩放（第 6 行） -->
<!--[if mso]><xml><o:OfficeDocumentSettings><o:PixelsPerInch>96</o:PixelsPerInch><o:AllowPNG/></o:OfficeDocumentSettings></xml><![endif]-->

<!-- 位置 2：Header 居中（第 20-26 行，156-161 行闭合） -->
<!-- Outlook doesn't respect max-width so we need an extra centered table -->
<!--[if mso]>
<tr>
  <td>
    <center>
      <table border="0" cellpadding="0" cellspacing="0" width="600">
<![endif]-->

<!-- ... header 内容 ... -->

<!-- 闭合（第 156-161 行） -->
<!--[if mso]>
            </table>
        </center>
    </td>
</tr>
<![endif]-->

<!-- 位置 3：Body 居中（第 167-173 行，187-192 行闭合） -->
<!-- Outlook doesn't respect max-width so we need an extra centered table -->
<!--[if mso]>
<tr>
  <td>
    <center>
      <table border="0" cellpadding="0" cellspacing="0" width="600">
<![endif]-->

<!-- ... body 内容 ... -->

<!-- 闭合（第 187-192 行） -->
<!--[if mso]>
            </table>
        </center>
    </td>
</tr>
<![endif]-->
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
    t: i18n.t,           // email-service-wrapper.js:94
    dir: i18n.dir.bind(i18n)  // email-service-wrapper.js:95
});
```

### 占位函数 vs 真实翻译函数

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:26-48`

```javascript
/**
 * Wrapper function so that i18next-parser can find these strings.
 * 仅用于静态分析提取翻译 key，不是实际运行时使用的函数！
 */
const t = (x) => {
    return x;
};  // email-renderer.js:33

// 用占位函数提取翻译 key（i18next-parser 静态分析）
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
    let locale = this.#settingsCache.get('locale') || DEFAULT_LOCALE;  // 273
    // 可能得到 'en'、'zh' 等简写
    
    locale = locale.trim();  // 276
    
    // 'en' 简写不被 Intl 完全支持，或 locale 无效 → 降级到 'en-gb'
    if (locale === 'en' || !isValidLocale(locale)) {
        locale = DEFAULT_LOCALE;  // email-renderer.js:280
    }
    
    return locale;
}  // 272-284
```

### 日期本地化

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:82-88`

```javascript
function formatDateLong(date, timezone, locale = DEFAULT_LOCALE) {
    return DateTime.fromJSDate(date)
        .setZone(timezone)
        .setLocale(locale)
        .toLocaleString({
            year: 'numeric',
            month: 'long',
            day: 'numeric'
        });
}
```

### RTL 方向支持

**文件**：`ghost/core/core/server/services/email-service/email-renderer.js:237-238, 1023-1024`

```javascript
// 构造函数注入
constructor({
    settingsCache,
    settingsHelpers,
    t,
    dir  // email-renderer.js:237 - 注入 i18n.dir
}) {
    this.#t = t;
    this.#dir = dir;  // email-renderer.js:238 - 赋值
}

// 使用
async getTemplateData({post, newsletter, html, addPaywall, segment}) {
    const locale = this.#getValidLocale();
    const direction = this.#dir(locale);  // email-renderer.js:1024
    
    const data = {
        site: {
            locale,
            direction
        }
    };
}
```

### 多语言兜底总结

| 场景 | 处理方式 | 文件 | 精确行号 |
|------|----------|------|----------|
| settings 存 `'en'` 简写 | `#getValidLocale()` 降级 | `email-renderer.js` | 278-280 |
| locale 无效 | `#getValidLocale()` 降级 | `email-renderer.js` | 278-280 |
| 文本翻译 | `i18n.t()` 执行 | `email-renderer.js` | 720, 778, 781 |
| 日期格式 | Luxon `setLocale()` | `email-renderer.js` | 82-88 |
| 文本方向 | `i18n.dir()` | `email-renderer.js` | 1024 |
| HTML lang/dir | 模板注入 | `email-wrapper.hbs` | 2 |
| RTL 布局调整 | 条件样式 | `styles.hbs` | 见样式文件 |

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
| `ghost/core/core/server/services/email-templates/partials/styles.hbs` | 动态样式注入 |

### 测试文件

| 文件 | 职责 |
|------|------|
| `ghost/core/test/unit/server/services/email-rendering/email-design.test.js` | 验证语义值被视为无效 HEX |
| `ghost/core/test/unit/server/services/email-service/batch-sending-service.test.js` | 批量发送单元测试 |
| `ghost/core/test/integration/services/email-service/batch-sending.test.js` | 批量发送集成测试 |
