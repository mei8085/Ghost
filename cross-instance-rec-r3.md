# Ghost 互荐同步链路重试边界与最终一致性深度分析

## 1. 核心发现总结

经过深入代码审查，发现以下关键问题：

| 问题 | 影响 |
|------|------|
| 测试环境强制禁用重试 | 测试无法真实验证重试逻辑 |
| `refreshMentions` 无分页 + limit=100 | 超过 100 条推荐时，后续推荐永远无法重验证 |
| 无周期性调度 | 站点长时间不重启，推荐状态永久不一致 |
| 无持久化任务队列 | 一次 HTTP 失败 = 永久失败（依赖下次启动） |
| 三类请求使用相同重试配置 | POST WebMention 与 GET 元数据无差异策略 |

---

## 2. externalRequest 与 got 重试机制深度分析

### 2.1 externalRequest 默认配置

`ghost/core/core/server/lib/request-external.js:275-290`

```javascript
const gotOpts = {
    headers: {
        'user-agent': 'Ghost(https://github.com/TryGhost/Ghost)'
    },
    timeout: {
        request: 10000  // 默认 10 秒超时
    },
    hooks: {
        // 关键点：测试环境强制禁用重试
        init: process.env.NODE_ENV?.startsWith('test') ? [disableRetries] : [],
        beforeRequest: [errorIfInvalidUrl, errorIfHostnameResolvesToPrivateIp, installSafeDnsLookup],
        beforeRedirect: [errorIfHostnameResolvesToPrivateIp, installSafeDnsLookup]
    }
};

const externalRequest = got.extend(gotOpts);
```

**测试环境禁用逻辑**（第 205-214 行）：

```javascript
async function disableRetries(options) {
    // Force disable retries
    options.retry = {
        limit: 0,
        calculateDelay: () => 0
    };
    options.timeout = {
        request: 5000
    };
}
```

**关键结论**：
- **生产环境**：使用 got 默认重试策略
- **测试环境**：`NODE_ENV.startsWith('test')` 时，强制 `retry.limit = 0`，完全禁用重试
- 这意味着**测试无法验证重试逻辑**，重试行为存在"测试盲区"

### 2.2 got 默认重试策略（生产环境）

根据 `got` 库（v12.x）的默认配置，`retry.limit` 未显式设置时：

| 错误类型 | 是否重试 | 说明 |
|----------|----------|------|
| `ENOTFOUND`, `ECONNRESET`, `ETIMEDOUT`, `ECONNREFUSED` | ✅ 重试 | 网络层错误 |
| `EADDRINUSE`, `EAI_AGAIN` | ✅ 重试 | DNS 和地址错误 |
| `EPIPE`, `ECONNABORTED` | ❌ 不重试 | 连接中断 |
| HTTP 408 (Request Timeout) | ✅ 重试 | 服务端超时 |
| HTTP 413 (Payload Too Large) | ❌ 不重试 | 请求过大 |
| HTTP 429 (Too Many Requests) | ✅ 重试（需 `retry-after` 头） | 速率限制 |
| HTTP 500, 502, 503, 504 | ✅ 重试 | 服务端错误 |
| HTTP 4xx（除 408、429 外） | ❌ 不重试 | 客户端错误 |
| HTTP 2xx, 3xx | ❌ 不重试 | 成功响应 |

**默认重试次数**：`got` 默认 `retry.limit = 2`（共 3 次尝试：1 次初始 + 2 次重试）

**默认重试间隔**：
- 指数退避：`Math.pow(2, attempt) * 1000` ms
- 第 1 次重试：1 秒后
- 第 2 次重试：2 秒后

---

## 3. 三类请求的重试配置对比

### 3.1 对比矩阵

| 请求类型 | 位置 | timeout | 显式 retry.limit | 方法 | 重试条件 | 失败后路径 |
|----------|------|---------|------------------|------|----------|------------|
| **POST WebMention** | `mention-sending-service.js:91-107` | 15s | ✅ `3` | POST | got 默认 | 仅记录日志，无队列持久化 |
| **GET 元数据** | `recommendation-metadata-service.ts:51-64` | 15s | ✅ `3` | GET | got 默认 | 启动时再次尝试 |
| **Endpoint 发现** | `mention-discovery-service.js:17-28` | 15s | ✅ `3` | GET | got 默认 | WebMention 发送被跳过 |

### 3.2 详细代码分析

#### 类 1：POST WebMention

`ghost/core/core/server/services/mentions/mention-sending-service.js:87-117`

```javascript
async send({source, target, endpoint}) {
    const response = await this.#externalRequest.post(endpoint.href, {
        form: {
            source: source.href,
            target: target.href,
            source_is_ghost: true
        },
        timeout: {
            request: 15000  // 15秒超时
        },
        retry: {
            limit: 3  // 最多重试 3 次（共 4 次尝试）
        }
    });
    return response;
}
```

**批量发送容错**（第 165-177 行）：

```javascript
async sendAll({url, links}) {
    for (const target of links) {
        const endpoint = await this.#discoveryService.getEndpoint(target);
        if (endpoint) {
            try {
                await this.send({source: url, target, endpoint});
            } catch (e) {
                // 仅记录日志，不重试整个批次
                logging.error('[Webmention] Failed sending via ' + endpoint.href + ': ' + e.message);
            }
        }
    }
}
```

**重试触发场景（会重试）**：
- 目标站点临时网络超时（`ETIMEDOUT`）
- 目标站点 DNS 临时解析失败（`ENOTFOUND`）
- 目标站点返回 500/502/503/504
- 目标站点返回 408 Request Timeout
- 目标站点返回 429 Too Many Requests（有 retry-after 头）

**不重试场景（直接失败）**：
- 目标站点返回 400 Bad Request
- 目标站点返回 401 Unauthorized
- 目标站点返回 403 Forbidden
- 目标站点返回 404 Not Found
- 目标站点返回 406 Not Acceptable
- 目标站点返回 409 Conflict
- 目标站点返回 410 Gone

**失败后路径**：
1. 日志记录
2. **无持久化重试队列**
3. 依赖：
   - 站点 B 重启时的 `refreshMentions`
   - 或站点 A 再次编辑推荐触发重新发送

#### 类 2：GET 推荐元数据

`ghost/core/core/server/services/recommendations/service/recommendation-metadata-service.ts:45-66`

```typescript
async #fetchJSON(url: URL, options) {
    const response = await this.#externalRequest.get(url.toString(), {
        responseType: 'json',
        timeout: {
            request: 15000
        },
        retry: {
            limit: 3
        },
        ...options
    });
    return JSON.parse(response.body);
}
```

**重试触发场景**：
- 目标站点 `members/api/site` 端点临时不可用
- 网络抖动、DNS 问题、5xx 响应

**失败后路径**：
1. 捕获异常，日志记录
2. 回退到 oEmbed（非 Ghost 站点）
3. 下次启动时 `updateAllRecommendationsMetadata` 会再次尝试

#### 类 3：GET Endpoint 发现

`ghost/core/core/server/services/mentions/mention-discovery-service.js:15-74`

```javascript
async getEndpoint(target) {
    try {
        const response = await this.#externalRequest(target.href, {
            followRedirect: true,
            timeout: {
                request: 15000
            },
            retry: {
                limit: 3
            }
        });

        // 1. 优先检查 HTTP Link Header
        const endpoint = parseWebmentionEndpoint(response.headers.link, target);
        if (endpoint) {
            return endpoint;
        }

        // 2. 解析 HTML 查找 <link rel="webmention">
        const $ = cheerio.load(response.body);
        const webmentionEndpoint = $('link[rel="webmention"]').attr('href') ||
                                  $('a[rel="webmention"]').attr('href');
        // ...
    } catch (e) {
        // 发现失败，返回 null
        return null;
    }
}
```

**重试触发场景**：
- 目标站点主页临时不可访问
- 网络问题、DNS 问题

**失败后路径**：
1. 返回 `null`
2. **该目标站点的 WebMention 发送被完全跳过**
3. 依赖下次推荐变更触发重新发现

### 3.3 三类请求一致性评估

| 维度 | 一致性 | 说明 |
|------|--------|------|
| timeout | ✅ 一致 | 都是 15 秒 |
| retry.limit | ✅ 一致 | 都是 3 |
| 重试条件 | ✅ 一致 | 都使用 got 默认策略 |
| 失败后路径 | ❌ 不一致 | 见下表 |

**失败后路径差异**：

| 请求类型 | 失败后行为 | 是否有"兜底"机制 |
|----------|------------|------------------|
| POST WebMention | 日志记录，无重试 | ✅ 有：站点 B 启动时 refreshMentions |
| GET 元数据 | 日志记录，oEmbed 回退 | ✅ 有：下次启动时刷新 |
| GET Endpoint 发现 | 返回 null，WebMention 被跳过 | ⚠️ 部分有：下次推荐变更时重新发现 |

**关键差异**：Endpoint 发现失败意味着**后续的 WebMention 发送完全不会尝试**，而不仅仅是 WebMention 本身失败。

---

## 4. `refreshMentions` 的 `limit=100` 限制深度分析

### 4.1 调用链

`incoming-recommendation-service.ts:99-105`

```typescript
async #updateIncomingRecommendations() {
    const filter = this.#getMentionFilter() + '+deleted:[true,false]';
    // 硬编码 limit: 100
    await this.#mentionsApi.refreshMentions({filter, limit: 100});
}
```

`mentions-api.js:173-182`

```javascript
async refreshMentions(options) {
    // 直接调用 getAll，没有分页循环
    const mentions = await this.#repository.getAll(options);

    for (const mention of mentions) {
        await this.#updateWebmention(mention, {
            source: mention.source,
            target: mention.target
        });
    }
}
```

`bookshelf-mention-repository.js:93-97`

```javascript
async getAll(options) {
    // Bookshelf findAll，limit 直接应用
    const models = await this.#MentionModel.findAll(options);
    return await Promise.all(models.map(model => this.#modelToMention(model)));
}
```

### 4.2 问题分析

**核心问题**：`refreshMentions` 方法没有分页循环逻辑，`limit=100` 是一次性限制，不是"每页 100 条"。

**数据流**：

```
数据库中 150 条推荐记录
         │
         ▼
    getAll({limit: 100})
         │
         ▼
    返回前 100 条（按默认排序，通常是 created_at ASC 或 DESC）
         │
         ▼
    只处理前 100 条
         │
         ▼
    第 101-150 条：永远不会被重验证！
```

### 4.3 排序不确定性

Bookshelf `findAll` 的默认排序取决于：

1. 模型定义的 `order` 属性
2. 数据库的自然顺序（通常是插入顺序）

如果默认是 `created_at ASC`（最早的在前）：
- 早期的 100 个推荐会被重验证
- 后来添加的推荐（超过 100 个后）**永远不会被重验证**

如果默认是 `created_at DESC`（最新的在前）：
- 最近的 100 个推荐会被重验证
- 早期的推荐**永远不会被重验证**（包括可能已失效的）

### 4.4 实际影响场景

**场景 A：站点被大量推荐**

```
时间线：
T1: 站点 C1 推荐 B → 记录 #1
T2: 站点 C2 推荐 B → 记录 #2
...
T100: 站点 C100 推荐 B → 记录 #100
T101: 站点 C101 推荐 B → 记录 #101
T102: 站点 C102 推荐 B → 记录 #102

然后：
- 站点 C1 删除对 B 的推荐（B 收到 WebMention 并标记删除）
- 站点 C101 删除对 B 的推荐（WebMention 发送失败或丢失）

站点 B 重启：
- refreshMentions(limit=100) 处理记录 #1-#100
- 记录 #1：验证发现 C1 已删除 → 确认删除状态 ✓
- 记录 #101-#102：未被处理 → 状态不一致 ✗
```

**场景 B：WebMention 丢失**

```
时间线：
T1: 站点 C 推荐 B → B 成功接收，记录 verified=true
T2: 站点 C 删除推荐 → WebMention 发送失败（网络分区）
T3: 站点 C 重启 → 重新发送失败（仍在分区中）
T4: 网络恢复

站点 B 重启：
- 如果 B 已有 >100 条推荐，且 C 的推荐不在前 100
- 则 B 永远不知道 C 已删除推荐！
```

### 4.5 与 `maxLimit=100` 全局配置的关系

`ghost/core/test/unit/shared/max-limit-cap.test.js:21`

```javascript
assert.equal(maxLimitCap.limitConfig.maxLimit, 100);
```

这是 API 层的限制，防止 API 一次返回过多数据。但 `refreshMentions` 是**内部服务调用**，不应受此限制，或应实现分页处理。

---

## 5. 仅启动触发对最终一致性的影响

### 5.1 触发时机分析

**传入推荐重验证触发点**：
`incoming-recommendation-service.ts:80-93`

```typescript
async init() {
    // 仅在生产环境触发
    if (!process.env.NODE_ENV?.startsWith('test') && 
        process.env.NODE_ENV !== 'development') {
        setTimeout(() => {
            // 延迟 15秒-3分钟
            this.#updateIncomingRecommendations().catch(...);
        }, 15 * 1000 + Math.random() * 5 * 60 * 1000);
    }
}
```

**开发环境跳过**：`process.env.NODE_ENV === 'development'` 时不会触发

### 5.2 无周期性调度

在 Ghost 代码库中搜索 `refreshMentions` 调用点：

```
Found 2 files:
1. ghost/core/core/server/services/recommendations/service/incoming-recommendation-service.ts
   - 仅在 init() 中调用

2. ghost/core/core/server/services/mentions/mentions-api.js
   - 方法定义
```

**结论**：没有任何定时任务（cron job、scheduleJob 等）定期调用 `refreshMentions`。

### 5.3 与其他调度任务对比

Ghost 中其他使用调度的服务：

| 服务 | 调度方式 | 文件 |
|------|----------|------|
| 帖子调度发布 | `scheduleJob` | `post-scheduling/post-scheduler-service.js` |
| Outbox 作业 | `scheduleJob` | `outbox/jobs/index.js` |
| Members 清理 | `scheduleJob` | `members/jobs/index.js` |
| **推荐重验证** | ❌ 无 | 仅启动触发 |

### 5.4 最终一致性影响矩阵

| 场景 | 当前行为 | 一致性达成时间 | 风险 |
|------|----------|----------------|------|
| 推荐添加 + WebMention 成功 | 即时同步 | 秒级 | 低 |
| 推荐添加 + WebMention 失败 | 等待下次启动 | 不确定（可能永远不） | 中 |
| 推荐删除 + WebMention 成功 | 即时验证删除 | 秒级 | 低 |
| 推荐删除 + WebMention 丢失 | 等待下次启动 + 在 limit=100 内 | 不确定 | **高** |
| 源站点变更域名 | 等待下次启动 + 在 limit=100 内 | 不确定 | **高** |
| 源站点永久下线 | 等待下次启动 + 在 limit=100 内 | 不确定 | **高** |
| 长时间不重启的站点（如数月） | 永不重验证 | 永远不一致 | **极高** |

### 5.5 现实场景分析

**场景：生产环境长期运行**

```
部署模式：Kubernetes/容器化，滚动更新
平均重启周期：每月 1-2 次（安全补丁、版本更新）
问题：
- 期间产生的推荐状态变更，最多可能延迟 30 天才能同步
- 如果有 >100 条推荐，部分永远不同步
```

**场景：高可用多实例部署**

```
部署模式：多实例共享数据库
实例 A 处理推荐变更 → 发送 WebMention
实例 B 从未重启 → 从未运行 refreshMentions
问题：
- 实例 B 查询 incoming recommendations 时可能看到不一致的状态
```

---

## 6. 可能的漏同步场景清单

### 6.1 场景一：推荐数量超过 100 条

**触发条件**：
- 站点 B 被 101+ 个站点推荐
- 其中某些推荐的 WebMention 丢失

**漏同步内容**：
- 第 101 条及以后的推荐永远不会被重验证
- 如果这些推荐被删除，B 永远不知道

**代码证据**：
- `incoming-recommendation-service.ts:104`: `limit: 100`
- `mentions-api.js:173-182`: 无分页循环

**影响范围**：高流量站点、被大量推荐的热门站点

### 6.2 场景二：WebMention 发送失败 + 源站点不再变更

**触发条件**：
- 站点 C 删除对 B 的推荐
- WebMention 发送时网络分区、超时或 4xx 错误
- 站点 C 之后不再有任何推荐变更（不会重新触发 WebMention 发送）

**漏同步内容**：
- B 一直认为 C 还在推荐自己
- 没有任何机制会重试这个"丢失"的 WebMention

**代码证据**：
- `mention-sending-service.js:170-175`: 仅记录日志，不持久化重试
- 无 `retry-after` 或持久化队列

**影响范围**：所有站点，特别是网络不稳定的环境

### 6.3 场景三：Endpoint 发现失败

**触发条件**：
- 站点 C 推荐 B，首次发送时 C 的主页暂时不可用
- Endpoint 发现失败（返回 null）
- WebMention 完全不发送

**漏同步内容**：
- B 从未收到任何 WebMention
- 只有当 C 再次编辑推荐时，才会重新尝试发现和发送

**代码证据**：
- `mention-discovery-service.js:15-74`: 失败返回 null
- `mention-sending-service.js:167`: `if (endpoint)` 才发送

**影响范围**：目标站点短暂不可用的情况

### 6.4 场景四：长时间不重启的站点

**触发条件**：
- 站点 B 运行 6 个月不重启（稳定运行）
- 期间有推荐被添加或删除
- WebMention 偶尔丢失

**漏同步内容**：
- 所有丢失的 WebMention 永远不会被补偿
- 6 个月后重启才能同步，但可能已经 >100 条

**代码证据**：
- 无 cron job 或定期调度
- 仅 `init()` 中的 `setTimeout`

**影响范围**：稳定运行的生产环境

### 6.5 场景五：4xx 响应不重试

**触发条件**：
- 站点 B 临时返回 400/401/403/404（可能是配置问题、临时限流等）
- 站点 C 发送 WebMention 收到 4xx
- got 不会重试 4xx

**漏同步内容**：
- WebMention 被丢弃
- 没有重试机制

**具体状态码**：
- 400 Bad Request：可能是 B 的临时 bug
- 401 Unauthorized：可能是临时认证问题
- 403 Forbidden：可能是临时 WAF 规则
- 404 Not Found：可能是 B 迁移端点路径

**代码证据**：
- `got` 默认策略：4xx 不重试
- 无自定义 `retry.statusCodes` 配置

**影响范围**：目标站点临时故障但可恢复的情况

### 6.6 场景六：开发环境完全跳过

**触发条件**：
- `NODE_ENV === 'development'`
- 本地开发或测试环境

**漏同步内容**：
- `refreshMentions` 完全不会执行
- 开发环境无法验证最终一致性逻辑

**代码证据**：
- `incoming-recommendation-service.ts:85`: `process.env.NODE_ENV !== 'development'`

**影响范围**：开发调试、本地测试

### 6.7 场景七：多实例部署的实例差异

**触发条件**：
- 多实例共享数据库
- 实例 A 刚重启（运行过 refreshMentions）
- 实例 B 已运行数月（未运行过 refreshMentions）

**漏同步内容**：
- 实例 A 和实例 B 返回的 incoming recommendations 可能不一致
- 取决于哪个实例处理 API 请求

**影响范围**：高可用部署、负载均衡环境

---

## 7. 代码层面的确认证据

### 7.1 测试确认 `limit=100`

`incoming-recommendation-service.test.ts:54-58`

```typescript
sinon.assert.calledWith(refreshMentions, {
    filter: `source:~$'/.well-known/recommendations.json'+deleted:[true,false]`,
    limit: 100  // 测试确认硬编码
});
```

### 7.2 测试确认忽略错误

`incoming-recommendation-service.test.ts:64-77`

```typescript
it('ignores errors when update incoming recommendations on boot', async function () {
    refreshMentions.rejects(new Error('test'));
    await service.init();
    clock.tick(1000 * 60 * 60 * 24);
    sinon.assert.calledOnce(refreshMentions);  // 只调用一次，不重试
});
```

### 7.3 测试确认开发环境不触发

测试中显式设置 `process.env.NODE_ENV = 'nottesting'` 才能触发：

`incoming-recommendation-service.test.ts:47-62`

```typescript
it('should update incoming recommendations on boot', async function () {
    const saved = process.env.NODE_ENV;
    try {
        process.env.NODE_ENV = 'nottesting';  // 不是 'development' 也不是 'test*'
        await service.init();
        // ...
    } finally {
        process.env.NODE_ENV = saved;
    }
});
```

---

## 8. 总结与建议

### 8.1 问题严重性排序

| 问题 | 严重性 | 影响范围 |
|------|--------|----------|
| `refreshMentions` 无分页 + limit=100 | 🔴 高 | 超过 100 条推荐的站点 |
| 无周期性调度 | 🔴 高 | 所有长时间运行的站点 |
| 4xx 不重试 | 🟡 中 | 目标站点临时故障 |
| Endpoint 发现失败导致 WebMention 跳过 | 🟡 中 | 目标站点短暂不可用 |
| 开发环境跳过 | 🟢 低 | 仅开发环境 |
| 测试禁用重试 | 🟢 低 | 仅测试盲区 |

### 8.2 建议改进方向

1. **分页处理**：`refreshMentions` 应实现分页循环，处理所有记录
2. **周期性调度**：添加每日/每周的 `refreshMentions` 定时任务
3. **持久化重试队列**：WebMention 发送失败应入队，后续重试
4. **自定义重试策略**：考虑对某些 4xx（如 408、429）进行重试
5. **Endpoint 发现重试**：发现失败应延迟重试，而不是永久跳过

---

## 9. 附录：关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| externalRequest 定义 | `core/server/lib/request-external.js` | 275-290 |
| 测试禁用重试 | `core/server/lib/request-external.js` | 205-214 |
| POST WebMention 配置 | `core/server/services/mentions/mention-sending-service.js` | 87-117 |
| GET 元数据配置 | `core/server/services/recommendations/service/recommendation-metadata-service.ts` | 45-66 |
| Endpoint 发现配置 | `core/server/services/mentions/mention-discovery-service.js` | 15-28 |
| refreshMentions 定义 | `core/server/services/mentions/mentions-api.js` | 173-182 |
| refreshMentions 调用（limit=100） | `core/server/services/recommendations/service/incoming-recommendation-service.ts` | 99-105 |
| 仅启动触发 | `core/server/services/recommendations/service/incoming-recommendation-service.ts` | 80-93 |
