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

### 核心三层架构

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

### 渲染流程总览

1. **触发发送**：`BatchSendingService.emailJob()` 被 JobsService 调度执行
2. **创建批次**：按 1000 人/批创建 `EmailBatch`，关联 `EmailRecipient`
3. **渲染邮件体**：`EmailRenderer.renderBody()` 执行完整渲染流水线
4. **构建收件人**：`SendingService.buildRecipients()` 绑定个性化变量
5. **发送**：通过 Mailgun 等 Provider 批量发送

---

## 后台 Email Designer 与主题模板共享版式

### 共享设计逻辑：`getEmailDesign` 函数

**核心文件**：`ghost/core/core/server/services/email-rendering/email-design.js:71`

后端和管理端共享同一套颜色计算逻辑，确保预览与实际发送一致：

```javascript
exports.getEmailDesign = (settings) => {
    // 1. 验证并规范化颜色值（#xxx 或 #xxxxxx）
    const accentColor = getHexColor(settings.accentColor, DEFAULT_ACCENT_COLOR);
    
    // 2. 计算对比度颜色（自动选择黑白文字）
    const accentContrastColor = textColorForBackgroundColor(accentColor).hex();
    
    // 3. 解析特殊值 'accent' → 实际颜色
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
    
    // 4. 按钮圆角映射
    let buttonBorderRadius;
    switch (buttonCorners) {
        case 'square': buttonBorderRadius = '0'; break;
        case 'pill': buttonBorderRadius = '9999px'; break;
        default: buttonBorderRadius = '6px'; break;
    }
};
```

### 管理端对应实现

**文件**：`apps/admin-x-settings/src/components/settings/email-design/design-utils.ts:88`

```typescript
export function resolveAllColors(settings: EmailDesignSettings, accentColor: string): ResolvedEmailColors {
    // 与后端完全一致的颜色解析逻辑
    const bgColor = resolveBackgroundColor(settings.background_color);
    const headerBgColor = resolveHeaderBackgroundColor(settings.header_background_color, accentColor);
    // ... 相同的颜色计算
}

export function resolveButtonCorners(corners: string | undefined): string {
    // 相同的圆角映射：square → rounded-none, pill → rounded-full
    switch (corners) {
    case 'square': return 'rounded-none';
    case 'pill': return 'rounded-full';
    case 'rounded':
    default: return 'rounded-[6px]';
    }
}
```

### 共享 Handlebars 模板

**核心模板结构**：

```
ghost/core/core/server/services/
├── email-rendering/
│   ├── email-design.js          # 共享设计计算逻辑
│   └── partials/
│       ├── email-wrapper.hbs    # 邮件外层包装（HTML doctype、head、body 骨架）
│       ├── base-styles.hbs      # 基础样式
│       ├── content-styles.hbs   # 内容样式
│       └── card-styles.hbs      # 卡片样式
└── email-service/
    └── email-templates/
        ├── template.hbs         # 主模板（内容区域 + 反馈按钮 + 页脚）
        └── partials/
            ├── styles.hbs       # 组装所有样式 + 动态颜色注入
            ├── feedback-button.hbs
            ├── latest-posts.hbs
            └── paywall.hbs
```

### 动态样式注入机制

**文件**：`ghost/core/core/server/services/email-renderer.js:937`

```javascript
#getEmailDesign(newsletter) {
    return getEmailDesign({
        accentColor: this.#settingsCache?.get('accent_color'),
        backgroundColor: newsletter?.get('background_color'),
        buttonColor: newsletter?.get('button_color'),
        buttonCorners: newsletter?.get('button_corners'),
        buttonStyle: newsletter?.get('button_style'),
        dividerColor: newsletter?.get('divider_color'),
        headerBackgroundColor: newsletter?.get('header_background_color'),
        imageCorners: newsletter?.get('image_corners'),
        linkColor: newsletter?.get('link_color'),
        linkStyle: newsletter?.get('link_style'),
        postTitleColor: newsletter?.get('post_title_color'),
        sectionTitleColor: newsletter?.get('section_title_color'),
        titleFontWeight: newsletter?.get('title_font_weight')
    });
}
```

**模板注入**：`email-templates/partials/styles.hbs`

```handlebars
/* 动态颜色通过 Handlebars 变量注入 */
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

### Newsletter 模型默认设计值

**文件**：`ghost/core/core/server/models/newsletter.js:9`

```javascript
defaults: function defaults() {
    return {
        // 排版
        title_font_category: 'sans_serif',
        body_font_category: 'sans_serif',
        title_alignment: 'center',
        title_font_weight: 'bold',
        
        // 颜色
        background_color: 'light',
        button_color: 'accent',
        button_style: 'fill',
        link_color: 'accent',
        link_style: 'underline',
        header_background_color: 'transparent',
        
        // 圆角
        button_corners: 'rounded',
        image_corners: 'square',
        
        // 开关
        show_badge: true,
        show_header_icon: true,
        show_header_title: true,
        show_feature_image: true,
        show_comment_cta: true,
        feedback_enabled: false
    };
}
```

---

## 个性化变量绑定到会员字段

### 变量语法

邮件模板使用 `%%{variable}%%` 语法标识个性化变量：

```handlebars
{{!-- template.hbs 中的使用示例 --}}
<a href="%%{unsubscribe_url}%%">{{t 'Unsubscribe'}}</a>
<p class="%%{name_class}%%">{{t 'Name'}}: %%{name, "not provided"}%%</p>
<p>{{t 'Email'}}: <a href="#">%%{email}%%</a></p>
<span>{{{t "You are receiving this because you are a <strong>%%{status}%% subscriber</strong>..." }}}</span>
```

### 支持的变量

**文件**：`ghost/core/core/server/services/email-renderer.js:719`

```javascript
const baseDefinitions = [
    {
        id: 'unsubscribe_url',
        getValue: (member) => this.createUnsubscribeUrl(member.uuid, {newsletterUuid})
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
        id: 'key',  // HMAC 签名，用于验证链接
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
        id: 'name_class',  // 用于条件隐藏
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
        id: 'status',  // free | paid | comped | gift | trialing | complimentary
        getValue: (member) => {
            if (member.status === 'comped') return t('complimentary');
            if (this.isMemberTrialing(member)) return t('trialing');
            return t(member.status);
        }
    },
    {
        id: 'status_text',  // 订阅状态的完整描述（含日期）
        getValue: (member) => this.getMemberStatusText(member)
    },
    {
        id: 'list_unsubscribe',  // 邮件头 List-Unsubscribe
        getValue: (member) => this.createUnsubscribeUrl(member.uuid, {newsletterUuid}),
        required: true
    },
    {
        id: 'uniqueid',  // 绕过 ESP 图片代理缓存
        getValue: () => crypto.randomUUID()
    }
];
```

### Fallback 机制

**文件**：`ghost/core/core/server/services/email-renderer.js:813`

变量语法支持带默认值：`%%{name, "not provided"}%%`

```javascript
const REPLACEMENT_STRING_REGEX = /^(?<recipientProperty>\w+?)(?:,? *(?:"|&quot;)(?<fallback>.*?)(?:"|&quot;))?$/;

// 解析时提取 fallback
const match = replacementStr.match(REPLACEMENT_STRING_REGEX);
if (match) {
    const {recipientProperty, fallback} = match.groups;
    const definition = baseDefinitions.find(d => d.id === recipientProperty);
    
    if (definition) {
        replacements.push({
            id: replacementStr,
            token: new RegExp(...),
            // 有 fallback 时：值为空则返回 fallback
            getValue: fallback ? (member => definition.getValue(member) || fallback) : definition.getValue
        });
    }
}
```

### 构建时扫描

**文件**：`ghost/core/core/server/services/email-renderer.js:820`

只构建 HTML 中实际使用的变量定义，避免不必要的计算：

```javascript
const EMAIL_REPLACEMENT_REGEX = /%%\{(.*?)\}%%/g;

let result;
while ((result = EMAIL_REPLACEMENT_REGEX.exec(html)) !== null) {
    const [replacementMatch, replacementStr] = result;
    
    // 去重
    if (replacements.find(r => r.id === replacementStr)) {
        continue;
    }
    
    // 解析并添加到 replacements
}

// 必填变量即使未使用也强制添加
for (const definition of baseDefinitions) {
    if (definition.required && !replacements.find(r => r.id === definition.id)) {
        replacements.push(...);
    }
}
```

### 发送时绑定

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
                    value: def.getValue(member) || ''  // 为每个会员计算实际值
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

**应用分段**：`email-renderer.js:393`

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

async retryDb(func, options) {
    try {
        return await func();
    } catch (e) {
        if (retryCount >= options.maxRetries) {
            throw e;
        }
        await new Promise(resolve => setTimeout(resolve, sleep));
        // 指数退避：sleep = sleep * 2
        return await this.retryDb(func, {...options, retryCount: retryCount + 1, sleep: sleep * 2});
    }
}
```

### 缓存渲染结果

**文件**：`ghost/core/core/server/services/email-service/sending-service.js:119`

同一批次的会员共享相同的 HTML 模板，只在替换变量时差异化：

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

将 `<style>` 中的样式内联到 `style` 属性，因为：
- Gmail、Outlook 等客户端会移除 `<style>` 标签
- 内联样式兼容性最好

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

**模板**：`email-rendering/partials/email-wrapper.hbs:6`

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

**样式文件**：`email-templates/partials/styles.hbs:916`

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

### Locale 验证

**文件**：`ghost/core/core/server/services/email-renderer.js:66`

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
    locale = locale.trim();
    
    // "en" 简写不被 Intl 完全支持，降级到 en-gb
    if (locale === 'en' || !isValidLocale(locale)) {
        locale = DEFAULT_LOCALE;
    }
    
    return locale;
}
```

### 日期本地化

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

**文件**：`ghost/core/core/server/services/email-renderer.js:1124`

```javascript
const direction = this.#dir(locale);  // i18next 的 i18n.dir()

// 注入模板
site: {
    locale,
    direction  // 'rtl' 或 'ltr'
}
```

**模板应用**：`email-wrapper.hbs:2`

```handlebars
<html lang="{{site.locale}}" dir="{{site.direction}}">
```

**布局调整**：`styles.hbs:498`

```css
.manage-subscription {
    text-align: {{#if (eq site.direction "rtl")}}left{{else}}right{{/if}};
}
```

### 翻译文本

**文件**：`ghost/core/core/server/services/email-renderer.js:36`

```javascript
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
};

getMemberStatusText(member) {
    const t = this.#t;
    const locale = this.#getValidLocale();
    
    // 使用 t() 函数进行翻译，支持插值变量 {date}
    return t(messages.subscriptionStatus.active, {date: formattedDate});
}
```

---

## 关键文件索引

### 核心服务

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-service/email-renderer.js` | 邮件渲染核心（模板、变量、兼容处理） |
| `ghost/core/core/server/services/email-service/sending-service.js` | 发送协调（缓存、收件人构建） |
| `ghost/core/core/server/services/email-service/batch-sending-service.js` | 批量调度（分批、并发、重试） |

### 设计共享

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-rendering/email-design.js` | 后端设计计算逻辑 |
| `apps/admin-x-settings/src/components/settings/email-design/design-utils.ts` | 前端设计计算逻辑（镜像） |
| `ghost/core/core/server/models/newsletter.js` | Newsletter 模型与默认设计值 |

### 模板文件

| 文件 | 职责 |
|------|------|
| `ghost/core/core/server/services/email-rendering/partials/email-wrapper.hbs` | HTML 骨架 + Outlook 条件注释 |
| `ghost/core/core/server/services/email-service/email-templates/template.hbs` | 内容区域 + 页脚 |
| `ghost/core/core/server/services/email-service/email-templates/partials/styles.hbs` | 动态样式注入 |

### 测试文件

| 文件 | 职责 |
|------|------|
| `ghost/core/test/unit/server/services/email-service/batch-sending-service.test.js` | 批量发送单元测试 |
| `ghost/core/test/unit/server/services/email-service/sending-service.test.js` | 发送服务单元测试 |
| `ghost/core/test/unit/server/services/lib/email-content-generator.test.js` | 邮件内容生成测试 |
| `ghost/core/test/integration/services/email-service/batch-sending.test.js` | 批量发送集成测试 |
