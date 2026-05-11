# Ghost 评论系统链路深度分析 (R2)

> 本文档是对 `comments-flow.md` 的补充，重点关注三个深入话题：
> 1. 审核状态变更后的前端刷新时机与最终一致性场景
> 2. 跨域认证失败路径与 origin 校验边界
> 3. 垃圾过滤与多语言的缺口与 fallback

---

## 一、审核状态变更后的前端刷新时机与最终一致性场景

### 1.1 刷新机制的核心架构

Ghost 评论系统**不使用 WebSocket 或 SSE 进行实时推送**，而是采用**拉取 + 乐观更新**的混合模式。

#### 1.1.1 缓存失效头的设置

后端在修改评论状态时，通过 `X-Cache-Invalidate` 响应头标记需要失效的 API 路径：

```javascript
// ghost/core/core/server/services/comments/comments-controller.js:227, 265, 308, 335
const pathsToInvalidate = [
    postId ? `/api/members/comments/post/${postId}/` : null,
    parentId ? `/api/members/comments/${parentId}/replies/` : null
].filter(Boolean);

frame.setHeader('X-Cache-Invalidate', pathsToInvalidate.join(', '));
```

这个响应头在以下操作后设置：
- `edit()`: 编辑评论内容
- `add()`: 新增评论
- `like()`: 点赞
- `unlike()`: 取消点赞

#### 1.1.2 前端对缓存头的处理

**注意**: 实际上，**comments-ui 前端并不消费 `X-Cache-Invalidate` 响应头**。这个头主要用于：

1. **CDN / 反向代理缓存失效**: 如果站点部署了 CDN，这个头告诉 CDN 哪些路径的缓存应该失效
2. **后端缓存层**: Ghost 内部缓存（如 Redis）可能使用这个头

前端依赖的是**乐观更新**（Optimistic Update）模式。

### 1.2 不同角色的刷新行为

#### 1.2.1 管理员（在站点评论区操作）

当管理员在同一浏览器会话中操作评论（如隐藏/显示）：

```typescript
// apps/comments-ui/src/actions.ts:145-177 (hideComment)
async function hideComment({state, data: comment}) {
    // 1. 先调用 API
    if (state.adminApi) {
        await state.adminApi.hideComment(comment.id);
    }
    
    // 2. 再更新本地状态（乐观更新，但这里是 API 成功后更新）
    return {
        comments: state.comments.map((c) => {
            // 更新状态为 'hidden'
            return {...c, status: 'hidden', replies};
        }),
        commentCount: state.commentCount - 1
    };
}
```

**刷新时机**: **立即**（API 调用完成后）
**一致性**: **强一致**（同一会话内）

#### 1.2.2 普通访客（页面保持打开）

当管理员在**另一个浏览器**或**后台管理页面**审核评论时：

```
场景: 访客 A 的页面保持打开，管理员 B 在后台隐藏了一条评论

访客 A 的浏览器:
┌────────────────────────────────────────────────────────────┐
│  1. 初始状态: 评论列表已加载到内存                          │
│  2. 管理员审核后: 数据库状态变为 'hidden'                   │
│  3. 但访客 A 的前端内存中，评论状态仍是 'published'         │
│  4. 没有 WebSocket 推送，没有通知                            │
└────────────────────────────────────────────────────────────┘

刷新时机: 仅在以下事件触发重新拉取
         ├── 页面刷新 (F5)
         ├── 切换排序方式 (Best ↔ Newest)
         ├── 点击 "Load more" 加载下一页
         ├── 评论被置顶导致重新渲染
         └── 用户自己操作 (评论/点赞) 触发重新拉取部分数据
```

**刷新时机**: **延迟**（依赖用户触发或页面刷新）
**一致性**: **最终一致**（可能存在几分钟到几天的不一致窗口）

### 1.3 最终一致性的具体场景

#### 场景 1: 评论被隐藏

```
时间线:
T0: 访客打开页面，加载评论 [C1, C2, C3]，状态都为 published
T1: 管理员在后台隐藏 C2 (status = 'hidden')
T2: 访客点击 C2 的点赞按钮
    ├── 前端调用 POST /api/members/comments/{C2}/like/
    └── 后端返回 404 或 403 (因为 hidden 评论对普通会员不可见)
    └── 前端点赞失败，但评论仍显示在列表中（内存状态未同步）
T3: 访客刷新页面
    ├── 重新 GET /api/members/comments/post/{postId}/
    ├── 后端过滤掉 status = 'hidden' 的评论
    └── C2 从列表中消失

不一致窗口: T1 ~ T3 (可能很长)
```

#### 场景 2: 评论被显示

```
时间线:
T0: 访客打开页面，评论列表中没有隐藏的评论
T1: 管理员在后台将 C4 从 hidden 改为 published
T2: 访客点击 "Load more" 加载下一页
    ├── 如果 C4 在"下一页"，会正常显示
    └── 如果 C4 在第一页（因排序变化），不会出现在新加载的数据中
T3: 访客切换排序方式 (Newest → Best)
    ├── 触发重新拉取第一页
    └── C4 出现在列表中

不一致窗口: T1 ~ 首次完整重新拉取
```

#### 场景 3: 评论被软删除 (status = 'deleted')

```
时间线:
T0: 评论 C1 有回复 R1, R2
T1: 访客删除 C1
T2: 前端判断: C1 有回复，不立即从列表移除，而是:
    └── 设置 status: 'deleted'，保留回复可见性
T3: 访客刷新页面
    ├── 后端: applyCustomQuery 会检查
    │   └── 父评论有可见回复时，即使 deleted 也返回
    └── C1 仍在列表中，但显示为 "[Deleted]"

特殊点: 有回复的已删除评论会一直保留，直到所有回复也被删除
```

### 1.4 刷新触发点汇总

| 操作 | 触发的前端刷新范围 | 数据来源 | 一致性 |
|------|------------------|---------|--------|
| 提交新评论 | 立即更新本地列表，新增项推入数组 | API 返回的新评论 | 强一致 |
| 编辑评论 | 立即替换本地状态中的该评论 | API 返回的更新后评论 | 强一致 |
| 删除评论 (无回复) | 触发 `setOrder()` 重新拉取整个评论区 | 完整重新拉取 | 强一致 |
| 删除评论 (有回复) | 仅本地标记为 deleted，保留回复 | 本地状态 | 最终一致 |
| 点赞/取消点赞 | 乐观更新本地 count.likes | 先更新本地，失败则回滚 | 最终一致 |
| 隐藏评论 (admin在站点操作) | API 完成后本地标记为 hidden | API 完成后更新本地 | 强一致 |
| 显示评论 (admin在站点操作) | 重新拉取单条评论后替换 | 单独 read API | 强一致 |
| 后台审核 (另一浏览器) | **无主动通知** | 等待用户触发 | 最终一致 |
| 切换排序方式 | 重新拉取第一页 | `/comments/post/{postId}/?order=...` | 强一致 |
| 加载更多 | 追加下一页数据 | `/comments/post/{postId}/?page=N` | 强一致 |
| 页面刷新 | 完整重新加载 | 首次初始化流程 | 强一致 |

### 1.5 设计权衡

**优点**:
- 无需维护 WebSocket 连接，服务器资源消耗低
- 乐观更新提供即时视觉反馈，用户体验好
- 拉取模式天然支持 CDN 缓存

**缺点**:
- 跨会话的审核操作存在可见性延迟
- 管理员和访客可能看到不同状态
- 点赞失败时的回滚逻辑可能产生闪烁

**改进空间**:
- 可考虑添加定时轮询（如每 30 秒检查新评论/状态变化）
- 或使用 Webhook 通知 + 前端 SSE（如果基础设施支持）

---

## 二、跨域认证失败路径与 origin 校验边界

### 2.1 跨域架构回顾

```
┌─────────────────────────────────────────────────────────────────┐
│  访客浏览器 (site.com)                                           │
│                                                                  │
│  ┌───────────────────────┐    postMessage    ┌────────────────┐ │
│  │  comments-ui iframe   │ ────────────────▶ │ admin-auth     │ │
│  │  (data-frame="comments")│  ◀────────────── │ iframe         │ │
│  │  site.com/ghost/...    │    postMessage    │ admin.com/...  │ │
│  └───────────────────────┘                   └────────────────┘ │
│           │                                (同源 Admin Cookie)  │
│           │                                                       │
│           ▼                                                       │
│  Members API (site.com/api/members/) ── Cookie 认证              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 失败路径分析

#### 路径 1: origin 不匹配（静默失败）

**位置**: `ghost/core/core/frontend/src/admin-auth/message-handler.js:41-44`

```javascript
window.addEventListener('message', async function (event) {
    if (event.origin !== siteOrigin) {
        console.warn('Ignored message to admin auth iframe because of mismatch in origin', 
            'expected', siteOrigin, 'got', event.origin, 'with data', event.data);
        return;  // 静默忽略，不返回响应
    }
    // ...
});
```

**触发条件**:
- `siteOrigin` 在编译时通过模板替换注入: `{{SITE_ORIGIN}}`
- 如果消息来自非预期的 origin（如恶意网站、错误配置的域名）

**失败表现**:
- 前端 `callApi()` Promise 永远不会 resolve/reject
- **内存泄漏风险**: `handlers[uid]` 永远不会被清理
- 用户看到的是"操作无响应"，没有错误提示

```typescript
// apps/comments-ui/src/utils/admin-api.ts:33
function callApi(action: string, args?: any): Promise<any> {
    return new Promise((resolve, reject) => {
        function handler(error, result) {
            if (error) return reject(error);
            return resolve(result);
        }
        uid += 1;
        handlers[uid] = handler;  // 永久泄漏，如果 origin 不匹配
        frame.contentWindow!.postMessage(JSON.stringify({
            uid, action, ...args
        }), adminOrigin);
    });
}
```

#### 路径 2: action 未知（静默失败）

**位置**: `message-handler.js:62-65`

```javascript
const handler = actions[data.action];
if (!handler) {
    return;  // 没有 handler，不返回响应
}
```

**触发条件**:
- 前端发送了不支持的 action
- 版本不匹配（旧版前端 + 新版后端，或反之）

**失败表现**:
- 同样是 Promise 永久 pending
- handlers 内存泄漏

#### 路径 3: 消息格式错误（静默失败）

**位置**: `message-handler.js:46-52`

```javascript
let data;
try {
    data = JSON.parse(event.data);
} catch (err) {
    console.error('Admin auth iframe failed to parse message from site origin:', event.data, err);
    return;  // 解析失败，不返回响应
}
```

**触发条件**:
- 前端发送了非 JSON 字符串
- 消息被中间人篡改

**失败表现**:
- Promise 永久 pending
- handlers 内存泄漏

#### 路径 4: 标识符验证失败（正常返回错误）

**位置**: `message-handler.js:6-11` + 测试用例

```javascript
function id(value) {
    if (typeof value !== 'string' || !/^[a-f0-9]{24}$/i.test(value)) {
        throw new Error('Invalid identifier');
    }
    return value;
}
```

**触发条件**:
- `postId`, `commentId`, `id` 不是有效的 ObjectID (24 位十六进制)
- 包含路径遍历字符: `../../users/me/token`
- 非字符串类型: `undefined`, `null`, 数字

**测试用例** (`ghost/core/test/unit/frontend/src/admin-auth-message-handler.test.js:65-69`):
```javascript
const INVALID_IDS = [
    {label: 'relative-path segment', value: '../../users/me/token'},
    {label: 'non-hex characters', value: 'ghijghijghijghijghijghij'},
    {label: 'wrong length', value: 'a'.repeat(23)},
    {label: 'non-string', value: undefined}
];
```

**失败表现**:
- **正常返回错误**: `respond(err, null)` → Promise reject
- handlers 正常清理
- 但前端可能没有错误处理 UI

#### 路径 5: Admin API 调用失败（正常返回错误）

**位置**: `message-handler.js:67-73`

```javascript
try {
    const res = await handler(data);
    const json = await res.json();
    respond(null, json);
} catch (err) {
    respond(err, null);  // 网络错误、HTTP 错误等
}
```

**触发条件**:
- 网络中断
- Admin API 返回 401（未登录）、403（无权限）、500（服务器错误）
- `res.json()` 解析失败（非 JSON 响应）

**失败表现**:
- Promise reject
- 前端可能显示错误，也可能静默失败

#### 路径 6: 前端 origin 校验（静默忽略）

**位置**: `admin-api.ts:9-13`

```typescript
window.addEventListener('message', function (event) {
    if (event.origin !== adminOrigin) {
        // Other message that is not intended for us
        return;  // 静默忽略
    }
    // ...
});
```

**触发条件**:
- 其他 iframe 或窗口发送了消息
- 恶意页面尝试伪造响应

**失败表现**:
- handlers 泄漏（如果是伪造的 uid 响应，真实的 handler 不会被调用）

### 2.3 安全边界分析

#### 2.3.1 siteOrigin 的注入机制

```javascript
// message-handler.js:4
const siteOrigin = '{{SITE_ORIGIN}}';
```

这个值在编译时（构建或部署时）被替换。可能的问题：

1. **环境变量配置错误**: 如果 Ghost 的 `url` 配置错误，siteOrigin 会错误
2. **多域名/自定义域名**: 每个站点需要单独构建，或使用通配符 origin（不安全）

#### 2.3.2 ObjectID 校验的防御目的

```javascript
/^[a-f0-9]{24}$/i  // 严格的 24 位十六进制
```

**防止的攻击**:
- **路径遍历**: `../../users/me/token` 会被拒绝
- **SQL 注入**: 虽然使用 ORM，但额外的格式校验是深度防御
- **API 滥用**: 不能用这个代理访问任意 Admin API 路径

**白名单 action**:
```javascript
const actions = {
    browseComments: d => fetch(...),  // 需要 postId (ObjectID)
    getReplies: d => fetch(...),      // 需要 commentId (ObjectID)
    readComment: d => fetch(...),     // 需要 commentId (ObjectID)
    getUser: () => fetch(...),        // 不需要 ID，但路径固定
    hideComment: d => fetch(...),     // 需要 id (ObjectID)
    showComment: d => fetch(...)      // 需要 id (ObjectID)
};
```

**能访问的 Admin API 路径**:
- `/api/admin/comments/post/{postId}/`
- `/api/admin/comments/{commentId}/replies/`
- `/api/admin/comments/{commentId}/`
- `/api/admin/users/me/?include=roles`
- `/api/admin/comments/{id}/` (PUT, 修改 status)

**不能访问的路径**:
- `/api/admin/users/` (用户管理)
- `/api/admin/posts/` (帖子管理)
- `/api/admin/settings/` (设置)
- 等等

#### 2.3.3 双向 postMessage 的信任链

```
信任链:
1. site.com 信任自己的 comments-ui iframe (同源)
2. comments-ui 信任 admin-auth iframe (通过预配置的 origin 白名单)
3. admin-auth 信任 site.com (通过编译时注入的 siteOrigin)
4. admin-auth 可以调用 Admin API (因为同源，携带 Cookie)

攻击面:
- 如果攻击者能控制 siteOrigin 的值（如部署配置泄露）
- 如果 Admin 登录态被盗用（XSS in admin）
- 如果 comments-ui 存在 XSS（可发送任意 action）
```

### 2.4 失败恢复机制

#### 前端超时机制

**当前状态**: **没有超时机制**

```typescript
// admin-api.ts 中没有超时
function callApi(action, args) {
    return new Promise((resolve, reject) => {
        // 没有 setTimeout，可能永远 pending
    });
}
```

**建议添加**:
```typescript
function callApi(action, args, timeout = 10000) {
    return new Promise((resolve, reject) => {
        const timer = setTimeout(() => {
            delete handlers[uid];
            reject(new Error('Admin API call timed out'));
        }, timeout);
        
        handlers[uid] = (error, result) => {
            clearTimeout(timer);
            // ...
        };
    });
}
```

#### 错误提示

**当前状态**: 部分操作有错误处理，部分没有

```typescript
// actions.ts 中的 hideComment 没有 try/catch
async function hideComment({state, data: comment}) {
    if (state.adminApi) {
        await state.adminApi.hideComment(comment.id);  // 失败会抛出，但上层可能没有 catch
    }
    // ...
}
```

---

## 三、垃圾过滤与多语言的缺口与 fallback

### 3.1 垃圾过滤的四层防护回顾

```
┌─────────────────────────────────────────────────────────────┐
│  第一层: 会员准入                                             │
│  ├── comments_enabled: off | members | paid                  │
│  ├── member.can_comment: false (单会员禁用)                  │
│  └── blocked_email_domains (注册时拦截，非评论时)             │
├─────────────────────────────────────────────────────────────┤
│  第二层: HTML 清理 (保存时)                                   │
│  ├── sanitize-html 白名单                                    │
│  ├── 链接 rel="ugc noopener noreferrer nofollow"            │
│  └── 空评论校验                                              │
├─────────────────────────────────────────────────────────────┤
│  第三层: 会员举报                                             │
│  ├── comment_reports 表                                      │
│  ├── 邮件通知管理员                                           │
│  └── count.reports 筛选                                      │
├─────────────────────────────────────────────────────────────┤
│  第四层: 管理员人工审核                                       │
│  ├── 隐藏/显示/删除评论                                       │
│  └── 禁用会员的评论权限                                       │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 垃圾过滤的缺口

#### 缺口 1: 缺乏自动化的垃圾检测

**现状**: 没有集成 Akismet、SpamAssassin 或类似的自动垃圾检测服务

**代码证据**:
```javascript
// ghost/core/core/server/services/comments/comments-service.js
// 没有调用外部垃圾检测服务的逻辑
// 只有基础的权限检查和内容清理

// 检查 can_comment 标志
// ghost/core/core/server/services/comments/comments-service.js:64-67
async checkCommentAccess(...) {
    if (member.can_comment === false) {
        throw new NotAllowedError({...});
    }
}
```

**可能的垃圾攻击**:
- 批量注册会员 + 自动化脚本提交评论
- 链接农场（大量评论中包含链接）
- 关键词垃圾（SEO 优化词堆砌）
- 重复内容（相同评论在多个帖子下）

#### 缺口 2: 速率限制不足

**现状**: 评论 API 没有明确的 per-member 速率限制

**代码检查**:
- `ghost/core/core/server/api/endpoints/comments-members.js` 没有 rate-limit 中间件
- 依赖可能存在的全局 API 限流，但未在评论模块中专门配置

**攻击面**: 单个会员可以在短时间内提交大量评论

#### 缺口 3: 内容审核规则不可配置

**现状**: sanitize-html 的规则硬编码在模型中

```javascript
// ghost/core/core/server/models/comment.js:99-112
sanitizeHtml(this.get('html'), {
    allowedTags: ['p', 'br', 'a', 'blockquote'],
    allowedAttributes: {
        a: ['href', 'target', 'rel']
    },
    // ...
});
```

**问题**:
- 站点管理员不能放宽或收紧允许的标签
- 不能添加自定义的内容过滤规则（如敏感词过滤）
- 不能配置最小/最大评论长度

#### 缺口 4: 链接数量限制

**现状**: 单条评论中可以包含任意数量的链接

**问题**: 链接农场垃圾评论无法被自动拦截

**建议**: 可考虑限制单条评论中的链接数量（如最多 2 个）

#### 缺口 5: blocked_email_domains 仅在注册时生效

**现状**:
```typescript
// apps/admin-x-settings/src/components/settings/advanced/spam-filters.tsx
// 这个设置在会员注册时检查，不在评论提交时检查
```

**问题**: 如果一个域名在会员注册后被添加到黑名单，已注册的会员仍可评论

#### 缺口 6: 已删除会员的评论

**现状**:
```javascript
// ghost/i18n/locales/en/comments.json
"Deleted member": "Name of a member used for comments when the member has been deleted"
```

**问题**: 已删除会员的评论仍保留在数据库中，无法批量清理或审核

### 3.3 多语言的 fallback 机制

#### 3.3.1 核心 fallback 逻辑

```javascript
// ghost/i18n/lib/i18n.js:12-30
function generateResources(locales, ns) {
    return locales.reduce((acc, locale) => {
        let res;
        try {
            res = require(`../locales/${locale}/${ns}.json`);
        } catch (err) {
            // 文件不存在 → 回退到英文
            res = require(`../locales/en/${ns}.json`);
        }
        // ...
    }, {});
}

// ghost/i18n/lib/i18n.js:136-139
fallbackLng: {
    no: ['nb', 'en'],  // 挪威语特殊处理
    default: ['en']     // 其他语言默认回退到英文
}
```

#### 3.3.2 fallback 链

```
请求语言: 'no' (挪威语，不存在)
    │
    ├── fallbackLng.no = ['nb', 'en']
    │       │
    │       ├── 尝试 'nb' (书面挪威语)
    │       └── 如果失败，尝试 'en'
    │
    └── 最终: 使用 'nb' 的翻译

请求语言: 'xx' (不存在的语言)
    │
    ├── 加载 locales/xx/comments.json → 失败
    ├── generateResources 捕获错误 → 回退到 en.json
    └── 最终: 使用英文翻译

请求语言: 'zh' (存在)，但某个键没有翻译
    │
    ├── i18next 查找 zh/comments.json 中的键
    ├── 没找到 → fallbackLng.default = ['en']
    └── 最终: 使用英文键本身作为 fallback
```

#### 3.3.3 测试用例验证

```javascript
// ghost/i18n/test/i18n.test.js:134-140
describe('it gracefully falls back to en if a file is missing', function () {
    it('should be able to translate a key that is missing in the locale', async function () {
        const resources = i18n.generateResources(['xx'], 'portal');
        const englishResources = i18n.generateResources(['en'], 'portal');
        assert.deepEqual(resources.xx, englishResources.en);
    });
});

// ghost/i18n/test/i18n.test.js:69-79
describe('Fallback will be nb when no is chosen', function () {
    it('Norwegian bokmål used when no is chosen', function () {
        const t = i18n('no', 'portal').t;
        assert.equal(t('Yearly'), 'Årlig');  // nb 的翻译
    });
});
```

#### 3.3.4 returnEmptyString = false 的作用

```javascript
// ghost/i18n/lib/i18n.js:133
returnEmptyString: false,
```

**含义**: 如果翻译值是空字符串 `""`，返回 key 本身而不是空字符串

**测试场景**:
```json
// zh/comments.json
{
    "Add comment": "添加评论",
    "New feature": ""  // 空字符串（翻译未完成）
}
```

```javascript
const t = i18n('zh', 'comments').t;
t('Add comment');      // → "添加评论"
t('New feature');      // → "New feature"（不是空字符串）
t('Unknown key');      // → "Unknown key"（回退到英文/key）
```

### 3.4 多语言的缺口

#### 缺口 1: 评论内容本身不支持多语言

**现状**: 评论内容是纯文本/HTML，没有语言标记

**问题**:
- 站点是多语言时，评论可能混杂不同语言
- 无法按语言过滤评论
- 无法针对语言应用不同的审核规则

#### 缺口 2: 管理员界面的评论内容不支持翻译

**现状**: 后台管理页面显示的评论内容是原始文本

**问题**: 如果评论是中文，英文管理员无法阅读

#### 缺口 3: 翻译键同步问题

**CI 检查**:
```bash
# ghost/i18n/package.json
"test": "... pnpm translate && git diff --exit-code"
```

**问题**:
- CI 会检查翻译键是否同步，但不检查翻译质量
- 空字符串被允许（returnEmptyString: false 只是返回 key）
- 部分翻译可能永远是英文

#### 缺口 4: RTL 语言支持

**现状**:
```javascript
// apps/sodo-search/src/app.js:11
const dir = i18n.dir() || 'ltr';
```

**问题**: comments-ui 中可能没有处理 RTL（从右到左）语言的文本方向

#### 缺口 5: 语言变体处理不完整

**支持的变体**:
- `zh` (简体中文)
- `zh-Hant` (繁体中文)
- `pt-BR` (巴西葡萄牙语)
- `de-CH` (瑞士德语)
- `sr-Cyrl` (塞尔维亚语西里尔字母)

**可能的问题**:
- 某些语言的区域变体没有独立翻译（如 `en-US`, `en-GB`）
- 如果请求 `pt-PT`（葡萄牙葡萄牙语），会回退到 `pt`（巴西葡萄牙语），可能存在术语差异

### 3.5 主题翻译的 fallback 增强

```javascript
// ghost/i18n/lib/i18n.js:32-102
function generateThemeResources(lng, themeLocalesPath) {
    // ...
    if (needsFallback) {
        if (locale !== 'en') {
            // 回退到英文主题翻译
            try {
                res = require(enPath);
            } catch {
                res = {};  // 空对象
            }
        } else {
            res = {};
        }
    }
    
    // 最终: 如果主题翻译都没有，i18next 会返回 key 本身
}
```

**测试验证**:
```javascript
// ghost/i18n/test/i18n.test.js:233-243
it('handles errors when both requested locale and English fallback files are invalid', async function () {
    const t = i18n('de', 'theme', {themePath});
    assert.equal(t('Read more'), 'Read more');  // 返回 key 本身
    assert.equal(t('Subscribe'), 'Subscribe');
});
```

---

## 四、总结与改进建议

### 4.1 刷新机制

| 问题 | 现状 | 建议 |
|------|------|------|
| 跨会话审核无实时通知 | 依赖页面刷新或用户操作 | 添加定时轮询（30s-5min）或 SSE |
| 乐观更新失败回滚 | 点赞有回滚，其他操作可能没有 | 统一错误处理和回滚逻辑 |
| 评论计数不一致 | 本地更新可能与服务器不同步 | 关键操作后重新拉取计数 |

### 4.2 跨域认证

| 问题 | 现状 | 建议 |
|------|------|------|
| 静默失败导致 handlers 泄漏 | origin/action 不匹配时不返回响应 | 添加超时机制，超时后 reject |
| 无错误提示 UI | 部分失败用户感知不到 | 统一错误处理，显示友好提示 |
| siteOrigin 编译时注入 | 多域名部署困难 | 考虑运行时 origin 白名单配置 |

### 4.3 垃圾过滤

| 问题 | 现状 | 建议 |
|------|------|------|
| 无自动化垃圾检测 | 仅人工审核 | 集成 Akismet 或自研规则引擎 |
| 内容规则硬编码 | sanitize-html 配置不可改 | 后台可配置过滤规则 |
| 无速率限制 | 单会员可大量提交 | 添加 per-member 速率限制 |
| 链接数量无限制 | 可用于链接农场 | 限制单条评论链接数量 |

### 4.4 多语言

| 问题 | 现状 | 建议 |
|------|------|------|
| 评论内容无语言标记 | 无法按语言过滤 | 记录评论语言，支持多语言站点 |
| 翻译质量无保证 | 空字符串回退到 key | 翻译审核流程，社区翻译平台 |
| RTL 支持不完整 | 部分应用有 dir 处理 | 统一 RTL 样式处理 |
| 区域变体不完善 | 部分语言只有一个翻译 | 增加常用变体支持 |

---

## 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 缓存失效头设置 | `ghost/core/core/server/services/comments/comments-controller.js:227` |
| 前端乐观更新 | `apps/comments-ui/src/actions.ts:145` |
| admin-auth origin 校验 | `ghost/core/core/frontend/src/admin-auth/message-handler.js:41` |
| admin-auth 测试 | `ghost/core/test/unit/frontend/src/admin-auth-message-handler.test.js` |
| 前端 postMessage 调用 | `apps/comments-ui/src/utils/admin-api.ts:33` |
| i18n fallback 实现 | `ghost/i18n/lib/i18n.js:12` |
| i18n 测试 | `ghost/i18n/test/i18n.test.js` |
| 评论 HTML 清理 | `ghost/core/core/server/models/comment.js:96` |
