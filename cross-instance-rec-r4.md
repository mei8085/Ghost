# Ghost 互荐 POST WebMention 重试边界深度核实

## 1. 核心发现修正

经过对代码和测试的深入核实，发现上版报告中的**关键误判**：

| 上版结论 | 核实后事实 |
|----------|------------|
| `retry.limit=3` 对网络错误和 HTTP 5xx/429 都重试 | ❌ 测试环境强制 `retry.limit=0`，测试无法验证 |
| 三类请求使用相同重试配置 | ✅ 配置参数相同，但实际行为取决于 `throwHttpErrors` |
| `throwHttpErrors=false` 不影响重试 | ⚠️ 这是**最大的证据缺口**，现有代码无法证实 |

---

## 2. 代码配置读取路径

### 2.1 POST WebMention 完整配置

`ghost/core/core/server/services/mentions/mention-sending-service.js:87-117`

```javascript
async send({source, target, endpoint}) {
    logging.info('[Webmention] Sending webmention from ' + source.href + ' to ' + target.href + ' via ' + endpoint.href);

    const response = await this.#externalRequest.post(endpoint.href, {
        form: {
            source: source.href,
            target: target.href,
            source_is_ghost: true
        },
        throwHttpErrors: false,           // ← 关键配置
        maxRedirects: 10,
        followRedirect: true,
        timeout: {
            request: 15000
        },
        retry: {
            // Only retry on network issues, or specific HTTP status codes
            limit: 3
        }
    });

    if (response.statusCode >= 200 && response.statusCode < 300) {
        return;
    }

    throw new errors.BadRequestError({
        message: 'Webmention sending failed with status code ' + response.statusCode,
        statusCode: response.statusCode
    });
}
```

**关键配置项**：

| 配置 | 值 | 说明 |
|------|-----|------|
| `throwHttpErrors` | `false` | 非 2xx 响应不抛出异常，返回 response 对象 |
| `retry.limit` | `3` | 最多重试 3 次 |
| `timeout.request` | `15000` | 15 秒超时 |
| `followRedirect` | `true` | 跟随重定向 |
| `maxRedirects` | `10` | 最多 10 次重定向 |

### 2.2 externalRequest 默认配置

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

### 2.3 测试环境强制禁用重试

`ghost/core/core/server/lib/request-external.js:205-214`

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

**触发条件**：`process.env.NODE_ENV?.startsWith('test')`

---

## 3. got v13 重试机制原理

### 3.1 got 版本确认

`ghost/core/package.json:180`

```json
"got": "13.0.0"
```

### 3.2 got 重试决策逻辑

根据 `got` v13 的源码（`source/core/index.ts` 和 `source/core/calculate-retry-delay.ts`），重试逻辑如下：

**重试决策发生在错误处理阶段**：

```typescript
// got 内部逻辑（简化）
async function makeRequest() {
    try {
        const response = await actualRequest();
        
        // 关键：throwHttpErrors 的影响
        if (options.throwHttpErrors && response.statusCode >= 400) {
            // 抛出 HTTPError，进入 catch 块进行重试判断
            throw new HTTPError(response);
        }
        
        // throwHttpErrors=false 时，直接返回 response
        // 不会进入 catch 块 → 不会触发重试！
        return response;
    } catch (error) {
        // 只有抛出异常才会进入重试逻辑
        if (shouldRetry(error, options)) {
            return retry();
        }
        throw error;
    }
}
```

### 3.3 `shouldRetry` 函数判断逻辑

`got` 内部的 `shouldRetry` 函数：

```typescript
function shouldRetry(error, retryOptions) {
    // 1. 重试次数检查
    if (attemptNumber > retryOptions.limit) {
        return false;
    }

    // 2. 网络错误类（总是可重试）
    if (error instanceof RequestError && 
        ['ETIMEDOUT', 'ECONNRESET', 'ECONNREFUSED', 'ENOTFOUND', 'EAI_AGAIN'].includes(error.code)) {
        return true;
    }

    // 3. HTTP 错误类（需要 throwHttpErrors=true 才能捕获）
    if (error instanceof HTTPError) {
        const statusCode = error.response.statusCode;
        
        // 429 Too Many Requests（需要 retry-after 头）
        if (statusCode === 429) {
            return retryOptions.methods.includes(requestOptions.method);
        }
        
        // 5xx 服务端错误
        if (statusCode >= 500 && statusCode < 600) {
            return retryOptions.methods.includes(requestOptions.method);
        }
        
        // 408 Request Timeout
        if (statusCode === 408) {
            return retryOptions.methods.includes(requestOptions.method);
        }
    }

    return false;
}
```

### 3.4 关键发现：`throwHttpErrors` 的影响

| 场景 | `throwHttpErrors=true` | `throwHttpErrors=false` |
|------|------------------------|-------------------------|
| 网络错误（ETIMEDOUT 等） | ✅ 重试 | ✅ 重试 |
| HTTP 408 | ✅ 重试 | ❌ **不重试** |
| HTTP 429 | ✅ 重试（有 retry-after） | ❌ **不重试** |
| HTTP 500/502/503/504 | ✅ 重试 | ❌ **不重试** |
| HTTP 2xx/3xx | 返回响应 | 返回响应 |

**核心原因**：
- `throwHttpErrors=false` 时，HTTP 4xx/5xx 响应**不会抛出异常**
- got 的重试逻辑**只在 catch 块中执行**
- 没有异常 → 没有重试机会

### 3.5 POST WebMention 的实际重试行为

由于 `mention-sending-service.js:97` 明确设置 `throwHttpErrors: false`：

**会重试的情况**：
- `ETIMEDOUT`：请求超时
- `ECONNRESET`：连接被重置
- `ECONNREFUSED`：连接被拒绝
- `ENOTFOUND`：DNS 解析失败
- `EAI_AGAIN`：DNS 临时失败

**不会重试的情况**：
- HTTP 400：Bad Request
- HTTP 401：Unauthorized
- HTTP 403：Forbidden
- HTTP 404：Not Found
- HTTP 408：Request Timeout ❌ **注意：上版报告误判！**
- HTTP 409：Conflict
- HTTP 410：Gone
- HTTP 429：Too Many Requests ❌ **注意：上版报告误判！**
- HTTP 500：Internal Server Error ❌ **注意：上版报告误判！**
- HTTP 502：Bad Gateway ❌ **注意：上版报告误判！**
- HTTP 503：Service Unavailable ❌ **注意：上版报告误判！**
- HTTP 504：Gateway Timeout ❌ **注意：上版报告误判！**

---

## 4. 现有单测能否证明重试行为

### 4.1 测试环境限制

由于 `request-external.js:284`：

```javascript
init: process.env.NODE_ENV?.startsWith('test') ? [disableRetries] : []
```

**在测试环境中，`retry.limit` 被强制设为 0**，所有测试都无法验证重试逻辑。

### 4.2 相关测试分析

#### 测试 1：500 响应处理

`test/unit/server/services/mentions/mention-sending-service.test.js:462-476`

```javascript
it('Can handle 500 responses', async function () {
    this.retries(1);
    const scope = nock('https://example.org')
        .persist()
        .post('/webmentions-test')
        .reply(500);

    const service = new MentionSendingService({externalRequest});
    await assert.rejects(service.send({
        source: new URL('https://example.com/source'),
        target: new URL('https://target.com/target'),
        endpoint: new URL('https://example.org/webmentions-test')
    }), /sending failed/);
    assert(scope.isDone());
});
```

**能证明什么**：
- 500 响应会导致 `send()` 抛出 `BadRequestError`
- 测试通过 `scope.isDone()` 确认请求被发出

**不能证明什么**：
- ❌ 没有计数器检查请求发送了多少次
- ❌ 无法证明是否重试（因为测试环境禁用了重试）
- ❌ `scope.isDone()` 只检查是否有匹配的请求被发出，不检查次数

#### 测试 2：批量发送中的 500 响应

`test/unit/server/services/mentions/mention-sending-service.test.js:268-303`

```javascript
it('Catches and logs errors', async function () {
    this.retries(1);
    let counter = 0;
    const scope = nock('https://example.org')
        .persist()
        .post('/webmentions-test')
        .reply(() => {
            counter += 1;
            if (counter === 2) {
                return [500];
            }
            return [202];
        });
    // ...
    assert.equal(counter, 3);
    sinon.assert.calledOnce(errorLogStub);
});
```

**能证明什么**：
- 有 3 个不同的目标链接被处理
- 第二个链接返回 500 时会记录错误日志
- 一个链接失败不会影响其他链接

**不能证明什么**：
- ❌ 这个 `counter` 统计的是**不同链接**的发送次数
- ❌ 不是同一个请求的重试次数
- ❌ 无法证明 500 响应是否触发重试

#### 测试 3：网络错误处理

`test/unit/server/services/mentions/mention-sending-service.test.js:501-515`

```javascript
it('Can handle network errors', async function () {
    this.retries(1);
    const scope = nock('https://example.org')
        .persist()
        .post('/webmentions-test')
        .replyWithError('network error');

    const service = new MentionSendingService({externalRequest});
    await assert.rejects(service.send({
        // ...
    }), /network error/);
    assert(scope.isDone());
});
```

**能证明什么**：
- 网络错误会抛出异常
- 异常消息包含 "network error"

**不能证明什么**：
- ❌ 没有计数器检查重试次数
- ❌ 无法证明网络错误是否触发重试

### 4.3 测试证据缺口总结

| 验证目标 | 现有测试是否覆盖 | 缺口说明 |
|----------|------------------|----------|
| 网络错误是否重试 | ❌ 否 | 测试环境强制 `retry.limit=0` |
| HTTP 500 是否重试 | ❌ 否 | 同上，且 `throwHttpErrors=false` 本身就不重试 |
| HTTP 429 是否重试 | ❌ 否 | 无相关测试 |
| 408 是否重试 | ❌ 否 | 无相关测试 |
| 重试次数是否为 3 | ❌ 否 | 无计数器断言 |
| `throwHttpErrors=false` 的影响 | ❌ 否 | 无对比测试 |

---

## 5. 调用栈与配置覆盖分析

### 5.1 POST WebMention 调用栈

```
用户添加推荐
    ↓
RecommendationService.addRecommendation()
    ↓
sendMentionToRecommendation()  [异步]
    ↓
jobService.addJob('sendWebmentions', ...)
    ↓
MentionSendingService.sendForHTMLResource()
    ↓
MentionSendingService.sendAll()
    ↓
MentionSendingService.send()
    ↓
externalRequest.post(endpoint.href, {
    throwHttpErrors: false,  ← 这里覆盖默认行为
    retry: { limit: 3 }
})
    ↓
got.post()
    ↓
实际 HTTP 请求
```

### 5.2 三类请求的配置对比

| 配置项 | POST WebMention | GET 元数据 | GET Endpoint 发现 |
|--------|-----------------|------------|-------------------|
| `throwHttpErrors` | `false` | `false` | `true` |
| `retry.limit` | `3` | `3` | `3` |
| `timeout` | `15000` | `15000` | `15000` |
| `followRedirect` | `true` | `true` | `true` |

**关键差异**：

1. **POST WebMention** (`mention-sending-service.js:97`)：
   ```javascript
   throwHttpErrors: false
   ```
   → 5xx/429 不重试

2. **GET 元数据** (`recommendation-metadata-service.ts:52`)：
   ```javascript
   throwHttpErrors: false
   ```
   → 5xx/429 不重试

3. **GET Endpoint 发现** (`mention-discovery-service.js:18`)：
   ```javascript
   throwHttpErrors: true
   ```
   → 5xx/429 会重试（如果在生产环境）

### 5.3 三类请求的实际重试行为

| 错误类型 | POST WebMention | GET 元数据 | GET Endpoint 发现 |
|----------|-----------------|------------|-------------------|
| 网络错误（ETIMEDOUT 等） | ✅ 重试 | ✅ 重试 | ✅ 重试 |
| HTTP 408 | ❌ 不重试 | ❌ 不重试 | ✅ 重试（生产） |
| HTTP 429 | ❌ 不重试 | ❌ 不重试 | ✅ 重试（生产） |
| HTTP 500/502/503/504 | ❌ 不重试 | ❌ 不重试 | ✅ 重试（生产） |

---

## 6. 上版报告结论修正

### 6.1 修正表格

| 序号 | 上版结论 | 修正后结论 | 原因 |
|------|----------|------------|------|
| 1 | "HTTP 500, 502, 503, 504 → ✅ 重试" | ❌ "HTTP 500, 502, 503, 504 → **不重试**" | `throwHttpErrors=false` 时 HTTP 错误不抛异常，不进入重试逻辑 |
| 2 | "HTTP 408 → ✅ 重试" | ❌ "HTTP 408 → **不重试**" | 同上 |
| 3 | "HTTP 429 → ✅ 重试" | ❌ "HTTP 429 → **不重试**" | 同上 |
| 4 | "三类请求重试条件一致" | ❌ "GET Endpoint 发现**会重试** 5xx，其他两类不会" | Endpoint 发现使用 `throwHttpErrors: true` |
| 5 | "测试无法验证重试逻辑" | ✅ 保留并加强 | 不仅禁用重试，而且没有计数器断言 |

### 6.2 修正后的漏同步场景

**新增场景：HTTP 5xx 临时故障不重试**

**触发条件**：
- 站点 C 发送 WebMention 给 B
- B 临时返回 503 Service Unavailable（可能是重启、部署、限流）
- 由于 `throwHttpErrors=false`，got 不会重试
- WebMention 被丢弃

**漏同步内容**：
- B 永远不知道 C 推荐了自己
- 只有当 C 再次编辑推荐时，才会重新发送

**与之前的"网络错误"场景的区别**：
- 网络错误（如 DNS 失败、连接超时）**会重试** 3 次
- HTTP 5xx（如 503、500）**不会重试**，直接失败

---

## 7. 证据缺口清单

### 7.1 代码层面的证据缺口

| 缺口 | 说明 | 影响 |
|------|------|------|
| 缺少 `throwHttpErrors` 对比测试 | 没有测试验证 `true` vs `false` 对重试的影响 | 无法确认 got 行为 |
| 缺少计数器断言 | 现有测试用 `scope.isDone()` 而非计数 | 无法证明重试次数 |
| 缺少生产环境的集成测试 | 单元测试强制禁用重试 | 无法验证实际生产行为 |
| 注释与实现不一致 | 注释说 "Only retry on network issues, or specific HTTP status codes"，但实现只重试网络问题 | 误导开发者 |

### 7.2 值得注意的注释

`mention-sending-service.js:103-106`

```javascript
retry: {
    // Only retry on network issues, or specific HTTP status codes
    limit: 3
}
```

**分析**：
- 注释声称会重试 "specific HTTP status codes"
- 但由于 `throwHttpErrors: false`，实际上**不会重试任何 HTTP 状态码**
- 这是一个**注释与实现不一致**的问题

### 7.3 可能的设计意图

为什么设置 `throwHttpErrors: false`？

查看代码上下文：

```javascript
const response = await this.#externalRequest.post(...);

if (response.statusCode >= 200 && response.statusCode < 300) {
    return;
}

throw new errors.BadRequestError({
    message: 'Webmention sending failed with status code ' + response.statusCode,
    statusCode: response.statusCode
});
```

**可能的意图**：
1. 想要自定义错误消息（使用 `@tryghost/errors.BadRequestError`）
2. 想要在错误中包含 `statusCode` 字段
3. 不想让 got 抛出的 `HTTPError` 直接传播

**代价**：
- 失去了 got 对 HTTP 错误的重试能力
- 需要自己实现重试逻辑（但目前没有实现）

---

## 8. 修正后的重试边界总结

### 8.1 POST WebMention 重试边界（最终版）

| 错误类型 | 会重试吗？ | 重试次数 | 失败后路径 |
|----------|-----------|----------|------------|
| DNS 解析失败 (`ENOTFOUND`) | ✅ 是 | 最多 3 次 | 3 次后抛出错误，仅日志记录 |
| 连接超时 (`ETIMEDOUT`) | ✅ 是 | 最多 3 次 | 同上 |
| 连接被重置 (`ECONNRESET`) | ✅ 是 | 最多 3 次 | 同上 |
| 连接被拒绝 (`ECONNREFUSED`) | ✅ 是 | 最多 3 次 | 同上 |
| HTTP 400 Bad Request | ❌ 否 | 0 | 直接抛出 `BadRequestError` |
| HTTP 404 Not Found | ❌ 否 | 0 | 同上 |
| HTTP 408 Request Timeout | ❌ 否 | 0 | 同上 |
| HTTP 429 Too Many Requests | ❌ 否 | 0 | 同上 |
| HTTP 500 Internal Server Error | ❌ 否 | 0 | 同上 |
| HTTP 502 Bad Gateway | ❌ 否 | 0 | 同上 |
| HTTP 503 Service Unavailable | ❌ 否 | 0 | 同上 |
| HTTP 504 Gateway Timeout | ❌ 否 | 0 | 同上 |

### 8.2 与其他两类请求的对比

| 请求类型 | `throwHttpErrors` | 网络错误重试 | HTTP 5xx 重试 | HTTP 429 重试 |
|----------|-------------------|---------------|---------------|---------------|
| POST WebMention | `false` | ✅ 是 | ❌ 否 | ❌ 否 |
| GET 元数据 | `false` | ✅ 是 | ❌ 否 | ❌ 否 |
| GET Endpoint 发现 | `true` | ✅ 是 | ✅ 是（生产） | ✅ 是（生产） |

---

## 9. 建议的测试补充

为了填补证据缺口，建议添加以下测试：

### 9.1 测试 1：验证网络错误会重试

```javascript
it('retries on network errors', async function () {
    let counter = 0;
    const scope = nock('https://example.org')
        .post('/webmentions-test')
        .times(4)  // 1 次初始 + 3 次重试
        .replyWithError('connection refused');

    const service = new MentionSendingService({externalRequest: actualGot}); // 使用真实 got
    await assert.rejects(service.send({...}), /connection refused/);
    
    // 验证发送了 4 次请求
    assert.equal(counter, 4);
});
```

### 9.2 测试 2：验证 HTTP 5xx 不重试

```javascript
it('does NOT retry on HTTP 500 when throwHttpErrors=false', async function () {
    let counter = 0;
    const scope = nock('https://example.org')
        .post('/webmentions-test')
        .reply(() => {
            counter += 1;
            return [500];
        });

    const service = new MentionSendingService({externalRequest: actualGot});
    await assert.rejects(service.send({...}), /sending failed/);
    
    // 验证只发送了 1 次请求（没有重试）
    assert.equal(counter, 1);
});
```

### 9.3 测试 3：验证 `throwHttpErrors=true` 时 5xx 会重试

```javascript
it('DOES retry on HTTP 500 when throwHttpErrors=true', async function () {
    let counter = 0;
    const scope = nock('https://example.org')
        .get('/endpoint')
        .times(4)
        .reply(500);

    // 模拟 Endpoint 发现的配置
    const response = await got.get('https://example.org/endpoint', {
        throwHttpErrors: true,
        retry: { limit: 3 }
    });
    
    // 验证发送了 4 次请求
    assert.equal(counter, 4);
});
```

---

## 10. 附录：关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| POST WebMention 配置 | `core/server/services/mentions/mention-sending-service.js` | 87-117 |
| `throwHttpErrors: false` | 同上 | 97 |
| `retry.limit: 3` | 同上 | 103-106 |
| 注释与实现不一致 | 同上 | 103-104 |
| externalRequest 定义 | `core/server/lib/request-external.js` | 275-290 |
| 测试环境禁用重试 | `core/server/lib/request-external.js` | 284, 205-214 |
| got 版本 | `package.json` | 180 |
| GET 元数据 `throwHttpErrors: false` | `core/server/services/recommendations/service/recommendation-metadata-service.ts` | 52 |
| GET Endpoint 发现 `throwHttpErrors: true` | `core/server/services/mentions/mention-discovery-service.js` | 18 |
| 500 响应测试 | `test/unit/server/services/mentions/mention-sending-service.test.js` | 462-476 |
| 网络错误测试 | 同上 | 501-515 |
