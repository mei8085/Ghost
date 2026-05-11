# Ghost 平台 Webhook 失败链路深度分析报告 (R2)

本文档是 `webhook-pipeline.md` 的补充，深入分析 webhook 失败投递的完整链路，包括重试耗尽后的状态落库、两种 webhook 的失败分流策略、410 与非 410 错误的后续状态差异，以及管理端查询展示的完整数据流。

---

## 目录
- [1. 失败投递完整链路总览](#1-失败投递完整链路总览)
- [2. 普通 Webhook 失败链路详解](#2-普通-webhook-失败链路详解)
- [3. Verification Webhook 失败链路详解](#3-verification-webhook-失败链路详解)
- [4. 两种 Webhook 失败分流对比](#4-两种-webhook-失败分流对比)
- [5. 410 删除与非 410 错误的状态差异](#5-410-删除与非-410-错误的状态差异)
- [6. 管理端查询展示完整链路](#6-管理端查询展示完整链路)
- [7. 状态字段流转图](#7-状态字段流转图)
- [8. 代码引用清单](#8-代码引用清单)

---

## 1. 失败投递完整链路总览

### 1.1 链路阶段分解

```
阶段 1: 初始请求
    ↓
阶段 2: 重试循环 (最多 5 次)
    ├─ 网络错误 → 重试
    ├─ 5xx 错误 → 重试
    └─ 4xx 错误 → 不重试
    ↓
阶段 3: 重试耗尽 / 最终失败
    ↓
阶段 4: 失败分流 (普通 Webhook vs Verification Webhook)
    ├─ 普通 Webhook: 状态落库 + 日志
    └─ Verification Webhook: 日志 + 业务分支
    ↓
阶段 5: 后续状态演进
    ├─ 410 Gone → 自动删除 webhook 记录
    └─ 其他错误 → 保留记录，等待下次事件触发
    ↓
阶段 6: 管理端查询展示
    ├─ 前端 API: useBrowseIntegrations(include=webhooks)
    ├─ 后端 API: Integration.findPage(withRelated=webhooks)
    └─ UI 展示: last_triggered_at, last_triggered_status
```

### 1.2 重试机制详解

基于 `got` HTTP 客户端的重试配置（`webhook-trigger.js:137-143`）：

```javascript
const opts = {
    timeout: {
        request: 2 * 1000  // 单次请求 2 秒超时
    },
    retry: {
        limit: process.env.NODE_ENV?.startsWith('test') ? 0 : 5  // 最多 5 次重试
    }
};
```

**got 默认重试策略**：
- **重试触发条件**：
  - 网络错误：`ECONNRESET`, `ECONNREFUSED`, `ETIMEDOUT`, `EAI_AGAIN` 等
  - DNS 解析失败
  - HTTP 5xx 状态码（500, 502, 503, 504）
  - 请求超时

- **不重试条件**：
  - HTTP 4xx 状态码（客户端错误）
  - 410 Gone（特殊处理，见第 5 节）

- **退避策略**：got 默认使用指数退避
  - 第 1 次重试：~100ms
  - 第 2 次重试：~200ms
  - 第 3 次重试：~400ms
  - 第 4 次重试：~800ms
  - 第 5 次重试：~1600ms

**最坏情况下的总耗时**：
- 单次请求：2 秒
- 重试间隔：~3.1 秒（100+200+400+800+1600ms）
- 总耗时：6 次请求 × 2 秒 + 3.1 秒 ≈ **15.1 秒**

---

## 2. 普通 Webhook 失败链路详解

### 2.1 触发入口

普通 Webhook 的触发入口在 `listen.js:47-68`：

```javascript
const listen = async () => {
    const webhookTrigger = new WebhookTrigger({models, payload, limitService});
    _.each(WEBHOOKS, (event) => {
        if (events.hasRegisteredListener(event, 'processWebhookTrigger')) {
            return;
        }
        events.on(event, function processWebhookTrigger(model, options) {
            if (options && options.importing) {
                return;
            }
            webhookTrigger.trigger(event, model);  // 异步执行，不阻塞主流程
        });
    });
};
```

**关键点**：
- `webhookTrigger.trigger()` 是**异步调用**（没有 await）
- 不会阻塞主业务流程（如文章保存、会员创建等）
- 失败不会影响原始操作

### 2.2 触发主流程

`WebhookTrigger.trigger()` 方法（`webhook-trigger.js:105-151`）：

```javascript
async trigger(event, model) {
    const response = {
        onSuccess: this.onSuccess.bind(this),
        onError: this.onError.bind(this)
    };

    const hooks = await this.getAll(event);  // 步骤 1: 获取匹配的 webhook 列表

    debug(`${hooks.models.length} webhooks found for ${event}.`);

    for (const webhook of hooks.models) {  // 步骤 2: 串行遍历每个 webhook
        const hookPayload = await this.payload(webhook.get('event'), model);  // 步骤 3: 生成 payload
        const reqPayload = JSON.stringify(hookPayload);
        const url = webhook.get('target_url');
        const secret = webhook.get('secret') || '';
        const ts = Date.now();

        const headers = { /* 构建请求头 */ };

        if (secret !== '') {
            headers['X-Ghost-Signature'] = `sha256=${crypto.createHmac('sha256', secret).update(`${reqPayload}${ts}`).digest('hex')}, t=${ts}`;
        }

        const opts = {
            method: 'POST',
            body: reqPayload,
            headers,
            timeout: { request: 2 * 1000 },
            retry: { limit: process.env.NODE_ENV?.startsWith('test') ? 0 : 5 }
        };

        logging.info(`Triggering webhook for "${webhook.get('event')}" with url "${url}"`);

        // 步骤 4: 发起请求，进入重试循环
        await this.request(url, opts)
            .then(response.onSuccess(webhook))  // 成功分支
            .catch(response.onError(webhook));  // 失败分支（重试耗尽后进入）
    }
}
```

### 2.3 失败回调 - onError

当请求失败（包括重试耗尽后），进入 `onError` 回调（`webhook-trigger.js:88-103`）：

```javascript
onError(webhook) {
    return (err) => {
        // 特殊状态码处理：410 Gone
        if (err.statusCode === 410) {
            logging.info(`Webhook destroyed (410 response) for "${webhook.get('event')}" with url "${webhook.get('target_url')}".`);
            return this.destroy(webhook);  // 分支 A: 删除 webhook
        }

        // 普通错误处理：更新状态字段
        this.update(webhook, {
            statusCode: err.statusCode,
            error: `Request failed: ${err.code || 'unknown'}`
        });  // 分支 B: 状态落库

        // 结构化错误日志
        logging.error(`[WEBHOOK_DELIVERY_FAILURE] url=${webhook.get('target_url') || 'unknown'} status=${err.statusCode || 'none'} error_code=${err.code || 'unknown'} message=${err.message || ''}`, err);
    };
}
```

### 2.4 状态落库 - update 方法

`WebhookTrigger.update()` 方法（`webhook-trigger.js:58-69`）：

```javascript
update(webhook, data) {
    this.models
        .Webhook
        .edit({
            last_triggered_at: Date.now(),           // 最后触发时间（毫秒时间戳）
            last_triggered_status: data.statusCode,  // HTTP 状态码或 undefined
            last_triggered_error: data.error || null // 错误描述或 null
        }, {id: webhook.id, autoRefresh: false})
        .catch(() => {
            logging.warn(`Unable to update "last_triggered" for webhook: ${webhook.id}`);
        });
}
```

**落库字段详解**：

| 字段 | 类型 | 成功时值 | 失败时值 | 说明 |
|------|------|----------|----------|------|
| `last_triggered_at` | dateTime | `Date.now()` | `Date.now()` | 每次触发都会更新 |
| `last_triggered_status` | string(50) | `res.statusCode` (如 "200") | `err.statusCode` (如 "500") 或 undefined | HTTP 响应状态码 |
| `last_triggered_error` | string(50) | `null` | `"Request failed: ${err.code}"` 或 null | 错误描述 |

**常见错误码**：
- `URL_PRIVATE_INVALID` - 目标 URL 解析到内网 IP（SSRF 防护）
- `URL_MISSING_INVALID` - URL 格式无效
- `ECONNREFUSED` - 连接被拒绝
- `ECONNRESET` - 连接被重置
- `ETIMEDOUT` - 请求超时
- `ENOTFOUND` - DNS 解析失败
- `unknown` - 其他未知错误

### 2.5 410 特殊处理 - destroy 方法

当收到 410 Gone 响应时（`webhook-trigger.js:71-78`）：

```javascript
destroy(webhook) {
    return this.models
        .Webhook
        .destroy({id: webhook.id}, {context: {internal: true}})
        .catch(() => {
            logging.warn(`Unable to destroy webhook ${webhook.id}.`);
        });
}
```

**410 的语义**：消费方明确表示该端点已永久移除，Ghost 自动清理 webhook 记录，避免后续无效推送。

---

## 3. Verification Webhook 失败链路详解

### 3.1 什么是 Verification Webhook

Verification Webhook 是 Ghost Pro（托管版）特有的机制，用于在会员导入或批量创建超过阈值时，向 Ghost 主机服务发送验证通知。

**触发场景**（`verification-trigger.js:67-107`）：
1. **API 创建会员**：超过 `apiTriggerThreshold`
2. **Admin 批量创建**：超过 `adminTriggerThreshold`
3. **CSV 导入会员**：超过 `importTriggerThreshold`

### 3.2 配置来源

`VerificationWebhookService` 从配置读取（`verification-webhook-service.ts:44-51`）：

```typescript
#readWebhookConfig() {
    return {
        webhookType: this.#config.get('hostSettings:emailVerification:webhookType'),
        webhookUrl: this.#config.get('hostSettings:emailVerification:webhookUrl'),
        webhookSecret: this.#config.get('hostSettings:emailVerification:webhookSecret') || '',
        siteId: this.#config.get('hostSettings:siteId') || null
    };
}
```

**注意**：这不是通过 Admin API 创建的普通 webhook，而是从 `hostSettings` 配置读取的**系统级 webhook**。

### 3.3 推送主流程

`VerificationWebhookService.sendVerificationWebhook()`（`verification-webhook-service.ts:69-135`）：

```typescript
async sendVerificationWebhook({
    amountTriggered,
    threshold,
    method
}: {
    amountTriggered: number;
    threshold: number;
    method: VerificationTriggerMethod;  // 'admin' | 'api' | 'import'
}): Promise<boolean> {
    const {webhookType, webhookUrl, webhookSecret, siteId} = this.#readWebhookConfig();

    // 前置检查：配置是否完整
    if (typeof webhookUrl !== 'string' || webhookUrl.length === 0) {
        this.#logging.warn('Verification webhook is not configured because webhookUrl is missing.');
        return false;
    }
    if (typeof webhookType !== 'string' || webhookType.length === 0) {
        this.#logging.warn('Verification webhook is not configured because webhookType is missing.');
        return false;
    }

    // 构建 payload
    const payload: VerificationWebhookBody = {
        type: webhookType,
        siteId: typeof siteId === 'string' ? siteId : null,
        amountTriggered,
        threshold,
        method
    };

    const requestBody = JSON.stringify(payload);
    const timestamp = Date.now().toString();

    // 签名（注意：与普通 webhook 算法不同！）
    const headers: Record<string, string | number> = {
        'Content-Length': Buffer.byteLength(requestBody),
        'Content-Type': 'application/json',
        'Content-Version': `v${ghostVersion.safe}`,
        'X-Ghost-Request-Timestamp': timestamp  // 单独的时间戳头
    };

    if (typeof webhookSecret === 'string' && webhookSecret !== '') {
        // 签名基串: `${timestamp}:${body}` (普通 webhook 是 `${body}${timestamp}`)
        headers['X-Ghost-Signature'] = this.#computeSignature(timestamp, requestBody, webhookSecret);
    }

    const requestOptions = {
        method: 'POST',
        body: requestBody,
        headers,
        timeout: {
            request: 30_000  // 30 秒超时（普通 webhook 是 2 秒）
        },
        retry: {
            limit: process.env.NODE_ENV?.startsWith('test') ? 0 : 5  // 同样 5 次重试
        }
    };

    const sanitizedWebhookUrl = this.#sanitizeWebhookUrl(webhookUrl);
    this.#logging.info(`Triggering verification webhook to "${sanitizedWebhookUrl}"`);

    try {
        await this.#request(webhookUrl, requestOptions);
        return true;  // 成功返回 true
    } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        this.#logging.error(`Failed to send verification webhook to "${sanitizedWebhookUrl}": ${message}`);
        throw error;  // 失败抛出异常，由上层处理
    }
}
```

### 3.4 签名算法差异

**Verification Webhook**（`verification-webhook-service.ts:53-56`）：
```typescript
#computeSignature(timestamp: string, body: string, secret: string): string {
    const baseString = `${timestamp}:${body}`;  // 冒号分隔
    return crypto.createHmac('sha256', secret).update(baseString).digest('base64');  // Base64 编码
}
```

**普通 Webhook**（`webhook-trigger.js:129-131`）：
```javascript
const signature = crypto.createHmac('sha256', secret)
    .update(`${reqPayload}${ts}`)  // 直接拼接
    .digest('hex');  // Hex 编码
```

| 对比项 | 普通 Webhook | Verification Webhook |
|--------|-------------|----------------------|
| 基串格式 | `${body}${timestamp}` | `${timestamp}:${body}` |
| 编码方式 | hex | base64 |
| 时间戳头 | 包含在 X-Ghost-Signature 中 | 单独的 X-Ghost-Request-Timestamp |

### 3.5 失败处理 - 上层业务分流

`VerificationTrigger._startVerificationProcess()`（`verification-trigger.js:217-252`）：

```javascript
async _startVerificationProcess({
    amount,
    threshold,
    method,
    throwOnTrigger,
    source
}) {
    // 前置检查 1: 是否已验证
    if (this._isVerified()) {
        return {needsVerification: false};
    }

    // 前置检查 2: 是否已触发过验证流程
    if (this._isVerificationRequired()) {
        return {needsVerification: false};
    }

    let webhookWasSent = false;

    try {
        webhookWasSent = await this._sendVerificationWebhook({
            amountTriggered: amount,
            threshold: threshold ?? amount,
            method: method ?? source ?? 'import'
        });
    } catch (error) {
        // 关键：webhook 失败时，catch 住并返回 false
        // `sendVerificationWebhook` already logs delivery failures.
        return {needsVerification: false};
    }

    // 只有 webhook 发送成功，才标记需要验证
    if (!webhookWasSent) {
        return {needsVerification: false};
    }

    // 成功分支：更新 settings
    await this._markVerificationRequired();
    return this._finishTrigger(throwOnTrigger);
}
```

### 3.6 Verification Webhook 失败的业务影响

```
会员创建/导入超过阈值
    ↓
触发 VerificationTrigger._startVerificationProcess()
    ↓
调用 sendVerificationWebhook()
    ↓
┌─ 成功 (返回 true) ───────────────────┐  ┌─ 失败 (抛出异常) ───────────────────┐
│                                      │  │                                      │
│  await _markVerificationRequired()   │  │  catch 住异常                       │
│  ├─ Settings.edit(email_verification │  │  记录日志（sendVerificationWebhook  │
│  │     _required: true)              │  │    内部已记录）                     │
│  └─ settingsCache.set()              │  │  return {needsVerification: false} │
│                                      │  │                                      │
│  _finishTrigger(throwOnTrigger)      │  │  不进入验证流程                     │
│  ├─ throwOnTrigger=true: 抛出异常    │  │  会员创建/导入继续进行              │
│  │   (阻止当前操作)                   │  │  不影响业务正常流程                 │
│  └─ throwOnTrigger=false: 返回对象   │  │                                      │
│     (继续当前操作，但标记需要验证)    │  │                                      │
└──────────────────────────────────────┘  └──────────────────────────────────────┘
```

**设计意图**：Verification Webhook 是**可选择降级**的机制。当主机服务不可用时，Ghost 站点继续正常运行，只是跳过验证流程。

---

## 4. 两种 Webhook 失败分流对比

### 4.1 对比矩阵

| 维度 | 普通 Webhook | Verification Webhook |
|------|-------------|----------------------|
| **创建方式** | Admin API / 集成页面 | hostSettings 配置 |
| **存储位置** | `webhooks` 数据库表 | 内存配置（无持久化） |
| **触发时机** | 模型事件（post.added 等） | 会员量超过阈值 |
| **请求超时** | 2 秒 | 30 秒 |
| **重试次数** | 5 次 | 5 次 |
| **失败后状态落库** | ✅ 更新 `last_triggered_*` 字段 | ❌ 无状态持久化 |
| **410 自动删除** | ✅ 支持 | ❌ 不支持（无数据库记录） |
| **失败日志格式** | `[WEBHOOK_DELIVERY_FAILURE] url=...` | `Failed to send verification webhook to ...` |
| **对业务影响** | 无（异步触发） | 降级处理（跳过验证） |
| **管理端可见性** | ✅ 集成页面展示 | ❌ 管理端不可见 |
| **签名算法** | `sha256(${body}${ts})` hex | `sha256(${ts}:${body})` base64 |
| **URL 安全检查** | request-external（SSRF 防护） | @tryghost/request（无 SSRF 防护） |

### 4.2 架构定位差异

```
┌─────────────────────────────────────────────────────────────────┐
│                     Ghost 平台架构分层                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              业务层 (Business Logic)                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │  文章服务   │  │  会员服务   │  │  导入服务       │  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │   │
│  │         │                │                   │            │   │
│  │         └────────────────┼───────────────────┘            │   │
│  │                          │                                │   │
│  │         领域事件总线      │   阈值检查                      │   │
│  │         (events.js)      │                                │   │
│  └─────────────┬────────────┴────────────────────────────────┘   │
│                │                                                 │
│     ┌──────────┴──────────┐                                      │
│     ▼                     ▼                                      │
│                                                                 │
│  ┌──────────────────┐   ┌──────────────────────────┐             │
│  │  普通 Webhook    │   │  Verification Webhook    │             │
│  │  系统            │   │  系统                    │             │
│  │                  │   │                          │             │
│  │  • 事件驱动      │   │  • 会员阈值触发          │             │
│  │  • 31 种事件类型 │   │  • 3 种触发方式          │             │
│  │  • 状态持久化    │   │  • 无状态持久化          │             │
│  │  • 管理端可见    │   │  • 管理端不可见          │             │
│  │  • SSRF 防护     │   │  • 信任内部服务          │             │
│  │  • 集成可配置    │   │  • 主机服务配置          │             │
│  └────────┬─────────┘   └────────────┬─────────────┘             │
│           │                          │                           │
│           ▼                          ▼                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              外部系统 (External Systems)                 │   │
│  │                                                          │   │
│  │  普通 Webhook 消费方:                                    │   │
│  │  • Zapier, Make, 自定义集成                              │   │
│  │  • 处理文章、会员等业务事件                               │   │
│  │                                                          │   │
│  │  Verification Webhook 消费方:                            │   │
│  │  • Ghost Pro 主机服务                                    │   │
│  │  • 处理会员验证、反垃圾邮件                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. 410 删除与非 410 错误的状态差异

### 5.1 状态机设计

```
                    ┌─────────────────┐
                    │  初始状态       │
                    │  (新创建)       │
                    │  last_triggered │
                    │  字段为 null    │
                    └────────┬────────┘
                             │
                             ▼ 事件触发
                    ┌─────────────────┐
                    │  发起 POST 请求 │
                    │  (含重试逻辑)   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  成功 (2xx) │  │  410 Gone   │  │  其他错误   │
    │             │  │             │  │  (4xx/5xx/  │
    │             │  │             │  │   网络错误) │
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                │
           ▼                ▼                ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  update()   │  │  destroy()  │  │  update()   │
    │             │  │             │  │             │
    │  status:200 │  │  删除记录    │  │  status:500 │
    │  error:null │  │  从数据库   │  │  error:...  │
    │             │  │  彻底移除    │  │             │
    └──────┬──────┘  └─────────────┘  └──────┬──────┘
           │                                  │
           ▼                                  ▼
    ┌─────────────┐                   ┌─────────────┐
    │  保留记录   │                   │  保留记录   │
    │  等待下次   │                   │  等待下次   │
    │  事件触发   │                   │  事件触发   │
    └─────────────┘                   └─────────────┘
```

### 5.2 410 Gone 处理逻辑

**触发条件**：`err.statusCode === 410`（`webhook-trigger.js:90-94`）

```javascript
if (err.statusCode === 410) {
    logging.info(`Webhook destroyed (410 response) for "${webhook.get('event')}" with url "${webhook.get('target_url')}".`);
    return this.destroy(webhook);  // 直接删除，不更新状态字段
}
```

**410 的语义**：
- HTTP 410 Gone 表示资源**永久不可用**
- 消费方明确告知："这个 endpoint 我不再维护了"
- Ghost 理解为："这个 webhook 订阅已失效，无需再推送"

**删除后的影响**：
1. Webhook 记录从 `webhooks` 表删除
2. 管理端集成页面不再显示该 webhook
3. 后续该事件触发时，`findAllByEvent` 不再返回此 webhook
4. 无任何历史记录保留（只保留删除前的日志）

### 5.3 非 410 错误处理逻辑

**触发条件**：其他所有错误（4xx 非 410、5xx、网络错误等）

```javascript
this.update(webhook, {
    statusCode: err.statusCode,
    error: `Request failed: ${err.code || 'unknown'}`
});
```

**非 410 的语义**：
- 4xx：消费方配置问题（鉴权失败、路由错误等）- 需要人工修复
- 5xx：消费方临时故障 - 可能下次就恢复
- 网络错误：临时性问题 - 可能下次就恢复

**保留记录的好处**：
1. **可诊断**：通过 `last_triggered_status` 和 `last_triggered_error` 定位问题
2. **可恢复**：消费方修复后，下次事件触发自动恢复推送
3. **可监控**：管理端可查看 webhook 健康状态

### 5.4 状态字段详细对比

| 场景 | `last_triggered_at` | `last_triggered_status` | `last_triggered_error` | 记录是否保留 |
|------|---------------------|------------------------|-----------------------|-------------|
| 推送成功 (200) | 当前时间 | `"200"` | `null` | ✅ 保留 |
| 推送成功 (201) | 当前时间 | `"201"` | `null` | ✅ 保留 |
| 客户端错误 (400) | 当前时间 | `"400"` | `"Request failed: unknown"` | ✅ 保留 |
| 未授权 (401) | 当前时间 | `"401"` | `"Request failed: unknown"` | ✅ 保留 |
| 禁止访问 (403) | 当前时间 | `"403"` | `"Request failed: unknown"` | ✅ 保留 |
| 未找到 (404) | 当前时间 | `"404"` | `"Request failed: unknown"` | ✅ 保留 |
| **永久移除 (410)** | **不更新** | **不更新** | **不更新** | ❌ **删除** |
| 服务端错误 (500) | 当前时间 | `"500"` | `"Request failed: unknown"` | ✅ 保留 |
| 网关错误 (502) | 当前时间 | `"502"` | `"Request failed: unknown"` | ✅ 保留 |
| 服务不可用 (503) | 当前时间 | `"503"` | `"Request failed: unknown"` | ✅ 保留 |
| 连接超时 | 当前时间 | `undefined` | `"Request failed: ETIMEDOUT"` | ✅ 保留 |
| 连接被拒绝 | 当前时间 | `undefined` | `"Request failed: ECONNREFUSED"` | ✅ 保留 |
| DNS 解析失败 | 当前时间 | `undefined` | `"Request failed: ENOTFOUND"` | ✅ 保留 |
| SSRF 防护拦截 | 当前时间 | `undefined` | `"Request failed: URL_PRIVATE_INVALID"` | ✅ 保留 |

### 5.5 日志记录对比

**410 日志**（INFO 级别）：
```
info: Webhook destroyed (410 response) for "post.added" with url "https://old-endpoint.com/webhook".
```

**非 410 错误日志**（ERROR 级别）：
```
error: [WEBHOOK_DELIVERY_FAILURE] url=https://example.com/hook status=500 error_code=unknown message=Internal Server Error
```

---

## 6. 管理端查询展示完整链路

### 6.1 链路全景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           管理端查询数据流                                    │
│                                                                              │
│  Admin UI (React)                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Integrations.tsx                                                   │    │
│  │  const {data} = useBrowseIntegrations();                           │    │
│  └────────────────────────────┬────────────────────────────────────────┘    │
│                               │                                              │
│                               ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  admin-x-framework/api/integrations.ts                             │    │
│  │  export const useBrowseIntegrations = createQuery({                │    │
│  │      path: '/integrations/',                                        │    │
│  │      defaultSearchParams: {include: 'api_keys,webhooks', limit:50} │    │
│  │  });                                                                │    │
│  └────────────────────────────┬────────────────────────────────────────┘    │
│                               │                                              │
│                               ▼ HTTP GET                                     │
│  Admin API (Express)                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  endpoints/integrations.js                                          │    │
│  │  browse: {                                                          │    │
│  │      validation: { include: { values: ['api_keys', 'webhooks'] } } │    │
│  │      query({options}) {                                             │    │
│  │          return models.Integration.findPage(options);              │    │
│  │      }                                                              │    │
│  │  }                                                                  │    │
│  └────────────────────────────┬────────────────────────────────────────┘    │
│                               │                                              │
│                               ▼                                              │
│  数据层 (Bookshelf ORM)                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  models/integration.js                                              │    │
│  │  webhooks: function webhooks() {                                    │    │
│  │      return this.hasMany('Webhook', 'integration_id');             │    │
│  │  }                                                                  │    │
│  │                                                                     │    │
│  │  findPage(options)  →  withRelated = ['webhooks']                  │    │
│  │    → SELECT * FROM integrations                                    │    │
│  │    → SELECT * FROM webhooks WHERE integration_id IN (...)          │    │
│  └────────────────────────────┬────────────────────────────────────────┘    │
│                               │                                              │
│                               ▼                                              │
│  数据库 (MySQL)                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  integrations表                    webhooks表                        │    │
│  │  ┌────┬──────┬────────┐          ┌────┬───────────┬─────────────┐  │    │
│  │  │ id │ name │ type   │          │ id │ event     │ target_url  │  │    │
│  │  ├────┼──────┼────────┤          ├────┼───────────┼─────────────┤  │    │
│  │  │ 1  │ Zap  │ builtin│          │ 1  │post.added │ https://... │  │    │
│  │  │ 2  │ Custom│ custom │   1:N    │ 2  │member.add │ https://... │  │    │
│  │  └────┴──────┴────────┘◄─────────┤ 3  │post.edited│ https://... │  │    │
│  │                                    └────┴───────────┴─────────────┘  │    │
│  │                                           ↑ last_triggered_at        │    │
│  │                                           ↑ last_triggered_status    │    │
│  │                                           ↑ last_triggered_error     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 前端数据获取层

**React Query Hook**（`admin-x-framework/src/api/integrations.ts:31-35`）：

```typescript
export const useBrowseIntegrations = createQuery<IntegrationsResponseType>({
    dataType,
    path: '/integrations/',
    defaultSearchParams: {include: 'api_keys,webhooks', limit: '50'}
});
```

**关键参数**：
- `include: 'api_keys,webhooks'` - 请求后端关联查询 webhooks
- `limit: '50'` - 分页大小

### 6.3 后端 API 层

**Integrations Controller**（`ghost/core/core/server/api/endpoints/integrations.js:18-37`）：

```javascript
browse: {
    headers: {
        cacheInvalidate: false
    },
    permissions: true,
    options: [
        'include',
        'limit'
    ],
    validation: {
        options: {
            include: {
                values: ['api_keys', 'webhooks']  // 白名单验证
            }
        }
    },
    query({options}) {
        return models.Integration.findPage(options);
    }
},
```

**include 参数白名单**：只允许 `api_keys` 和 `webhooks`，防止任意关联查询。

### 6.4 ORM 关联层

**Integration 模型**（`ghost/core/core/server/models/integration.js:6-68`）：

```javascript
const Integration = ghostBookshelf.Model.extend({
    tableName: 'integrations',

    relationships: ['api_keys', 'webhooks'],  // 声明可关联关系
    relationshipConfig: {
        api_keys: { editable: true },
        webhooks: { editable: true }
    },
    relationshipBelongsTo: {
        api_keys: 'api_keys',
        webhooks: 'webhooks'
    },

    // ...

    // 关联定义
    api_keys: function apiKeys() {
        return this.hasMany('ApiKey', 'integration_id');
    },

    webhooks: function webhooks() {
        return this.hasMany('Webhook', 'integration_id');  // 一对多关联
    }
});
```

### 6.5 Webhook 类型定义

**前端类型**（`admin-x-framework/src/api/webhooks.ts:6-19`）：

```typescript
export type Webhook = {
    id: string;
    event: string;
    target_url: string;
    name: string;
    secret: string | null;
    api_version: string;
    integration_id: string;
    last_triggered_at: string | null;      // 最后触发时间（ISO 字符串）
    last_triggered_status: string | null;  // HTTP 状态码（字符串）
    last_triggered_error: string | null;   // 错误信息
    created_at: string;
    updated_at: string;
}
```

### 6.6 UI 展示层

**Webhook 表格组件**（`admin-x-settings/src/components/settings/advanced/integrations/webhooks-table.tsx:34-97`）：

```tsx
const WebhooksTable: React.FC<{integration: Integration}> = ({integration}) => {
    // ...

    return (<div>
        <Table>
            <TableRow bgOnHover={false}>
                <TableHead>{integration.webhooks?.length || 0} webhook(s)</TableHead>
                <TableHead>Last triggered</TableHead>
            </TableRow>
            {integration.webhooks?.map(webhook => (
                <TableRow 
                    action={<Button color='red' label='Delete' link ... />}
                    onClick={() => { NiceModal.show(WebhookModal, {webhook, integrationId}); }}
                >
                    <TableCell className='w-3/4'>
                        <div className='text-sm font-semibold'>{webhook.name}</div>
                        <div className='grid grid-cols-[max-content_1fr] gap-x-1 text-xs leading-snug'>
                            <span className='text-grey-600'>Event:</span>
                            <span>{getWebhookEventLabel(webhook.event)}</span>
                            <span className='text-grey-600'>URL:</span>
                            <span className='line-clamp-3 break-all'>{webhook.target_url}</span>
                            {/* 注意：UI 只展示 last_triggered_at，不展示 status 和 error */}
                        </div>
                    </TableCell>
                    <TableCell className='w-1/4 text-sm'>
                        {webhook.last_triggered_at && new Date(webhook.last_triggered_at).toLocaleString(
                            'default', {
                                weekday: 'short',
                                month: 'short',
                                day: 'numeric',
                                year: 'numeric',
                                hour: '2-digit',
                                minute: '2-digit',
                                second: '2-digit'
                            }
                        )}
                    </TableCell>
                </TableRow>
            ))}
        </Table>
        {/* 添加 webhook 按钮 */}
    </div>);
};
```

**当前 UI 的限制**：
- ✅ 展示 `last_triggered_at`（格式化的日期时间）
- ❌ **不展示** `last_triggered_status`（HTTP 状态码）
- ❌ **不展示** `last_triggered_error`（错误信息）
- ❌ 没有健康状态指示器（成功/失败图标）
- ❌ 没有重试按钮或手动触发功能

### 6.7 编辑弹窗

**Webhook Modal**（`admin-x-settings/src/components/settings/advanced/integrations/webhook-modal.tsx`）：

```tsx
const WebhookModal: React.FC<WebhookModalProps> = ({webhook, integrationId}) => {
    const {formState, updateForm, handleSave, errors, clearError} = useForm<Partial<Webhook>>({
        initialState: webhook || {},  // 编辑时传入 webhook 对象
        onSave: async () => {
            if (formState.id) {
                await editWebhook(formState as Webhook);
            } else {
                await createWebhook({...formState, integration_id: integrationId});
            }
        },
        // ...
    });

    return <Modal ...>
        <Form>
            <TextField title='Name' value={formState.name} ... />
            <Select title='Event' selectedOption={...} options={webhookEventOptions} ... />
            <TextField title='Target URL' value={formState.target_url} ... />
            <TextField title='Secret' value={formState.secret || undefined} ... />
            {/* 注意：编辑弹窗也不展示 last_triggered_* 字段 */}
        </Form>
    </Modal>;
};
```

### 6.8 数据流转时序图

```
用户打开集成页面
        │
        ▼
┌─────────────────────┐
│ useBrowseIntegrations│
│ (React Query)       │
└──────────┬──────────┘
           │
           │ GET /ghost/api/admin/integrations/?include=api_keys,webhooks&limit=50
           ▼
┌─────────────────────┐
│ Admin API Controller│
│ integrations.js     │
└──────────┬──────────┘
           │
           │ options.include = ['api_keys', 'webhooks']
           ▼
┌─────────────────────┐
│ Bookshelf findPage()│
│ withRelated 加载关联│
└──────────┬──────────┘
           │
           │ SQL 查询
           ▼
┌────────────────────────────────────────┐
│  SELECT * FROM integrations LIMIT 50;  │
│                                        │
│  SELECT * FROM webhooks                │
│  WHERE integration_id IN (?, ?, ...);  │
│                                        │
│  webhooks 表包含:                       │
│  - last_triggered_at                   │
│  - last_triggered_status               │
│  - last_triggered_error                │
└──────────────────┬─────────────────────┘
                   │
                   │ JSON 序列化
                   ▼
┌────────────────────────────────────────────────────────┐
│ API Response:                                          │
│ {                                                      │
│   "integrations": [                                    │
│     {                                                  │
│       "id": "int_123",                                 │
│       "name": "My Integration",                        │
│       "webhooks": [                                    │
│         {                                              │
│           "id": "wh_456",                              │
│           "event": "post.added",                       │
│           "target_url": "https://...",                 │
│           "last_triggered_at": "2026-05-10T10:30:00.000Z",│
│           "last_triggered_status": "500",              │
│           "last_triggered_error": "Request failed: ECONNREFUSED"│
│         }                                              │
│       ]                                                │
│     }                                                  │
│   ]                                                    │
│ }                                                      │
└──────────────────────────┬─────────────────────────────┘
                           │
                           │ React Query 缓存
                           ▼
┌────────────────────────────────────────────────────────┐
│ Admin UI Render:                                        │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ My Integration                         [Delete] │   │
│  │ Event: Post created                              │   │
│  │ URL: https://example.com/webhook                 │   │
│  │                                      May 10, 10:30│   │  ← 只展示时间
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  (状态码和错误信息在 API 响应中，但 UI 未展示)           │
└────────────────────────────────────────────────────────┘
```

---

## 7. 状态字段流转图

### 7.1 普通 Webhook 完整状态流转

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     普通 Webhook 状态流转图                                  │
└─────────────────────────────────────────────────────────────────────────────┘

                    初始状态
      last_triggered_at: null
      last_triggered_status: null
      last_triggered_error: null
                    │
                    │ 事件触发 (post.added, member.edited, etc.)
                    ▼
          ┌─────────────────────┐
          │   发起 POST 请求    │
          │   timeout: 2s      │
          │   retry: 5 次      │
          └──────────┬──────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  成功   │  │  410    │  │  失败   │
   │  2xx    │  │  Gone   │  │  其他   │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        │            │            │
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ update()│  │destroy()│  │ update()│
   │         │  │         │  │         │
   │ at: now │  │ 删除记录 │  │ at: now │
   │ status: │  │ 从DB移除 │  │status:err│
   │  "200"  │  │         │  │error:... │
   │ error:n │  │         │  │         │
   │  ull    │  │         │  │         │
   └────┬────┘  └─────────┘  └────┬────┘
        │                         │
        │                         │
        │                         │
        ▼                         ▼
   ┌─────────┐             ┌─────────┐
   │ 记录保留│             │ 记录保留│
   │ 等待下次│             │ 等待下次│
   │ 事件触发│             │ 事件触发│
   └────┬────┘             └────┬────┘
        │                         │
        └───────────┬─────────────┘
                    │
                    ▼ 下次事件触发
          ┌─────────────────────┐
          │   重新尝试推送       │
          │   (循环往复)        │
          └─────────────────────┘
```

### 7.2 Verification Webhook 完整状态流转

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  Verification Webhook 状态流转图                             │
└─────────────────────────────────────────────────────────────────────────────┘

                    触发条件
      会员创建/导入超过阈值
      (api/admin/import 三种来源)
                    │
                    ▼
          ┌─────────────────────┐
          │ 检查前置条件        │
          │ - isVerified?      │
          │ - isVerification   │
          │   Required?       │
          └──────────┬──────────┘
                     │
          ┌──────────┴──────────┐
          │ 任一条件为 true     │
          │ 跳过后续流程        │
          │ return {needsVerif │
          │ ication: false}    │
          └─────────────────────┘
                     │
                     │ 两个条件都为 false
                     ▼
          ┌─────────────────────┐
          │ 读取配置            │
          │ webhookUrl?        │
          │ webhookType?       │
          └──────────┬──────────┘
                     │
          ┌──────────┴──────────┐
          │ 配置缺失            │
          │ return false       │
          │ (needsVerification │
          │ : false)           │
          └─────────────────────┘
                     │
                     │ 配置完整
                     ▼
          ┌─────────────────────┐
          │ 发起 POST 请求      │
          │ timeout: 30s       │
          │ retry: 5 次        │
          └──────────┬──────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
   ┌─────────┐              ┌─────────┐
   │  成功   │              │  失败   │
   │ return  │              │ throw   │
   │ true    │              │ error   │
   └────┬────┘              └────┬────┘
        │                         │
        │                         │
        ▼                         ▼
   ┌─────────┐              ┌─────────┐
   │ 更新    │              │ catch   │
   │ Settings│              │ 住异常  │
   │ email_v │              │         │
   │ erifica │              │ return  │
   │ tion_re │              │ {needsV │
   │ quired: │              │ erifica │
   │ true}   │              │ tion:   │
   │         │              │ false}  │
   └────┬────┘              └────┬────┘
        │                         │
        │                         │
        ▼                         ▼
   ┌─────────┐              ┌─────────┐
   │ 标记需要│              │ 不标记  │
   │ 验证    │              │ 需要验证│
   │         │              │         │
   │ 影响后续│              │ 业务正常│
   │ 邮件发送│              │ 继续    │
   └─────────┘              └─────────┘
```

---

## 8. 代码引用清单

### 8.1 核心文件

| 文件路径 | 功能 |
|----------|------|
| `ghost/core/core/server/services/webhooks/webhook-trigger.js` | 普通 Webhook 触发、重试、状态更新 |
| `ghost/core/core/server/services/webhooks/listen.js` | 事件监听器注册 |
| `ghost/core/core/server/services/verification/verification-webhook-service.ts` | Verification Webhook 实现 |
| `ghost/core/core/server/services/verification-trigger.js` | Verification 触发逻辑、阈值检查 |
| `ghost/core/core/server/models/integration.js` | Integration 模型、webhooks 关联定义 |
| `ghost/core/core/server/api/endpoints/integrations.js` | Integrations API 端点 |
| `ghost/core/core/server/api/endpoints/utils/serializers/output/mappers/integrations.js` | API 响应序列化 |
| `apps/admin-x-framework/src/api/integrations.ts` | 前端 API Hooks |
| `apps/admin-x-framework/src/api/webhooks.ts` | Webhook 类型定义 |
| `apps/admin-x-settings/src/components/settings/advanced/integrations.tsx` | 集成页面主组件 |
| `apps/admin-x-settings/src/components/settings/advanced/integrations/webhooks-table.tsx` | Webhook 表格展示 |
| `apps/admin-x-settings/src/components/settings/advanced/integrations/webhook-modal.tsx` | Webhook 编辑弹窗 |

### 8.2 关键函数

| 函数 | 文件:行号 | 功能 |
|------|-----------|------|
| `WebhookTrigger.trigger()` | `webhook-trigger.js:105` | 普通 Webhook 触发主流程 |
| `WebhookTrigger.onError()` | `webhook-trigger.js:88` | 普通 Webhook 失败处理 |
| `WebhookTrigger.update()` | `webhook-trigger.js:58` | 状态字段落库 |
| `WebhookTrigger.destroy()` | `webhook-trigger.js:71` | 410 时删除记录 |
| `VerificationWebhookService.sendVerificationWebhook()` | `verification-webhook-service.ts:69` | Verification Webhook 发送 |
| `VerificationTrigger._startVerificationProcess()` | `verification-trigger.js:217` | Verification 流程控制 |
| `VerificationTrigger._handleMemberCreatedEvent()` | `verification-trigger.js:67` | 会员创建事件处理 |
| `useBrowseIntegrations` | `integrations.ts:31` | 前端查询 Hook |
| `Integration.webhooks()` | `integration.js:65` | ORM 关联定义 |

### 8.3 数据库字段

| 表名 | 字段 | 类型 | 用途 |
|------|------|------|------|
| `webhooks` | `last_triggered_at` | dateTime | 最后触发时间 |
| `webhooks` | `last_triggered_status` | string(50) | HTTP 状态码 |
| `webhooks` | `last_triggered_error` | string(50) | 错误描述 |
| `settings` | `email_verification_required` | boolean | Verification 流程标记 |

---

*报告生成时间：2026-05-11*
*基于 Ghost 代码库 commit: 工作区当前状态*
