# Ghost 互荐订阅状态跨站同步机制深度分析

## 1. 核心概念澄清

在深入分析之前，需要明确区分两个独立的"订阅"概念：

| 概念 | 说明 | 维护方 |
|------|------|--------|
| **推荐关系**（Recommendation） | 站点 A 推荐站点 B 的关系 | 推荐方站点 A 维护 |
| **互荐状态**（Recommending Back） | 站点 B 是否也在推荐站点 A | 被推荐方站点 B 动态计算 |
| **一键订阅能力**（One-Click Subscribe） | 站点是否支持外部用户一键订阅 | 被推荐站点自身维护 |
| **用户订阅事件**（Subscribe Event） | 用户点击"订阅"按钮的行为记录 | 推荐方站点本地记录 |

---

## 2. 状态维护权责划分

### 2.1 推荐关系（互荐关系）

**维护方：推荐方站点**

当站点 A 推荐站点 B 时：

1. **站点 A 保存推荐记录**
   `ghost/core/core/server/services/recommendations/service/recommendation-service.ts:123-143`

   ```typescript
   async addRecommendation(addRecommendation: AddRecommendation): Promise<RecommendationPlain> {
       const recommendation = Recommendation.create(addRecommendation);
       await this.repository.save(recommendation);
       // 更新 .well-known 让外部可见
       await this.updateWellknown(recommendations);
       // 异步通知站点 B
       this.sendMentionToRecommendation(recommendation);
   }
   ```

2. **站点 A 公开推荐列表**
   通过 `/.well-known/recommendations.json` 公开，供任何站点查询：

   ```json
   [
     {
       "url": "https://site-b.com/",
       "updated_at": "2024-01-15T10:30:00.000Z",
       "created_at": "2024-01-10T08:00:00.000Z"
     }
   ]
   ```

3. **站点 B 接收并验证**
   `ghost/core/core/server/services/mentions/mentions-api.js:270-277`

   ```javascript
   async processWebmention(webmention) {
       // 通过 source-target 唯一键去重
       let mention = await this.#repository.getBySourceAndTarget(
           webmention.source,  // A 的 .well-known 地址
           webmention.target   // B 的站点地址
       );
       return await this.#updateWebmention(mention, webmention);
   }
   ```

**关键结论**：推荐关系的"真相源"在推荐方站点 A，站点 B 只是被动接收和缓存。

### 2.2 互荐检测状态

**维护方：被推荐方站点（动态计算，不持久化）**

互荐状态是一个**计算字段**，而非持久化存储的字段：

`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:107-128`

```typescript
async #mentionToIncomingRecommendation(mention: Mention): Promise<IncomingRecommendation|null> {
    // 从 source 提取站点地址
    const url = new URL(mention.source.toString().replace(/\/.well-known\/recommendations\.json$/, ''));
    
    // 查询本地 recommendations 表，判断是否也在推荐对方
    const existing = await this.#recommendationService.readRecommendationByUrl(url);
    const recommendingBack = !!existing;  // 动态计算
    
    return {
        ...
        recommendingBack  // 不在 mentions 表中存储
    };
}
```

**API 响应**：`incoming-recommendations` 端点返回 `recommending_back` 字段

**关键结论**：互荐状态从不跨站同步，而是每次查询时在本地动态计算。

### 2.3 一键订阅能力

**维护方：被推荐站点自身**

一键订阅能力是站点的公共配置，通过 `members/api/site` 端点公开：

`ghost/core/core/server/services/public-config/site.js:6-19`

```javascript
module.exports = function getSiteProperties() {
    return {
        title: settingsCache.get('title'),
        description: settingsCache.get('description'),
        // ...
        allow_external_signup: settingsCache.get('allow_self_signup') && 
            !(settingsCache.get('portal_signup_checkbox_required') && 
              settingsCache.get('portal_signup_terms_html')),
        site_uuid: settingsCache.get('site_uuid')
    };
};
```

**发现流程**（站点 A 检测站点 B 是否支持一键订阅）：

`ghost/core/core/server/services/recommendations/service/recommendation-metadata-service.ts:89-119`

```typescript
async fetch(url: URL): Promise<RecommendationMetadata> {
    // 1. 尝试访问目标站点的 Ghost 专用 API
    let ghostSiteData = await this.#fetchJSON(
        new URL('members/api/site', url),
        options
    );
    
    if (ghostSiteData?.site?.allow_external_signup !== undefined) {
        return {
            oneClickSubscribe: !!ghostSiteData.site.allow_external_signup,
            // ... 其他元数据
        };
    }
    
    // 2. 非 Ghost 站点回退到 oEmbed
    const oembed = await this.#oembedService.fetchOembedDataFromUrl(...);
    return {
        oneClickSubscribe: false,  // 非 Ghost 站点不支持
        // ...
    };
}
```

**关键结论**：一键订阅能力由目标站点自身控制，推荐方站点定期检测并缓存。

### 2.4 用户订阅事件

**维护方：推荐方站点（本地私有数据）**

当用户在站点 A 上点击站点 B 的"订阅"按钮时：

`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:268-271`

```typescript
async trackSubscribed({id, memberId}: { id: string, memberId: string }) {
    const subscribeEvent = SubscribeEvent.create({
        recommendationId: id,
        memberId: memberId
    });
    await this.subscribeEventRepository.save(subscribeEvent);
}
```

**事件记录**：`recommendation_subscribe_events` 表

| 字段 | 说明 | 是否跨站同步 |
|------|------|-------------|
| `id` | 事件 ID | 否 |
| `recommendation_id` | 关联的推荐 ID | 否 |
| `member_id` | 订阅用户的本地 member ID | **否，绝对私密** |
| `created_at` | 事件时间 | 否 |

**访问控制**：
`ghost/core/core/server/api/endpoints/recommendations-public.js:45-64`

```javascript
trackSubscribed: {
    permissions: true,  // 需要会员认证
    async query(frame) {
        await recommendations.controller.trackSubscribed(frame);
    }
}
```

**控制器验证**：
`ghost/core/core/server/services/recommendations/service/recommendation-controller.ts:181-190`

```typescript
async trackSubscribed(frame: Frame) {
    const member = this.#authMember(frame);  // 强制认证
    // memberId 从会话中获取，不是前端传入
    await this.service.trackSubscribed({
        id,
        memberId: member.id
    });
}
```

**关键结论**：用户订阅事件（以及 member_id）是推荐方站点的私有数据，**绝对不会跨站同步**。

---

## 3. 跨实例同步字段清单

### 3.1 推荐关系同步

**通过 WebMention + .well-known 协议同步的字段**

| 字段 | 数据流向 | 同步方式 | 代码位置 |
|------|----------|----------|----------|
| `source` | A → B | WebMention payload | `mention-sending-service.js:91-96` |
| `target` | A → B | WebMention payload | 同上 |
| `source_title` | A → B | 源站点元数据抓取 | `mentions-api.js:205-212` |
| `source_site_title` | A → B | 源站点元数据抓取 | 同上 |
| `source_excerpt` | A → B | 源站点元数据抓取 | 同上 |
| `source_author` | A → B | 源站点元数据抓取 | 同上 |
| `source_favicon` | A → B | 源站点元数据抓取 | 同上 |
| `source_featured_image` | A → B | 源站点元数据抓取 | 同上 |
| `verified` | B 本地 | 验证源是否真的包含目标 | `mention.js:43-84` |
| `deleted` | A → B（间接） | B 验证时发现源不再包含目标 | `mention.js:53-56` |

### 3.2 推荐元数据同步

**通过 `members/api/site` + oEmbed 定期抓取的字段**

| 字段 | 数据流向 | 同步方式 | 代码位置 |
|------|----------|----------|----------|
| `title` | B → A（A 主动抓取） | 启动后延迟刷新 | `recommendation-service.ts:55-70` |
| `excerpt` | B → A | 同上 | `recommendation-metadata-service.ts:113` |
| `featured_image` | B → A | 同上 | 同上 |
| `favicon` | B → A | 同上 | 同上 |
| `one_click_subscribe` | B → A | 同上（从 `allow_external_signup` 映射） | `recommendation-metadata-service.ts:117` |

### 3.3 本地私有字段（永不同步）

**站点 A 的私有数据**（recommendations 表）：

| 字段 | 存储位置 | 说明 |
|------|----------|------|
| `description` | 推荐方本地 | 管理员自定义描述 |
| `created_at` | 推荐方本地 | 创建时间 |
| `updated_at` | 推荐方本地 | 更新时间 |

**站点 A 的统计数据**（关联表，仅本地聚合）：

| 统计项 | 计算方式 | 代码位置 |
|--------|----------|----------|
| `clickCount` | 本地 COUNT(`recommendation_click_events`) | `bookshelf-recommendation-repository.ts:31-35` |
| `subscriberCount` | 本地 COUNT(`recommendation_subscribe_events`) | `bookshelf-recommendation-repository.ts:37-41` |

**站点 A 的点击事件**（`recommendation_click_events` 表）：

| 字段 | 是否同步 | 说明 |
|------|----------|------|
| `member_id` | **否** | 可空，用于去重统计 |
| `recommendation_id` | 否 | 本地外键 |

**站点 A 的订阅事件**（`recommendation_subscribe_events` 表）：

| 字段 | 是否同步 | 说明 |
|------|----------|------|
| `member_id` | **否** | 必填，私有用户标识 |
| `recommendation_id` | 否 | 本地外键 |

**站点 B 的接收数据**（mentions 表，部分字段本地计算）：

| 字段 | 是否同步 | 说明 |
|------|----------|------|
| `recommending_back` | **否** | 动态计算，查询时判断 |
| `resource_id` | 否 | 本地资源关联 |
| `resource_type` | 否 | 本地资源类型 |

---

## 4. 跨站更新触发时机

### 4.1 推荐关系变更触发

#### 场景 1：添加推荐

**触发方**：站点 A（推荐方）

**时机**：管理员通过 Admin API 添加推荐

`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:123-143`

```typescript
async addRecommendation(addRecommendation: AddRecommendation) {
    // 1. 保存本地
    await this.repository.save(recommendation);
    
    // 2. 立即更新 .well-known
    await this.updateWellknown(recommendations);
    
    // 3. 异步发送 WebMention（不阻塞响应）
    this.sendMentionToRecommendation(recommendation);
}
```

**WebMention 发送流程**：
`ghost/core/core/server/services/mentions/mention-sending-service.js:165-177`

```javascript
async sendAll({url, links}) {
    for (const target of links) {
        // 发现目标站点的 WebMention endpoint
        const endpoint = await this.#discoveryService.getEndpoint(target);
        if (endpoint) {
            try {
                await this.send({source: url, target, endpoint});
            } catch (e) {
                // 失败只记录日志，不重试整个批次
                logging.error('[Webmention] Failed sending...', e);
            }
        }
    }
}
```

#### 场景 2：删除推荐

**触发方**：站点 A（推荐方）

**时机**：管理员删除推荐

`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:219-236`

```typescript
async deleteRecommendation(id: string) {
    existing.delete();
    await this.repository.save(existing);
    
    // .well-known 中不再包含该推荐
    await this.updateWellknown(recommendations);
    
    // 仍发送 WebMention，让目标站点验证
    this.sendMentionToRecommendation(existing);
}
```

**站点 B 的处理**（验证发现源不再包含目标）：
`ghost/core/core/server/services/mentions/mention.js:43-84`

```javascript
verify(html, contentType) {
    if (contentType.includes('application/json')) {
        // 检查 .well-known JSON 中是否包含目标 URL
        this.#verified = !!html.includes(JSON.stringify(this.target.href));
        
        if (wasVerified && !this.#verified) {
            // 之前验证通过，现在不通过 → 标记删除
            this.#deleted = true;
            this.#verified = true;  // 保持 verified=true 表示"已确认删除"
        }
    }
}
```

#### 场景 3：编辑推荐

**触发方**：站点 A（推荐方）

**时机**：管理员编辑推荐的描述等字段

```typescript
async editRecommendation(id: string, recommendationEdit) {
    existing.edit(recommendationEdit);
    await this.repository.save(existing);
    await this.updateWellknown(recommendations);
    this.sendMentionToRecommendation(existing);  // 也发送 WebMention
}
```

### 4.2 启动时重验证

**触发方**：所有站点（启动后延迟执行）

**时机 1**：推荐元数据刷新（站点 A）
`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:55-70`

```typescript
async init() {
    // 延迟 2-7 分钟（随机化避免惊群）
    if (!process.env.NODE_ENV?.startsWith('test')) {
        setTimeout(async () => {
            await this.updateAllRecommendationsMetadata();
        }, 2 * 60 * 1000 + Math.random() * 5 * 60 * 1000);
    }
}
```

**刷新内容**：
- 重新访问 `{url}/members/api/site`
- 更新 `title`, `excerpt`, `featured_image`, `favicon`, `one_click_subscribe`

**时机 2**：传入推荐重验证（站点 B）
`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:80-93`

```typescript
async init() {
    // 延迟 15 秒到 3 分钟
    if (!process.env.NODE_ENV?.startsWith('test') && 
        process.env.NODE_ENV !== 'development') {
        setTimeout(() => {
            this.#updateIncomingRecommendations();
        }, 15 * 1000 + Math.random() * 5 * 60 * 1000);
    }
}
```

**重验证内容**：
`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:99-105`

```typescript
async #updateIncomingRecommendations() {
    // 包含已删除的记录
    const filter = `source:~$'/.well-known/recommendations.json'+deleted:[true,false]`;
    await this.#mentionsApi.refreshMentions({filter, limit: 100});
}
```

---

## 5. 去重机制

### 5.1 WebMention 接收去重

**去重键**：`source` + `target` 组合

`ghost/core/core/server/services/mentions/mentions-api.js:270-277`

```javascript
async processWebmention(webmention) {
    // 通过 source 和 target 唯一标识一条推荐关系
    let mention = await this.#repository.getBySourceAndTarget(
        webmention.source,
        webmention.target
    );
    
    if (mention) {
        // 已存在 → 更新而非创建
        return await this.#updateWebmention(mention, webmention);
    } else {
        // 不存在 → 创建新记录
        return await this.#updateWebmention(null, webmention);
    }
}
```

**Repository 查询**：
`ghost/core/core/server/services/mentions/bookshelf-mention-repository.js:104-115`

```javascript
async getBySourceAndTarget(source, target) {
    const model = await this.#MentionModel.findOne({
        source: source.href,
        target: target.href
    }, {require: false});
    return model ? this.#modelToMention(model) : null;
}
```

### 5.2 推荐 URL 去重

**去重键**：URL 的 `hostname` + `pathname`（忽略协议、www、查询参数）

**添加时检查**：
`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:126-132`

```typescript
async addRecommendation(addRecommendation: AddRecommendation) {
    const existing = await this.repository.getByUrl(recommendation.url);
    if (existing) {
        throw new errors.ValidationError({
            message: 'A recommendation with this URL already exists.'
        });
    }
}
```

**Repository 匹配逻辑**：
`ghost/core/core/server/services/recommendations/service/bookshelf-recommendation-repository.ts:100-116`

```typescript
async getByUrl(url: URL): Promise<Recommendation | null> {
    // 只匹配 hostname 和 pathname
    const existing = recommendations.find((r) => {
        return r.url.hostname.replace('www.', '') === url.hostname.replace('www.', '') &&
               r.url.pathname.replace(/\/$/, '') === url.pathname.replace(/\/$/, '');
    }) || null;
    return existing;
}
```

**忽略的部分**：
- 协议（http vs https）
- www 前缀
- 尾部斜杠
- 查询参数
- hash 片段

### 5.3 列表显示去重

**场景**：同一来源站点多次发送 WebMention

`ghost/core/core/server/services/mentions/bookshelf-mention-repository.js:76-79`

```javascript
if (options.unique) {
    _options.whereRaw = 'NOT EXISTS (select id from mentions as m where m.id > mentions.id and m.source = mentions.source)';
}
```

**效果**：同一 `source` 只显示最新的一条记录。

### 5.4 点击/订阅事件去重

**注意**：事件表**不做去重**，允许重复记录

`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:263-271`

```typescript
async trackClicked({id, memberId}) {
    // 每次点击都创建新事件
    const clickEvent = ClickEvent.create({recommendationId: id, memberId});
    await this.clickEventRepository.save(clickEvent);
}

async trackSubscribed({id, memberId}) {
    // 每次订阅都创建新事件
    const subscribeEvent = SubscribeEvent.create({recommendationId: id, memberId});
    await this.subscribeEventRepository.save(subscribeEvent);
}
```

**原因**：
- 统计指标需要原始事件数据
- `member_id` 可用于应用层去重（如同一用户多次点击只算一次）
- 时序分析需要完整事件流

---

## 6. 重试边界

### 6.1 HTTP 请求层重试

**配置**：`retry.limit = 3`

**应用位置 1**：WebMention 发送
`ghost/core/core/server/services/mentions/mention-sending-service.js:91-107`

```javascript
const response = await this.#externalRequest.post(endpoint.href, {
    timeout: {request: 15000},  // 15秒超时
    retry: {
        limit: 3
    }
});
```

**应用位置 2**：元数据抓取
`ghost/core/core/server/services/recommendations/service/recommendation-metadata-service.ts:51-64`

```typescript
const response = await this.#externalRequest.get(url.toString(), {
    timeout: {request: 15000},
    retry: {
        limit: 3
    }
});
```

**应用位置 3**：Endpoint 发现
`ghost/core/core/server/services/mentions/mention-discovery-service.js:17-28`

```javascript
const response = await this.#externalRequest(url.href, {
    timeout: {request: 15000},
    retry: {
        limit: 3
    }
});
```

### 6.2 重试触发条件

根据 `externalRequest` 的默认行为（基于 `got` 库）：

| 触发条件 | 是否重试 |
|----------|----------|
| 网络错误（ECONNRESET, ETIMEDOUT 等） | ✅ 重试 |
| DNS 解析失败 | ✅ 重试 |
| HTTP 5xx 错误 | ✅ 重试 |
| HTTP 4xx 错误 | ❌ 不重试 |
| 请求成功但业务逻辑失败 | ❌ 不重试 |

### 6.3 重试间隔

`got` 库默认使用**指数退避**策略：
- 第 1 次重试：~1 秒后
- 第 2 次重试：~2 秒后
- 第 3 次重试：~4 秒后

### 6.4 失败后的处理

**WebMention 发送失败**：
`ghost/core/core/server/services/mentions/mention-sending-service.js:170-175`

```javascript
try {
    await this.send({source: url, target, endpoint});
} catch (e) {
    // 仅记录日志，不再重试
    logging.error('[Webmention] Failed sending via ' + endpoint.href + ': ' + e.message);
}
```

**关键点**：
1. **无持久化队列**：失败后不会重新入队
2. **依赖启动重验证**：下次启动时 `refreshMentions` 会重新验证
3. **单向通知**：WebMention 是单向协议，目标站点不会主动回推状态

**元数据刷新失败**：
`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:72-83`

```typescript
async updateAllRecommendationsMetadata() {
    for (const recommendation of recommendations) {
        try {
            await this._updateRecommendationMetadata(recommendation);
            await this.repository.save(recommendation);
        } catch (e) {
            // 单个失败不影响其他
            logging.error('[Recommendations] Failed to save...', e);
        }
    }
}
```

---

## 7. 完整数据流图

### 7.1 推荐创建数据流

```
站点 A（推荐方）                              站点 B（被推荐方）
    |                                            |
    | 1. 管理员添加推荐                           |
    |    POST /ghost/api/admin/recommendations   |
    |                                            |
    | 2. 写入 recommendations 表                 |
    |    - id, url, title, description          |
    |    - one_click_subscribe (默认 false)     |
    |                                            |
    | 3. 更新 /.well-known/recommendations.json |
    |    [{"url": "https://site-b.com/", ...}]  |
    |                                            |
    | 4. 异步发送 WebMention ------------------> |
    |    source: A/.well-known/...              |
    |    target: B/                             |
    |                                            |
    |                                            | 5. 接收 WebMention
    |                                            |    POST /webmention
    |                                            |
    |                                            | 6. 去重检查
    |                                            |    getBySourceAndTarget()
    |                                            |
    |                                            | 7. 验证源
    |                                            |    GET A/.well-known/...
    |                                            |    检查是否包含 B 的 URL
    |                                            |
    |                                            | 8. 写入 mentions 表
    |                                            |    - source, target
    |                                            |    - source_title, source_excerpt
    |                                            |    - verified: true
    |                                            |
    |                                            | 9. 触发 MentionCreatedEvent
    |                                            |    → 可能发送通知邮件
    |
    | 10. 启动后延迟刷新元数据                     |
    |     GET B/members/api/site --------------> |
    |     <-- {allow_external_signup, title,...} |
    |                                            |
    | 11. 更新本地缓存                            |
    |     one_click_subscribe = true             |
```

### 7.2 订阅事件数据流（纯本地）

```
站点 A（推荐方）
    |
    | 1. 会员用户点击"订阅"按钮
    |    POST /members/api/recommendations/:id/subscribed
    |
    | 2. 认证会员身份
    |    loadMemberSession → #authMember(frame)
    |
    | 3. 写入 recommendation_subscribe_events 表
    |    - id (自动生成)
    |    - recommendation_id (本地外键)
    |    - member_id (从会话获取)
    |    - created_at
    |
    | 4. 返回 204 No Content
    |
    | [注意] 这一步完全是本地操作，不会向站点 B 发送任何请求
    | [注意] member_id 绝对不会离开站点 A
```

---

## 8. 关键设计决策总结

| 决策点 | 选择 | 原因 |
|--------|------|------|
| 推荐关系真相源 | 推荐方站点 | 符合 WebMention 协议设计，源拥有最终话语权 |
| 互荐状态 | 动态计算，不持久化 | 避免数据不一致，查询时实时判断 |
| 用户订阅事件 | 推荐方本地私有 | 隐私保护，member_id 绝不同步 |
| 一键订阅能力 | 被推荐方控制 | 站点自主决定是否开放外部订阅 |
| 去重键 | source + target | 同一对站点间只维护一条推荐关系 |
| URL 去重 | hostname + pathname | 协议、www 前缀等不影响识别 |
| 重试策略 | HTTP 层 3 次 + 启动重验证 | 平衡及时性与资源消耗 |
| 失败处理 | 仅记录日志 | WebMention 是单向协议，不保证送达 |
| 元数据刷新 | 启动后延迟执行 | 避免启动风暴，给外部站点留准备时间 |

---

## 9. 边界情况

| 场景 | 处理方式 |
|------|----------|
| 站点 B 宕机时收到推荐 | WebMention 发送失败，日志记录；下次启动重验证时恢复 |
| 站点 A 删除推荐但 WebMention 丢失 | 站点 B 启动重验证时发现源不再包含目标，自动标记删除 |
| 同一站点多次推荐 | URL 去重检查抛出错误 |
| 用户多次点击订阅按钮 | 每次都记录事件，但 COUNT 统计会包含重复（可通过 member_id 去重） |
| 非 Ghost 站点推荐 Ghost 站点 | WebMention 协议兼容，但元数据走 oEmbed 回退，one_click_subscribe=false |
| Ghost 站点推荐非 Ghost 站点 | 同上，元数据走 oEmbed，无一键订阅能力 |
