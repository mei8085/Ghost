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

### 5.1 事件获取方式

Ghost **不使用 Webhook**，而是通过**定时轮询** Mailgun Events API 获取投递反馈：

```javascript
// email-analytics-provider-mailgun.js:4
const DEFAULT_EVENT_FILTER = 'delivered OR opened OR failed OR unsubscribed OR complained';
```

### 5.2 定时任务类型

| 任务 | 频率 | 事件类型 | 说明 |
|------|------|----------|------|
| `fetchLatestOpenedEvents` | 高频 | opened | 打开事件（更新频繁） |
| `fetchLatestNonOpenedEvents` | 高频 | delivered, failed, unsubscribed, complained | 其他事件 |
| `fetchMissing` | 低频 | 全部 | 补漏（30分钟后 Mailgun 存储稳定） |
| `fetchScheduled` | 按需 | 全部 | 手动调度的历史数据拉取 |

### 5.3 信任阈值与补漏机制

```javascript
// email-analytics-service.js:38-39
const TRUST_THRESHOLD_MS = 30 * 60 * 1000;  // 30分钟
const FETCH_LATEST_END_MARGIN_MS = 1 * 60 * 1000;  // 1分钟
```

**设计原因**：Mailgun 的事件存储有最终一致性延迟，所以：
- `fetchLatest` 只拉取到 **1分钟前** 的事件
- `fetchMissing` 拉取 **30分钟前** 到上一次拉取的事件（此时数据已稳定）

### 5.4 事件规范化

```javascript
// mailgun-client.js:304-328
normalizeEvent(event) {
    return {
        id: event.id,
        type: event.event,           // delivered / opened / failed / unsubscribed / complained
        severity: event.severity,    // permanent / temporary (仅 failed)
        recipientEmail: event.recipient,
        emailId: event['user-variables']?.['email-id'],  // Ghost 的 email ID
        providerId: event?.message?.headers['message-id'],  // Mailgun message-id
        timestamp: new Date(event.timestamp * 1000),
        error: event['delivery-status'] ? {
            code: event['delivery-status'].code,
            message: event['delivery-status'].message,
            enhancedCode: event['delivery-status']['enhanced-code']
        } : null
    };
}
```

### 5.5 事件匹配与归并

#### 5.5.1 收件人匹配策略

```javascript
// email-event-processor.js:208-244
async getRecipient(emailIdentification, recipientCache) {
    // 1. 获取 emailId (从 v:email-id 或通过 provider_id 查 email_batches)
    const emailId = emailIdentification.emailId ?? await this.getEmailId(providerId);
    
    // 2. 通过 (email, emailId) 匹配 email_recipients
    const {id, member_id} = await this.#db.knex('email_recipients')
        .select('id', 'member_id')
        .where('member_email', emailIdentification.email)
        .where('email_id', emailId)
        .first();
}
```

#### 5.5.2 批量查询优化

```javascript
// email-event-processor.js:288-356
async batchGetRecipients(emailIdentifications) {
    // Step 1: 批量解析 providerId → emailId
    // Step 2: 构建 OR 查询批量获取所有 recipient
    // Step 3: 构建 Map<email:emailId, recipientInfo> 缓存
}
```

### 5.6 事件存储与批量更新

#### 5.6.1 批量处理模式

```javascript
// email-event-storage.js:21-25
this.#pendingUpdates = {
    delivered: new Map(),  // recipientId → timestamp
    opened: new Map(),
    failed: new Map()
};
```

#### 5.6.2 批量 SQL 更新（CASE 语句）

```javascript
// email-event-storage.js:289-298
const sql = `
    UPDATE email_recipients
    SET delivered_at = CASE id 
        WHEN 'id1' THEN '2024-05-10 10:00:00'
        WHEN 'id2' THEN '2024-05-10 10:00:01'
        ...
    END
    WHERE id IN (?, ?, ...)
    AND delivered_at IS NULL  -- 乱序保护
`;
```

#### 5.6.3 乱序保护

```javascript
// email-event-storage.js:44-47
// 保持最早时间戳
if (!existing || timestamp < existing) {
    this.#pendingUpdates.delivered.set(event.emailRecipientId, timestamp);
}

// SQL 层面也保护
AND delivered_at IS NULL
```

### 5.7 事件类型处理

| 事件类型 | 处理方法 | 数据库操作 |
|----------|----------|------------|
| **delivered** | `handleDelivered()` | 更新 `email_recipients.delivered_at` |
| **opened** | `handleOpened()` | 更新 `email_recipients.opened_at` |
| **failed (permanent)** | `handlePermanentFailed()` | 更新 `failed_at` + 写入 `email_recipient_failures` |
| **failed (temporary)** | `handleTemporaryFailed()` | 仅写入 `email_recipient_failures` |
| **unsubscribed** | `handleUnsubscribed()` | 更新成员订阅关系 + 移除 Mailgun 退订列表 |
| **complained** | `handleComplained()` | 写入 `email_spam_complaint_events` + 移除 Mailgun 投诉列表 |

### 5.8 统计聚合

```javascript
// email-analytics-service.js:424-444
// 每 5 分钟或 5000 个会员聚合一次
if ((Date.now() - lastAggregation > 5 * 60 * 1000 || processingResult.memberIds.length > 5000) && eventCount > 0) {
    await this.aggregateStats(processingResult, includeOpenedEvents);
    processingResult = new EventProcessingResult();
}
```

聚合内容：
- **emails 表**：`delivered_count`, `opened_count`, `failed_count`
- **members 表**：会员邮件发送统计

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
