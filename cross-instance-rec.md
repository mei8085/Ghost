# Ghost 跨站点互荐协作机制分析

## 1. 概述

Ghost 实现了基于 WebMention 协议的跨站点推荐系统，允许不同 Ghost 实例（或支持 WebMention 的站点）之间互相推荐并保持状态同步。该系统的核心目标是：
- 站点 A 可以推荐站点 B
- 站点 B 能够感知到被站点 A 推荐（"互荐"检测）
- 推荐状态变化时能够及时同步
- 失效推荐能够被自动清理

---

## 2. 实例发现机制

### 2.1 双向发现协议

#### 2.1.1 公开推荐列表（.well-known）

每个 Ghost 实例通过 `/.well-known/recommendations.json` 公开其推荐列表：

**文件位置**：`ghost/core/core/server/services/recommendations/service/well-known-service.ts:33-46`

```json
[
  {
    "url": "https://other-site.com/",
    "updated_at": "2024-01-15T10:30:00.000Z",
    "created_at": "2024-01-10T08:00:00.000Z"
  }
]
```

**更新时机**：
- 启动时初始化
- 添加/编辑/删除推荐后立即更新

#### 2.1.2 WebMention Endpoint 发现

当站点 A 添加对站点 B 的推荐时，需要先发现 B 的 WebMention 接收端点。

**发现流程**（`ghost/core/core/server/services/mentions/mention-discovery-service.js:15-74`）：

1. **HTTP Link Header 优先**：检查响应头中的 `Link` 字段
   ```
   Link: <https://site-b.com/webmention>; rel="webmention"
   ```

2. **HTML 回退**：解析 HTML 页面查找
   ```html
   <link rel="webmention" href="https://site-b.com/webmention">
   <!-- 或 -->
   <a rel="webmention" href="https://site-b.com/webmention">
   ```

3. **请求参数**：
   - 超时：15秒
   - 重试：3次（仅网络问题或特定 HTTP 状态码）
   - 重定向：最多 10 次

---

## 3. 推荐状态同步流程

### 3.1 推荐创建与通知

#### 场景：站点 A 推荐站点 B

**步骤 1：站点 A 保存推荐**
`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:123-143`

```typescript
async addRecommendation(addRecommendation: AddRecommendation): Promise<RecommendationPlain> {
    // 1. 创建推荐实体
    const recommendation = Recommendation.create(addRecommendation);
    
    // 2. 检查 URL 唯一性
    const existing = await this.repository.getByUrl(recommendation.url);
    
    // 3. 保存到数据库
    await this.repository.save(recommendation);
    
    // 4. 更新 .well-known/recommendations.json
    const recommendations = await this.#listRecommendations();
    await this.updateWellknown(recommendations);
    
    // 5. 发送 WebMention 通知（异步，不阻塞）
    this.sendMentionToRecommendation(recommendation);
    
    return recommendation.plain;
}
```

**步骤 2：站点 A 发送 WebMention**
`ghost/core/core/server/services/mentions/mention-sending-service.js:87-117`

```javascript
async send({source, target, endpoint}) {
    const response = await this.#externalRequest.post(endpoint.href, {
        form: {
            source: source.href,           // A 的 .well-known 地址
            target: target.href,           // B 的站点地址
            source_is_ghost: true          // 标识来源是 Ghost
        },
        timeout: {request: 15000},
        retry: {limit: 3}
    });
}
```

**步骤 3：站点 B 接收并处理**
`ghost/core/core/server/services/mentions/mentions-api.js:270-277`

```javascript
async processWebmention(webmention) {
    // 1. 查找是否已存在该 source-target 对
    let mention = await this.#repository.getBySourceAndTarget(
        webmention.source,
        webmention.target
    );
    
    // 2. 更新或创建
    return await this.#updateWebmention(mention, webmention);
}
```

### 3.2 互荐检测（Recommending Back）

站点 B 收到来自 A 的推荐后，会检查自己是否也在推荐 A：

`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:107-128`

```typescript
async #mentionToIncomingRecommendation(mention: Mention): Promise<IncomingRecommendation|null> {
    // 从 source URL 提取站点地址（移除 /.well-known/recommendations.json）
    const url = new URL(mention.source.toString().replace(/\/.well-known\/recommendations\.json$/, ''));
    
    // 检查本站点是否也推荐了这个 URL
    const existing = await this.#recommendationService.readRecommendationByUrl(url);
    const recommendingBack = !!existing;
    
    return {
        ...
        recommendingBack  // 这是关键字段
    };
}
```

**API 暴露**：`incoming-recommendations` 端点返回 `recommending_back` 字段，供 Admin UI 显示"互荐"状态。

### 3.3 元数据与一键订阅同步

推荐的元数据（标题、描述、图标、一键订阅支持）会定期更新：

**初始化延迟更新**：`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:55-70`

```typescript
async init() {
    // 启动后延迟 2-7 分钟（随机化避免 thundering herd）
    if (!process.env.NODE_ENV?.startsWith('test')) {
        setTimeout(async () => {
            await this.updateAllRecommendationsMetadata();
        }, 2 * 60 * 1000 + Math.random() * 5 * 60 * 1000);
    }
}
```

**元数据获取优先级**：`ghost/core/core/server/services/recommendations/service/recommendation-metadata-service.ts:89-132`

1. **Ghost 站点检测**：尝试访问 `{url}/members/api/site`
   - 检查 `allow_external_signup` 字段确定是否支持一键订阅
   - 提取 `title`, `description`, `cover_image`, `icon/logo`

2. **OEmbed 回退**：对非 Ghost 站点使用 oEmbed 协议获取元数据

---

## 4. 推荐失效后的清理机制

### 4.1 主动删除时的通知

**场景：站点 A 删除对站点 B 的推荐**

`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:219-236`

```typescript
async deleteRecommendation(id: string) {
    // 1. 软删除（标记删除状态）
    existing.delete();
    await this.repository.save(existing);
    
    // 2. 更新 .well-known（该推荐不再出现在列表中）
    const recommendations = await this.#listRecommendations();
    await this.updateWellknown(recommendations);
    
    // 3. 发送 WebMention 通知站点 B
    // 根据 WebMention 规范：源不再包含链接时，目标应删除 mention
    this.sendMentionToRecommendation(existing);
}
```

### 4.2 被动检测与清理

站点 B 通过以下机制检测推荐是否失效：

#### 4.2.1 启动时重验证

`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:80-93`

```typescript
async init() {
    // 启动后延迟 15 秒到 3 分钟（随机化）
    if (!process.env.NODE_ENV?.startsWith('test') && process.env.NODE_ENV !== 'development') {
        setTimeout(() => {
            this.#updateIncomingRecommendations().catch(...);
        }, 15 * 1000 + Math.random() * 5 * 60 * 1000);
    }
}
```

**重验证逻辑**：`ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts:99-105`

```typescript
async #updateIncomingRecommendations() {
    // 包含已删除的记录，因为可能需要恢复或确认删除
    const filter = this.#getMentionFilter() + '+deleted:[true,false]';
    await this.#mentionsApi.refreshMentions({filter, limit: 100});
}
```

#### 4.2.2 WebMention 验证流程

`ghost/core/core/server/services/mentions/mentions-api.js:184-260`

```javascript
async #updateWebmention(mention, webmention) {
    const isNew = !mention;
    const wasDeleted = mention?.deleted ?? false;
    
    // 1. 检查目标 URL 是否存在于本站点
    const targetExists = await this.#routingService.pageExists(webmention.target);
    
    if (!targetExists) {
        if (mention) {
            // 目标不存在，标记删除
            mention.delete();
        }
    }
    
    // 2. 验证源 URL 是否真的包含目标链接
    if (targetExists) {
        try {
            const metadata = await this.#webmentionMetadata.fetch(webmention.source);
            // 更新元数据
        } catch (err) {
            if (mention) {
                // 源无法访问或验证失败，标记删除
                mention.delete();
            }
        }
        
        // 3. 验证源内容中是否真的有目标链接
        if (metadata?.body) {
            mention.verify(metadata.body, metadata.contentType);
        }
    }
    
    await this.#repository.save(mention);
}
```

**验证内容**：
- 源 URL 是否可访问
- 源内容中是否真的包含对目标的链接
- 目标 URL 在本站点是否仍然有效

### 4.3 数据库表结构

**推荐表（recommendations）**：`ghost/core/core/server/data/schema/schema.js:1105-1116`

```javascript
recommendations: {
    id: {type: 'string', maxlength: 24, primary: true},
    url: {type: 'string', maxlength: 2000},           // 被推荐的站点 URL
    title: {type: 'string', maxlength: 2000},
    excerpt: {type: 'string', maxlength: 2000},
    featured_image: {type: 'string', maxlength: 2000},
    favicon: {type: 'string', maxlength: 2000},
    description: {type: 'string', maxlength: 2000},
    one_click_subscribe: {type: 'boolean', defaultTo: false},
    created_at: {type: 'dateTime'},
    updated_at: {type: 'dateTime'}
}
```

**Mention 表（接收的推荐）**：`ghost/core/core/server/data/schema/schema.js:1063-1079`

```javascript
mentions: {
    id: {type: 'string', maxlength: 24, primary: true},
    source: {type: 'string', maxlength: 2000},        // 来源站点的 .well-known 地址
    source_title: {type: 'string', maxlength: 2000},
    source_site_title: {type: 'string', maxlength: 2000},
    source_excerpt: {type: 'string', maxlength: 2000},
    source_author: {type: 'string', maxlength: 2000},
    source_featured_image: {type: 'string', maxlength: 2000},
    source_favicon: {type: 'string', maxlength: 2000},
    target: {type: 'string', maxlength: 2000},        // 本站点被推荐的 URL
    resource_id: {type: 'string', maxlength: 24},
    resource_type: {type: 'string', maxlength: 50},
    created_at: {type: 'dateTime'},
    payload: {type: 'text', maxlength: 65535},
    deleted: {type: 'boolean', defaultTo: false},    // 软删除标记
    verified: {type: 'boolean', defaultTo: false}    // 是否已验证源包含目标
}
```

---

## 5. 跨实例延迟处理策略

### 5.1 异步任务队列

所有跨站点 HTTP 请求都通过异步任务执行：

`ghost/core/core/server/services/mentions/mention-sending-service.js:66-76`

```javascript
if (html || previousHtml) {
    await this.#jobService.addJob('sendWebmentions', async () => {
        await this.sendForHTMLResource({
            url: new URL(this.#getPostUrl(post)),
            html: html,
            previousHtml: previousHtml
        });
    });
}
```

**优势**：
- 不阻塞用户操作的响应
- 失败时可以重试

### 5.2 重试机制

**网络请求重试**：`ghost/core/core/server/services/mentions/mention-sending-service.js:91-107`

```javascript
const response = await this.#externalRequest.post(endpoint.href, {
    ...
    timeout: {
        request: 15000  // 15秒超时
    },
    retry: {
        limit: 3        // 最多重试 3 次
    }
});
```

### 5.3 启动延迟与随机化

为避免快速重启导致的重复请求和 thundering herd 问题：

| 服务 | 延迟范围 | 目的 |
|------|----------|------|
| 推荐元数据更新 | 2-7 分钟 | 避免启动时大量外部请求 |
| 传入推荐重验证 | 15秒-3分钟 | 给外部站点留出启动时间 |

### 5.4 错误隔离

**单个推荐失败不影响整体**：
`ghost/core/core/server/services/recommendations/service/recommendation-service.ts:72-83`

```typescript
async updateAllRecommendationsMetadata() {
    for (const recommendation of recommendations) {
        try {
            await this._updateRecommendationMetadata(recommendation);
            await this.repository.save(recommendation);
        } catch (e) {
            // 只记录错误，继续处理下一个
            logging.error('[Recommendations] Failed to save...', e);
        }
    }
}
```

### 5.5 最终一致性

由于 WebMention 是异步协议，系统采用**最终一致性**模型：

1. **推荐添加**：立即保存本地，异步发送通知
2. **状态同步**：依赖目标站点接收和处理 WebMention
3. **失效检测**：通过启动时重验证 + 定期刷新实现
4. **数据恢复**：即使错过某些 WebMention，启动时的重验证也能恢复正确状态

---

## 6. 完整时序图

### 6.1 推荐创建流程

```
站点 A (推荐方)                    站点 B (被推荐方)
    |                                   |
    | 1. 用户添加推荐                    |
    |    (RecommendationService.add)    |
    |                                   |
    | 2. 保存到 recommendations 表      |
    |                                   |
    | 3. 更新 /.well-known/...          |
    |                                   |
    | 4. 发现 B 的 webmention endpoint  |
    |    (MentionDiscoveryService)      |
    |                                   |
    | 5. POST webmention -------------> |
    |    source: A/.well-known/...      |
    |    target: B/                     |
    |                                   |
    |                                   | 6. 处理 webmention
    |                                   |    (MentionsAPI.processWebmention)
    |                                   |
    |                                   | 7. 保存到 mentions 表
    |                                   |    verified=true
    |                                   |
    |                                   | 8. 检查是否也推荐 A
    |                                   |    (recommendingBack)
    |                                   |
    | 9. 返回成功（无需等待 B 响应）     |
```

### 6.2 推荐删除流程

```
站点 A (推荐方)                    站点 B (被推荐方)
    |                                   |
    | 1. 用户删除推荐                    |
    |    (RecommendationService.delete) |
    |                                   |
    | 2. 从列表移除（软删除）            |
    |                                   |
    | 3. 更新 /.well-known/...          |
    |    (不再包含 B 的 URL)             |
    |                                   |
    | 4. 发送 webmention -------------> |
    |    (同添加时的 payload)            |
    |                                   |
    |                                   | 5. 验证源是否仍包含目标
    |                                   |    - 访问 A/.well-known/...
    |                                   |    - 发现 B 不在列表中
    |                                   |
    |                                   | 6. 标记 mention.deleted = true
    |                                   |
    |                                   | 7. 或启动时重验证发现失效
```

---

## 7. 关键组件清单

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| RecommendationService | `recommendations/service/recommendation-service.ts` | 推荐的 CRUD、元数据更新、WebMention 触发 |
| IncomingRecommendationService | `recommendations/service/incoming-recommendation-service.ts` | 处理传入推荐、互荐检测、启动重验证 |
| WellknownService | `recommendations/service/well-known-service.ts` | 管理 `/.well-known/recommendations.json` |
| RecommendationMetadataService | `recommendations/service/recommendation-metadata-service.ts` | 获取推荐站点的元数据（Ghost API + oEmbed） |
| MentionSendingService | `mentions/mention-sending-service.js` | 发送 WebMention、异步任务调度 |
| MentionDiscoveryService | `mentions/mention-discovery-service.js` | 发现目标站点的 WebMention endpoint |
| MentionsAPI | `mentions/mentions-api.js` | 处理接收的 WebMention、验证、删除逻辑 |

---

## 8. 设计要点总结

1. **基于标准协议**：使用 WebMention W3C 推荐标准，兼容非 Ghost 站点
2. **最终一致性**：不追求强一致，通过异步通知 + 定期重验证达成一致
3. **容错设计**：超时、重试、错误隔离、启动恢复
4. **随机化延迟**：避免 thundering herd 问题
5. **软删除**：保留历史记录，支持恢复和审计
6. **双向感知**：通过 `recommendingBack` 实现互荐关系检测
7. **元数据缓存**：本地缓存标题、图标等，定期刷新
