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

## 6. 前端反馈协作与状态同步

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
