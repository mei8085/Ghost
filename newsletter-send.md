# Ghost Newsletter 发送流程完整分析

本文档详细分析 Ghost 中 newsletter 从草稿生成到通过 Mailgun 批量投递再回流反馈的完整过程。

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [草稿生成与个性化变量渲染](#2-草稿生成与个性化变量渲染)
3. [批次切片策略](#3-批次切片策略)
4. [Mailgun 批量投递](#4-mailgun-批量投递)
5. [投递反馈回流机制](#5-投递反馈回流机制)
6. [退订处理流程](#6-退订处理流程)
7. [失败重试机制](#7-失败重试机制)
8. [数据模型与关键表](#8-数据模型与关键表)

---

## 1. 整体架构概览

### 1.1 核心服务组件

| 服务 | 文件路径 | 职责 |
|------|----------|------|
| **BatchSendingService** | `ghost/core/core/server/services/email-service/batch-sending-service.js` | 批次管理、并发控制、发送调度 |
| **SendingService** | `ghost/core/core/server/services/email-service/sending-service.js` | 邮件内容准备、收件人构建 |
| **MailgunEmailProvider** | `ghost/core/core/server/services/email-service/mailgun-email-provider.js` | Mailgun API 调用封装 |
| **EmailRenderer** | `ghost/core/core/server/services/email-service/email-renderer.js` | HTML/Plaintext 渲染、变量定义 |
| **EmailAnalyticsService** | `ghost/core/core/server/services/email-analytics/email-analytics-service.js` | 事件获取、聚合统计 |
| **EmailEventProcessor** | `ghost/core/core/server/services/email-service/email-event-processor.js` | 事件处理、收件人匹配 |
| **EmailEventStorage** | `ghost/core/core/server/services/email-service/email-event-storage.js` | 事件持久化、批量更新 |
| **MailgunEmailSuppressionList** | `ghost/core/core/server/services/email-suppression-list/mailgun-email-suppression-list.js` | 退订/投诉/硬弹处理 |
| **MailgunClient** | `ghost/core/core/server/services/lib/mailgun-client.js` | 底层 Mailgun API 客户端 |

### 1.2 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Newsletter 发送完整流程                                │
└─────────────────────────────────────────────────────────────────────────────┘

  [1. 草稿生成]                          [2. 批次切片]
       │                                      │
       ▼                                      ▼
  ┌──────────┐      ┌───────────┐      ┌──────────────┐      ┌───────────┐
  │ Post     │─────▶│  Email    │─────▶│ EmailBatch[] │─────▶│ 收件人分割 │
  │ (文章)   │      │ (邮件记录)│      │  (1000人/批) │      │ (分segment)│
  └──────────┘      └───────────┘      └──────────────┘      └───────────┘
                                                           │
                                                           ▼
  [3. Mailgun 投递]                      ┌──────────────────────────────┐
       │                                │ 个性化变量替换 (%%{name}%%)   │
       │                                │ 转为 Mailgun 变量语法         │
       ▼                                │ %recipient.name%             │
  ┌──────────────┐                      └──────────────────────────────┘
  │ Mailgun API  │◀─────────────────────────────┘
  │ /messages    │
  └──────┬───────┘
         │
         │ Mailgun 事件
         ▼
  [4. 反馈回流]
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │  EmailAnalyticsService (定时任务轮询)                    │
  │  - fetchLatestOpenedEvents()                            │
  │  - fetchLatestNonOpenedEvents()                         │
  │  - fetchMissing() (补漏)                                │
  └─────────────────────────────────────────────────────────┘
       │
       ▼
  [5. 事件处理]
       │
       ├──▶  delivered  ──▶ 更新 email_recipients.delivered_at
       ├──▶  opened     ──▶ 更新 email_recipients.opened_at
       ├──▶  failed     ──▶ 硬弹/软弹 ──▶ 写入 email_recipient_failures
       ├──▶  unsubscribed ──▶ 取消订阅 ──▶ 更新成员订阅关系
       └──▶  complained  ──▶ 投诉记录 ──▶ 写入 email_spam_complaint_events
       │
       ▼
  [6. 统计聚合]
       │
       ▼
  ┌─────────────────────────────────────────────────────────┐
  │  聚合到 emails 表:                                       │
  │  - delivered_count / opened_count / failed_count        │
  │  聚合到 members 表:                                      │
  │  - 邮件发送统计                                          │
  └─────────────────────────────────────────────────────────┘
```

---

## 2. 草稿生成与个性化变量渲染

### 2.1 邮件草稿创建

当用户在 Ghost 后台发布一篇文章并选择通过 newsletter 发送时，系统会创建一条 `Email` 记录：

```javascript
// email.js:7-19
defaults: function defaults() {
    return {
        uuid: crypto.randomUUID(),
        status: 'pending',           // 初始状态：待发送
        recipient_filter: 'status:-free',  // 默认发送给付费会员
        track_opens: false,
        track_clicks: false,
        feedback_enabled: false,
        delivered_count: 0,
        opened_count: 0,
        failed_count: 0,
        source_type: 'html'
    };
}
```

### 2.2 个性化变量定义

`EmailRenderer.buildReplacementDefinitions()` 方法负责构建可替换变量列表，支持以下变量：

| 变量 ID | 说明 | 示例值 |
|---------|------|--------|
| `uuid` | 会员唯一标识 | `a1b2c3d4-...` |
| `key` | 邮件验证 HMAC（SHA256） | `hex 字符串` |
| `first_name` | 名字（空格前第一部分） | `John` |
| `name` | 完整姓名 | `John Doe` |
| `name_class` | CSS class（有名则为空，否则为 'hidden'） | `''` 或 `'hidden'` |
| `email` | 邮箱地址 | `john@example.com` |
| `created_at` | 会员创建日期（格式化） | `10 May 2024` |
| `status` | 会员状态（free/paid/comped/gift/trialing） | `paid` |
| `status_text` | 订阅状态详细文本 | `Your subscription will renew...` |
| `unsubscribe_url` | 退订链接 | `https://.../unsub/...` |
| `manage_account_url` | 账户管理链接 | `https://.../#/portal/account` |
| `list_unsubscribe` | List-Unsubscribe 头（必需） | 同退订链接 |
| `uniqueid` | 随机 UUID（绕过 ESP 图片代理） | 每次生成随机值 |

### 2.3 变量替换语法

Ghost 使用 `%%{variable}%%` 语法支持带 fallback 的变量：

```
%%{name,"Friend"}%%      // 如果 name 为空，使用 "Friend"
%%{first_name}%%          // 无 fallback
```

在 `email-renderer.js:813-842` 中通过正则解析：

```javascript
const EMAIL_REPLACEMENT_REGEX = /%%\{(.*?)\}%%/g;
const REPLACEMENT_STRING_REGEX = /^(?<recipientProperty>\w+?)(?:,? *(?:"|&quot;)(?<fallback>.*?)(?:"|&quot;))?$/;
```

### 2.4 会员分段（Segments）

邮件内容可能因会员类型而异，支持以下分段：

- **`status:free`** - 免费会员
- **`status:-free`** - 付费/赠送会员
- **`null`** - 所有会员内容相同

分段检测逻辑在 `EmailRenderer.getSegments()` 中实现：

1. 检查是否有 `<!--members-only-->` 标记（付费墙）
2. 检查是否有 `data-gh-segment` 属性的元素

### 2.5 邮件渲染流程

```
renderBody(post, newsletter, segment, options)
    │
    ├──▶ 渲染文章基础 HTML (lexical/mobiledoc)
    │
    ├──▶ 处理付费墙 (paywall)
    │    ├── paid 文章 + free 会员: 显示 paywall + 移除付费内容
    │    └── 其他: 显示完整内容
    │
    ├──▶ 按 segment 移除不相关内容
    │    └── $('[data-gh-segment]').remove() if not match
    │
    ├──▶ 渲染 Handlebars 模板
    │    ├── base-styles.hbs
    │    ├── email-wrapper.hbs
    │    ├── template.hbs
    │    └── 其他 partials
    │
    ├──▶ 链接追踪 (click tracking)
    │    ├── 添加 ?m= 参数用于会员归因
    │    ├── 添加 newsletter ref 参数
    │    ├── --uuid-- 占位符 → %%{uuid}%%
    │    └── 外部链接替换
    │
    ├──▶ Juice CSS 内联
    │    └── 兼容 Outlook 等邮件客户端
    │
    ├──▶ 构建替换变量定义
    │    └── buildReplacementDefinitions()
    │
    └──▶ 生成 plaintext 版本
         └── htmlToPlaintext.email(html)
```

### 2.6 邮件头信息

```javascript
// mailgun-client.js:71-73
messageData['h:Sender'] = message.from;
messageData['h:Auto-Submitted'] = 'auto-generated';
messageData['h:X-Auto-Response-Suppress'] = 'OOF, AutoReply';

// mailgun-client.js:78-81
if (recipientData[...].list_unsubscribe) {
    messageData['h:List-Unsubscribe'] = '<%recipient.list_unsubscribe%>, <%tag_unsubscribe_email%>';
    messageData['h:List-Unsubscribe-Post'] = 'List-Unsubscribe=One-Click';
}
```

---

## 3. 批次切片策略

### 3.1 批次大小配置

```javascript
// mailgun-client.js:11
static DEFAULT_BATCH_SIZE = 1000;

// mailgun-client.js:395-397
getBatchSize() {
    return this.#config.get('bulkEmail')?.batchSize ?? this.DEFAULT_BATCH_SIZE;
}
```

**默认每批 1000 人**，可通过配置 `bulkEmail.batchSize` 调整。

### 3.2 批次创建流程 (`BatchSendingService.createBatches()`)

```
createBatches({email, post, newsletter})
    │
    ├──▶ 获取 segments (按会员类型)
    │    └── getSegments(post) → ['status:free', 'status:-free'] 或 [null]
    │
    ├──▶ 遍历每个 segment
    │    │
    │    ├──▶ 构建会员过滤器
    │    │    └── getMemberFilterForSegment(newsletter, recipient_filter, segment)
    │    │
    │    ├──▶ 分页获取会员 (按 ID 倒序，每次 BATCH_SIZE+1)
    │    │    └── 使用 ObjectId 特性：只获取 email 创建之前的会员
    │    │
    │    └──▶ 域预热 (Domain Warming) 处理
    │         ├── 自定义域名配额内 → useFallbackDomain: false
    │         └── 超出配额后 → useFallbackDomain: true (使用备用域名)
    │
    └──▶ 验证计数一致性
         └── email_count vs 实际会员数
```

### 3.3 域预热机制

当启用域名预热时，系统会控制通过自定义域名发送的邮件数量：

```javascript
// batch-sending-service.js:240-244
let domainWarmupLimit = Infinity;
if (this.#domainWarmingService.isEnabled()) {
    domainWarmupLimit = Number.isInteger(email.get('csd_email_count')) 
        ? email.get('csd_email_count') 
        : Infinity;
}
```

### 3.4 并发控制

```javascript
// batch-sending-service.js:10
const MAX_SENDING_CONCURRENCY = 2;

// batch-sending-service.js:466-467
// 同时最多 2 个批次在发送
await Promise.all(new Array(MAX_SENDING_CONCURRENCY).fill(0).map(() => runNext()));
```

### 3.5 定时投递窗口

支持配置 `targetDeliveryWindow`（秒），将批次均匀分布在指定时间窗口内：

```javascript
// batch-sending-service.js:756-773
calculateDeliveryTimes(email, numBatches) {
    const deadline = this.getDeliveryDeadline(email);
    if (!deadline || now >= deadline) {
        return new Array(numBatches).fill(undefined);  // 立即发送
    }
    const timeToDeadline = deadline.getTime() - now.getTime();
    const batchDelay = timeToDeadline / numBatches;
    // 每个批次间隔 batchDelay 发送
}
```

---

## 4. Mailgun 批量投递

### 4.1 变量语法转换

Ghost 变量 → Mailgun 变量语法转换：

```javascript
// mailgun-email-provider.js:59-67
#updateRecipientVariables(data, replacementDefinitions) {
    for (const def of replacementDefinitions) {
        data = data.replace(
            def.token,              // /%%\{name\}%%/g
            `%recipient.${def.id}%`  // %recipient.name%
        );
    }
    return data;
}
```

### 4.2 收件人数据构建

```javascript
// mailgun-email-provider.js:122-125
const recipientData = recipients.reduce((acc, recipient) => {
    acc[recipient.email] = this.#createRecipientData(recipient.replacements);
    return acc;
}, {});

// 结果格式:
// {
//   'john@example.com': {
//     name: 'John',
//     unsubscribe_url: 'https://...',
//     uuid: 'a1b2c3...',
//     ...
//   },
//   'jane@example.com': { ... }
// }
```

### 4.3 Mailgun API 调用

```javascript
// mailgun-client.js:63-74
messageData = {
    to: Object.keys(recipientData),           // 所有收件人邮箱
    from: message.from,
    'h:Reply-To': message.replyTo,
    subject: messageContent.subject,
    html: messageContent.html,
    text: messageContent.plaintext,
    'recipient-variables': JSON.stringify(recipientData),  // 个性化变量
    'h:Sender': message.from,
    'h:Auto-Submitted': 'auto-generated',
    'h:X-Auto-Response-Suppress': 'OOF, AutoReply'
};

// mailgun-client.js:84-86
if (message.id) {
    messageData['v:email-id'] = message.id;  // 关联 Ghost 的 email ID
}

// mailgun-client.js:88-92
const tags = ['bulk-email', 'ghost-email'];
if (bulkEmailConfig?.mailgun?.tag) {
    tags.push(bulkEmailConfig.mailgun.tag);
}
messageData['o:tag'] = tags;
```

### 4.4 追踪配置

```javascript
// mailgun-client.js:99-101
if (message.track_opens) {
    messageData['o:tracking-opens'] = true;
}
```

### 4.5 错误处理

```javascript
// mailgun-email-provider.js:149-176
try {
    const response = await this.#mailgunClient.send(...);
} catch (e) {
    ghostError = new errors.EmailError({
        statusCode: error.status,
        message: this.#createMailgunErrorMessage(error),
        errorDetails: JSON.stringify({error, messageData}),
        code: 'BULK_EMAIL_SEND_FAILED'
    });
    throw ghostError;
}
```

---

## 5. 投递反馈回流机制

### 5.1 事件获取方式：API 轮询 vs Webhook

Ghost **不使用 Webhook** 推送，而是通过**定时轮询** Mailgun Events API 获取投递反馈。这是一个重要的架构决策：

```javascript
// email-analytics-provider-mailgun.js:4
const DEFAULT_EVENT_FILTER = 'delivered OR opened OR failed OR unsubscribed OR complained';
```

**为什么选择轮询而非 Webhook**：
1. **可靠性**：轮询不依赖外部服务的推送可靠性，网络波动不会丢失事件
2. **最终一致性处理**：Mailgun Events API 有 30 分钟的稳定延迟，轮询+补漏机制更好处理
3. **幂等性**：通过时间游标控制拉取范围，天然支持断点续传和重复拉取

### 5.2 事件真实性校验机制

由于不使用 Webhook（无签名校验），事件真实性依赖以下多层校验：

#### 5.2.1 第一层：API 凭证认证

```javascript
// mailgun-client.js:360-377
getInstance() {
    const mailgunConfig = this.#getConfig();
    if (!mailgunConfig) {
        return null;
    }

    const formData = require('form-data');
    const Mailgun = require('mailgun.js').default;

    const baseUrl = new URL(mailgunConfig.baseUrl);
    const mailgun = new Mailgun(formData);

    return mailgun.client({
        username: 'api',
        key: mailgunConfig.apiKey,  // Mailgun API Key 认证
        url: baseUrl.origin,
        timeout: 60000
    });
}
```

**认证保障**：
- 使用配置的 `mailgun_api_key` 通过 HTTPS 调用 Mailgun API
- 只有持有有效 API Key 的请求才能获取事件数据
- 这是事件来源真实性的基础保障

#### 5.2.2 第二层：域范围过滤

```javascript
// mailgun-client.js:177-202
#fetchEventsFromDomain(domain, mailgunInstance, mailgunOptions, batchHandler, {maxEvents}) {
    // 从配置的域名获取事件
    let page = await this.getEventsFromMailgun(mailgunInstance, domain, mailgunOptions);
    ...
}

// mailgun-client.js:192-202
#getDomainsToFetch(mailgunConfig) {
    const domains = [mailgunConfig.domain];  // 主域名

    const fallbackDomain = this.#config.get('hostSettings:managedEmail:fallbackDomain');
    if (fallbackDomain && fallbackDomain !== mailgunConfig.domain) {
        domains.push(fallbackDomain);  // 备用域名（域预热场景）
    }

    return domains;  // 只从 Ghost 配置的域名拉取事件
}
```

**域范围保障**：
- 只从配置的 `mailgun_domain`（和可选的 fallbackDomain）拉取事件
- 不会获取到其他域名的事件
- 通过 `tags: 'bulk-email AND ghost-email'` 进一步过滤

```javascript
// email-analytics-provider-mailgun.js:10-16
constructor({config, settings, labs}) {
    this.mailgunClient = new MailgunClient({config, settings, labs});
    this.tags = [...DEFAULT_TAGS];  // ['bulk-email', 'ghost-email']

    if (config.get('bulkEmail:mailgun:tag')) {
        this.tags.push(config.get('bulkEmail:mailgun:tag'));
    }
}

// email-analytics-provider-mailgun.js:31-39
const mailgunOptions = {
    limit: PAGE_LIMIT,
    event: options?.events ? options.events.join(' OR ') : DEFAULT_EVENT_FILTER,
    tags: this.tags.join(' AND '),  // 标签过滤
    ...
};
```

#### 5.2.3 事件 ID 的真实使用：仅失败记录写入，未用于去重

**代码对账结论**：Ghost **没有**使用 `event.id` 做去重（如 `ON CONFLICT(event_id) DO NOTHING` 或存储已处理 event.id 的表）。

`event.id` 的唯一用途是写入 `email_recipient_failures.event_id` 字段：

```javascript
// email-event-storage.js:153
event_id: event.id

// email-event-storage.js:173
event_id: event.id
```

**去重依赖的真实机制**：

| 机制 | 实现方式 | 代码位置 |
|------|----------|----------|
| **时间游标** | `[lastEventTimestamp, end]` 窗口轮询，游标前进 | `queries.getLastEventTimestamp()` → `jobs.finished_at` |
| **WHERE IS NULL** | 只更新 NULL 字段，幂等写入 | `email-event-storage.js:54, 80, 106` |
| **事件时间戳比较** | 取最早时间戳，忽略更晚的重复事件 | `email-event-storage.js:45, 71, 97` |

**时间游标机制详解**：

```javascript
// queries.js:40-78
async getLastEventTimestamp(jobName, events = ['delivered', 'opened', 'failed']) {
    // 优先级：jobs.finished_at > jobs.started_at > 各字段 MAX()
    const lastJobRunTimestamp = await this.getLastJobRunTimestamp(jobName);

    if (lastJobRunTimestamp) {
        // 优先使用 jobs 表记录的游标
        maxOpenedAt = events.includes('opened') ? lastJobRunTimestamp : null;
        maxDeliveredAt = events.includes('delivered') ? lastJobRunTimestamp : null;
        maxFailedAt = events.includes('failed') ? lastJobRunTimestamp : null;
    } else {
        // 首次运行 fallback：查询各字段最大值
        maxOpenedAt = MAX(email_recipients.opened_at);
        maxDeliveredAt = MAX(email_recipients.delivered_at);
        maxFailedAt = MAX(email_recipients.failed_at);
    }

    return _.max([maxOpenedAt, maxDeliveredAt, maxFailedAt]);
}

// queries.js:109-128
async setJobTimestamp(jobName, field, date) {
    // 更新 jobs 表游标
    await db.knex('jobs')
        .update({[updateField]: date, status: status})
        .where('name', jobName);
}
```

**时间游标前进逻辑**：
```
[lastEventTs] ────────────────▶ [end = now - 1min]
                    │
                    ▼
      拉取此时间窗口内的所有事件
                    │
                    ▼
        事件消费完？
            │
            ├── 是 ──▶ 游标前进 = lastEventTs + 1 秒（或到处理的最大事件时间）
            └── 否 ──▶ 游标不前进，下次继续处理同窗口
```

**可能的重复处理边界**：

| 场景 | 重复风险 | 保护机制 |
|------|----------|----------|
| 同一秒内的事件跨 job 边界 | **可能重复拉取** | `WHERE xxx_at IS NULL` |
| `fetchLatest` 和 `fetchMissing` 时间窗口重叠 | **同事件被两个 job 拉取** | `WHERE xxx_at IS NULL` |
| job 中途崩溃，游标未前进 | **整个窗口重新处理** | `WHERE xxx_at IS NULL` |

**结论**：事件去重完全依赖**幂等写入**（`WHERE IS NULL` + 时间戳比较），而非 event.id 级别去重。

#### 5.2.4 第四层：事件关联校验

事件必须能关联到本地已有的邮件记录才算有效：

```javascript
// mailgun-client.js:304-328
normalizeEvent(event) {
    const providerId = event?.message?.headers['message-id'];

    // 必须有 email-id (user-variables) 或 message-id (providerId)
    if (!providerId && !(event['user-variables'] && event['user-variables']['email-id'])) {
        logging.error('Received invalid event from Mailgun');
        logging.error(event);
        return null;  // 丢弃无法关联的事件
    }

    return {
        id: event.id,
        type: event.event,
        severity: event.severity,
        recipientEmail: event.recipient,
        emailId: event['user-variables']?.['email-id'],
        providerId: providerId,
        timestamp: new Date(event.timestamp * 1000),
        ...
    };
}
```

**关联校验**：
- 事件必须包含 `v:email-id`（发送时注入的 user-variable）或 `message-id`
- 后续处理中会进一步校验 emailId 是否在本地 `emails` 表存在

#### 5.2.5 第五层：乱序保护（时间戳校验）

Mailgun 事件可能乱序到达，Ghost 通过多重机制保护：

```javascript
// email-event-storage.js:36-59
async handleDelivered(event) {
    const useBatchProcessing = config.get('emailAnalytics:batchProcessing');

    if (useBatchProcessing) {
        // 内存层：保持最早时间戳
        const timestamp = moment.utc(event.timestamp).format('YYYY-MM-DD HH:mm:ss');
        const existing = this.#pendingUpdates.delivered.get(event.emailRecipientId);
        
        // 如果已有更新，只保留更早的时间戳
        if (!existing || timestamp < existing) {
            this.#pendingUpdates.delivered.set(event.emailRecipientId, timestamp);
        }
    } else {
        // 数据库层：只更新 NULL 值
        const rowCount = await this.#db.knex('email_recipients')
            .where('id', '=', event.emailRecipientId)
            .whereNull('delivered_at')  // 关键：已设置则不覆盖
            .update({
                delivered_at: moment.utc(event.timestamp).format('YYYY-MM-DD HH:mm:ss')
            });
    }
}
```

**乱序保护**：
1. **内存层**：Map 中只保留最早时间戳
2. **数据库层**：`WHERE xxx_at IS NULL` 确保只写入一次
3. **事件时间戳**：`failed_at` 更新时也比较 `existing.get('failed_at') > event.timestamp`

### 5.3 定时任务类型与时间窗口策略

| 任务 | Job Name | 频率 | 事件类型 | 时间窗口 | 说明 |
|------|----------|------|----------|----------|------|
| `fetchLatestOpenedEvents` | `email-analytics-latest-opened` | 高频 | opened | `[lastEventTs, now-1min]` | 打开事件频繁，快速更新 |
| `fetchLatestNonOpenedEvents` | `email-analytics-latest-others` | 高频 | delivered, failed, unsubscribed, complained | `[lastEventTs, now-1min]` | 其他状态事件 |
| `fetchMissing` | `email-analytics-missing` | 低频 | 全部 | `[lastJobTs, min(now-30min, lastFetchBegin)]` | 补漏（30分钟后 Mailgun 存储稳定） |
| `fetchScheduled` | `email-analytics-scheduled` | 按需 | 全部 | 手动指定 | 历史数据回溯 |

```javascript
// email-analytics-service.js:38-39
const TRUST_THRESHOLD_MS = 30 * 60 * 1000;  // 30分钟 - Mailgun 存储稳定时间
const FETCH_LATEST_END_MARGIN_MS = 1 * 60 * 1000;  // 1分钟 - 避免秒级边界

// email-analytics-service.js:165-175
async fetchLatestOpenedEvents({maxEvents = Infinity} = {}) {
    const begin = await this.getLastOpenedEventTimestamp();
    const end = new Date(Date.now() - FETCH_LATEST_END_MARGIN_MS);  // 只拉取 1 分钟前

    if (end <= begin) {
        return createEmptyResult();  // 时间窗口无效则跳过
    }
    return await this.#fetchEvents(this.#fetchLatestOpenedData, {begin, end, maxEvents, eventTypes: ['opened']});
}

// email-analytics-service.js:203-221
async fetchMissing({maxEvents = Infinity} = {}) {
    const begin = await this.getLastMissingEventTimestamp();
    
    // 结束时间取「30分钟前」和「上一次 fetchLatest 开始时间」的较小值
    const end = new Date(
        Math.min(
            Date.now() - TRUST_THRESHOLD_MS,
            this.#fetchLatestNonOpenedData?.lastBegin?.getTime() || Date.now()
        )
    );

    if (end <= begin) {
        return createEmptyResult();
    }
    return await this.#fetchEvents(this.#fetchMissingData, {begin, end, maxEvents});
}
```

**双窗口设计原因**：
- `fetchLatest` 快速获取新事件（延迟 1 分钟）
- `fetchMissing` 在 30 分钟后回扫补漏（此时 Mailgun 数据已稳定，不会遗漏）

### 5.4 事件规范化

```javascript
// mailgun-client.js:304-328
normalizeEvent(event) {
    const providerId = event?.message?.headers['message-id'];

    if (!providerId && !(event['user-variables'] && event['user-variables']['email-id'])) {
        logging.error('Received invalid event from Mailgun');
        logging.error(event);
        return null;
    }

    return {
        id: event.id,                              // Mailgun 事件唯一 ID
        type: event.event,                         // delivered / opened / failed / unsubscribed / complained
        severity: event.severity,                  // permanent / temporary (仅 failed)
        recipientEmail: event.recipient,           // 收件人邮箱
        emailId: event['user-variables']?.['email-id'],  // Ghost 的 email ID（发送时注入）
        providerId: providerId,                    // Mailgun message-id（批次级别）
        timestamp: new Date(event.timestamp * 1000),  // Mailgun Unix 时间戳 → Date
        error: event['delivery-status'] ? {
            code: event['delivery-status'].code,
            message: (event['delivery-status'].message || event['delivery-status'].description).substring(0, 2000),
            enhancedCode: event['delivery-status']['enhanced-code']?.toString()?.substring(0, 50) ?? null
        } : null
    };
}
```

### 5.5 事件匹配与归并：email-id 缺失时的 provider_id 回查

#### 5.5.1 关联键的两种来源

发送时 Ghost 注入了两种关联键：

```javascript
// mailgun-client.js:84-86
if (message.id) {
    messageData['v:email-id'] = message.id;  // 1. User Variable: email-id（邮件级别）
}

// Mailgun 返回时还带：
// 2. Message-Id Header: <20240510...@mg.example.com>（批次级别）
```

#### 5.5.2 收件人匹配策略

```javascript
// email-event-processor.js:208-244
async getRecipient(emailIdentification, recipientCache) {
    // 前置校验：必须有 email 且有 emailId 或 providerId
    if (!emailIdentification.emailId && !emailIdentification.providerId) {
        return;  // 无法关联，丢弃事件
    }

    // 1. 优先使用 emailId (v:email-id)，否则通过 providerId 回查
    const emailId = emailIdentification.emailId 
        ?? await this.getEmailId(emailIdentification.providerId);
    
    if (!emailId) {
        return;  // 回查失败，丢弃
    }

    // 2. 查缓存（批量模式）
    if (recipientCache) {
        const key = `${emailIdentification.email}:${emailId}`;
        const cached = recipientCache.get(key);
        if (cached) {
            return cached;
        }
    }

    // 3. 通过 (member_email, email_id) 二元组定位收件人
    const {id: emailRecipientId, member_id: memberId} = await this.#db.knex('email_recipients')
        .select('id', 'member_id')
        .where('member_email', emailIdentification.email)
        .where('email_id', emailId)
        .first() || {};

    if (emailRecipientId && memberId) {
        return {
            emailRecipientId,  // email_recipients 表主键
            memberId,          // members 表主键
            emailId            // emails 表主键
        };
    }
}
```

#### 5.5.3 provider_id 回查机制（email-id 缺失时）

```javascript
// email-event-processor.js:260-281
async getEmailId(providerId) {
    // 1. 内存缓存：避免重复查询
    if (this.providerIdEmailIdMap[providerId]) {
        return this.providerIdEmailIdMap[providerId];
    }

    // 2. 数据库查询：email_batches 表存储了 provider_id → email_id 映射
    const {emailId} = await this.#db.knex('email_batches')
        .select('email_id as emailId')
        .where('provider_id', providerId)
        .first() || {};

    if (!emailId) {
        return;  // 查不到，可能是测试邮件或过期数据
    }

    // 3. 写入缓存
    this.providerIdEmailIdMap[providerId] = emailId;
    return emailId;
}
```

**关联链**：
```
Mailgun Event
    │
    ├──▶ v:email-id (user-variable) ──▶ emails.id ──┐
    │                                               ├──▶ email_recipients
    └──▶ Message-Id (provider_id) ──▶ email_batches.email_id ──┘
                                                │
                                                └──▶ (member_email, email_id) 定位具体收件人
```

#### 5.5.4 批量查询优化

```javascript
// email-event-processor.js:288-356
async batchGetRecipients(emailIdentifications) {
    const recipientCache = new Map();

    // Step 1: 批量解析 providerId → emailId（一次查询所有）
    const providerIds = [...new Set(
        emailIdentifications
            .filter(e => e.providerId && !e.emailId)
            .map(e => e.providerId)
    )];

    if (providerIds.length > 0) {
        const providerIdMapping = await this.#db.knex('email_batches')
            .select('provider_id', 'email_id')
            .whereIn('provider_id', providerIds);  // 批量 IN 查询

        for (const row of providerIdMapping) {
            this.providerIdEmailIdMap[row.provider_id] = row.email_id;
        }
    }

    // Step 2: 构建所有 (email, emailId) 查找对
    const lookups = [];
    for (const identification of emailIdentifications) {
        const emailId = identification.emailId ?? this.providerIdEmailIdMap[identification.providerId];
        if (emailId && identification.email) {
            lookups.push({email: identification.email, emailId});
        }
    }

    // Step 3: 构建 OR 查询，批量获取所有 recipient
    const recipientQuery = this.#db.knex('email_recipients')
        .select('id', 'member_id', 'email_id', 'member_email');

    recipientQuery.where(function () {
        for (const lookup of lookups) {
            this.orWhere(function () {
                this.where('member_email', lookup.email)
                    .andWhere('email_id', lookup.emailId);
            });
        }
    });

    const recipients = await recipientQuery;

    // Step 4: 构建 Map<email:emailId, recipientInfo> 缓存
    for (const recipient of recipients) {
        const key = `${recipient.member_email}:${recipient.email_id}`;
        recipientCache.set(key, {
            emailRecipientId: recipient.id,
            memberId: recipient.member_id,
            emailId: recipient.email_id
        });
    }

    return recipientCache;
}
```

**性能优化**：
- 避免 N+1 查询：所有 provider_id 一次 IN 查询
- 所有收件人一次 OR 查询
- 内存 Map 缓存，O(1) 查找

### 5.6 事件存储与批量更新

#### 5.6.1 批量处理模式

```javascript
// email-event-storage.js:21-25
this.#pendingUpdates = {
    delivered: new Map(),  // emailRecipientId → timestamp string
    opened: new Map(),
    failed: new Map()
};
```

#### 5.6.2 批量 SQL 更新（CASE 语句）

```javascript
// email-event-storage.js:277-298
async #flushDeliveredUpdates() {
    const updates = Array.from(this.#pendingUpdates.delivered.entries());
    
    // 构建 CASE 语句
    const recipientIds = updates.map(([id]) => id);
    const caseClauses = updates.map(([id, timestamp]) => {
        return `WHEN '${id}' THEN '${timestamp}'`;
    }).join(' ');

    const sql = `
        UPDATE email_recipients
        SET delivered_at = CASE id ${caseClauses} END
        WHERE id IN (${recipientIds.map(() => '?').join(',')})
        AND delivered_at IS NULL  -- 数据库层乱序保护
    `;

    const rowCount = await this.#db.knex.raw(sql, recipientIds);
    this.recordEventStored('delivered', updates.length);
    return rowCount;
}
```

#### 5.6.3 三层乱序保护

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| **内存层** | Map 中只保留最早时间戳 | `email-event-storage.js:44-47` |
| **SQL 层** | `WHERE delivered_at IS NULL` | `email-event-storage.js:54` |
| **失败记录层** | 比较时间戳，永久失败不可覆盖 | `email-event-storage.js:156-164` |

```javascript
// email-event-storage.js:125-176 失败记录的乱序保护
async saveFailure(severity, event, options) {
    const existing = await this.#models.EmailRecipientFailure.findOne({
        email_recipient_id: event.emailRecipientId
    }, {...options, require: false, forUpdate: true});

    if (!existing) {
        // 新建失败记录
        await this.#models.EmailRecipientFailure.add({...});
    } else {
        if (existing.get('severity') === 'permanent') {
            // 已永久失败 → 不再更新
            return;
        }

        if (existing.get('failed_at') > event.timestamp) {
            // 新事件时间更早 → 忽略（乱序保护）
            return;
        }

        // 可以更新（temporary → permanent，或更新为更近的 temporary）
        await existing.save({
            severity,
            message: event.error.message,
            ...
        }, {...options, patch: true});
    }
}
```

### 5.7 事件类型处理：Temporary vs Permanent Failed

#### 5.7.1 事件处理矩阵

| 事件类型 | severity | 处理方法 | email_recipients | email_recipient_failures | 本地统计 |
|----------|----------|----------|------------------|--------------------------|----------|
| **delivered** | - | `handleDelivered()` | `delivered_at = ts` | - | `delivered_count++` |
| **opened** | - | `handleOpened()` | `opened_at = ts` | - | `opened_count++` |
| **failed** | `permanent` | `handlePermanentFailed()` | `failed_at = ts` | 插入 | `failed_count++` |
| **failed** | `temporary` | `handleTemporaryFailed()` | - | 插入 | **不增加** |
| **unsubscribed** | - | `handleUnsubscribed()` | - | - | - |
| **complained** | - | `handleComplained()` | - | - | - |

#### 5.7.2 Permanent Failed（硬弹）- 计入统计

```javascript
// email-event-storage.js:88-112
async handlePermanentFailed(event) {
    const useBatchProcessing = config.get('emailAnalytics:batchProcessing');

    if (useBatchProcessing) {
        // 1. 更新 email_recipients.failed_at（计入统计）
        const timestamp = moment.utc(event.timestamp).format('YYYY-MM-DD HH:mm:ss');
        const existing = this.#pendingUpdates.failed.get(event.emailRecipientId);
        if (!existing || timestamp < existing) {
            this.#pendingUpdates.failed.set(event.emailRecipientId, timestamp);
        }
    } else {
        // 直接更新，只写 NULL
        await this.#db.knex('email_recipients')
            .where('id', '=', event.emailRecipientId)
            .whereNull('failed_at')
            .update({failed_at: timestamp});
    }
    
    // 2. 写入详细失败记录
    await this.saveFailure('permanent', event);
}
```

#### 5.7.3 Temporary Failed（软弹）- 不计入统计

```javascript
// email-event-storage.js:114-116
async handleTemporaryFailed(event) {
    // 仅写入失败记录，不更新 email_recipients.failed_at
    await this.saveFailure('temporary', event);
}
```

**设计原因**：
- **Permanent**：邮箱不存在、域名不存在等致命错误 → 计入失败率
- **Temporary**：收件箱满、连接超时、灰名单等临时错误 → Mailgun 会自动重试，不计入本地失败统计

#### 5.7.4 统计聚合逻辑

```javascript
// queries.js:207-222
async aggregateEmailStats(emailId, updateOpenedCount) {
    // delivered_count = COUNT(email_recipients WHERE delivered_at IS NOT NULL)
    const [deliveredCount] = await db.knex('email_recipients')
        .count('id as count')
        .whereRaw('email_id = ? AND delivered_at IS NOT NULL', [emailId]);
    
    // failed_count = COUNT(email_recipients WHERE failed_at IS NOT NULL)
    // 注意：只有 permanent failed 会设置 failed_at
    const [failedCount] = await db.knex('email_recipients')
        .count('id as count')
        .whereRaw('email_id = ? AND failed_at IS NOT NULL', [emailId]);

    const updateData = {
        delivered_count: deliveredCount.count,
        failed_count: failedCount.count  // 只统计 permanent failed
    };

    if (updateOpenedCount) {
        const [openedCount] = await db.knex('email_recipients')
            .count('id as count')
            .whereRaw('email_id = ? AND opened_at IS NOT NULL', [emailId]);
        updateData.opened_count = openedCount.count;
    }

    await db.knex('emails').update(updateData).where('id', emailId);
}
```

#### 5.7.5 失败重试截止判断

邮件发送阶段的重试与邮件投递后的失败是两个不同概念：

**发送阶段重试**（BatchSendingService）：
```javascript
// batch-sending-service.js:151-158
// 计算重试截止时间
const expectedBatchCount = Math.ceil(email.get('email_count') / 1000);
const minimumSecondsPerBatch = 26;  // 每批预估最低耗时
const stopAfter = Math.max(
    expectedBatchCount * minimumSecondsPerBatch * 1000,
    this.#BEFORE_RETRY_CONFIG.maxTime  // 至少 10 分钟
);
email._retryCutOffTime = new Date(startTime + stopAfter);

// 在重试配置中应用截止时间
#getBeforeRetryConfig(email) {
    if (email._retryCutOffTime) {
        return {...this.#BEFORE_RETRY_CONFIG, stopAfterDate: email._retryCutOffTime};
    }
    return this.#BEFORE_RETRY_CONFIG;
}
```

**投递后失败**（Mailgun 处理）：
- **Temporary Failed**：Mailgun 会自动重试（通常重试 8 小时，间隔递增）
- **Permanent Failed**：Mailgun 停止重试，Ghost 计入 `failed_count`
- Ghost 不主动重试已提交给 Mailgun 的邮件

### 5.8 EventProcessingResult 事件计数

```javascript
// event-processing-result.js:1-64
class EventProcessingResult {
    constructor(result = {}) {
        this.delivered = 0;
        this.opened = 0;
        this.temporaryFailed = 0;   // 软弹单独计数
        this.permanentFailed = 0;   // 硬弹单独计数
        this.unsubscribed = 0;
        this.complained = 0;
        this.unhandled = 0;
        this.unprocessable = 0;     // 无法关联的事件
        
        this.emailIds = [];    // 需要聚合的 email ID
        this.memberIds = [];   // 需要聚合的 member ID
    }

    get totalEvents() {
        return this.delivered
            + this.opened
            + this.temporaryFailed
            + this.permanentFailed
            + this.unsubscribed
            + this.complained
            + this.unhandled
            + this.unprocessable;
    }
}
```

### 5.9 统计聚合触发时机

```javascript
// email-analytics-service.js:424-444
// 每 5 分钟或 5000 个会员触发一次中间聚合
if ((Date.now() - lastAggregation > 5 * 60 * 1000 
    || processingResult.memberIds.length > 5000) 
    && eventCount > 0) {
    
    const aggregationTimings = await this.aggregateStats(processingResult, includeOpenedEvents);
    
    // 清空已聚合的 ID，避免重复聚合
    processingResult.emailIds.forEach(id => allEmailIds.delete(id));
    processingResult.memberIds.forEach(id => allMemberIds.delete(id));
    processingResult = new EventProcessingResult();
}
```

聚合内容：
- **emails 表**：`delivered_count`（permanent+delivered? 不，只有 delivered）、`opened_count`、`failed_count`（仅 permanent）
- **members 表**：`email_count`、`email_opened_count`、`email_open_rate`（需 ≥5 封追踪邮件才计算）

---

## 6. 退订处理流程

### 6.1 退订入口

邮件中包含多重退订机制：

1. **邮件内退订链接**（`%%{unsubscribe_url}%%`）
2. **List-Unsubscribe Header**（一键退订）
3. **Mailgun List-Unsubscribe Email**（`<mailto:...>`）

### 6.2 退订事件处理

```javascript
// email-event-storage.js:178-189
async handleUnsubscribed(event) {
    try {
        // 1. 查找该会员需要保留的 newsletter（排除当前这个）
        const newsletters = await this.findNewslettersToKeep(event);
        
        // 2. 更新会员订阅关系
        await this.#membersRepository.update({newsletters}, {id: event.memberId});
        
        // 3. 从 Mailgun suppression list 移除（避免影响同域名其他站点）
        await this.#emailSuppressionList.removeUnsubscribe(event.email);
    } catch (err) {
        logging.error(err);
    }
}
```

### 6.3 保留其他 Newsletter

```javascript
// email-event-storage.js:208-225
async findNewslettersToKeep(event) {
    // 获取会员当前订阅的所有 newsletter
    const existingNewsletters = member.related('newsletters');
    
    // 获取当前邮件对应的 newsletter
    const email = await this.#models.Email.findOne({id: event.emailId});
    const newsletterToRemove = email.get('newsletter_id');
    
    // 过滤掉当前 newsletter，保留其他
    return existingNewsletters.models
        .filter(n => n.id !== newsletterToRemove)
        .map(n => ({id: n.id}));
}
```

### 6.4 垃圾邮件投诉处理

```javascript
// email-event-storage.js:191-206
async handleComplained(event) {
    // 1. 记录投诉事件
    await this.#models.EmailSpamComplaintEvent.add({
        member_id: event.memberId,
        email_id: event.emailId,
        email_address: event.email
    });
    
    // 2. 从 Mailgun suppression list 移除
    await this.#emailSuppressionList.removeComplaint(event.email);
}
```

### 6.5 Suppression List 同步

```javascript
// mailgun-email-suppression-list.js:117-152
async init() {
    // 监听硬弹和投诉事件
    DomainEvents.subscribe(EmailBouncedEvent, handleEvent('bounce'));
    DomainEvents.subscribe(SpamComplaintEvent, handleEvent('spam'));
}

// handleEvent 内部:
await this.Suppression.add({
    email: event.email,
    email_id: event.emailId,
    reason: reason,  // 'bounce' 或 'spam'
    created_at: event.timestamp
});
```

---

## 7. 失败重试机制

### 7.1 多层重试策略

```javascript
// batch-sending-service.js:37-39
#BEFORE_RETRY_CONFIG = {maxRetries: 10, maxTime: 10 * 60 * 1000, sleep: 2000};  // 发送前
#AFTER_RETRY_CONFIG = {maxRetries: 20, maxTime: 30 * 60 * 1000, sleep: 2000};     // 发送后
#MAILGUN_API_RETRY_CONFIG = {sleep: 10 * 1000, maxRetries: 6};                     // Mailgun API
```

### 7.2 指数退避重试

```javascript
// batch-sending-service.js:675-728
async retryDb(func, options) {
    try {
        return await func();
    } catch (e) {
        if (retryCount >= options.maxRetries || ...) {
            throw e;  // 超过最大重试次数
        }
        
        // 指数退避: sleep * 2^retryCount
        await new Promise(resolve => setTimeout(resolve, sleep));
        return await this.retryDb(func, {
            ...options, 
            retryCount: retryCount + 1, 
            sleep: sleep * 2
        });
    }
}
```

### 7.3 发送前重试（数据库操作）

包括：
- `updateStatusLock` - 状态锁（确保只有一个 job 在处理）
- `getLazyRelation` - 加载关联数据
- `getBatches` / `createBatch` - 批次操作
- `getBatchMembers` - 获取批次会员

### 7.4 发送后重试（状态持久化）

包括：
- Email 状态更新 (`pending` → `submitting` → `submitted`/`failed`)
- EmailBatch 状态更新
- EmailRecipient `processed_at` 更新

### 7.5 Mailgun API 重试

```javascript
// batch-sending-service.js:522-536
const response = await this.retryDb(async () => {
    return await this.#sendingService.send({...}, {...});
}, {...this.#MAILGUN_API_RETRY_CONFIG, ...});
```

### 7.6 状态锁机制

```javascript
// batch-sending-service.js:646-659
async updateStatusLock(Model, id, status, allowedStatuses) {
    await Model.transaction(async (transacting) => {
        // 1. 行锁 (FOR UPDATE)
        model = await Model.findOne({id}, {require: true, transacting, forUpdate: true});
        
        // 2. 检查当前状态
        if (!allowedStatuses.includes(model.get('status'))) {
            model = undefined;  // 其他进程已在处理
            return;
        }
        
        // 3. 更新状态
        await model.save({status}, {patch: true, transacting, autoRefresh: false});
    });
    return model;
}
```

状态流转：
```
Email: pending ──▶ submitting ──▶ submitted (成功)
                          │
                          └───▶ failed (失败)

EmailBatch: pending ──▶ submitting ──▶ submitted (成功)
                               │
                               └───▶ failed (失败)
```

### 7.7 失败记录

```javascript
// email-event-storage.js:125-176
async saveFailure(severity, event, options) {
    const existing = await this.#models.EmailRecipientFailure.findOne({
        email_recipient_id: event.emailRecipientId
    }, {...options, forUpdate: true});
    
    if (!existing) {
        // 新建失败记录
        await this.#models.EmailRecipientFailure.add({
            email_id: event.emailId,
            member_id: event.memberId,
            email_recipient_id: event.emailRecipientId,
            severity,              // 'permanent' 或 'temporary'
            message: event.error.message,
            code: event.error.code,
            enhanced_code: event.error.enhancedCode,
            failed_at: event.timestamp,
            event_id: event.id
        });
    } else {
        // 已永久失败则忽略；否则更新为最新失败
        if (existing.get('severity') === 'permanent') return;
        if (existing.get('failed_at') > event.timestamp) return;
        await existing.save({...}, {patch: true});
    }
}
```

### 7.8 重试截止时间

```javascript
// batch-sending-service.js:151-158
const expectedBatchCount = Math.ceil(email.get('email_count') / 1000);
const minimumSecondsPerBatch = 26;
const stopAfter = Math.max(
    expectedBatchCount * minimumSecondsPerBatch * 1000,
    this.#BEFORE_RETRY_CONFIG.maxTime
);
email._retryCutOffTime = new Date(startTime + stopAfter);
```

---

## 8. 数据模型与关键表

### 8.1 核心表结构

```
┌─────────────────────────────────────────────────────────────────┐
│                           emails                                 │
├─────────────────────────────────────────────────────────────────┤
│ id                │ ObjectId, PK                                │
│ uuid              │ varchar(36), unique                         │
│ post_id           │ FK → posts                                  │
│ newsletter_id     │ FK → newsletters                            │
│ status            │ enum: pending/submitting/submitted/failed   │
│ recipient_filter  │ NQL, e.g., 'status:-free'                   │
│ email_count       │ 预计收件人数                                 │
│ csd_email_count   │ 自定义域名发送数量                           │
│ track_opens       │ boolean                                      │
│ track_clicks      │ boolean                                      │
│ delivered_count   │ 已送达数量                                   │
│ opened_count      │ 已打开数量                                   │
│ failed_count      │ 失败数量                                     │
│ submitted_at      │ datetime                                     │
│ created_at        │ datetime                                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        email_batches                             │
├─────────────────────────────────────────────────────────────────┤
│ id                    │ ObjectId, PK                            │
│ email_id              │ FK → emails                             │
│ member_segment        │ 'status:free' | 'status:-free' | null   │
│ status                │ enum: pending/submitting/submitted/failed│
│ provider_id           │ Mailgun message-id (批次级别)            │
│ fallback_sending_domain │ boolean                                │
│ error_status_code     │ int                                      │
│ error_message         │ text                                     │
│ error_data            │ json                                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      email_recipients                            │
├─────────────────────────────────────────────────────────────────┤
│ id                    │ ObjectId, PK                            │
│ email_id              │ FK → emails                             │
│ batch_id              │ FK → email_batches                      │
│ member_id             │ FK → members                            │
│ member_uuid           │ varchar(36)                             │
│ member_email          │ varchar(191)                            │
│ member_name           │ varchar(191)                            │
│ processed_at          │ datetime (批次处理时间)                  │
│ delivered_at          │ datetime (送达时间)                      │
│ opened_at             │ datetime (打开时间)                      │
│ failed_at             │ datetime (失败时间)                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   email_recipient_failures                       │
├─────────────────────────────────────────────────────────────────┤
│ id                    │ ObjectId, PK                            │
│ email_id              │ FK → emails                             │
│ member_id             │ FK → members                            │
│ email_recipient_id    │ FK → email_recipients, unique           │
│ severity              │ enum: permanent / temporary             │
│ code                  │ int (SMTP error code)                    │
│ enhanced_code         │ varchar(50)                             │
│ message               │ varchar(2000)                           │
│ failed_at             │ datetime                                 │
│ event_id              │ varchar(255) (Mailgun event id)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  email_spam_complaint_events                     │
├─────────────────────────────────────────────────────────────────┤
│ id              │ ObjectId, PK                                  │
│ member_id       │ FK → members                                  │
│ email_id        │ FK → emails                                   │
│ email_address   │ varchar(191)                                  │
│ created_at      │ datetime                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 关联查询

通过 `provider_id` 关联 Mailgun 批次和 Ghost 批次：

```sql
-- 通过 Mailgun message-id 找到对应的 email
SELECT email_id 
FROM email_batches 
WHERE provider_id = '<mailgun-message-id>';

-- 通过 (email, email_id) 找到具体收件人
SELECT id, member_id 
FROM email_recipients 
WHERE member_email = 'john@example.com' 
  AND email_id = '<email-id>';
```

---

## 9. 配置项

### 9.1 Mailgun 配置

```javascript
// config 或 settings:
bulkEmail: {
    mailgun: {
        apiKey: 'xxx',
        domain: 'mg.example.com',
        baseUrl: 'https://api.mailgun.net',
        tag: 'optional-tag',
        testmode: false
    },
    batchSize: 1000,              // 每批人数
    targetDeliveryWindow: 0       // 投递窗口（秒），0 = 立即发送
}
```

### 9.2 邮件分析配置

```javascript
emailAnalytics: {
    batchProcessing: true  // 是否启用批量更新模式
}
```

---

## 10. 关键代码位置速查

| 功能模块 | 文件路径 | 关键方法 |
|----------|----------|----------|
| 批次管理 | `batch-sending-service.js` | `createBatches()`, `sendBatches()`, `sendBatch()` |
| 个性化渲染 | `email-renderer.js` | `renderBody()`, `buildReplacementDefinitions()`, `getSegments()` |
| 发送服务 | `sending-service.js` | `send()`, `buildRecipients()` |
| Mailgun 集成 | `mailgun-email-provider.js` | `send()` |
| Mailgun 客户端 | `mailgun-client.js` | `send()`, `fetchEvents()`, `normalizeEvent()` |
| 事件获取 | `email-analytics-service.js` | `fetchLatestOpenedEvents()`, `fetchMissing()`, `processEvent()` |
| 事件处理 | `email-event-processor.js` | `handleDelivered()`, `getRecipient()`, `batchGetRecipients()` |
| 事件存储 | `email-event-storage.js` | `handleDelivered()`, `flushBatchedUpdates()`, `handleUnsubscribed()` |
| 退订列表 | `mailgun-email-suppression-list.js` | `init()`, `removeUnsubscribe()` |

---

## 总结

Ghost 的 newsletter 发送系统是一个设计完善的批量邮件系统，具有以下特点：

1. **灵活的分段渲染** - 支持免费/付费会员差异化内容
2. **丰富的个性化变量** - 12+ 内置变量，支持 fallback
3. **批次化投递** - 每批 1000 人，并发控制，支持域预热
4. **定时投递窗口** - 可配置在指定时间窗口内均匀发送
5. **事件轮询机制** - 不依赖 Webhook，通过轮询+补漏保证数据完整性
6. **批量更新优化** - 使用 SQL CASE 语句批量更新，支持乱序保护
7. **多层重试策略** - 指数退避，状态锁防止重复处理
8. **完善的退订/投诉处理** - List-Unsubscribe、一键退订、投诉记录

整个系统设计充分考虑了大规模邮件发送的可靠性、可观测性和可维护性。
