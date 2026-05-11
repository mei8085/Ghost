# Ghost 定价模型与外部支付资源同步机制分析

## 概述

本文档分析 Ghost 博客平台的后台定价模型如何与 Stripe 支付系统进行双向同步。重点关注以下三个核心问题：

1. **Tier 与 Offer 的本地与 Stripe 双向映射**
2. **价格变更时已有订阅的处理机制**
3. **折扣码的生效与过期清理机制**

---

## 一、数据模型架构

### 1.1 核心数据表关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            本地数据模型                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  products (Tier)                  stripe_products                           │
│  ┌──────────────┐                ┌────────────────────┐                    │
│  │ id (PK)      │◄───────────────┤ id (PK)            │                    │
│  │ name         │  product_id    │ product_id (FK)    │                    │
│  │ type (paid/free) │            │ stripe_product_id  │                    │
│  │ currency     │                └────────────────────┘                    │
│  │ monthly_price│                         │                                │
│  │ yearly_price │                         │                                │
│  └──────────────┘                         ▼                                │
│                              ┌────────────────────┐                         │
│                              │ stripe_prices      │                         │
│                              │ id (PK)            │                         │
│                              │ stripe_price_id    │                         │
│                              │ stripe_product_id  │◄─────────┐              │
│                              │ active             │          │              │
│                              │ currency           │          │              │
│                              │ amount             │          │              │
│                              │ type               │          │              │
│                              │ interval (month/year)│       │              │
│                              └────────────────────┘          │              │
│                                             │                 │              │
│                                             │                 │              │
│  offers                         ┌────────────▼─────────────┐  │              │
│  ┌──────────────────┐          │ members_stripe_customers_ │  │              │
│  │ id (PK)          │          │ subscriptions            │  │              │
│  │ name             │          │ id (PK)                  │  │              │
│  │ code             │          │ subscription_id          │  │              │
│  │ product_id (FK)  │◄─────────┤ offer_id (FK)            │  │              │
│  │ stripe_coupon_id │          │ stripe_price_id          │──┘              │
│  │ discount_type    │          │ status                   │                 │
│  │ discount_amount  │          │ current_period_end       │                 │
│  │ duration         │          │ discount_start           │                 │
│  │ redemption_type  │          │ discount_end             │                 │
│  └──────────────────┘          └─────────────────────────┘                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据表字段

#### products (Tier) 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 本地产品 ID（Tier ID） |
| name | string | 套餐名称 |
| type | string | 类型：paid / free |
| currency | string | 货币代码（3 位 ISO） |
| monthly_price | integer | 月价格（分） |
| yearly_price | integer | 年价格（分） |
| active | boolean | 是否激活 |

参考: `ghost/core/core/server/data/schema/schema.js:449`

#### stripe_products 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 本地 ID |
| product_id | string | 关联 products.id |
| stripe_product_id | string | Stripe 产品 ID |

参考: `ghost/core/core/server/data/schema/schema.js:794`

#### stripe_prices 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 本地 ID |
| stripe_price_id | string | Stripe 价格 ID |
| stripe_product_id | string | 关联 stripe_products.stripe_product_id |
| active | boolean | 是否激活 |
| currency | string | 货币代码 |
| amount | integer | 价格（分） |
| type | string | recurring / one_time / donation |
| interval | string | month / year |

参考: `ghost/core/core/server/data/schema/schema.js:801`

#### offers 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 本地优惠 ID |
| name | string | 优惠名称 |
| code | string | 优惠码 |
| product_id | string | 关联 products.id（可为 null） |
| stripe_coupon_id | string | Stripe Coupon ID |
| discount_type | string | percent / amount / trial |
| discount_amount | integer | 折扣金额/百分比 |
| duration | string | trial / once / repeating / forever |
| duration_in_months | integer | repeating 类型的持续月数 |
| redemption_type | string | signup / retention |
| active | boolean | 是否激活 |

参考: `ghost/core/core/server/data/schema/schema.js:483`

#### members_stripe_customers_subscriptions 表

| 字段 | 类型 | 说明 |
|------|------|------|
| subscription_id | string | Stripe 订阅 ID |
| stripe_price_id | string | 关联 stripe_prices.stripe_price_id |
| offer_id | string | 关联 offers.id |
| status | string | 订阅状态 |
| discount_start | dateTime | 折扣开始时间 |
| discount_end | dateTime | 折扣结束时间 |
| trial_start_at | dateTime | 试用开始时间 |
| trial_end_at | dateTime | 试用结束时间 |

参考: `ghost/core/core/server/data/schema/schema.js:690`

---

## 二、Tier 与 Offer 的双向映射

### 2.1 Tier (Product) 与 Stripe 的双向映射

#### 2.1.1 Tier → Stripe 正向映射链

```
Tier (Domain Model)
    ↓
Product (本地数据库表)
    ↓
StripeProduct (映射表)
    ↓
Stripe Product (Stripe API)
    ↓
Stripe Price (Stripe API)
    ↓
StripePrice (本地映射表)
```

#### 2.1.2 Stripe → Tier 回流定位链路

当 Stripe Webhook 触发（如订阅创建、更新）时，系统需要从 Stripe Subscription 反推到本地 Tier/Product：

```
Stripe Subscription (from Webhook)
    ↓
items.data[0].price.product (获取 stripe_product_id)
    ↓
stripe_products 表: WHERE stripe_product_id = ?
    ↓
stripeProduct.related('product').fetch()
    ↓
本地 Product/Tier
```

**核心定位逻辑** (`product-repository.js:90-100`):

```javascript
async get(data, options) {
    // ...
    if ('stripe_product_id' in data) {
        // 1. 先查映射表
        const stripeProduct = await this._StripeProduct.findOne({
            stripe_product_id: data.stripe_product_id
        }, options);

        if (!stripeProduct) {
            return null;
        }

        // 2. 通过 Bookshelf 关系获取本地 Product
        return await stripeProduct.related('product').fetch(options);
    }
    // ...
}
```

**调用链路** (`member-repository.js:1084-1090`):

```javascript
const subscriptionPriceData = _.get(stripeSubscriptionData, 'items.data[0].price');

// 从 Stripe Subscription 获取 stripe_product_id
ghostProduct = await this._productRepository.get(
    {stripe_product_id: subscriptionPriceData.product}, 
    options
);

// 兜底：找不到映射时使用默认付费产品
if (!ghostProduct) {
    ghostProduct = await this._productRepository.getDefaultProduct(options);
}
```

**回流后的同步操作** (`member-repository.js:1093-1110`):

找到或使用默认 Product 后，调用 `productRepository.update({stripe_prices: [...]})` 确保 Stripe Price 记录存在于本地 `stripe_prices` 表中。

参考:
- `ghost/core/core/server/services/members/members-api/repositories/product-repository.js:81-144`
- `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1084-1118`

#### 2.1.2 同步触发机制

Tier 的价格变更通过**领域事件**系统触发同步：

**事件订阅** (`payments-service.js:40-52`):

```javascript
DomainEvents.subscribe(TierCreatedEvent, async (event) => {
    if (event.data.tier.type === 'paid') {
        await this.getPriceForTierCadence(event.data.tier, 'month');
        await this.getPriceForTierCadence(event.data.tier, 'year');
    }
});

DomainEvents.subscribe(TierPriceChangeEvent, async (event) => {
    if (event.data.tier.type === 'paid') {
        await this.getPriceForTierCadence(event.data.tier, 'month');
        await this.getPriceForTierCadence(event.data.tier, 'year');
    }
});
```

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:40-52`

#### 2.1.3 价格同步逻辑

**getPriceForTierCadence** 方法执行以下步骤 (`payments-service.js:476-513`):

1. **获取或创建 Stripe Product**
   - 先查询 `stripe_products` 表是否已有映射
   - 如果没有，调用 Stripe API 创建 Product
   - 保存映射到 `stripe_products` 表

2. **查找匹配的 Stripe Price**
   - 查询条件：`stripe_product_id` + `currency` + `interval` + `amount` + `active: true`
   - 验证 Stripe 端价格状态和属性

3. **创建新价格（如果需要）**
   - 如果没有匹配的价格，在 Stripe 创建新 Price
   - 保存到 `stripe_prices` 表

**关键设计：不修改已有价格，只创建新价格**

```javascript
// 查找匹配价格
const rows = await this.StripePriceModel.where({
    stripe_product_id: product.id,
    currency,
    interval: cadence,
    amount,  // ← 价格也是匹配条件
    active: true,
    type: 'recurring'
}).query().select('id', 'stripe_price_id');
```

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:480-487`

#### 2.1.4 Tier 变更事件

Tier 模型在更新价格时触发事件 (`tier.js:151-173`):

```javascript
updatePricing({currency, monthlyPrice, yearlyPrice}) {
    // ... 验证逻辑
    
    // 检测价格是否实际变化
    if (newCurrency === this.#currency && 
        newMonthlyPrice === this.#monthlyPrice && 
        newYearlyPrice === this.#yearlyPrice) {
        return;
    }

    // 更新价格
    this.#currency = newCurrency;
    this.#monthlyPrice = newMonthlyPrice;
    this.#yearlyPrice = newYearlyPrice;

    // 触发价格变更事件
    this.events.push(TierPriceChangeEvent.create({
        tier: this
    }));
}
```

参考: `ghost/core/core/server/services/tiers/tier.js:151-173`

### 2.2 Offer 到 Stripe Coupon 的映射

#### 2.2.1 映射关系

```
Offer (Domain Model)
    ↓
offers (本地数据库表)
    ↓
stripe_coupon_id 字段
    ↓
Stripe Coupon (Stripe API)
```

#### 2.2.2 Coupon 创建时机

Coupon 创建采用**双重保障机制**：`OfferCreatedEvent` 事件触发创建作为主路径，`stripe_coupon_id` 为空时的实时创建作为兜底。

**路径 1：事件触发创建（主路径）** (`payments-service.js:36-38`)

当 Offer 通过 `offersAPI.createOffer()` 创建后，系统立即发出 `OfferCreatedEvent`，事件处理器异步调用 `getCouponForOffer()` 尝试创建 Coupon：

```javascript
DomainEvents.subscribe(OfferCreatedEvent, async (event) => {
    await this.getCouponForOffer(event.data.offer.id);
});
```

**触发时机**：
- 管理员在后台创建 Offer 并保存成功后
- 事件异步执行，不阻塞 Offer 创建的 API 响应

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:36-38`

**路径 2：`stripe_coupon_id` 为空时实时创建（兜底）** (`payments-service.js:549-562`)

`getCouponForOffer()` 方法内部实现了懒加载逻辑：**当需要 Coupon 且 `stripe_coupon_id` 为空时，实时创建**。

```javascript
async getCouponForOffer(offerId) {
    const row = await this.OfferModel.where({id: offerId})
        .query().select('stripe_coupon_id', 'discount_type').first();
    
    if (!row || row.discount_type === 'trial') {
        return null;  // Trial 类型无需 Coupon
    }
    
    if (!row.stripe_coupon_id) {
        // 兜底创建：数据库中 stripe_coupon_id 为空，实时创建 Coupon
        const offer = await this.offersAPI.getOffer({id: offerId});
        await this.createCouponForOffer(offer);
        return this.getCouponForOffer(offerId);  // 递归重试
    }
    
    return { id: row.stripe_coupon_id };
}
```

**兜底触发条件**（代码实际逻辑）：
- 需要使用 Coupon（调用 `getCouponForOffer`）
- Offer 存在且 `discount_type !== 'trial'`
- 数据库中 `stripe_coupon_id` 为空

**命中兜底的业务场景**：
- **场景 1：生成支付链接** (`payments-service.js:92`)
  - 调用 `getPaymentLink()` 创建新订阅时，若 `offer` 参数传入且不为 trial 类型
  - 代码：`coupon = await this.getCouponForOffer(offer.id);`
- **场景 2：应用到已有订阅** (`member-controller.js:247`)
  - 调用 `applyOfferToSubscription()` 应用 Retention Offer 时
  - 代码：`const coupon = await this._paymentsService.getCouponForOffer(offerId);`

**注意**：
- `discount_type === 'trial'` 的 Offer **不需要** Stripe Coupon，直接通过 Stripe 的 `trial_period_days` 参数实现
- 事件路径是创建 Coupon 的主动方式，兜底是被动保障（事件可能未执行或执行失败）

参考:
- `ghost/core/core/server/services/members/members-api/services/payments-service.js:549-562`
- `ghost/core/core/server/services/members/members-api/services/payments-service.js:92`
- `ghost/core/core/server/services/members/members-api/controllers/member-controller.js:247`

#### 2.2.3 Coupon 创建逻辑

```javascript
async createCouponForOffer(offer) {
    const couponData = {
        name: offer.name,
        duration: offer.duration
    };

    if (offer.duration === 'repeating') {
        couponData.duration_in_months = offer.duration_in_months;
    }

    if (offer.type === 'percent') {
        couponData.percent_off = offer.amount;
    } else {
        couponData.amount_off = offer.amount;
        couponData.currency = offer.currency;
    }

    const coupon = await this.stripeAPIService.createCoupon(couponData);

    await this.OfferModel.edit({
        stripe_coupon_id: coupon.id
    }, { id: offer.id });
}
```

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:567-592`

#### 2.2.4 从 Stripe Coupon 反向创建 Offer

当订阅带有 Stripe Coupon 时，系统会自动创建对应的 Offer (`member-repository.js:1120-1150`):

```javascript
let stripeCouponId = stripeSubscriptionData.discount?.coupon?.id;

if (stripeCouponId && !offerId && ghostProduct) {
    const coupon = stripeSubscriptionData.discount.coupon;
    const cadence = _.get(subscriptionPriceData, 'recurring.interval');
    const tier = {id: ghostProduct.get('id'), name: ghostProduct.get('name')};

    try {
        const offer = await this._offersAPI.ensureOfferForStripeCoupon(
            coupon,
            cadence,
            tier,
            options
        );
        offerId = offer.id;
    } catch (e) {
        // 处理不兼容的 Coupon
        if (e.code === 'INVALID_YEARLY_DURATION') {
            logging.error(`Failed to create offer for Stripe coupon - ${stripeCouponId}`);
        } else {
            throw e;
        }
    }
}
```

参考: `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1120-1150`

**Offer.createFromStripeCoupon** 方法 (`offer.js:407-454`):

- 从 Stripe Coupon 提取：`percent_off` / `amount_off` / `duration` / `duration_in_months`
- 自动生成 Offer 名称和 code
- **状态设置为 archived**（避免被用于新订阅）

```javascript
// Create the offer as archived, so that it can't be used for new signups
const status = 'archived';
```

参考: `ghost/core/core/server/services/offers/domain/models/offer.js:431`

---

## 三、价格变更时已有订阅的处理

### 3.1 核心策略：已有订阅保留原价格

**设计原则**：Tier 价格变更时，**不自动迁移已有订阅**。

#### 3.1.1 价格变更的实现方式

Stripe 的 Price 资源是**不可变**的（immutable）——一旦创建，`amount`、`currency`、`interval` 等字段无法修改。

因此 Ghost 的策略是：
1. **Tier 价格变更 → 创建新的 Stripe Price**
2. **已有订阅继续使用旧 Price**
3. **新订阅使用新 Price**

#### 3.1.2 证据分析

**PaymentsService 中的价格匹配逻辑** (`payments-service.js:476-513`):

```javascript
async getPriceForTierCadence(tier, cadence) {
    const product = await this.getProductForTier(tier);
    const currency = tier.currency.toLowerCase();
    const amount = tier.getPrice(cadence);
    
    // 精确匹配：包括 amount
    const rows = await this.StripePriceModel.where({
        stripe_product_id: product.id,
        currency,
        interval: cadence,
        amount,  // ← 价格必须完全匹配
        active: true,
        type: 'recurring'
    }).query().select('id', 'stripe_price_id');

    // ... 验证 Stripe 端价格

    // 没有匹配价格 → 创建新价格
    const price = await this.createPriceForTierCadence(tier, cadence);
    return { id: price.id };
}
```

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:476-513`

**结论**：价格变更会导致 `amount` 不匹配，从而触发创建新 Price。

#### 3.1.3 会员升级/降级时的价格处理

会员在 Billing Portal 中切换价格时 (`member-repository.js:1627-1704`):

```javascript
async updateSubscription(data, options) {
    // ...
    
    if (data.subscription.price) {
        const subscription = await this._stripeAPIService.getSubscription(
            data.subscription.subscription_id
        );

        const subscriptionItem = subscription.items.data[0];

        if (data.subscription.price !== subscription.price) {
            // 更新订阅价格
            updatedSubscription = await this._stripeAPIService.updateSubscriptionItemPrice(
                subscription.id,
                subscriptionItem.id,
                data.subscription.price
            );
            
            // 移除优惠券
            updatedSubscription = await this._stripeAPIService.removeCouponFromSubscription(
                subscription.id
            );

            // 如果是试用中，立即取消试用
            if (subscriptionModel.get('status') === SUBSCRIPTION_STATUS_TRIALING) {
                updatedSubscription = await this._stripeAPIService.cancelSubscriptionTrial(
                    subscription.id
                );
            }
        }
    }
    
    // ... 同步更新到本地
    await this.linkSubscription({
        id: member.id,
        subscription: updatedSubscription
    }, options);
}
```

参考: `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1627-1704`

**关键行为**：
- 价格变更后**自动移除优惠券**
- 试用期内切换价格**立即结束试用**
- 通过 `updateSubscriptionItemPrice` 调用 Stripe API

#### 3.1.4 Stripe API 调用

```javascript
async updateSubscriptionItemPrice(subscriptionId, id, price, options = {}) {
    const subscription = await this._stripe.subscriptions.update(subscriptionId, {
        proration_behavior: options.prorationBehavior || 'always_invoice',
        items: [{
            id,
            price
        }],
        cancel_at_period_end: false,
        metadata: {
            cancellation_reason: options.cancellationReason ?? null
        }
    });
    return subscription;
}
```

参考: `ghost/core/core/server/services/stripe/stripe-api.js:942-956`

**按比例计费 (Proration)**：
- 默认行为：`always_invoice`（立即生成按比例发票）
- 可选：`create_prorations`（创建按比例项目但不立即收费）、`none`（不按比例）

---

## 四、折扣码的生效与过期清理

### 4.1 折扣码生效机制

#### 4.1.1 新订阅时应用 Offer

**支付链接生成** (`payments-service.js:75-131`):

```javascript
async getPaymentLink({tier, cadence, offer, member, ...}) {
    let coupon = null;
    let trialDays = null;
    
    if (offer) {
        // 验证 Offer 与 Tier 匹配
        if (!offer.tier) {
            throw new BadRequestError({
                message: 'Offer does not have a tier'
            });
        }
        if (!tier.id.equals(offer.tier.id)) {
            throw new BadRequestError({
                message: 'This Offer is not valid for the Tier'
            });
        }
        
        // Trial 类型 Offer
        if (offer.type === 'trial') {
            trialDays = offer.amount;
        } else {
            // 获取或创建 Stripe Coupon
            coupon = await this.getCouponForOffer(offer.id);
        }
    }

    // 获取价格
    const price = await this.getPriceForTierCadence(tier, cadence);

    // 创建 Checkout Session
    const session = await this.stripeAPIService.createCheckoutSession(
        price.id, 
        customer, 
        {
            trialDays: trialDays ?? tier.trialDays,
            coupon: coupon?.id
        }
    );

    return session.url;
}
```

参考: `ghost/core/core/server/services/members/members-api/services/payments-service.js:75-131`

#### 4.1.2 已有订阅应用 Retention Offer

**applyOfferToSubscription** 方法 (`member-repository.js:1716-1869`):

**验证步骤**：

1. **订阅必须激活**
   ```javascript
   const subscriptionStatus = subscriptionModel.get('status');
   if (!this.isActiveSubscriptionStatus(subscriptionStatus)) {
       throw new errors.BadRequestError({
           message: tpl(messages.subscriptionNotActive)
       });
   }
   ```

2. **不能已有活跃 Offer**
   ```javascript
   if (await hasActiveOffer(subscriptionModel, this._offersAPI, options)) {
       throw new errors.BadRequestError({
           message: tpl(messages.subscriptionHasOffer)
       });
   }
   ```

3. **Offer 类型限制**
   - Trial Offer 不能用于已有订阅
   - Signup Offer 不能用于已有订阅
   - Retention Offer 不能用于即将取消的订阅

4. **Tier 和 Cadence 匹配**
   ```javascript
   if (offer.tier && offer.tier.id !== tierId) {
       throw new errors.BadRequestError({
           message: tpl(messages.offerTierMismatch)
       });
   }
   if (offer.cadence !== cadence) {
       throw new errors.BadRequestError({
           message: tpl(messages.offerCadenceMismatch)
       });
   }
   ```

5. **不能重复兑换**
   ```javascript
   const existingRedemption = await this._OfferRedemption.findOne({
       offer_id: data.offerId,
       subscription_id: subscriptionModel.id
   }, options);
   if (existingRedemption) {
       throw new errors.BadRequestError({
           message: tpl(messages.offerAlreadyRedeemed)
       });
   }
   ```

参考: `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1716-1869`

### 4.2 活跃 Offer 检测

**hasActiveOffer** 工具函数 (`has-active-offer.js:1-51`):

```javascript
module.exports = async function hasActiveOffer(subscriptionModel, offersAPI, options = {}) {
    const subscriptionData = {
        discount_start: subscriptionModel.get('discount_start'),
        discount_end: subscriptionModel.get('discount_end'),
        start_date: subscriptionModel.get('start_date'),
        current_period_end: subscriptionModel.get('current_period_end')
    };

    // 检查活跃试用
    const trialEndAt = subscriptionModel.get('trial_end_at');
    if (trialEndAt && new Date(trialEndAt) > new Date()) {
        return true;
    }

    // 检查活跃 Offer
    const offerId = subscriptionModel.get('offer_id');
    if (!offerId) {
        return false;
    }

    // 计算折扣窗口
    try {
        const offer = await offersAPI.getOffer({id: offerId}, options);
        if (!offer) {
            return false;
        }

        const discountWindow = getDiscountWindow(subscriptionData, offer);
        if (discountWindow) {
            return !discountWindow.end || new Date(discountWindow.end) > new Date();
        }

        return false;
    } catch (e) {
        return true;  // 查询失败时保守处理
    }
};
```

参考: `ghost/core/core/server/services/members/members-api/utils/has-active-offer.js:1-51`

### 4.3 折扣过期处理

#### 4.3.1 通过 Webhook 同步

Stripe 会在折扣过期时发送 `customer.subscription.updated` 事件，Ghost 通过 Webhook 处理：

**SubscriptionEventService** (`subscription-event-service.js:12-59`):

```javascript
async handleSubscriptionEvent(subscription) {
    const subscriptionPriceData = _.get(subscription, 'items.data');
    
    const member = await memberRepository.get({
        customer_id: subscription.customer
    });

    if (member) {
        // 调用 linkSubscription 同步最新状态
        await memberRepository.linkSubscription({
            id: member.id,
            subscription
        });
        
        // ...
    }
}
```

参考: `ghost/core/core/server/services/stripe/services/webhook/subscription-event-service.js:12-59`

**linkSubscription 中的折扣处理** (`member-repository.js:1219-1230`):

```javascript
const previousOfferId = stripeCustomerSubscriptionModel.get('offer_id');

// CASE: Only preserve offer_id for active trials (trial offers don't have Stripe discounts)
// Otherwise, allow offer_id to be cleared when the Stripe discount expires
if (!subscriptionData.offer_id) {
    const trialEndAt = subscriptionData.trial_end_at;
    const hasActiveTrial = trialEndAt && new Date(trialEndAt) > new Date();

    if (hasActiveTrial) {
        // 试用活跃 → 保留 offer_id
        delete subscriptionData.offer_id;
    }
    // 否则 → offer_id 会被更新为 null
}
```

参考: `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1219-1230`

#### 4.3.2 订阅数据中的折扣字段

```javascript
const subscriptionData = {
    // ...
    offer_id: offerId,
    discount_start: stripeSubscriptionData.discount?.start 
        ? new Date(stripeSubscriptionData.discount.start * 1000) 
        : null,
    discount_end: stripeSubscriptionData.discount?.end 
        ? new Date(stripeSubscriptionData.discount.end * 1000) 
        : null
};
```

参考: `ghost/core/core/server/services/members/members-api/repositories/member-repository.js:1152-1184`

#### 4.3.3 无独立的定时清理任务

**分析结论**：Ghost 没有专门的定时任务来清理过期的折扣码。

**清理依赖**：
1. **Stripe Webhook**：折扣过期时 Stripe 发送事件，`discount.end` 更新或 `discount` 变为 null
2. **主动同步**：调用 `linkSubscription` 时从 Stripe 获取最新状态
3. **Offer 状态管理**：通过 `active` 字段手动停用

**相关清理任务**：
- `clean-expired-comped.js`：清理过期的 complimentary 订阅（不是折扣码）
- 无专门的 Offer 过期清理任务

参考: `ghost/core/core/server/services/members/jobs/clean-expired-comped.js`

### 4.4 Offer 状态管理

#### 4.4.1 Retention Offer 的唯一性约束

**archiveActiveRetentionOffers** 方法 (`offers-api.js:33-47`):

```javascript
async archiveActiveRetentionOffers(offerId, cadence, options = {}) {
    const activeRetentionOffers = await this.repository.getAll({
        transacting: options.transacting,
        filter: 'status:active+redemption_type:retention'
    }, {withRedemptionStats: false});

    for (const activeRetentionOffer of activeRetentionOffers) {
        // 同一 cadence 只保留一个活跃的 Retention Offer
        if (activeRetentionOffer.id === offerId || 
            activeRetentionOffer.cadence.value !== cadence) {
            continue;
        }

        activeRetentionOffer.status = OfferStatus.create('archived');
        await this.repository.save(activeRetentionOffer, options);
    }
}
```

参考: `ghost/core/core/server/services/offers/application/offers-api.js:33-47`

**触发时机**：
- 创建新的 Retention Offer 时
- 更新 Retention Offer 状态为 active 时

---

## 五、关键代码位置汇总

### 5.1 核心服务

| 功能 | 文件位置 |
|------|----------|
| Stripe API 封装 | `ghost/core/core/server/services/stripe/stripe-api.js` |
| 支付服务（价格/优惠券同步） | `ghost/core/core/server/services/members/members-api/services/payments-service.js` |
| 会员订阅管理 | `ghost/core/core/server/services/members/members-api/repositories/member-repository.js` |
| Tier 领域模型 | `ghost/core/core/server/services/tiers/tier.js` |
| Offer 领域模型 | `ghost/core/core/server/services/offers/domain/models/offer.js` |
| Offer API | `ghost/core/core/server/services/offers/application/offers-api.js` |

### 5.2 数据库模型

| 模型 | 文件位置 |
|------|----------|
| Product (Tier) | `ghost/core/core/server/models/product.js` |
| StripeProduct | `ghost/core/core/server/models/stripe-product.js` |
| StripePrice | `ghost/core/core/server/models/stripe-price.js` |
| 数据库 Schema | `ghost/core/core/server/data/schema/schema.js` |

### 5.3 Webhook 处理

| 事件 | 文件位置 |
|------|----------|
| 订阅事件处理 | `ghost/core/core/server/services/stripe/services/webhook/subscription-event-service.js` |
| Stripe Webhook 控制器 | `ghost/core/core/server/services/stripe/webhook-controller.js` |

---

## 六、总结

### 6.1 Tier 与 Offer 的双向映射

| 方向 | 本地实体 | Stripe 实体 | 同步触发 |
|------|----------|-------------|----------|
| Tier → Stripe | Product + monthly/yearly_price | Product + Price | TierCreatedEvent / TierPriceChangeEvent |
| Offer → Stripe | Offer | Coupon | OfferCreatedEvent（主路径）/ 首次使用（兜底容错） |
| Stripe → Offer | - | Coupon | linkSubscription（处理现有订阅的折扣） |

### 6.2 价格变更处理

| 场景 | 行为 |
|------|------|
| Tier 价格变更 | 创建新的 Stripe Price，旧 Price 保持不变 |
| 已有订阅 | 继续使用旧 Price，不自动迁移 |
| 新订阅 | 使用新 Price |
| 会员主动切换价格 | 通过 updateSubscriptionItemPrice，移除优惠券，按比例计费 |

### 6.3 折扣码机制

| 方面 | 实现方式 |
|------|----------|
| 生效检测 | hasActiveOffer 检查 discount_end 时间 |
| 过期处理 | 依赖 Stripe Webhook 事件同步 |
| 定时清理 | 无独立定时任务，依赖 Webhook 和状态字段 |
| 状态管理 | 手动设置 active 字段，Retention Offer 有唯一性约束 |

### 6.4 关键设计决策

1. **Price 不可变性**：遵循 Stripe 的 Price 不可变设计，价格变更时创建新 Price
2. **Tier 回流定位链**：从 Stripe Subscription → `stripe_products` 映射表 → 本地 Product，缺失时使用默认付费产品兜底
3. **Coupon 双重保障**：OfferCreatedEvent 事件触发创建（主路径）+ 首次使用懒加载兜底（容错）
4. **Webhook 驱动同步**：订阅状态、折扣过期等变化通过 Stripe Webhook 驱动 `linkSubscription()` 同步
5. **保守的 Offer 反向创建**：从 Stripe Coupon 创建的 Offer 状态设为 archived，避免被误用于新订阅
