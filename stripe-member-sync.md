# Stripe 订阅事件与会员状态同步分析报告

## 概述

本报告详细分析 Ghost 中 Stripe 订阅事件如何同步为会员状态变更，包括 webhook 接收机制、幂等处理、状态映射、订阅生命周期事件处理以及通知系统的协作方式。

## 1. Webhook 接收与验证

### 1.1 事件监听

Ghost 监听以下 Stripe webhook 事件（定义于 `webhook-manager.js:44-51`）：

| 事件类型 | 处理服务 | 说明 |
|---------|---------|------|
| `checkout.session.completed` | `CheckoutSessionEventService` | 结账完成，包括订阅、捐赠、礼品购买 |
| `customer.subscription.created` | `SubscriptionEventService` | 订阅创建 |
| `customer.subscription.updated` | `SubscriptionEventService` | 订阅更新（状态变更、价格变更等） |
| `customer.subscription.deleted` | `SubscriptionEventService` | 订阅取消 |
| `invoice.payment_succeeded` | `InvoiceEventService` | 发票支付成功 |
| `charge.refunded` | `ChargeRefundedEventService` | 退款处理 |

### 1.2 签名验证流程

Webhook 签名验证确保请求确实来自 Stripe，防止伪造请求。

**流程**（`webhook-controller.js:76-109`）：
```
1. 检查请求体和 stripe-signature 头
   ↓
2. WebhookManager.parseWebhook() 调用 Stripe SDK
   ↓
3. Stripe.webhooks.constructEvent() 验证签名
   ↓
4. 成功 → 处理事件；失败 → 返回 401
```

**关键点**：
- 使用 Stripe webhook secret 进行 HMAC 签名验证
- 验证失败立即返回 401 Unauthorized
- 配置支持 `webhookSecret`（用于 stripe-cli 本地测试）和动态创建的 webhook endpoint

### 1.3 路由处理

**Webhook 处理入口** (`webhook-controller.js:117-159`)：

```javascript
async handleEvent(event) {
    if (!this.handlers[event.type]) {
        return;  // 忽略未注册的事件类型
    }
    await this.handlers[event.type].call(this, event.data.object);
}
```

**事件忽略列表**：
- 支持 `stripeWebhookCustomerIgnoreList` 配置，可忽略特定 customer 的 `customer.subscription.updated` 事件
- 用于测试或特殊场景

## 2. 幂等处理机制

Stripe webhook 可能重复发送或乱序到达，Ghost 采用多层幂等保护策略。

### 2.1 数据库唯一约束

**核心策略**：利用数据库唯一约束 + 捕获重复错误

**示例** (`subscription-event-service.js:40-51`)：
```javascript
try {
    await memberRepository.linkSubscription({
        id: member.id,
        subscription
    });
} catch (err) {
    if (err.code !== 'ER_DUP_ENTRY' && err.code !== 'SQLITE_CONSTRAINT') {
        throw err;
    }
    throw new errors.ConflictError({err});
}
```

**覆盖场景**：
- 相同 `subscription_id` 重复插入
- 相同 `customer_id` 重复关联
- 多个 webhook 事件并发处理

### 2.2 Upsert 操作

使用 upsert（insert or update）而非单纯 insert，确保重复事件安全处理：

**示例** (`member-repository.js:974-981`)：
```javascript
async upsertCustomer(data) {
    return await this._StripeCustomer.upsert({
        customer_id: data.customer_id,
        member_id: data.member_id,
        name: data.name,
        email: data.email
    });
}
```

### 2.3 事务 + 行级锁

在 `linkSubscription` 中使用：
- **数据库事务** (`member-repository.js:1036-1043`)：确保操作原子性
- **行级锁** (`forUpdate: true`)：防止并发修改同一成员状态

```javascript
return this._Member.transaction((transacting) => {
    return this.linkSubscription(data, {
        ...options,
        transacting
    });
});
```

### 2.4 事件去重

**Offer 赎回事件去重** (`member-repository.js:113-133`)：
```javascript
DomainEvents.subscribe(OfferRedemptionEvent, async function (event) {
    const existingRedemption = await OfferRedemption.findOne({
        member_id: event.data.memberId,
        subscription_id: event.data.subscriptionId,
        offer_id: event.data.offerId
    });
    
    if (!existingRedemption) {
        await OfferRedemption.add({...});
    }
});
```

## 3. Tier 与价格的状态映射

### 3.1 会员状态枚举

**会员状态** (`member-repository.js:42`)：
```javascript
const MEMBER_STATUSES = ['free', 'paid', 'comped', 'gift'];
```

| 状态 | 说明 |
|-----|------|
| `free` | 免费会员 |
| `paid` | 付费订阅会员 |
| `comped` | 赠送/内部会员（Complimentary） |
| `gift` | 礼品兑换会员 |

### 3.2 订阅状态到会员状态映射

**订阅状态判断** (`member-repository.js:164-166`)：
```javascript
isActiveSubscriptionStatus(status) {
    return ['active', 'trialing', 'unpaid', 'past_due'].includes(status);
}
```

**内部订阅状态** (`member-repository.js:1186-1203`)：
```javascript
const getStatus = (modelToCheck) => {
    const status = modelToCheck.get('status');
    const canceled = modelToCheck.get('cancel_at_period_end');
    
    if (status === 'canceled') {
        return 'expired';           // 已过期
    }
    if (canceled) {
        return 'canceled';          // 已取消但在期限内
    }
    if (this.isActiveSubscriptionStatus(status)) {
        return 'active';            // 活跃
    }
    return 'inactive';              // 非活跃
};
```

### 3.3 会员状态计算逻辑

**核心逻辑** (`member-repository.js:1390-1484`)：

```javascript
// 默认状态
let status = memberProducts.length === 0 ? 'free' : 'comped';

// 有活跃订阅
if (this.isActiveSubscriptionStatus(stripeSubscriptionData.status)) {
    if (this.isComplimentarySubscription(stripeSubscriptionData)) {
        status = 'comped';    // 赠送订阅
    } else {
        status = 'paid';      // 付费订阅
    }
    // ... 关联产品
} else {
    // 无活跃订阅
    const subscriptions = await memberModel.related('stripeSubscriptions').fetch(options);
    let activeSubscriptionForGhostProduct = false;
    
    for (const subscriptionModel of subscriptions.models) {
        if (this.isActiveSubscriptionStatus(subscriptionModel.get('status'))) {
            status = 'paid';  // 其他活跃订阅
        }
    }
    
    if (memberProducts.length === 0) {
        status = 'free';      // 无产品则免费
    }
}
```

### 3.4 Tier (Product) 与 Stripe 的关联

**ProductRepository** 管理 Ghost Tier 与 Stripe Product/Price 的映射：

- **StripeProduct**: 关联 Ghost Product ↔ Stripe Product
- **StripePrice**: 关联 Stripe Price，包含：
  - `stripe_price_id` / `stripe_product_id`
  - `currency`, `amount`, `interval` (month/year/week/day)
  - `nickname`, `active`

**MRR 计算** (`member-repository.js:250-294`)：
- trial/incomplete/canceled 状态 MRR = 0
- year interval ÷ 12 = 月收入
- week interval × 4 = 月收入
- day interval × 30 = 月收入
- 永久折扣 (forever) 自动应用

## 4. 订阅生命周期事件处理

### 4.1 新订阅流程

**事件序列**（可乱序到达）：
```
1. checkout.session.completed → CheckoutSessionEventService
2. customer.subscription.created → SubscriptionEventService  
3. customer.subscription.updated → SubscriptionEventService
4. invoice.payment_succeeded → InvoiceEventService
```

**CheckoutSession 处理** (`checkout-session-event-service.js:198-304`)：

```
新会员路径：
  1. 通过 email 查找会员
  2. 未找到 → 创建新会员 + 关联 Stripe Customer
  3. linkSubscription() 关联订阅
  4. 移除赠送订阅（如已有）
  5. 发送 Magic Link 登录邮件

老会员路径：
  1. 更新姓名（如为空）
  2. upsertCustomer() 更新客户信息
  3. linkSubscription() 关联订阅
  4. 移除赠送订阅
```

### 4.2 订阅状态变更处理

**核心函数**：`linkSubscription()` (`member-repository.js:1031-1545`)

**步骤**：

1. **获取最新订阅数据**：从 Stripe API 获取完整订阅信息（含支付方式）
2. **关联/更新 Product**：通过 `stripe_product_id` 查找 Ghost Product
3. **处理 Offer**：将 Stripe coupon 映射为 Ghost Offer
4. **Upsert 订阅数据**：更新 `StripeCustomerSubscription` 表
5. **状态变更检测**：对比新旧状态
6. **更新会员状态**：重新计算会员的 `status` 和关联的 `products`
7. **发送事件**：根据状态变化分发 Domain Events

### 4.3 关键状态转换

**订阅激活** (`member-repository.js:1286-1299`)：
```javascript
if (originalStatus !== 'active' && updatedStatus === 'active') {
    const subscriptionActivatedEvent = SubscriptionActivatedEvent.create({
        source,
        tierId: ghostProduct?.get('id'),
        memberId: memberModel.id,
        subscriptionId: updatedStripeCustomerSubscriptionModel.get('id'),
        offerId: offerId,
        batchId: options.batch_id
    });
    this.dispatchEvent(subscriptionActivatedEvent, options);
}
```

**订阅取消/过期** (`member-repository.js:1301-1322`)：
```javascript
if (this.isActiveSubscriptionStatus(originalStatus) && 
    (updatedStatus === 'canceled' || updatedStatus === 'expired')) {
    const subscriptionCancelledEvent = SubscriptionCancelledEvent.create({
        source,
        tierId: ghostProduct?.get('id'),
        memberId: memberModel.id,
        subscriptionId: updatedStripeCustomerSubscriptionModel.get('id'),
        cancelNow,        // 立即取消 vs 到期取消
        canceledAt,
        expiryAt
    });
    this.dispatchEvent(subscriptionCancelledEvent, options);
}
```

**新订阅创建** (`member-repository.js:1324-1387`)：
- `SubscriptionCreatedEvent`: 订阅记录创建
- `SubscriptionActivatedEvent`: 订阅已激活（如状态为 active）
- `OfferRedemptionEvent`: 有优惠码时

### 4.4 付费订阅事件记录

**MemberPaidSubscriptionEvent** (`member-repository.js:1272-1281`)：
记录每次订阅状态变更的详细信息：
- `type`: created / updated / reactivated / active / canceled / expired
- `from_plan` / `to_plan`: 价格变更
- `mrr_delta`: 月收入变化
- `currency`

## 5. 事件通知与协作

### 5.1 Domain Events 架构

Ghost 使用发布-订阅模式 (`@tryghost/domain-events`)：

**事件分发** (`member-repository.js:143-162`)：
```javascript
dispatchEvent(event, options) {
    if (options?.transacting) {
        // 事务提交后再分发
        options.transacting.executionPromise.then(async () => {
            DomainEvents.dispatch(event);
        }).catch(...);
    } else {
        DomainEvents.dispatch(event);
    }
}
```

**关键事件**：
- `MemberCreatedEvent`: 新会员创建
- `SubscriptionCreatedEvent`: 订阅创建
- `SubscriptionActivatedEvent`: 订阅激活
- `SubscriptionCancelledEvent`: 订阅取消
- `OfferRedemptionEvent`: 优惠码兑换
- `MemberSubscribeEvent`: 订阅通讯

### 5.2 员工通知 (StaffService)

**订阅者注册** (`staff-service.js:142-178`)：
```javascript
subscribeEvents() {
    this.DomainEvents.subscribe(MemberCreatedEvent, async (event) => {
        await this.emails.notifyFreeMemberSignup({...});
    });
    
    this.DomainEvents.subscribe(SubscriptionActivatedEvent, async (event) => {
        await this.emails.notifyPaidSubscriptionStarted({...});
    });
    
    this.DomainEvents.subscribe(SubscriptionCancelledEvent, async (event) => {
        await this.emails.notifyPaidSubscriptionCanceled({...});
    });
}
```

**通知邮件包含**：
- **付费订阅开始**: 会员信息、tier 信息、订阅详情、优惠信息、归因数据
- **付费订阅取消**: 取消原因、到期时间、金额
- **免费会员注册**: 会员信息、归因数据

### 5.3 会员欢迎邮件

**欢迎邮件自动化** (`member-repository.js:191-225`)：
```javascript
async enqueueWelcomeEmailRun(memberId, slug, options = {}) {
    const automation = await this._WelcomeEmailAutomation.findOne({slug}, {...});
    
    if (isActive) {
        const run = await this._WelcomeEmailAutomationRun.add({
            welcome_email_automation_id: automation.id,
            member_id: memberId,
            ready_at: new Date(),
            ...
        });
        this.dispatchEvent(StartAutomationsPollEvent.create(), options);
    }
}
```

**触发条件** (`member-repository.js:1534-1544`)：
- 状态变更为 `paid`
- 前置状态不是 `gift`（礼品会员已在兑换时触发）
- 来源为 `member`（正常注册流程）

### 5.4 登录邮件 (Magic Link)

**Checkout 完成后** (`checkout-session-event-service.js:291-304`)：
```javascript
if (checkoutType !== 'upgrade') {
    const shouldSkip = canWelcomeEmailReplaceSignupPaidEmail(ghostSignupContext);
    
    if (shouldSkip) {
        const isPaidWelcomeEmailActive = await this.deps.isPaidWelcomeEmailActive();
        if (!isPaidWelcomeEmailActive) {
            this.deps.sendSignupEmail(customer.email);  // 无欢迎邮件时发送
        }
    } else {
        this.deps.sendSignupEmail(customer.email);      // 直接发送
    }
}
```

**优先级**：
1. 欢迎邮件自动化 → 优先
2. Magic Link 登录邮件 → 作为 fallback

### 5.5 支付记录

**Invoice 支付成功** (`invoice-event-service.js:27-69`)：
```javascript
async handleInvoiceEvent(invoice) {
    if (invoice.paid && invoice.amount_paid !== 0) {
        await eventRepository.registerPayment({
            member_id: member.id,
            currency: invoice.currency,
            amount: invoice.amount_paid
        });
    }
}
```

## 6. 数据流协作图

### 6.1 完整订阅流程

```
┌─────────────┐     Webhook Events (可乱序)     ┌─────────────────┐
│   Stripe    │─────────────────────────────────▶│  Ghost Backend  │
└─────────────┘                                  └────────┬────────┘
                                                          │
              ┌───────────────────────────────────────────┼───────────────────────────────────┐
              │                                           │                                   │
              ▼                                           ▼                                   ▼
  ┌──────────────────────┐                    ┌──────────────────────┐            ┌──────────────────────┐
  │   WebhookController  │                    │  MemberRepository    │            │     StaffService     │
  │   - 签名验证         │                    │  - linkSubscription  │            │  - 订阅 DomainEvents │
  │   - 事件路由         │                    │  - 状态计算          │            │  - 发送通知邮件       │
  └──────────┬───────────┘                    └──────────┬───────────┘            └──────────────────────┘
             │                                           │
             ▼                                           ▼
  ┌──────────────────────┐                    ┌──────────────────────┐
  │ SubscriptionEvent    │                    │   DomainEvents       │
  │ CheckoutSessionEvent │                    │  - 事务后分发        │
  │ InvoiceEvent         │                    │  - 解耦处理          │
  └──────────────────────┘                    └──────────────────────┘
```

### 6.2 状态变更时序

```
Stripe 事件                    Ghost 处理                        会员状态                  通知
──────────                   ───────────                      ─────────                ──────
                             ┌──────────┐
checkout.session.completed ─▶│ 创建会员 │ ──▶ free → paid ──▶ SubscriptionCreatedEvent
                             │ 关联订阅 │                      SubscriptionActivatedEvent
                             └──────────┘                                         │
                                                                                    ▼
                                                                           欢迎邮件入队
                                                                           登录邮件发送
                                                                           员工通知邮件

customer.subscription.updated ─▶ linkSubscription() ──▶ 计算状态 ──▶ MemberStatusEvent
    (cancel_at_period_end)                                             │
                                                                       ▼
                                                                SubscriptionCancelledEvent
                                                                       │
                                                                       ▼
                                                                员工取消通知邮件

customer.subscription.updated ─▶ linkSubscription() ──▶ paid → comped ─▶ MemberStatusEvent
    (complimentary plan)

customer.subscription.deleted ─▶ linkSubscription() ──▶ paid → free ──▶ MemberStatusEvent
                                                                       │
                                                                       ▼
                                                                SubscriptionCancelledEvent
```

## 7. 关键代码位置

| 功能 | 文件路径 | 核心函数 |
|-----|---------|---------|
| Webhook 接收 | `webhook-controller.js` | `handle()`, `handleEvent()` |
| Webhook 管理 | `webhook-manager.js` | `parseWebhook()`, `start()` |
| 订阅事件处理 | `services/webhook/subscription-event-service.js` | `handleSubscriptionEvent()` |
| Checkout 事件处理 | `services/webhook/checkout-session-event-service.js` | `handleSubscriptionEvent()` |
| 会员状态同步 | `members-api/repositories/member-repository.js` | `linkSubscription()` |
| 状态计算 | `member-repository.js` | `getStatus()`, `isActiveSubscriptionStatus()` |
| Product/Tier 管理 | `members-api/repositories/product-repository.js` | `get()`, `create()`, `update()` |
| 员工通知 | `staff/staff-service.js` | `subscribeEvents()`, `handleEvent()` |
| 欢迎邮件 | `member-repository.js` | `enqueueWelcomeEmailRun()` |
| 事件定义 | `shared/events/*.js` | 各类 Event 类 |

## 8. 总结

Ghost 的 Stripe 订阅同步系统具有以下特点：

1. **高可靠性**：多层幂等保护（签名验证、唯一约束、upsert、行级锁）
2. **事件驱动**：通过 Domain Events 解耦订阅状态变更与通知处理
3. **状态一致**：事务提交后才分发事件，确保数据与通知一致
4. **乱序兼容**：Stripe 事件可能乱序到达，upsert + 状态比较确保最终一致性
5. **智能通知**：区分欢迎邮件、登录邮件、员工通知，避免重复打扰

这个设计确保了即使在网络不稳定、Stripe 重复推送、或并发处理场景下，会员状态依然能够正确同步。
