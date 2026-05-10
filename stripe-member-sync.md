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

## 6. 事件级映射表与状态偏差分析

本节详细分析每个 Stripe webhook 事件触发的连锁反应，包括 Ghost Domain Events、会员通知、前端反馈，以及乱序/重复事件时可能出现的状态偏差。

### 6.1 Stripe 事件 → 会员通知 → 前端反馈 完整映射表

#### 6.1.1 事件流转总览

```
Stripe Webhook Event
       ↓
WebhookController 接收 + 签名验证
       ↓
对应 EventService 处理
       ↓
memberRepository.linkSubscription()
       ├── 更新数据库状态
       ├── 记录 MemberPaidSubscriptionEvent
       └── 分发 DomainEvents
               ↓
        各类订阅者处理
               ├── StaffService → 员工通知邮件
               ├── MemberRepository → 欢迎邮件
               ├── VerificationTrigger → 验证触发
               └── OfferRedemption → 优惠记录
       ↓
前端主动拉取 (sessionData) 或 URL 回调刷新
```

#### 6.1.2 事件级详细映射表

| Stripe Webhook 事件 | 触发场景 | Ghost Domain Events | 会员通知 | Admin 通知 | 前端可见反馈 |
|---------------------|---------|--------------------|----------|------------|-------------|
| **checkout.session.completed** | 新订阅/升级/礼品购买 | `MemberCreatedEvent` (新会员) → `SubscriptionCreatedEvent` → `SubscriptionActivatedEvent` (如 active) → `OfferRedemptionEvent` (如有优惠) | 1. 欢迎邮件自动化 (paid welcome)<br>2. 或 Magic Link 登录邮件 | 1. `notifyFreeMemberSignup` (free)<br>2. `notifyPaidSubscriptionStarted` (paid) | Portal: `?stripe=success` → "Success!" 通知 → 关闭时刷新会员数据 |
| **customer.subscription.created** | Stripe 创建订阅记录 | `SubscriptionCreatedEvent` → `SubscriptionActivatedEvent` (如 active) | 无 (checkout 场景已处理) | 同 activated 时 | 无直接反馈 (依赖 checkout 回调或主动刷新) |
| **customer.subscription.updated** (状态: incomplete→active) | 3D Secure 验证完成 | `SubscriptionActivatedEvent` | 欢迎邮件 (如之前未发送) | `notifyPaidSubscriptionStarted` | 无直接反馈 (需刷新页面) |
| **customer.subscription.updated** (状态: active→past_due) | 首次扣款失败 | 无 DomainEvent (仅更新状态为 unpaid) | 无 | 无 | Portal: 仍显示 paid，但状态可能异常 |
| **customer.subscription.updated** (cancel_at_period_end=true) | 用户在 Portal 或 Stripe 取消 | `SubscriptionCancelledEvent` (cancelNow=false) | 无 | `notifyPaidSubscriptionCanceled` (到期取消) | Portal: 显示 "Renew subscription" 按钮，显示到期日期 |
| **customer.subscription.updated** (状态: canceled→active) | 到期前恢复订阅 | `SubscriptionActivatedEvent` | 无 | 无 | Portal: "Renew subscription" 按钮变回 "Cancel subscription" |
| **customer.subscription.deleted** | 立即取消 (退款/管理员操作) | `SubscriptionCancelledEvent` (cancelNow=true) | 无 | `notifyPaidSubscriptionCanceled` (立即取消) | Portal: 状态变为 free，显示升级选项 |
| **invoice.payment_succeeded** | 周期性扣款成功 | 无 (仅记录 payment) | 无 | 无 | 无直接反馈 |
| **charge.refunded** | 退款处理 | 无 | 无 | 无 | 无直接反馈 |

#### 6.1.3 MemberPaidSubscriptionEvent 数据库记录

每次订阅状态变更都会在 `members_paid_subscription_events` 表记录一行：

| 字段 | 说明 | 示例值 |
|-----|------|-------|
| `source` | 事件来源 | `stripe` |
| `type` | 事件类型 | `created` / `updated` / `reactivated` / `active` / `canceled` / `expired` |
| `from_plan` | 变更前价格 ID | `price_xxx` (null 表示新建) |
| `to_plan` | 变更后价格 ID | `price_xxx` (null 表示取消) |
| `currency` | 货币 | `usd` |
| `mrr_delta` | MRR 变化量 | `999` (月收入增加 ¥9.99) |
| `created_at` | 事件时间 | `2026-05-10 10:00:00` |

**type 字段映射** (`member-repository.js:1254-1264`)：
```javascript
const getEventType = (originalStatus, updatedStatus) => {
    if (originalStatus === updatedStatus) return 'updated';      // 价格/优惠变更
    if (originalStatus === 'canceled' && updatedStatus === 'active') return 'reactivated';  // 恢复订阅
    return updatedStatus;  // active / canceled / expired
};
```

#### 6.1.4 事件 → 观测点 → 定位动作 对照表

本节为每个 Stripe 事件提供最小排查路径，包括日志关键字、数据库核对字段、前端可见症状，以及 Portal 与 Admin 的反馈差异。

**核心排查原则**：
1. **日志先行**：先确认事件是否被接收和处理
2. **数据库核对**：确认状态是否正确持久化
3. **前端症状**：区分 Portal（会员视角）与 Admin（运营视角）的差异
4. **定位动作**：给出可直接执行的检查步骤

---

**对照表总览**：

| Stripe Webhook 事件 | 日志关键字 | 数据库核对字段 | Portal 可见症状 | Admin 可见症状 | 最小定位动作 |
|---------------------|-----------|---------------|----------------|----------------|-------------|
| **checkout.session.completed** | `Handling webhook checkout.session.completed` | `members.status` = 'paid'<br>`members_stripe_customers_subscriptions.status` = 'active' | ✅ 显示"Success!"通知<br>✅ 关闭后刷新会员数据<br>✅ `?stripe=success` URL 参数 | ✅ 收到 `notifyPaidSubscriptionStarted` 邮件<br>✅ 会员列表状态变为 Paid | 1. 查日志确认 webhook 到达<br>2. 查 `members_paid_subscription_events.type` = 'created'<br>3. 查 `members.status` |
| **customer.subscription.created** | `Handling webhook customer.subscription.created` | `members_stripe_customers_subscriptions` 新增记录 | ❌ 无直接反馈（依赖 checkout 回调） | ✅ 如状态为 active，收到激活通知 | 1. 查日志确认 webhook 到达<br>2. 查订阅表是否新增记录 |
| **customer.subscription.updated** (incomplete→active) | `Handling webhook customer.subscription.updated` | `members_stripe_customers_subscriptions.status` 从 'incomplete' → 'active' | ❌ 无直接反馈（需刷新页面） | ✅ 收到 `notifyPaidSubscriptionStarted` 邮件 | 1. 查日志确认 webhook 到达<br>2. 查 `members_paid_subscription_events.type` = 'active'<br>3. 对比前后 `status` 变化 |
| **customer.subscription.updated** (active→past_due) | `Handling webhook customer.subscription.updated` | `members_stripe_customers_subscriptions.status` = 'past_due'<br>`mrr` = 0 | ⚠️ 仍显示 Paid 状态<br>⚠️ `isActiveSubscriptionStatus()` 仍返回 true | ❌ 无邮件通知<br>⚠️ 会员列表仍显示 Paid | 1. 查日志确认 webhook 到达<br>2. 查订阅表 `status` 和 `mrr` 字段<br>3. 注意：前端可能无感知 |
| **customer.subscription.updated** (cancel_at_period_end=true) | `Handling webhook customer.subscription.updated` | `members_stripe_customers_subscriptions.cancel_at_period_end` = true | ✅ 显示"Renew subscription"按钮<br>✅ 显示到期日期 | ✅ 收到 `notifyPaidSubscriptionCanceled` 邮件（到期取消）<br>✅ 会员列表显示 Canceling | 1. 查日志确认 webhook 到达<br>2. 查 `cancel_at_period_end` 字段<br>3. 查 `members_paid_subscription_events.type` = 'canceled' |
| **customer.subscription.updated** (canceled→active) | `Handling webhook customer.subscription.updated` | `members_stripe_customers_subscriptions.cancel_at_period_end` = false<br>`status` = 'active' | ✅ "Renew subscription" 按钮消失<br>✅ 恢复正常付费 UI | ❌ 无邮件通知<br>✅ 会员列表状态恢复 Paid | 1. 查日志确认 webhook 到达<br>2. 查 `members_paid_subscription_events.type` = 'reactivated' |
| **customer.subscription.deleted** | `Handling webhook customer.subscription.deleted` | `members_stripe_customers_subscriptions.status` = 'deleted' 或记录被移除<br>`members.status` = 'free' | ✅ 状态变为 Free<br>✅ 显示升级选项 | ✅ 收到 `notifyPaidSubscriptionCanceled` 邮件（立即取消）<br>✅ 会员列表显示 Canceled | 1. 查日志确认 webhook 到达<br>2. 查 `members.status` = 'free'<br>3. 查 `members_paid_subscription_events.type` = 'expired' |
| **invoice.payment_succeeded** | `Handling webhook invoice.payment_succeeded` | `members_payments` 新增记录 | ❌ 无直接反馈 | ❌ 无邮件通知 | 1. 查日志确认 webhook 到达<br>2. 查 `members_payments` 表是否有记录 |
| **charge.refunded** | `Handling webhook charge.refunded` | 无直接字段更新（需业务逻辑处理） | ❌ 无直接反馈 | ❌ 无邮件通知 | 1. 查日志确认 webhook 到达<br>2. 需手动核对 Stripe Dashboard |

---

#### 6.1.5 Portal 与 Admin 前端反馈差异对照表

**设计差异根源**：
- **Portal**：面向付费会员，关注"我有什么权限"、"我的订阅状态"
- **Admin**：面向运营人员，关注"谁订阅了"、"收入多少"、"是否需要跟进"

| 场景 | Portal 反馈（会员视角） | Admin 反馈（运营视角） | 差异原因 |
|-----|------------------------|----------------------|---------|
| **新订阅成功** | ✅ "Success!" 通知（3秒自动消失）<br>✅ 关闭后自动刷新会员数据<br>✅ 登录后显示"Welcome, {name}!" | ✅ 邮件通知："New paid subscription"<br>✅ 会员列表：状态从 Free → Paid<br>✅ 仪表盘：MRR 增加 | Portal：即时视觉反馈 + 数据刷新<br>Admin：邮件 + 数据列表更新 |
| **订阅取消（到期取消）** | ✅ "Renew subscription" 按钮<br>✅ 显示到期日期<br>✅ 仍可访问付费内容 | ✅ 邮件通知："Paid subscription canceled"<br>✅ 会员列表：状态 "Canceling"<br>✅ 仪表盘：预期 MRR 下降 | Portal：强调"还能继续用，要续费吗？"<br>Admin：强调"有流失风险，需要跟进" |
| **订阅取消（立即取消）** | ✅ 状态变为 Free<br>✅ 无法访问付费内容<br>✅ 显示"Upgrade"按钮 | ✅ 邮件通知："Paid subscription canceled (immediate)"<br>✅ 会员列表：状态 "Canceled"<br>✅ 仪表盘：MRR 立即下降 | Portal：即时权限降级<br>Admin：明确区分到期取消 vs 立即取消 |
| **付款失败 (past_due)** | ⚠️ 仍显示 Paid 状态<br>⚠️ 无任何提示<br>⚠️ 仍可访问付费内容 | ❌ 无邮件通知<br>⚠️ 会员列表仍显示 Paid<br>⚠️ 仪表盘 MRR 已下降 (mrr=0) | **⚠️ 风险点**：双方都无感知<br>Portal：`isActiveSubscriptionStatus()` 包含 past_due<br>Admin：依赖外部支付失败提醒 |
| **恢复订阅** | ✅ "Renew subscription" 按钮消失<br>✅ 恢复正常付费 UI | ❌ 无邮件通知<br>✅ 会员列表恢复 Paid 状态 | Portal：即时 UI 变化<br>Admin：无主动通知，需查看列表 |
| **价格变更/优惠变更** | ✅ 显示新价格信息<br>✅ 显示优惠码（如有） | ❌ 无邮件通知<br>✅ 会员详情显示新价格 | Portal：关注"我付多少钱"<br>Admin：关注"收入变化"（MRR delta 记录） |

---

#### 6.1.6 每个事件的最小排查路径（详细版）

**排查路径格式**：
```
日志确认 → 数据库核对 → 前端验证 → 定位动作
```

**事件 1: checkout.session.completed**

```
日志确认:
  搜索: "Handling webhook checkout.session.completed"
  正常: 日志存在，无 Error 级别日志
  异常:
    - 找不到日志 → webhook 未到达或被忽略（检查 ignore list）
    - "ConflictError" → 重复事件（数据库唯一约束）
    - "No member found" → 乱序事件（subscription.updated 先到）

数据库核对:
  1. SELECT * FROM members_stripe_webhook_events 
     WHERE type = 'checkout.session.completed'
     ORDER BY created_at DESC LIMIT 1;
  2. SELECT id, email, status FROM members 
     WHERE email = '{会员邮箱}';
     → 期望: status = 'paid'
  3. SELECT id, status, cancel_at_period_end 
     FROM members_stripe_customers_subscriptions
     WHERE member_id = '{会员ID}';
     → 期望: status = 'active'
  4. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'created'

前端验证 (Portal):
  - 地址栏有 ?stripe=success 参数
  - 显示 "Success! Your account is fully activated..."
  - 关闭通知后刷新，member.paid = true
  - 显示当前订阅价格

前端验证 (Admin):
  - 邮件：收到 "New paid subscription from {email}"
  - 会员列表：状态 = Paid
  - 会员详情：显示订阅信息

定位动作:
  1. 查 Stripe Dashboard → Webhooks → 确认事件已发送
  2. 查 Ghost 日志 → 确认事件被接收
  3. 查数据库 → 确认状态已更新
  4. 查 Domain Events → 确认邮件已发送
```

**事件 2: customer.subscription.updated (cancel_at_period_end=true)**

```
日志确认:
  搜索: "Handling webhook customer.subscription.updated"
  正常: 日志存在
  异常: "No member found for Stripe customer" → 会员未关联

数据库核对:
  1. SELECT id, status, cancel_at_period_end, current_period_end
     FROM members_stripe_customers_subscriptions
     WHERE member_id = '{会员ID}';
     → 期望: cancel_at_period_end = 1
  2. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'canceled'
  3. SELECT id, email, status FROM members 
     WHERE id = '{会员ID}';
     → 期望: status = 'paid' (注意：仍为 paid，只是 cancel_at_period_end)

前端验证 (Portal):
  - 显示 "Your subscription will end on {日期}"
  - "Cancel subscription" 按钮变为 "Renew subscription"
  - 仍可访问付费内容

前端验证 (Admin):
  - 邮件：收到 "Paid subscription canceled"
  - 会员列表：状态 = Canceling
  - 会员详情：显示 "Expires on {日期}"

定位动作:
  1. 查日志确认 webhook 到达
  2. 查 cancel_at_period_end 字段
  3. 查 cancel_now 参数（区分立即取消 vs 到期取消）
  4. 确认取消来源（Portal 操作 vs Stripe Dashboard 操作）
```

**事件 3: customer.subscription.deleted（立即取消）**

```
日志确认:
  搜索: "Handling webhook customer.subscription.deleted"

数据库核对:
  1. SELECT id, email, status FROM members 
     WHERE id = '{会员ID}';
     → 期望: status = 'free'
  2. SELECT id, status FROM members_stripe_customers_subscriptions
     WHERE member_id = '{会员ID}';
     → 期望: 记录不存在 或 status = 'deleted'
  3. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'expired'

前端验证 (Portal):
  - 状态变为 Free
  - 无法访问付费内容（返回 403 或重定向）
  - 显示 "Upgrade" 按钮

前端验证 (Admin):
  - 邮件：收到 "Paid subscription canceled (immediate)"
  - 会员列表：状态 = Canceled
  - 仪表盘：MRR 立即下降

定位动作:
  1. 确认是立即取消（cancel_now = true）还是到期取消
  2. 查 members.status 是否已变为 free
  3. 确认 MRR 计算正确（mrr_delta 应为负数）
  4. 确认取消原因（退款？欺诈？管理员操作？）
```

**事件 4: customer.subscription.updated (active→past_due)**

```
日志确认:
  搜索: "Handling webhook customer.subscription.updated"
  → 注意：无错误日志，无 DomainEvent 分发

数据库核对:
  1. SELECT id, status, mrr FROM members_stripe_customers_subscriptions
     WHERE member_id = '{会员ID}';
     → 期望: status = 'past_due', mrr = 0
  2. SELECT id, email, status FROM members 
     WHERE id = '{会员ID}';
     → 期望: status = 'paid' (⚠️ 注意：仍为 paid！)
  3. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'past_due', mrr_delta = -{原金额}

前端验证 (Portal):
  ⚠️ 仍显示 Paid 状态
  ⚠️ 无任何提示
  ⚠️ 仍可访问付费内容
  ⚠️ isActiveSubscriptionStatus() 返回 true

前端验证 (Admin):
  ❌ 无邮件通知
  ⚠️ 会员列表仍显示 Paid
  ⚠️ 但仪表盘 MRR 已下降

定位动作:
  1. ⚠️ 这是高风险场景：前后端都无感知
  2. 查订阅表 mrr 字段（应为 0）
  3. 查 Stripe Dashboard → 是否有付款失败提醒
  4. 建议：建立付款失败的主动提醒机制
  5. 查 members_paid_subscription_events.mrr_delta
```

**事件 5: invoice.payment_succeeded**

```
日志确认:
  搜索: "Handling webhook invoice.payment_succeeded"

数据库核对:
  1. SELECT * FROM members_payments 
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: 新增记录，amount = invoice.amount_paid

前端验证 (Portal):
  ❌ 无直接反馈

前端验证 (Admin):
  ❌ 无邮件通知
  ✅ 会员详情显示最新付款记录

定位动作:
  1. 查 members_payments 表是否有记录
  2. 查 invoice.paid 和 invoice.amount_paid 字段
  3. 确认金额和币种正确
  4. 对于订阅场景：此事件通常伴随 subscription.updated
```

**事件 6: customer.subscription.created**

```
日志确认:
  搜索: "Handling webhook customer.subscription.created"
  正常: 日志存在
  异常:
    - 找不到日志 → webhook 未到达
    - "No member found for Stripe customer" → 会员未关联（可能是新会员，checkout 事件还未到）
    - "ConflictError" → 重复事件

数据库核对:
  1. SELECT * FROM members_stripe_webhook_events 
     WHERE type = 'customer.subscription.created'
     ORDER BY created_at DESC LIMIT 1;
  2. SELECT id, status, stripe_subscription_id, stripe_price_id, stripe_plan_id
     FROM members_stripe_customers_subscriptions
     WHERE stripe_subscription_id = '{subscription_id}';
     → 期望: 存在新记录
  3. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE subscription_id = (SELECT id FROM members_stripe_customers_subscriptions 
                             WHERE stripe_subscription_id = '{subscription_id}')
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'created'
  4. SELECT id, email, status FROM members 
     WHERE id = (SELECT member_id FROM members_stripe_customers_subscriptions 
                  WHERE stripe_subscription_id = '{subscription_id}');
     → 期望: 存在会员记录

前端验证 (Portal):
  ❌ 无直接反馈
  ⚠️ 注意：此事件通常在 checkout.session.completed 之后或同时到达
  ⚠️ 如果会员已登录，刷新页面后应能看到订阅状态

前端验证 (Admin):
  ✅ 如订阅状态为 active，会收到 SubscriptionActivatedEvent 对应的邮件通知
  ✅ 会员列表应显示新的付费会员
  ✅ 会员详情应显示订阅信息

定位动作:
  1. 查 Stripe Dashboard → Subscriptions → 确认订阅已创建
  2. 查 Ghost 日志 → 确认事件被接收
  3. 查订阅表是否新增记录
  4. 检查是否有对应的 checkout.session.completed 事件
  5. 如果是新会员：检查 members 表是否创建
  6. 如果 status=active：检查 SubscriptionCreatedEvent 是否分发
```

**事件 7: customer.subscription.updated (incomplete → active)**

**触发场景**：3D Secure 验证完成、首次付款成功、pending 状态转为 active

```
日志确认:
  搜索: "Handling webhook customer.subscription.updated"
  → 需要配合检查事件数据中的 previous_attributes.status
  正常: 日志存在，无 Error
  异常:
    - "No member found" → 会员未关联
    - "ConflictError" → 重复事件

数据库核对:
  1. SELECT data->'$.data.object.status' as current_status,
            data->'$.data.previous_attributes.status' as previous_status
     FROM members_stripe_webhook_events
     WHERE type = 'customer.subscription.updated'
     AND data->'$.data.previous_attributes.status' = 'incomplete'
     ORDER BY created_at DESC LIMIT 1;
     → 期望: previous_status = 'incomplete', current_status = 'active'
  2. SELECT id, status, mrr, cancel_at_period_end 
     FROM members_stripe_customers_subscriptions
     WHERE stripe_subscription_id = '{subscription_id}';
     → 期望: status = 'active'
  3. SELECT id, email, status FROM members 
     WHERE id = (SELECT member_id FROM members_stripe_customers_subscriptions 
                  WHERE stripe_subscription_id = '{subscription_id}');
     → 期望: status = 'paid'
  4. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE subscription_id = (SELECT id FROM members_stripe_customers_subscriptions 
                             WHERE stripe_subscription_id = '{subscription_id}')
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'active'

前端验证 (Portal):
  ❌ 无直接反馈（需用户主动刷新页面）
  ⚠️ 注意：3D Secure 完成后，用户可能仍在 Stripe 页面
  ⚠️ 用户返回后需要刷新才能看到最新状态
  ✅ 如果已登录且刷新，member.paid = true
  ✅ 显示当前订阅价格和 tier

前端验证 (Admin):
  ✅ 收到 "New paid subscription" 邮件（SubscriptionActivatedEvent）
  ✅ 会员列表：状态从 Free → Paid（或从不完整 → Paid）
  ✅ 会员详情：显示完整的订阅信息
  ✅ 仪表盘：MRR 增加

定位动作:
  1. 确认这是 incomplete → active 的转换（查 previous_attributes）
  2. 查 Stripe Dashboard → 确认付款是否成功
  3. 查订阅表 status 是否已变为 'active'
  4. 查 members.status 是否已变为 'paid'
  5. 查 members_paid_subscription_events.type = 'active'
  6. 检查 SubscriptionActivatedEvent 是否分发
  7. 如果邮件未发送：检查 StaffService 是否正常监听
```

**事件 8: customer.subscription.updated (canceled → active - 恢复订阅)**

**触发场景**：用户在 Portal 点击 "Renew subscription"、或在期限内恢复取消的订阅

```
日志确认:
  搜索: "Handling webhook customer.subscription.updated"
  → 需要检查 previous_attributes.cancel_at_period_end 或 previous_attributes.status
  正常: 日志存在
  异常:
    - "No member found" → 会员未关联

数据库核对:
  1. SELECT data->'$.data.object.cancel_at_period_end' as current_cancel,
            data->'$.data.previous_attributes.cancel_at_period_end' as previous_cancel
     FROM members_stripe_webhook_events
     WHERE type = 'customer.subscription.updated'
     AND data->'$.data.previous_attributes.cancel_at_period_end' = true
     ORDER BY created_at DESC LIMIT 1;
     → 期望: previous_cancel = true, current_cancel = false
  2. SELECT id, status, cancel_at_period_end, current_period_end 
     FROM members_stripe_customers_subscriptions
     WHERE stripe_subscription_id = '{subscription_id}';
     → 期望: cancel_at_period_end = 0 (false)
     → 期望: status = 'active'
  3. SELECT type, from_plan, to_plan, mrr_delta 
     FROM members_paid_subscription_events
     WHERE subscription_id = (SELECT id FROM members_stripe_customers_subscriptions 
                             WHERE stripe_subscription_id = '{subscription_id}')
     ORDER BY created_at DESC LIMIT 1;
     → 期望: type = 'reactivated' 或 'active'
  4. SELECT id, email, status FROM members 
     WHERE id = (SELECT member_id FROM members_stripe_customers_subscriptions 
                  WHERE stripe_subscription_id = '{subscription_id}');
     → 期望: status = 'paid'

前端验证 (Portal):
  ✅ "Renew subscription" 按钮消失
  ✅ 恢复正常付费 UI 显示
  ✅ 不再显示到期日期警告
  ✅ "Cancel subscription" 按钮重新出现
  ✅ member.paid = true（刷新后确认）

前端验证 (Admin):
  ❌ 无邮件通知（没有专门的恢复订阅通知）
  ✅ 会员列表：状态从 Canceling → Paid
  ✅ 会员详情：不再显示 "Expires on {日期}"
  ✅ 仪表盘：预期流失率下降，MRR 保持稳定

定位动作:
  1. 确认这是恢复操作（cancel_at_period_end 从 true 变 false）
  2. 查订阅表 cancel_at_period_end 是否已变为 0
  3. 查 members_paid_subscription_events.type
  4. 注意：此事件不触发 SubscriptionActivatedEvent（因为原本就是 active）
  5. 确认恢复来源：Portal 操作 vs Stripe Dashboard 操作
  6. 检查 current_period_end 是否已延长
```

**事件 9: charge.refunded**

**触发场景**：全额退款、部分退款、争议退款

```
日志确认:
  搜索: "Handling webhook charge.refunded"
  正常: 日志存在
  异常:
    - 找不到日志 → webhook 未到达
    - "No member found" → 会员未关联

数据库核对:
  1. SELECT * FROM members_stripe_webhook_events 
     WHERE type = 'charge.refunded'
     ORDER BY created_at DESC LIMIT 1;
  2. 查 data 字段：
     - data.object.amount_refunded: 退款金额
     - data.object.amount: 原始金额
     - data.object.refunded: 是否全额退款
     - data.object.customer: Stripe customer ID
  3. SELECT id, email, status FROM members 
     WHERE id = (SELECT member_id FROM members_stripe_customers 
                  WHERE customer_id = '{customer_id}');
     ⚠️ 注意：退款事件本身不会自动更新会员状态
  4. SELECT id, status, cancel_at_period_end 
     FROM members_stripe_customers_subscriptions
     WHERE member_id = '{会员ID}';
     ⚠️ 注意：退款事件本身不会自动取消订阅
  5. SELECT * FROM members_payments 
     WHERE member_id = '{会员ID}'
     ORDER BY created_at DESC;
     ⚠️ 注意：当前版本可能不会自动记录退款

前端验证 (Portal):
  ❌ 无直接反馈（除非订阅也被取消）
  ⚠️ 如果是全额退款并取消订阅：
     → 状态变为 Free
     → 显示 "Upgrade" 按钮
  ⚠️ 如果只是部分退款：
     → 仍显示 Paid 状态
     → 无任何退款提示

前端验证 (Admin):
  ❌ 无专门的退款邮件通知
  ⚠️ 如果同时取消订阅：
     → 收到 "Paid subscription canceled" 邮件
     → 会员列表状态 = Canceled
  ⚠️ 如果只是退款：
     → 会员列表仍显示 Paid
     → 无退款标记
  ⚠️ 注意：需要手动核对 Stripe Dashboard

定位动作:
  1. ⚠️ 重要：charge.refunded 事件本身不会自动处理会员状态
  2. 查 data.object.refunded 确认是全额还是部分退款
  3. 查 data.object.amount_refunded 确认退款金额
  4. 查是否有对应的 subscription.deleted 或 subscription.updated 事件
  5. 如果是全额退款：需要手动确认订阅是否应该取消
  6. 查 Stripe Dashboard → Payments → 确认退款详情
  7. 查 members_payments 表：当前版本可能需要手动记录退款
  8. 如果应该取消订阅：检查是否有管理员手动取消操作
```

---

#### 6.1.7 快速排查命令汇总

**日志排查**：
```bash
# 查看最近 1 小时的 Stripe webhook 日志
grep -i "stripe webhook\|Handling webhook\|checkout.session\|subscription\|invoice" logs/*.log | head -50

# 查看特定事件类型
grep "Handling webhook checkout.session.completed" logs/*.log

# 查看错误日志
grep -i "error\|failed\|conflict" logs/*.log | grep -i stripe
```

**数据库排查**：
```sql
-- 1. 查看最近的 webhook 事件
SELECT type, created_at, 
       data->'$.data.object.id' as object_id,
       data->'$.data.object.customer' as customer_id
FROM members_stripe_webhook_events
WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY created_at DESC;

-- 2. 查看某会员的完整状态
SELECT 
    m.id, m.email, m.status as member_status,
    s.id as sub_id, s.status as sub_status, 
    s.cancel_at_period_end, s.mrr,
    e.type as last_event_type, e.mrr_delta
FROM members m
LEFT JOIN members_stripe_customers_subscriptions s ON m.id = s.member_id
LEFT JOIN members_paid_subscription_events e ON s.id = e.subscription_id
WHERE m.email = '{会员邮箱}'
ORDER BY e.created_at DESC
LIMIT 5;

-- 3. 查找状态不一致的会员
SELECT m.id, m.email, m.status, 
       s.status as sub_status, s.mrr
FROM members m
JOIN members_stripe_customers_subscriptions s ON m.id = s.member_id
WHERE 
    (m.status = 'paid' AND s.status NOT IN ('active', 'trialing', 'unpaid', 'past_due'))
    OR (m.status = 'free' AND s.status IN ('active', 'trialing'))
    OR (s.status = 'past_due' AND s.mrr != 0);

-- 4. 查看某会员的付款历史
SELECT * FROM members_payments 
WHERE member_id = '{会员ID}'
ORDER BY created_at DESC;

-- 5. 查看某会员的事件历史
SELECT type, from_plan, to_plan, mrr_delta, created_at
FROM members_paid_subscription_events
WHERE member_id = '{会员ID}'
ORDER BY created_at DESC;
```

**API 验证**：
```bash
# 验证 Portal sessionData API
curl -H "Cookie: ghost-members-ssr={cookie}" \
     http://localhost:2368/members/api/session/

# 验证 Admin 会员 API
curl -H "Authorization: GhostSession {token}" \
     http://localhost:2368/ghost/api/admin/members/{member_id}/
```

### 6.2 乱序事件场景分析

Stripe webhook 不保证按顺序到达，以下是常见乱序场景及用户可见影响：

#### 6.2.1 场景 1: subscription.updated 先于 checkout.session.completed

**时间轴**：
```
T0: 用户完成 Stripe checkout
T1: subscription.updated (status=active) 到达 → 但会员还未创建
T2: checkout.session.completed 到达 → 创建会员 + 关联订阅
```

**可能偏差**：
- T1 时：`memberRepository.getByCustomerId()` 返回 `null` → 事件被忽略
- T2 时：checkout 处理流程正常执行 → 最终状态正确
- **用户感知**：无偏差，checkout 事件通常先到达

**风险等级**：低 (checkout 事件通常先到达)

#### 6.2.2 场景 2: subscription.created 与 subscription.updated 并发

**时间轴**：
```
T0: 两个事件并发到达
    Thread A: subscription.created → 开始 linkSubscription
    Thread B: subscription.updated → 开始 linkSubscription
```

**可能偏差**：
- 事务 + 行级锁 (`forUpdate: true`) → 其中一个等待
- 先完成的执行 insert，后完成的执行 update
- **用户感知**：无偏差，事务保证原子性

**风险等级**：极低 (数据库锁保护)

#### 6.2.3 场景 3: subscription.deleted 先于 canceled 状态更新

**时间轴**：
```
T0: 用户立即取消订阅 (退款)
T1: subscription.updated (status=canceled) 到达
T2: subscription.deleted 到达 (有时差)
```

**可能偏差**：
- T1: 状态变为 `canceled` (cancel_at_period_end=true)
- T2: 状态变为 `expired` (cancelNow=true)
- **用户感知**：可能短暂看到"已取消但在期限内"，然后变为"已过期"
- **Admin 通知**：可能收到两封取消邮件？

**代码检查**：
```javascript
// member-repository.js:1304
if (this.isActiveSubscriptionStatus(originalStatus) && 
    (updatedStatus === 'canceled' || updatedStatus === 'expired')) {
    // 发送取消通知
}
```

- 如果 originalStatus 在 T2 时已不是 active (T1 已改为 canceled)
- **则 T2 不会再次发送通知**

**风险等级**：低 (状态检查过滤重复通知)

#### 6.2.4 场景 4: 3D Secure 场景 - updated 与 checkout 乱序

**时间轴**：
```
T0: 用户开始 checkout → 跳转 Stripe
T1: Stripe 创建 subscription (status=incomplete)
T2: subscription.created 到达 → 记录但 status 非 active
T3: 用户完成 3D Secure
T4: checkout.session.completed 到达
T5: subscription.updated (incomplete→active) 到达
```

**可能偏差**：
- **正常顺序** T4 → T5：
  - T4: checkout 处理，subscription 已 active → 发送激活通知
  - T5: updated 处理，originalStatus=active → 不重复发送

- **乱序 T5 → T4**：
  - T5: subscription.updated，会员还不存在 → 被忽略
  - T4: checkout 处理，subscription 已 active → 发送激活通知
- **用户感知**：无偏差

**风险等级**：低 (checkout 事件兜底)

### 6.3 重复事件场景分析

Stripe 保证至少一次 (at-least-once) 投递，事件可能重复。

#### 6.3.1 重复 checkout.session.completed

**保护机制**：
```javascript
// member-repository.js:925-960 - createSubscription
// 对 StripeCustomer 执行 upsert
await this.upsertCustomer({
    customer_id: customer.id,
    member_id: member.id,
    ...
});

// 对 subscription 也会有唯一约束保护
// subscription-event-service.js:40-51
try {
    await memberRepository.linkSubscription({...});
} catch (err) {
    if (err.code !== 'ER_DUP_ENTRY' && err.code !== 'SQLITE_CONSTRAINT') {
        throw err;
    }
    throw new errors.ConflictError({err});  // Webhook 返回 409，但 Stripe 忽略
}
```

**可能偏差**：
- 数据库层：无重复数据 (唯一约束 + upsert)
- 通知层：可能重复发送邮件！
  - `SubscriptionCreatedEvent` / `SubscriptionActivatedEvent` 可能被多次分发
  - StaffService 收到多次 → 多封通知邮件
  - 欢迎邮件可能重复入队

**用户感知**：
- 员工收到多封通知邮件
- 会员可能收到多封欢迎邮件/登录邮件

**风险等级**：中 (邮件可能重复)

**Offer 赎回保护** (有专门去重)：
```javascript
// member-repository.js:113-133
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

#### 6.3.2 重复 subscription.updated

**保护机制**：
```javascript
// member-repository.js:1250-1281
// 只有当 MRR、plan_id、status、cancel_at_period_end 变化时才记录事件
if (stripeCustomerSubscriptionModel.get('mrr') !== updated... ||
    stripeCustomerSubscriptionModel.get('plan_id') !== updated... ||
    stripeCustomerSubscriptionModel.get('status') !== updated... ||
    stripeCustomerSubscriptionModel.get('cancel_at_period_end') !== updated...) {
    
    // 记录 MemberPaidSubscriptionEvent
    // 分发 DomainEvents
}
```

**可能偏差**：
- 第一次处理：状态已更新，事件已分发
- 第二次处理：状态对比发现无变化 → 跳过
- **通知层**：可能在第二次处理时不重复发送，但第一次可能重复？

**风险等级**：低 (状态对比过滤)

#### 6.3.3 重复 invoice.payment_succeeded

**保护机制**：
```javascript
// invoice-event-service.js:27-69
// 简单的 insert，无去重
await eventRepository.registerPayment({
    member_id: member.id,
    currency: invoice.currency,
    amount: invoice.amount_paid
});
```

**可能偏差**：
- 同一发票可能记录多条 payment 记录
- MRR 统计不受影响 (不重复计算)
- 但 `members_payments` 表有重复数据

**风险等级**：低 (数据冗余但功能正常)

### 6.4 延迟场景的用户可见状态偏差

当 Stripe webhook 因网络问题延迟时，前端可能看到不一致状态。

#### 6.4.1 新订阅延迟

**用户操作时间轴**：
```
T0: 用户点击 "Subscribe" → 跳转 Stripe
T1: 用户完成支付 → Stripe 返回 successUrl
T2: 用户回到 Ghost → 看到 "Success!" 通知
T3: 用户打开 Portal
    ⚠️ Webhook 延迟到达
    → 后端 member.status 仍是 'free'
    → Portal 显示免费账户 UI
T4: 用户关闭通知 → refreshMemberData()
    → 后端可能仍未处理 → 还是 free
T5: 5 分钟后 Webhook 到达 → member.status = 'paid'
T6: 用户刷新页面 → 正常显示 paid
```

**用户感知偏差**：
| 时间点 | 预期状态 | 实际显示 | 用户操作 |
|-------|---------|---------|---------|
| T2-T3 | paid | free 或部分 paid | 困惑：支付成功但仍显示免费 |
| T4 | paid | free | 刷新无效 |
| T6 | paid | paid | 恢复正常 |

**可能的用户投诉**：
> "我已经付费了，但账户还是免费的！"

**兜底机制**：
- Stripe 会重试 webhook (最多几天)
- 用户刷新页面最终会看到正确状态
- 但用户体验差

#### 6.4.2 取消订阅延迟

**用户操作时间轴**：
```
T0: 用户在 Portal 点击 "Cancel subscription"
T1: API 调用 Stripe 取消 → 立即返回成功
T2: 本地调用 sessionData() → 状态已更新
T3: 弹窗关闭 → reloadOnPopupClose → 页面刷新
    ⚠️ Stripe webhook 延迟
T4: 用户看到已取消状态 (本地状态已更新)
T5: 几分钟后 Webhook 到达 → 确认更新
```

**关键差异**：
- **用户发起的操作**：本地先更新，再等 webhook
- **Stripe 发起的状态变更**：完全依赖 webhook

**用户感知**：无偏差 (本地状态优先)

#### 6.4.3 付款失败 (past_due)

**场景**：
```
T0: 自动扣款失败 → subscription.status = past_due
T1: Stripe 发送 webhook → Ghost 收到，更新状态
T2: 用户访问 Portal
    → isActiveSubscriptionStatus() 仍认为 active
    → 显示正常付费 UI
    → 但 MRR = 0
```

**状态判断逻辑** (`member-repository.js:164-166`)：
```javascript
isActiveSubscriptionStatus(status) {
    return ['active', 'trialing', 'unpaid', 'past_due'].includes(status);
}
```

**用户感知偏差**：
- Portal 仍显示"付费会员"
- 但实际上付款已失败
- 用户可能不知道需要更新支付方式

**风险等级**：中 (用户无感知)

### 6.5 排查观测点指南

当出现状态不一致时，按以下顺序排查：

#### 6.5.1 第一级：Webhook 接收状态

**检查点**：
1. **Stripe Dashboard** → Developers → Webhooks
   - 查看 webhook endpoint 状态
   - 查看失败重试列表
   - 检查 event 时间戳

2. **Ghost 日志** (应用日志)
   ```
   搜索: "stripe webhook" 或 "WebhookController"
   
   正常日志:
   - "Handling Stripe webhook event: checkout.session.completed"
   
   异常日志:
   - "Failed to parse Stripe webhook" (签名错误)
   - "ConflictError" (重复事件)
   - "No member found for Stripe customer" (乱序事件被忽略)
   ```

3. **数据库: members_stripe_webhook_events**
   - 记录所有收到的 webhook 事件
   - 可与 Stripe Dashboard 对比
   - 检查 `created_at` 时间顺序

#### 6.5.2 第二级：状态同步状态

**检查点**：
1. **Stripe API 直连确认**
   ```javascript
   // Stripe Dashboard 或 CLI
   stripe subscriptions retrieve sub_xxx
   → 确认 Stripe 端真实状态
   ```

2. **Ghost 数据库对比**
   ```sql
   -- 对比会员状态
   SELECT id, email, status 
   FROM members 
   WHERE id = 'member_xxx';
   
   -- 对比订阅状态
   SELECT id, status, cancel_at_period_end, current_period_end
   FROM members_stripe_customers_subscriptions
   WHERE member_id = 'member_xxx';
   ```

3. **事件记录表**
   ```sql
   -- 查看订阅事件历史
   SELECT type, from_plan, to_plan, mrr_delta, created_at
   FROM members_paid_subscription_events
   WHERE member_id = 'member_xxx'
   ORDER BY created_at;
   ```

4. **Domain Events 分发**
   ```
   搜索日志: "DomainEvents.dispatch" 或事件名
   
   应看到:
   - "SubscriptionActivatedEvent dispatched"
   - "StaffService handled SubscriptionActivatedEvent"
   ```

#### 6.5.3 第三级：前端状态同步

**检查点**：
1. **Portal sessionData API**
   ```
   浏览器 DevTools → Network
   找到: /members/api/session/
   检查 Response: { member: { paid: true/false, status: "paid", subscriptions: [...] } }
   ```

2. **URL 参数**
   ```
   检查地址栏: ?stripe=success, ?stripe=cancel 等
   验证: NotificationParser 是否正确解析
   ```

3. **React Query 缓存 (Admin)**
   ```javascript
   // DevTools → React Query 面板
   检查 Query Key: ["MembersResponseType", "/members/"]
   验证: 数据是否与数据库一致
   检查: invalidateQueries 是否已触发
   ```

#### 6.5.4 第四级：通知状态

**检查点**：
1. **邮件队列**
   ```sql
   -- 检查邮件是否入队
   SELECT * FROM emails 
   WHERE member_id = 'member_xxx' 
   ORDER BY created_at DESC;
   ```

2. **欢迎邮件自动化**
   ```sql
   -- 检查自动化运行
   SELECT * FROM members_welcome_email_automation_runs
   WHERE member_id = 'member_xxx';
   ```

3. **Mailpit (开发环境)**
   ```
   http://localhost:8025
   搜索: 会员邮箱地址
   ```

#### 6.5.5 快速排查清单

| 症状 | 优先排查点 |
|-----|-----------|
| 付费成功但 Portal 仍显示免费 | 1. Stripe Webhook 状态<br>2. `members_stripe_webhook_events` 表<br>3. `members.status` 字段 |
| 收到多封通知邮件 | 1. 重复事件日志<br>2. `members_paid_subscription_events` 重复记录<br>3. `emails` 表重复条目 |
| Admin 列表状态与详情不一致 | 1. React Query 缓存<br>2. Ember Bridge 事件同步<br>3. `emberDataChange` 事件触发 |
| 取消后仍可访问付费内容 | 1. `members_stripe_customers_subscriptions.status`<br>2. `cancel_at_period_end` 字段<br>3. 主题层权限判断逻辑 |
| 3D Secure 后状态未更新 | 1. `checkout.session.completed` 时间<br>2. `subscription.updated` 时间<br>3. 哪个先到达 |

#### 6.5.6 关键 SQL 查询

```sql
-- 1. 查看某会员的完整订阅历史
SELECT 
    m.id, m.email, m.status,
    s.id as subscription_id, s.status as sub_status, s.cancel_at_period_end,
    e.type, e.from_plan, e.to_plan, e.mrr_delta, e.created_at
FROM members m
LEFT JOIN members_stripe_customers_subscriptions s ON m.id = s.member_id
LEFT JOIN members_paid_subscription_events e ON s.id = e.subscription_id
WHERE m.email = 'user@example.com'
ORDER BY e.created_at DESC;

-- 2. 查看最近的 webhook 事件
SELECT 
    type, 
    created_at, 
    data->'$.id' as event_id,
    data->'$.data.object.id' as object_id
FROM members_stripe_webhook_events
WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY created_at DESC;

-- 3. 查找状态不一致的会员
SELECT m.id, m.email, m.status, s.status as subscription_status
FROM members m
JOIN members_stripe_customers_subscriptions s ON m.id = s.member_id
WHERE 
    (m.status = 'paid' AND s.status NOT IN ('active', 'trialing', 'unpaid', 'past_due'))
    OR (m.status = 'free' AND s.status IN ('active', 'trialing'));

-- 4. 查找重复的支付记录
SELECT member_id, invoice_id, COUNT(*) as cnt
FROM members_payments
GROUP BY member_id, invoice_id
HAVING cnt > 1;
```

## 7. 前端反馈协作与状态同步

Stripe webhook 处理完成后，前端（Portal 和 Admin）需要感知会员状态变化并更新 UI。本节分析状态如何从后端同步到前端界面。

### 6.1 Portal 前端状态同步

#### 6.1.1 数据获取时机

Portal 是一个嵌入在主题页面中的 React 应用，有三个核心时机获取会员状态：

**1. 初始化时** (`app.js:264-323` - `initSetup()`)：
```javascript
async initSetup() {
    // 初始化时从 API 拉取最新数据
    const {site, member, offers} = await this.fetchData();
    // ...
}
```

`fetchApiData()` (`api.js:856-895`) 并行请求：
- `api.member.sessionData()` → 获取当前登录会员详情
- `api.site.settings()` → 获取站点设置
- `api.site.tiers()` → 获取可用 tier/价格
- `api.site.newsletters()` → 获取通讯列表
- 付费会员额外调用 `api.member.offers()` → 获取可用优惠

**2. 操作后主动刷新** (`actions.js` 多个位置)：

| 操作 | 刷新时机 | 代码位置 |
|-----|---------|---------|
| 更新订阅 | `updateSubscription:success` | `actions.js:372` |
| 取消订阅 | `cancelSubscription:success` | `actions.js:400` |
| 恢复订阅 | `continueSubscription:success` | `actions.js:425` |
| 应用优惠 | `applyOffer:success` | `actions.js:451` |
| 礼品兑换 | `redeemGift:success` | `actions.js:227` |
| 通知关闭后 | URL 来源通知关闭时 | `notification.js:297` |

**刷新模式**：
```javascript
// 模式 1: 操作成功后立即调用 sessionData()
const member = await api.member.sessionData();
return { member, ... };  // 直接更新状态

// 模式 2: 通过 action 触发
this.context.doAction('refreshMemberData');

// refreshMemberData 实现 (actions.js:648-669)
async function refreshMemberData({state, api}) {
    if (state.member) {
        const member = await api.member.sessionData();
        return { member, success: true };
    }
    return null;
}
```

**3. 页面刷新时**：
- Stripe checkout 完成后通过 `successUrl` 带参数返回
- 或 `reloadOnPopupClose: true` 触发 `window.location.reload()`

#### 6.1.2 会员状态判断

Portal 通过 `helpers.js` 中的工具函数判断会员状态：

**付费会员判断** (`helpers.js:70-72`)：
```javascript
export function isPaidMember({member = {}}) {
    return (member && member.paid);  // 简单判断 member.paid 布尔值
}
```

**活跃订阅查找** (`helpers.js:42-54`)：
```javascript
export function getMemberSubscription({member = {}}) {
    if (isPaidMember({member})) {
        const subscriptions = member.subscriptions || [];
        const activeSubscription = subscriptions.find((sub) => {
            return ['active', 'trialing', 'unpaid', 'past_due'].includes(sub.status);
        });
        return activeSubscription;
    }
    return null;
}
```

**赠送/礼品会员判断**：
```javascript
// Complimentary (amount === 0)
export function isComplimentaryMember({member = {}}) {
    const subscription = getMemberSubscription({member});
    if (subscription) {
        const {price} = subscription;
        return (price && price.amount === 0);  // 价格为 0 即为赠送
    } else if (!subscription && !!member.paid) {
        return true;  // paid 但无订阅信息也视为 comped
    }
    return false;
}

// Gift membership
export function isGiftMember({member = {}}) {
    return member?.status === 'gift';  // 直接判断 status 字段
}
```

#### 6.1.3 状态影响的 UI 变化

| 状态 | Portal 行为 | 影响页面 |
|-----|------------|---------|
| `free` (未登录) | 显示 signup/signin 页面 | 默认页面 |
| `free` (已登录) | 显示 Account Home，可升级 | `accountHome`, `accountPlan` |
| `paid` | 显示 Account Home，显示当前 tier | 所有 account 页面 |
| `comped` | 同 paid，但隐藏价格等 UI | 所有 account 页面 |
| `gift` | 显示"Continue gift subscription"按钮 | `accountHome` |

**关键状态依赖**：
- **accountHome 欢迎信息** (`account-welcome.js`)：
  - `isPaidMember()` → 显示付费欢迎 vs 免费欢迎
  - `isComplimentaryMember()` → 调整文案（无价格展示）
  
- **accountHome 操作按钮** (`paid-account-actions.js`)：
  - 活跃订阅 → 显示"Manage subscription"、"Cancel subscription"
  - `cancel_at_period_end === true` → 显示"Renew subscription"
  - 礼品会员 → 显示"Continue gift subscription"（调用 `continueGiftCheckout`）

- **accountPlan 页面** (`account-plan-page.js`)：
  - 显示当前价格信息
  - 显示"Upgrade" / "Switch plan" 选项
  - 显示订阅到期时间（`getSubscriptionExpiry`）

### 6.2 Admin 前端状态同步

Admin 是 Ember.js + React 的混合应用，通过 **Ember Bridge** 实现状态同步。

#### 6.2.1 React Query 缓存架构

Admin 使用 `@tanstack/react-query` 进行数据管理，核心 API 定义于 `admin-x-framework/src/api/members.ts`：

**会员列表查询** (`members.ts:107-110`)：
```javascript
export const useBrowseMembers = createQuery<MembersResponseType>({
    dataType,
    path: '/members/'
});
```

**会员详情查询** (`members.ts:126-129`)：
```javascript
export const getMember = createQueryWithId<MembersResponseType>({
    dataType,
    path: id => `/members/${id}/`
});
```

**数据类型映射** (`ember-bridge.tsx:68-84`)：
```javascript
const EMBER_TO_REACT_TYPE_MAPPING: Record<string, string> = {
    'member': 'MembersResponseType',
    'tier': 'TiersResponseType',
    // ...
};
```

#### 6.2.2 Ember → React 状态同步

**useEmberDataSync Hook** (`ember-bridge.tsx:145-170`)：

```javascript
export function useEmberDataSync() {
    const queryClient = useQueryClient();

    useEffect(() => {
        const handleEmberDataChange = (event: EmberDataChangeEvent) => {
            const { modelName, operation } = event;  // update/create/delete
            const reactDataType = EMBER_TO_REACT_TYPE_MAPPING[modelName];

            if (reactDataType) {
                // 使 React Query 缓存失效，触发重新获取
                void queryClient.invalidateQueries({
                    predicate: (query) => {
                        return query.queryKey[0] === reactDataType;
                    }
                });
            }
        };

        return onEmberStateBridgeEvent('emberDataChange', handleEmberDataChange);
    }, [queryClient]);
}
```

**同步机制**：
```
Ember Data Store 修改
       ↓
window.EmberBridge.state.onUpdate('ember', {modelName, id, data})
       ↓
onEmberStateBridgeEvent('emberDataChange', handler) 被触发
       ↓
queryClient.invalidateQueries({predicate})
       ↓
React Query 重新获取数据 → UI 更新
```

#### 6.2.3 站点订阅状态同步

**useSubscriptionStatus Hook** (`ember-bridge.tsx:197-209`)：

用于跟踪 Ghost 自身的订阅状态（如试用、升级提示）：

```javascript
export function useSubscriptionStatus() {
    const [subscriptionStatus, setSubscriptionStatus] = useState(null);

    useEffect(() => {
        const handleSubscriptionChange = (payload) => {
            // payload = {subscription: {isActiveTrial, trial_end, status}}
            setSubscriptionStatus(payload);
        };

        return onEmberStateBridgeEvent('subscriptionChange', handleSubscriptionChange);
    }, []);

    return subscriptionStatus;
}
```

**应用场景** (`use-upgrade-status.ts`)：
```javascript
export function useUpgradeStatus() {
    const subscriptionStatus = useSubscriptionStatus();
    const showUpgradeBanner = !!subscriptionStatus?.subscription?.isActiveTrial;
    const trialDaysRemaining = subscriptionStatus?.subscription?.trial_end 
        ? Math.ceil((new Date(subscriptionStatus.subscription.trial_end).getTime() - Date.now()) / (1000 * 60 * 60 * 24)) 
        : 0;
    // ...
}
```

### 6.3 Stripe Checkout 回调与状态同步

#### 6.3.1 成功回调 URL 参数

Stripe checkout 完成后，Portal 通过 `successUrl` 返回，携带状态参数：

**URL 参数定义** (`api.js` 多个位置)：

| 操作 | successUrl 参数 | 含义 |
|-----|-----------------|------|
| 普通订阅 | `?stripe=success` | 订阅成功 |
| 取消订阅 | `?stripe=cancel` | 用户取消了 checkout |
| 礼品购买 | `?stripe=gift-purchase-success` | 礼品购买成功 |
| 更新账单 | `?stripe=billing-update-success` | 账单信息更新成功 |
| 取消账单更新 | `?stripe=billing-update-cancel` | 用户取消了账单更新 |
| 关闭账单门户 | `?stripe=billing-portal-closed` | Stripe billing portal 关闭 |

#### 6.3.2 URL 通知解析

`NotificationParser` (`notifications.js:84-112`) 解析 URL 参数并生成通知配置：

```javascript
export default function NotificationParser({billingOnly = false} = {}) {
    const searchParams = new URLSearchParams(window.location.search || '');
    const stripeStatus = searchParams.get('stripe');
    
    if (stripeStatus) {
        return handleStripeActions({status: stripeStatus, billingOnly});
    }
    // ...
}
```

**Stripe 通知处理** (`notifications.js:42-63`)：
```javascript
export const handleStripeActions = ({status, billingOnly}) => {
    // 普通 checkout 结果
    if (!billingOnly && ['success'].includes(status)) {
        return {
            type: 'stripe:checkout',
            status: 'success',
            duration: 3000,
            autoHide: true
        };
    }
    
    // 账单门户操作结果
    if (billingOnly && ['billing-portal-closed', 'billing-update-success', 'billing-update-cancel'].includes(status)) {
        const statusVal = status === 'billing-update-cancel' ? 'warning' : 'success';
        return {
            type: 'stripe:billing-update',
            status: statusVal,
            duration: 3000,
            autoHide: true,
            closeable: true
        };
    }
};
```

#### 6.3.3 通知关闭后的状态刷新

**Notification 组件关闭逻辑** (`notification.js:283-306`)：

```javascript
onHideNotification() {
    const {type, source} = this.state;

    if (source === 'url') {
        // 通知来自 URL 参数（如 Stripe checkout 回调）
        const deleteParams = [];
        if (['stripe:checkout'].includes(type)) {
            deleteParams.push('stripe');
        }
        
        // 1. 清除 URL 参数（避免刷新后重复显示）
        clearURLParams(deleteParams);
        
        // 2. 刷新会员数据（确保状态同步）
        this.context.doAction('refreshMemberData');
        
    } else if (source === 'state') {
        // 通知来自应用内状态
        this.context.doAction('closeNotification');
    }

    this.setState({ active: false, source: null });
}
```

**关键协作点**：
- 通知关闭时主动调用 `refreshMemberData` → 调用 `sessionData()` API
- URL 参数被清除 → `window.history.replaceState`
- 这确保了即使 webhook 有延迟，前端也会**主动拉取**最新状态

### 6.4 延迟与失败场景的用户体验

#### 6.4.1 延迟场景的处理

**场景 1: Stripe checkout 完成，但 webhook 延迟到达**

```
用户操作                              系统行为
─────────                            ─────────
1. 点击 "Subscribe"                  → 重定向到 Stripe
2. 完成支付                          → Stripe 发送 successUrl (?stripe=success)
3. 返回 Ghost 页面
   a. URL 参数触发 "Success!" 通知
   b. member.paid 可能仍为 false（webhook 未到达）
   c. 用户关闭通知 → refreshMemberData() 被调用
   d. 如果后端已处理 → 状态更新
   e. 如果后端未处理 → 仍显示为 free
4. 用户手动刷新页面
   → initSetup() 重新获取
   → 若 webhook 已处理 → 状态正确
   → 若仍未处理 → 需等待或联系支持
```

**场景 2: Portal 内取消/恢复订阅**

```
1. 用户点击 "Cancel subscription"
2. 调用 PUT /subscriptions/{id}/
3. API 返回成功
4. 立即调用 sessionData() 获取最新状态
5. reloadOnPopupClose = true → 弹窗关闭时刷新页面
```

**核心机制**：
- **操作后立即刷新**：`sessionData()` 确保本地状态与后端一致
- **弹窗关闭时刷新**：`window.location.reload()` 作为最终兜底
- **双重保障**：API 返回 + 页面刷新

#### 6.4.2 失败场景的用户提示

**Portal 操作失败提示** (`actions.js` 统一模式)：

```javascript
// 模式：所有异步操作都有 :failed 分支
async function cancelSubscription({data, state, api}) {
    try {
        await api.member.updateSubscription({...});
        const member = await api.member.sessionData();
        return {
            action: 'cancelSubscription:success',
            member,
            reloadOnPopupClose: true
        };
    } catch (e) {
        return {
            action: 'cancelSubscription:failed',
            popupNotification: createPopupNotification({
                type: 'cancelSubscription:failed',
                autoHide: false,        // 不自动隐藏
                closeable: true,         // 用户可关闭
                status: 'error',
                message: t('Failed to cancel subscription, please try again')
            })
        };
    }
}
```

**失败消息设计**：
| 属性 | 值 | 含义 |
|-----|---|------|
| `autoHide` | `false` | 不自动消失，确保用户看到 |
| `closeable` | `true` | 用户可手动关闭 |
| `status` | `'error'` | 红色警告图标 |
| `message` | 国际化文案 | 清晰的错误描述 |

**通知组件展示** (`notification.js`)：
```
┌──────────────────────────────────────────────┐
│ ⚠️  Failed to cancel subscription,            │
│     please try again                    [×]   │
└──────────────────────────────────────────────┘
  红底白字图标 + 错误信息 + 关闭按钮
```

#### 6.4.3 兜底策略

**策略 1: 页面刷新兜底**

`reloadOnPopupClose` 标记（`app.js:116-118`）：
```javascript
if (!this.state.showPopup) {
    // 弹窗关闭时
    if (this.state.reloadOnPopupClose) {
        window.location.reload();  // 强制刷新
    }
}
```

**适用操作**：
- `cancelSubscription:success`
- `continueSubscription:success`
- `applyOffer:success`

**策略 2: API 错误时的通用错误处理**

`dispatchAction` 兜底（`app.js:902-922`）：
```javascript
} catch (error) {
    if (data && data.throwErrors) {
        throw error;
    }

    const popupNotification = createPopupNotification({
        type: `${action}:failed`,
        autoHide: true, 
        closeable: true, 
        status: 'error', 
        state: this.state,
        meta: { error }
    });
    
    this.setState({
        action: `${action}:failed`,
        actionErrorMessage: chooseBestErrorMessage(
            error, 
            t('An unexpected error occured. Please try again or <a>contact support</a> if the error persists.')
        ),
        popupNotification
    });
}
```

**策略 3: Stripe checkout 取消的友好提示**

用户在 Stripe 页面点击取消时（`notification.js:133-146`）：
```javascript
} else if (type === 'stripe:checkout' && status === 'warning') {
    // Stripe checkout flow was cancelled
    if (context.member) {
        return (
            <p>{t('Plan upgrade was cancelled.')}</p>
        );
    }
    return (
        <p>{t('Plan checkout was cancelled.')}</p>
    );
}
```

这是一个 **warning** 状态（非 error），用户主动取消不视为失败。

#### 6.4.4 成员会话失效处理

`sessionData()` API 返回 `null` 或 204：
- `api.js:246-247`: `if (!res.ok || res.status === 204) return null;`
- Portal 会将 `member` 设为 `null`，UI 降级到未登录状态
- 用户需要重新通过 Magic Link 登录

### 6.5 端到端协作流程图

```
┌─────────────┐
│   前端      │  (Portal / Admin)
└──────┬──────┘
       │
       │ 1. 用户操作 (点击订阅/取消/升级)
       │
       ▼
┌───────────────────────────┐
│  checkoutPlan /           │
│  updateSubscription       │  → 调用 Stripe API 或 Ghost API
└───────────────┬───────────┘
                │
                ├────────── Stripe Checkout 路径 ──────────┐
                │                                         │
                │ 2. 重定向到 Stripe                      │
                │    (window.location.assign)             │
                ▼                                         ▼
        ┌─────────────┐                          ┌──────────────────┐
        │  Stripe UI  │  用户完成支付            │  Stripe 失败/取消 │
        └──────┬──────┘                          └────────┬─────────┘
               │                                          │
               │ 3. webhook 异步发送                      │
               │    (checkout.session.completed)          │
               ▼                                          │
        ┌─────────────┐                                    │
        │  Webhook    │  同步会员状态                      │
        │  Controller │                                    │
        └──────┬──────┘                                    │
               │                                          │
               │ 4. Stripe successUrl 回调                │
               │    (?stripe=success / ?stripe=cancel)    │
               ▼                                          ▼
        ┌─────────────────────────────────────────────────┐
        │              Portal 返回 Ghost 页面              │
        ├─────────────────────────────────────────────────┤
        │  a. NotificationParser 解析 URL 参数            │
        │  b. 显示通知 (Success! / Plan cancelled)        │
        │  c. 用户关闭通知                                 │
        │     → clearURLParams()                          │
        │     → doAction('refreshMemberData')             │
        │     → sessionData() 拉取最新状态                │
        └───────────────────────────┬─────────────────────┘
                                    │
                                    │ 5. 弹窗关闭时
                                    │    (reloadOnPopupClose = true)
                                    ▼
                            ┌──────────────┐
                            │ window.      │  最终兜底
                            │ location.    │
                            │ reload()     │
                            └──────────────┘
```

### 6.6 状态延迟感知的关键时间点

```
时间轴 ──────────────────────────────────────────────────────────▶

T0  用户点击 "Subscribe"
    ├── Portal 调用 create-stripe-checkout-session API
    └── 重定向到 Stripe checkout 页面

T1  用户完成支付（Stripe 端）
    ├── Stripe 发送 checkout.session.completed webhook
    │   └── (异步，通常 < 2s，但可能延迟)
    └── Stripe 重定向到 successUrl

T2  用户返回 Ghost 页面（successUrl 回调）
    ├── Portal 初始化 / URL 解析
    ├── 显示 "Success! Your account is fully activated..."
    │
    ├── ⚠️ 关键：此时 member.paid 可能仍为 false！
    │   (如果 webhook 还未处理完成)
    │
    └── 用户关闭通知
        ├── refreshMemberData() 被调用
        ├── sessionData() API 请求
        │   └── 后端返回最新状态（如果 webhook 已处理）
        └── URL 参数被清除

T3  Webhook 最终处理完成（如 T2 时还未完成）
    ├── linkSubscription() 执行
    ├── member.status = 'paid'
    ├── DomainEvents 分发
    └── 邮件通知发送

T4  用户主动刷新 / 重新打开 Portal
    ├── initSetup() → fetchApiData()
    ├── sessionData() 获取最新状态
    └── UI 正确显示 paid
```

**关键洞察**：
- **无实时推送**：Portal 和 Admin 都没有 WebSocket/SSE 实时推送机制
- **依赖主动拉取**：状态同步完全依赖 `sessionData()` API 的主动调用
- **通知触发刷新**：URL 通知关闭是关键的刷新触发点
- **页面刷新兜底**：`reloadOnPopupClose` 确保复杂操作后状态一致

## 7. 数据流协作图

### 7.1 完整订阅流程

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

### 8.2 状态变更时序

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

## 9. 总结

### 9.1 后端同步特性

Ghost 的 Stripe 订阅同步系统具有以下特点：

1. **高可靠性**：多层幂等保护（签名验证、唯一约束、upsert、行级锁）
2. **事件驱动**：通过 Domain Events 解耦订阅状态变更与通知处理
3. **状态一致**：事务提交后才分发事件，确保数据与通知一致
4. **乱序兼容**：Stripe 事件可能乱序到达，upsert + 状态比较确保最终一致性
5. **智能通知**：区分欢迎邮件、登录邮件、员工通知，避免重复打扰

### 9.2 前端反馈协作特性

前端状态同步采用**主动拉取 + 多层兜底**策略：

1. **无实时推送**：Portal 和 Admin 都没有 WebSocket/SSE，完全依赖主动拉取
2. **三层刷新机制**：
   - 操作成功后立即调用 `sessionData()`
   - URL 通知关闭时触发 `refreshMemberData`
   - 复杂操作后 `reloadOnPopupClose` 强制页面刷新
3. **统一错误处理**：所有异步操作都有 `:failed` 分支，错误提示不自动消失
4. **用户友好的失败状态**：
   - `autoHide: false` 确保用户看到错误
   - `closeable: true` 允许用户手动关闭
   - warning 状态区分主动取消与真实失败
5. **Admin 数据同步**：通过 Ember Bridge + React Query 缓存失效实现 Ember→React 状态同步

### 9.3 延迟场景的最终一致性

当 Stripe webhook 延迟时，系统确保最终一致性：

```
用户视角（可能的体验）：
1. 支付完成 → 看到 "Success!" 通知
2. 打开 Portal → 可能还显示免费状态（webhook 未到）
3. 关闭通知 → refreshMemberData() 主动拉取
4. 若已同步 → 状态更新；若未同步 → 仍显示免费
5. 用户刷新页面 → initSetup() 再次获取
```

这个设计确保了即使在网络不稳定、Stripe 重复推送、或并发处理场景下，会员状态依然能够正确同步。
