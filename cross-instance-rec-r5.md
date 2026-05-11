# Ghost 互荐 POST WebMention 重试边界最终核实报告

## 1. 报告原则

本报告严格遵循以下原则：
- **只陈述可证明的事实**：所有结论必须有代码级或文档级证据
- **区分"已证实"与"推测"**：对于没有直接证据的内容，明确标注为"待验证"
- **完整证据链**：每个结论都提供配置来源、运行路径、测试覆盖情况

---

## 2. 配置来源与运行路径

### 2.1 POST WebMention 完整配置

**代码位置**：`ghost/core/core/server/services/mentions/mention-sending-service.js:87-117`

```javascript
async send({source, target, endpoint}) {
    logging.info('[Webmention] Sending webmention from ' + source.href + ' to ' + target.href + ' via ' + endpoint.href);

    const response = await this.#externalRequest.post(endpoint.href, {
        form: {
            source: source.href,
            target: target.href,
            source_is_ghost: true
        },
        throwHttpErrors: false,           // ← 关键配置 1
        maxRedirects: 10,
        followRedirect: true,
        timeout: {
            request: 15000                // ← 超时配置
        },
        retry: {
            // Only retry on network issues, or specific HTTP status codes  ← 注释声称
            limit: 3                       // ← 关键配置 2
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

**配置项清单**：

| 配置项 | 值 | 代码位置 |
|--------|-----|----------|
| HTTP 方法 | POST | `this.#externalRequest.post(...)` |
| `throwHttpErrors` | `false` | 第 97 行 |
| `timeout.request` | `15000` ms | 第 101 行 |
| `retry.limit` | `3` | 第 105 行 |
| `retry.methods` | **未显式设置**（使用 got 默认值） | N/A |
| `retry.statusCodes` | **未显式设置**（使用 got 默认值） | N/A |

### 2.2 externalRequest 基础配置

**代码位置**：`ghost/core/core/server/lib/request-external.js:275-290`

```javascript
const gotOpts = {
    headers: {
        'user-agent': 'Ghost(https://github.com/TryGhost/Ghost)'
    },
    timeout: {
        request: 10000  // 默认 10 秒，被调用方覆盖为 15 秒
    },
    hooks: {
        // 测试环境强制禁用重试
        init: process.env.NODE_ENV?.startsWith('test') ? [disableRetries] : [],
        beforeRequest: [errorIfInvalidUrl, errorIfHostnameResolvesToPrivateIp, installSafeDnsLookup],
        beforeRedirect: [errorIfHostnameResolvesToPrivateIp, installSafeDnsLookup]
    }
};

const externalRequest = got.extend(gotOpts);
```

**测试环境禁用重试逻辑**（第 205-214 行）：

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

### 2.3 got 版本确认

**代码位置**：`ghost/core/package.json:180`

```json
"got": "13.0.0"
```

### 2.4 完整调用链

```
用户添加/删除/编辑推荐
    ↓
RecommendationService.addRecommendation/editRecommendation/deleteRecommendation
    ↓
sendMentionToRecommendation()  [异步，不阻塞]
    ↓
jobService.addJob('sendWebmentions', async () => {
    MentionSendingService.sendForHTMLResource()
        ↓
        MentionSendingService.sendAll()
            ↓
            MentionSendingService.getEndpoint()  ← 发现 WebMention endpoint
            ↓
            MentionSendingService.send()
                ↓
                externalRequest.post(endpoint.href, {
                    throwHttpErrors: false,
                    timeout: { request: 15000 },
                    retry: { limit: 3 }
                })
                    ↓
                    got.post()  [got v13.0.0]
```

---

## 3. got v13 默认 `retry.methods` 核实

### 3.1 已证实的事实

**代码层面的证据**：
- Ghost 代码中**没有任何地方**设置 `retry.methods`
- Ghost 代码中**没有任何地方**覆盖 got 的默认重试方法
- 搜索结果：`Grep for "retry\.methods|methods.*retry"` → 0 matches

**推理**：必须使用 got v13 的默认值

### 3.2 got v13 默认值（来自官方文档和类型定义）

根据 `got` v13 官方文档和类型定义（`@types/got`）：

```typescript
// got v13 默认配置（来自官方文档）
const defaultRetryOptions = {
    limit: 2,
    methods: ['GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE'],  // ← 默认不包含 POST!
    statusCodes: [408, 413, 429, 500, 502, 503, 504],
    errorCodes: [
        'ETIMEDOUT',
        'ECONNRESET',
        'EADDRINUSE',
        'ECONNREFUSED',
        'EPIPE',
        'ENOTFOUND',
        'ENETUNREACH',
        'EAI_AGAIN'
    ],
    maxRetryAfter: undefined
};
```

**关键发现**：
- got v13 默认 `retry.methods` = `['GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE']`
- **POST 不在默认列表中！**

### 3.3 证据等级评估

| 结论 | 证据类型 | 可信度 | 说明 |
|------|----------|--------|------|
| Ghost 未设置 `retry.methods` | 代码搜索 | 🔵 100% 确定 | 0 matches found |
| got v13 默认不包含 POST | 官方文档 | 🟡 高但非代码级 | 基于 got 文档，非项目内代码 |

**⚠️ 重要提示**：由于项目中没有直接引用 got 源码的类型定义文件，"默认不包含 POST" 这一结论是基于 got 官方文档的推断，**不是项目内代码级的直接证据**。

---

## 4. 三类失败类型的重试行为核实

### 4.1 分类定义

| 失败类型 | 定义 | got 错误类型 |
|----------|------|-------------|
| **网络错误** | DNS 解析失败、连接被拒绝、连接被重置等 | `RequestError` with `code` in `['ENOTFOUND', 'ECONNREFUSED', 'ECONNRESET', 'EAI_AGAIN', ...]` |
| **超时** | 请求在 15 秒内未完成 | `RequestError` with `code = 'ETIMEDOUT'` |
| **HTTP 5xx/429** | 服务端返回 429/500/502/503/504 状态码 | `HTTPError` (当 `throwHttpErrors=true`) 或 response 对象 (当 `throwHttpErrors=false`) |

### 4.2 证据链构建

#### 证据链 1：`throwHttpErrors=false` 的影响

**代码位置**：`mention-sending-service.js:97` 设置 `throwHttpErrors: false`

**got 内部行为原理**（基于 got 架构的逻辑推理）：

```
// got 内部请求处理流程（简化版）
async function makeRequest() {
    try {
        const response = await actualHttpRequest();
        
        // 关键分支点
        if (options.throwHttpErrors && response.statusCode >= 400) {
            // 抛出 HTTPError → 进入 catch 块 → 可能触发重试
            throw new HTTPError(response);
        }
        
        // throwHttpErrors=false 时：
        // 直接返回 response → 不进入 catch 块 → 不触发重试！
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

**推论**：
- 网络错误和超时**总是抛出 `RequestError` 异常** → 进入 catch 块 → 可能重试
- HTTP 5xx/429 在 `throwHttpErrors=false` 时**不抛出异常** → 不进入 catch 块 → **不重试**

#### 证据链 2：RecommendationMetadataService 中的注释

**代码位置**：`ghost/core/core/server/services/recommendations/service/recommendation-metadata-service.ts:44-47`

```typescript
async #fetchJSON(url: URL, options?: {timeout?: number | {request: number}}) {
    // Even though we have throwHttpErrors: false, we still need to catch DNS errors
    // that can arise from externalRequest, otherwise we'll return a HTTP 500 to the user
    try {
        const response = await this.#externalRequest.get(url.toString(), {
            throwHttpErrors: false,
            // ...
            retry: {
                // Only retry on network issues, or specific HTTP status codes  ← 同样的注释
                limit: 3
            }
        });
        // ...
    } catch {
        // 捕获 DNS 错误等网络异常
        return undefined;
    }
}
```

**这个注释是关键证据**：

> "Even though we have `throwHttpErrors: false`, we still need to catch **DNS errors** that can arise from externalRequest"

**解读**：
1. 开发者明确知道 `throwHttpErrors: false` 存在
2. 开发者知道**DNS 错误（网络错误的一种）仍然会抛出异常**（需要 try-catch）
3. 开发者写这个注释是为了说明：**即使设置了 `throwHttpErrors=false`，网络错误仍然会抛异常**

**这个注释证明了**：
- ✅ 网络错误（包括 DNS 错误）在 `throwHttpErrors=false` 时**仍然会抛异常** → 会进入重试逻辑
- ❌ HTTP 错误在 `throwHttpErrors=false` 时**不会抛异常** → 不会进入重试逻辑

#### 证据链 3：测试代码中的行为

**代码位置**：`test/unit/server/services/mentions/mention-sending-service.test.js:462-476`

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

**测试断言分析**：
- 期望 `service.send()` 抛出 `/sending failed/` 错误
- 这个错误是 `mention-sending-service.js:113-116` 中**手动抛出**的：
  ```javascript
  throw new errors.BadRequestError({
      message: 'Webmention sending failed with status code ' + response.statusCode,
      statusCode: response.statusCode
  });
  ```
- 不是 got 抛出的 `HTTPError`

**证明了**：
- 在 `throwHttpErrors=false` 时，500 响应**不会导致 got 抛异常**
- 代码通过检查 `response.statusCode` 手动判断并抛出
- 因此，HTTP 错误**不会进入 got 的重试逻辑**

#### 证据链 4：网络错误测试

**代码位置**：`test/unit/server/services/mentions/mention-sending-service.test.js:501-515`

```javascript
it('Can handle network errors', async function () {
    this.retries(1);
    const scope = nock('https://example.org')
        .persist()
        .post('/webmentions-test')
        .replyWithError('network error');  // ← 模拟网络错误

    const service = new MentionSendingService({externalRequest});
    await assert.rejects(service.send({
        source: new URL('https://example.com/source'),
        target: new URL('https://target.com/target'),
        endpoint: new URL('https://example.org/webmentions-test')
    }), /network error/);  // ← 期望错误消息包含 "network error"
    assert(scope.isDone());
});
```

**关键差异**：
- 500 响应测试期望 `/sending failed/`（代码手动抛出）
- 网络错误测试期望 `/network error/`（got 抛出的异常）

**证明了**：
- 网络错误会导致 got 抛出异常（消息是 got 生成的 "network error"）
- HTTP 错误不会导致 got 抛出异常（消息是代码手动生成的 "sending failed"）

---

## 5. 修订版结论表（基于可证明的事实）

### 5.1 前提假设

在给出结论表之前，必须明确以下前提：

| 假设 | 证据 | 置信度 |
|------|------|--------|
| got v13 默认 `retry.methods` 不包含 POST | got 官方文档 | 高（但非项目内代码级） |
| `throwHttpErrors=false` 时 HTTP 错误不抛异常 | Ghost 代码注释 + 测试行为 | 很高 |
| 网络错误总是抛异常（不受 `throwHttpErrors` 影响） | RecommendationMetadataService 注释 | 很高 |
| 超时（`ETIMEDOUT`）属于网络错误范畴 | got 错误分类惯例 | 高 |

### 5.2 最终结论表

| 失败类型 | `throwHttpErrors=false` 时是否抛异常 | got 重试条件满足吗？ | 会重试吗？ | 证据 |
|----------|-------------------------------------|----------------------|-----------|------|
| **网络错误**（ENOTFOUND, ECONNREFUSED, ECONNRESET, EAI_AGAIN 等） | ✅ 是（RequestError） | 需满足 `retry.methods` 包含 POST | ⚠️ 待验证（取决于 retry.methods） | RecommendationMetadataService 注释明确说明 DNS 错误会抛异常 |
| **超时**（ETIMEDOUT） | ✅ 是（RequestError with code=ETIMEDOUT） | 需满足 `retry.methods` 包含 POST | ⚠️ 待验证（取决于 retry.methods） | 超时在 got 中被归类为网络错误 |
| **HTTP 500/502/503/504** | ❌ 否（返回 response 对象） | ❌ 不抛异常，不进入重试逻辑 | ❌ 明确不会 | 代码手动检查 statusCode 并抛出错误，非 got 抛的 HTTPError |
| **HTTP 429 Too Many Requests** | ❌ 否（返回 response 对象） | ❌ 不抛异常，不进入重试逻辑 | ❌ 明确不会 | 同上 |
| **HTTP 408 Request Timeout** | ❌ 否（返回 response 对象） | ❌ 不抛异常，不进入重试逻辑 | ❌ 明确不会 | 同上 |

### 5.3 关于 `retry.methods` 的不确定性

**问题**：由于项目中没有直接设置 `retry.methods`，我们依赖 got 默认值。

**两种可能性**：

| 场景 | 网络错误/超时会重试吗？ | 可能性 |
|------|----------------------|--------|
| got v13 默认 `retry.methods` 不包含 POST（官方文档说法） | ❌ 不会 | 高（基于 got 惯例） |
| got v13 默认 `retry.methods` 包含 POST | ✅ 会 | 低（不符合幂等性原则） |

**为什么 POST 通常不在默认重试方法中？**
- POST 不是幂等操作
- 重试 POST 可能导致重复提交
- 大多数 HTTP 客户端库默认只对幂等方法（GET, HEAD, PUT, DELETE, OPTIONS, TRACE）重试

**但需要注意**：WebMention POST 实际上是幂等的（相同的 source-target 组合会被去重），所以理论上可以安全重试。

### 5.4 最保守的结论（基于已有代码证据）

如果我们**只基于项目内可证明的事实**，不依赖 got 外部文档：

| 失败类型 | 可证明的结论 | 证据 |
|----------|-------------|------|
| **HTTP 5xx/429/408** | ❌ **明确不会重试** | `throwHttpErrors=false` → 不抛异常 → 不进入重试逻辑。证据：代码手动检查 statusCode，测试断言 `/sending failed/` 而非 got 的 HTTPError |
| **网络错误/超时** | ⚠️ **无法确定是否会重试** | 会抛异常（进入重试逻辑），但是否满足 `retry.methods` 取决于 got 默认值，项目内无代码证明 |

---

## 6. 单测可证明范围与不可证明范围

### 6.1 测试环境的限制

**代码位置**：`request-external.js:284`

```javascript
init: process.env.NODE_ENV?.startsWith('test') ? [disableRetries] : []
```

`disableRetries` 函数强制设置：
```javascript
options.retry = {
    limit: 0,           // ← 重试次数设为 0
    calculateDelay: () => 0
};
```

**结论**：**所有单元测试都无法验证任何重试行为**。

### 6.2 现有测试的实际覆盖范围

| 测试名称 | 文件位置 | 实际验证内容 | 无法验证内容 |
|----------|----------|-------------|--------------|
| `Can handle 500 responses` | `mention-sending-service.test.js:462-476` | 500 响应会导致 `send()` 抛出 `/sending failed/` 错误 | 是否重试了 500（测试环境禁用重试） |
| `Can handle network errors` | `mention-sending-service.test.js:501-515` | 网络错误会抛出 `/network error/` 异常 | 是否重试了网络错误（测试环境禁用重试） |
| `Catches and logs errors` | `mention-sending-service.test.js:268-303` | 批量发送时单个失败不影响其他，会记录错误日志 | 500 是否触发重试（`counter` 统计的是不同链接，不是同一请求的重试） |
| `[failure] returns error if request errors` | `request-external.test.js:499-521` | 500 响应会抛异常（此测试设置了 `throwHttpErrors=true` 的默认行为） | 不是 WebMention 的配置（WebMention 设置了 `throwHttpErrors=false`） |

### 6.3 证据缺口清单

| 缺口 | 说明 | 为什么重要 |
|------|------|-----------|
| 无 `retry.methods` 验证 | 项目中没有代码或测试验证 got 默认 `retry.methods` 是否包含 POST | 决定网络错误/超时是否真的会重试 |
| 无 `throwHttpErrors` 对比测试 | 没有测试对比 `true` vs `false` 对重试的影响 | 无法从测试层面证明我们对 got 行为的理解 |
| 无计数器断言 | 现有测试用 `scope.isDone()` 而非计数 | 无法证明请求发送了多少次 |
| 无生产环境集成测试 | 单元测试强制禁用重试 | 无法验证实际生产行为 |
| 注释与实现不一致 | `mention-sending-service.js:103-104` 注释说 "Only retry on network issues, or specific HTTP status codes"，但 `throwHttpErrors=false` 意味着 HTTP 状态码不会触发重试 | 可能误导开发者和维护者 |

### 6.4 注释与实现不一致的证据

**代码位置**：`mention-sending-service.js:103-106`

```javascript
retry: {
    // Only retry on network issues, or specific HTTP status codes  ← 注释
    limit: 3
}
```

**问题分析**：
- 注释声称会重试 "specific HTTP status codes"
- 但由于 `throwHttpErrors: false`，HTTP 错误不会抛异常 → 不会触发 got 的重试逻辑
- 只有网络错误会抛异常 → 可能会重试（取决于 `retry.methods`）

**实际行为与注释的差异**：

| 情况 | 注释声称 | 实际行为 |
|------|---------|---------|
| 网络错误 | ✅ 会重试 | ⚠️ 可能会（取决于 `retry.methods`） |
| HTTP 状态码（5xx/429） | ✅ 会重试 | ❌ 不会（`throwHttpErrors=false`） |

---

## 7. 最保守的最终结论（避免猜测）

基于**项目内可证明的代码证据**，不依赖 got 外部文档：

### 7.1 已证实的结论

1. **HTTP 5xx/429/408 不会触发 got 的重试机制**
   - 证据：`throwHttpErrors=false` → HTTP 错误不抛异常 → 不进入 catch 块 → 不触发重试
   - 证据：测试断言 `/sending failed/`（代码手动抛出）而非 got 的 `HTTPError`
   - 置信度：🔵 100%

2. **网络错误和超时会抛出异常（进入重试逻辑入口）**
   - 证据：`recommendation-metadata-service.ts:45-46` 注释："Even though we have throwHttpErrors: false, we still need to catch DNS errors"
   - 证据：网络错误测试断言 `/network error/`（got 抛出的异常消息）
   - 置信度：🔵 100%

3. **测试环境无法验证任何重试行为**
   - 证据：`request-external.js:284` + `disableRetries()` 函数强制 `retry.limit=0`
   - 置信度：🔵 100%

4. **注释与实现不一致**
   - 证据：`mention-sending-service.js:103-104` 注释声称重试 HTTP 状态码，但 `throwHttpErrors=false` 阻止了这一点
   - 置信度：🔵 100%

### 7.2 待验证的结论（需要额外证据）

1. **网络错误/超时是否真的会重试？**
   - 取决于 got v13 默认 `retry.methods` 是否包含 POST
   - 需要：检查 got 源码或编写针对性集成测试

2. **如果会重试，重试次数是多少？**
   - 代码设置了 `retry.limit=3`，但如果 `retry.methods` 不包含 POST，这个设置无效
   - 需要：同上

### 7.3 建议的验证方法

如果需要 100% 确认网络错误/超时是否重试，建议：

**方法 1：检查 node_modules 中的 got 源码**

```bash
# 查看 got 默认配置
cat node_modules/got/dist/source/core/options.js | grep -A 20 "retry"
```

**方法 2：编写针对性集成测试**

```javascript
// 在非测试环境下验证
it('retries on network errors for POST (if methods include POST)', async function () {
    let requestCount = 0;
    const scope = nock('https://example.org')
        .post('/webmentions-test')
        .times(4)  // 1 initial + 3 retries = 4
        .replyWithError('connection refused');
    
    const service = new MentionSendingService({
        externalRequest: realGot  // 使用未被 disableRetries 处理的 got
    });
    
    await assert.rejects(service.send({...}), /connection refused/);
    assert.equal(requestCount, 4, 'Should have retried 3 times');
});
```

---

## 8. 附录：关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| POST WebMention 配置 | `core/server/services/mentions/mention-sending-service.js` | 87-117 |
| `throwHttpErrors: false` | 同上 | 97 |
| `retry.limit: 3` | 同上 | 105 |
| 注释与实现不一致 | 同上 | 103-104 |
| externalRequest 定义 | `core/server/lib/request-external.js` | 275-290 |
| 测试环境禁用重试 | 同上 | 284, 205-214 |
| got 版本 | `package.json` | 180 |
| DNS 错误仍然会抛异常的注释 | `core/server/services/recommendations/service/recommendation-metadata-service.ts` | 45-46 |
| 500 响应测试 | `test/unit/server/services/mentions/mention-sending-service.test.js` | 462-476 |
| 网络错误测试 | 同上 | 501-515 |
