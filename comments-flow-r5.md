# Ghost 评论审核同步机制 - 最终决策文档 (R5)

> **版本**: 1.0  
> **日期**: 2026-05-11  
> **状态**: 待评审  
> **文档目的**: 提供可落地的审核回拉流程步骤、失败处理决策、实施顺序与验收清单

---

## 第一部分: 同会话审核回拉流程

### 1.1 前提条件

同会话审核生效的前置条件：
- 用户已登录 Ghost 后台 (`/ghost`)
- 用户角色为 `Owner` / `Administrator` / `Super Editor`
- 评论组件已通过 `AuthFrame` 完成 `initAdminAuth` 初始化
- `state.adminApi !== null` 且 `state.isAdmin === true`

### 1.2 hideComment - 隐藏评论可落地步骤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         hideComment 执行步骤                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤 1: 用户点击 "Hide" 按钮                                             │
│  └── 触发: dispatchAction('hideComment', {id: commentId})                │
│                                                                         │
│  步骤 2: 校验 adminApi 是否可用                                          │
│  ├── 文件: actions.ts:145-148                                           │
│  ├── 代码: if (state.adminApi) { ... }                                  │
│  └── 判定:                                                              │
│      ├── 可用: 继续步骤 3                                               │
│      └── 不可用: 跳过 API 调用，仅执行本地更新 (但实际无意义)              │
│                                                                         │
│  步骤 3: 调用跨域 API                                                    │
│  ├── 文件: admin-api.ts:59-61                                           │
│  ├── 代码: await state.adminApi.hideComment(comment.id)                 │
│  └── 调用链:                                                            │
│      └── callApi('hideComment', {id})                                   │
│          └── frame.contentWindow.postMessage(...)                       │
│              └── (跨 iframe) message-handler.js                        │
│                                                                         │
│  步骤 4: message-handler 处理                                            │
│  ├── 文件: message-handler.js:36                                        │
│  ├── 代码: hideComment: d => setCommentStatus(d.id, 'hidden')           │
│  └── 实际请求:                                                          │
│      └── PUT /ghost/api/admin/comments/{id}/                           │
│          └── Body: {comments: [{id, status: 'hidden'}]}                │
│                                                                         │
│  步骤 5: API 响应                                                       │
│  ├── 成功路径:                                                          │
│  │   ├── res.json() 成功                                               │
│  │   └── respond(null, json) → 前端 resolve                            │
│  │                                                                     │
│  └── 失败路径:                                                          │
│      ├── fetch 网络错误 → respond(err, null) → 前端 reject             │
│      └── HTTP 错误 (4xx/5xx) → res.json() 可能成功/失败                │
│                                                                         │
│  步骤 6: 本地状态更新 (API 成功后执行)                                    │
│  ├── 文件: actions.ts:149-176                                          │
│  ├── 变更 1: 评论状态更新                                                │
│  │   └── c.status = 'hidden' (主评论或回复)                             │
│  │                                                                     │
│  └── 变更 2: 计数更新                                                   │
│      └── commentCount: state.commentCount - 1                          │
│                                                                         │
│  步骤 7: React 重新渲染                                                  │
│  └── 视觉效果:                                                          │
│      ├── 评论内容显示 "Hidden for members"                              │
│      ├── 评论计数减 1                                                   │
│      └── 按钮从 "Hide" 变为 "Show"                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 showComment - 恢复评论可落地步骤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         showComment 执行步骤                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤 1: 用户点击 "Show" 按钮                                             │
│  └── 触发: dispatchAction('showComment', {id: commentId})                │
│                                                                         │
│  步骤 2: 校验 adminApi 是否可用                                          │
│  ├── 文件: actions.ts:179-182                                          │
│  └── 同 hideComment 步骤 2                                              │
│                                                                         │
│  步骤 3: 调用跨域 API (恢复状态)                                          │
│  ├── 文件: admin-api.ts:62-64                                           │
│  ├── 代码: await state.adminApi.showComment({id})                       │
│  └── 调用链:                                                            │
│      └── callApi('showComment', {id})                                   │
│          └── message-handler.js:37                                      │
│              └── PUT status='published'                                 │
│                                                                         │
│  步骤 4: 重新拉取完整评论数据 (关键差异!)                                   │
│  ├── 文件: actions.ts:183-190                                          │
│  ├── 原因: 需要以当前会员身份重新加载关系数据                               │
│  │   (点赞状态、回复等)                                                 │
│  │                                                                     │
│  └── 调用:                                                             │
│      └── adminApi.read({commentId, memberUuid})                        │
│          └── callApi('readComment', {...})                             │
│              └── GET /ghost/api/admin/comments/{id}/                   │
│                  └── ?impersonate_member_uuid={memberUuid}             │
│                                                                         │
│  步骤 5: 本地状态替换                                                    │
│  ├── 文件: actions.ts:194-214                                         │
│  ├── 用完整数据替换本地状态:                                             │
│  │   └── c.id === comment.id ? updatedComment : c                      │
│  │                                                                     │
│  └── 计数更新:                                                          │
│      └── commentCount: state.commentCount + 1                          │
│                                                                         │
│  步骤 6: React 重新渲染                                                  │
│  └── 视觉效果:                                                          │
│      ├── 评论完整内容恢复显示                                             │
│      ├── 评论计数加 1                                                   │
│      └── 按钮从 "Show" 变为 "Hide"                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.4 同会话关键参数汇总

| 参数 | hideComment | showComment |
|------|-------------|-------------|
| API 调用次数 | 1 次 | 2 次 (PUT + GET) |
| 本地更新方式 | 直接修改 status | 完整替换对象 |
| 影响计数 | -1 | +1 |
| 失败回滚 | **无** (当前代码) | **无** (当前代码) |
| 乐观更新 | 否 (API 成功后更新) | 否 (API 成功后更新) |

---

## 第二部分: 跨会话审核回拉流程

### 2.1 触发事件与回拉条件

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    跨会话触发事件优先级 (从高到低)                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  P0: 页面刷新 (F5 / 重新导航)                                            │
│  ├── 触发条件: 用户主动刷新页面                                          │
│  ├── 回拉动作: initSetup() 完整重新初始化                                │
│  └── firstCommentCreatedAt: 重置为新值                                  │
│                                                                         │
│  P1: 切换排序方式                                                        │
│  ├── 触发条件: 用户点击 "Sort by" 下拉框                                 │
│  ├── 回拉动作: setOrder({order})                                        │
│  └── firstCommentCreatedAt: 不生效 (有 order 参数)                      │
│                                                                         │
│  P2: 加载更多评论                                                        │
│  ├── 触发条件: 用户点击 "Load more"                                     │
│  ├── 回拉动作: loadMoreComments()                                       │
│  └── firstCommentCreatedAt: 仍在使用 (无 order 时)                      │
│                                                                         │
│  P3: 用户自身操作 (发表/点赞/编辑/删除)                                   │
│  ├── 触发条件: 用户执行评论操作                                          │
│  └── 回拉动作: 仅影响操作的单条评论                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 P0: 页面刷新可落地步骤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      页面刷新 - 完整回拉步骤                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤 1: 页面加载 → App 组件挂载                                         │
│  └── 入口: app.tsx:30                                                   │
│                                                                         │
│  步骤 2: 懒加载检测 (IntersectionObserver)                               │
│  ├── 文件: app.tsx:317-348                                              │
│  ├── 触发条件: 评论区进入视口 (threshold: 0.1)                           │
│  └── 例外: 有 permalink 时立即加载                                       │
│                                                                         │
│  步骤 3: initSetup() 执行                                                │
│  ├── 文件: app.tsx:263-314                                              │
│  ├── 子步骤 3.1: api.init()                                             │
│  │   └── 获取会员信息、labs 配置、supportEmail                          │
│  │                                                                     │
│  ├── 子步骤 3.2: fetchComments()                                       │
│  │   ├── api.comments.browse({page:1})                                 │
│  │   │   └── GET /members/api/comments/post/{postId}/                  │
│  │   │       └── 后端过滤: status NOT IN ('hidden', 'deleted')         │
│  │   │                                                                 │
│  │   └── api.comments.count({postId})                                  │
│  │       └── GET /members/api/comments/post/{postId}/count/            │
│  │                                                                     │
│  └── 子步骤 3.3: (如有 permalink) fetchScrollTarget()                  │
│      └── 检查目标评论是否仍存在且为 published                           │
│                                                                         │
│  步骤 4: AuthFrame 加载 → initAdminAuth()                               │
│  ├── 文件: app.tsx:357                                                 │
│  └── 仅当 comments.length > 0 时加载                                    │
│                                                                         │
│  步骤 5: adminApi.browse() (如管理员登录)                                │
│  ├── 文件: app.tsx:143                                                  │
│  └── 使用管理员 API 重新拉取 (含 hidden/deleted 状态)                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 P1: 切换排序可落地步骤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      切换排序 - 第 1 页回拉步骤                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤 1: 用户选择新排序方式                                               │
│  └── 触发: dispatchAction('setOrder', {order: 'new_order'})             │
│                                                                         │
│  步骤 2: 设置加载状态                                                    │
│  ├── 文件: actions.ts:34-35                                            │
│  └── dispatchAction('setCommentsIsLoading', true)                       │
│      └── UI: 评论区半透明 (opacity: 0.5)                                │
│                                                                         │
│  步骤 3: 调用 API (第 1 页全量拉取)                                       │
│  ├── 文件: actions.ts:38-43                                            │
│  ├── 管理员: state.adminApi.browse({page:1, postId, order})            │
│  └── 普通用户: api.comments.browse({page:1, postId, order})            │
│                                                                         │
│  步骤 4: 关键: firstCommentCreatedAt 不生效                              │
│  ├── 文件: admin-api.ts:68-70                                          │
│  ├── 代码: if (firstCommentCreatedAt && !order) { ... }                 │
│  └── order 存在 → 跳过锚点过滤 → 拉取最新数据                            │
│                                                                         │
│  步骤 5: 更新状态                                                        │
│  ├── 文件: actions.ts:45-50                                            │
│  └── 返回: {comments, pagination, order, commentsIsLoading: false}      │
│                                                                         │
│  步骤 6: 异常处理                                                        │
│  ├── 文件: actions.ts:51-55                                            │
│  ├── console.error('Failed to set order:', error)                      │
│  ├── state.commentsIsLoading = false                                   │
│  └── throw error → 向上传播但无 UI 提示                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 P2: 加载更多可落地步骤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      加载更多 - 追加页回拉步骤                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤 1: 用户点击 "Load more ({amount})"                                 │
│  └── 触发: dispatchAction('loadMoreComments')                            │
│                                                                         │
│  步骤 2: 计算下一页                                                      │
│  ├── 文件: actions.ts:7-10                                             │
│  └── page = state.pagination.page + 1                                   │
│                                                                         │
│  步骤 3: 调用 API                                                        │
│  ├── 文件: actions.ts:12-16                                            │
│  └── 注意: **无 order 参数** (使用当前 state.order)                      │
│                                                                         │
│  步骤 4: 关键: firstCommentCreatedAt 锚点生效                            │
│  ├── 文件: admin-api.ts:68-70                                          │
│  └── filter = `created_at:<=${firstCommentCreatedAt}`                   │
│      └── **问题**: 已加载页的审核变更无法发现!                           │
│                                                                         │
│  步骤 5: 合并数据                                                        │
│  ├── 文件: actions.ts:18-25                                            │
│  ├── updatedComments = [...state.comments, ...data.comments]           │
│  └── dedupedComments = 去重 (按 id)                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 跨会话一致性对比

| 触发事件 | 数据范围 | firstCommentCreatedAt 状态 | 一致性 | 备注 |
|---------|---------|---------------------------|--------|------|
| 页面刷新 | 完整重新加载 | 重置 | **强一致** ✓ | P0 |
| 切换排序 | 第 1 页重新拉取 | 不生效 | **部分一致** ✓ | P1 |
| 加载更多 | 追加下一页 | **仍生效** | **部分一致** ✗ | P2 - 已加载页不刷新 |
| 发表评论 | 新增评论 | 不重置 | 仅新评论 | P3 |
| 点赞/编辑 | 单条评论 | 不影响 | 仅该评论 | P3 |
| 删除评论 (无回复) | 触发 setOrder | 重置 | **强一致** ✓ | P3 - 特殊情况 |
| 删除评论 (有回复) | 本地标记 | 不重置 | 仅该评论 | P3 |

---

## 第三部分: 失败类型决策表

### 3.1 失败类型分类

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         失败类型分类                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  类 A: 可重试失败 (Retryable)                                            │
│  ├── 特征: 临时问题，重试可能成功                                        │
│  └── 策略: 指数退避重试 → 仍失败 → 降级                                 │
│                                                                         │
│  类 B: 不可重试失败 (Non-Retryable)                                     │
│  ├── 特征: 配置/权限问题，重试也不会成功                                │
│  └── 策略: 立即降级 + 错误提示                                         │
│                                                                         │
│  类 C: 静默失败 (Silent Failure)                                        │
│  ├── 特征: message-handler 不返回响应，Promise 永久 pending            │
│  └── 策略: 超时机制 → 超时后 reject + 清理 handlers                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 详细决策表 (可落地参数)

#### 类 A: 可重试失败

| ID | 失败类型 | 判定信号 | 默认超时 | 重试上限 | 重试间隔 | 降级动作 | 提示文案 (i18n key) |
|----|---------|---------|---------|---------|---------|---------|---------------------|
| A1 | fetch 网络失败 | `fetch()` 抛出 `TypeError` | 浏览器默认 | **3 次** | 1s, 2s, 4s | `adminApi = null`<br>`adminAuthDegraded = true` | "审核功能暂时不可用" |
| A2 | HTTP 5xx | `res.status >= 500` | 浏览器默认 | **2 次** | 2s, 4s | 同 A1 | 同 A1 |
| A3 | HTTP 429 | `res.status === 429` | `Retry-After` 响应头 | **3 次** | Retry-After 或 5s/10s/20s | 同 A1 | "操作太频繁，请稍后再试" |
| A4 | iframe 时序问题 | `postMessage` 抛出异常 | 无 | **5 次** | 100ms/200ms/300ms/400ms/500ms | 同 A1 | 无 (静默重试) |

#### 类 B: 不可重试失败

| ID | 失败类型 | 判定信号 | 默认超时 | 重试上限 | 重试间隔 | 降级动作 | 提示文案 (i18n key) |
|----|---------|---------|---------|---------|---------|---------|---------------------|
| B1 | HTTP 401 | `res.status === 401` | 无 | **0 次** | 无 | `admin = null`<br>`adminApi = null`<br>`isAdmin = false` | "请先登录后台以使用审核功能" |
| B2 | HTTP 403 | `res.status === 403` | 无 | **0 次** | 无 | 同 B1 | "您没有审核评论的权限" |
| B3 | HTTP 404 | `res.status === 404` | 无 | **0 次** | 无 | 本地移除该评论 | "该评论不存在或已被删除"<br>(现有: `The linked comment is no longer available.`) |
| B4 | 标识符校验失败 | `!/^[a-f0-9]{24}$/i.test(value)` | 无 | **0 次** | 无 | 无 (bug/攻击) | "操作失败，请刷新页面重试" |
| B5 | 响应 JSON 解析失败 | `res.json()` 抛出异常 | 无 | **0 次** | 无 | `adminApi = null` | "服务器异常，请刷新页面重试" |
| B6 | 角色校验失败 | `!ALLOWED_MODERATORS.includes(role.name)` | 无 | **0 次** | 无 | `admin = null` (设计如此) | 无 (静默降级) |

#### 类 C: 静默失败 (需超时机制兜底)

| ID | 失败类型 | 判定信号 | 默认超时 | 重试上限 | 重试间隔 | 降级动作 | 提示文案 (i18n key) |
|----|---------|---------|---------|---------|---------|---------|---------------------|
| C1 | Origin 不匹配 (发送方) | `event.origin !== siteOrigin` | **10s** | **1 次** | 无 | `adminApi = null` | "审核功能配置异常，请联系站长" |
| C2 | JSON 解析失败 (发送方) | `JSON.parse(event.data)` 失败 | **10s** | **0 次** | 无 | 同 C1 | "操作失败，请刷新页面重试" |
| C3 | Action 不匹配 | `actions[data.action] === undefined` | **10s** | **0 次** | 无 | 同 C1 | "操作失败，请刷新页面重试" |
| C4 | Origin 不匹配 (接收方) | `event.origin !== adminOrigin` | **10s** | **0 次** | 无 | 无 (安全防护) | 无 (静默忽略) |
| C5 | UID 不匹配 | `handlers[data.uid] === undefined` | 无 | **0 次** | 无 | 无 | 无 |

### 3.3 用户可见提示文案 (i18n 新增键)

```json
{
    "Moderation temporarily unavailable": "审核功能暂时不可用，稍后刷新页面重试",
    "Too many requests. Please try again later.": "操作太频繁，请稍后再试",
    "Please sign in to the admin panel to moderate comments.": "请先登录后台以使用审核功能",
    "You don't have permission to moderate comments.": "您没有审核评论的权限",
    "Server error. Please refresh the page and try again.": "服务器异常，请刷新页面重试",
    "Moderation configuration error. Please contact the site owner.": "审核功能配置异常，请联系站长",
    "Operation failed. Please refresh the page and try again.": "操作失败，请刷新页面重试",
    "Moderation unavailable": "审核功能不可用"
}
```

### 3.4 降级状态机

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          降级状态机                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  初始状态: adminAuthReady                                               │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  state.adminApi !== null                                          │ │
│  │  state.adminAuthFailed === false                                  │ │
│  │  state.adminAuthDegraded === false                                │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│           │                                                             │
│           │ 首次失败 (可重试类型: A1-A4)                                  │
│           ▼                                                             │
│  adminAuthRetrying                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  静默重试 (指数退避)                                               │ │
│  │  用户无感知                                                        │ │
│  │  重试上限: 2-3 次 (根据类型)                                       │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│           │                                                             │
│           ├── 重试成功 ──▶ adminAuthReady                              │
│           │                                                             │
│           │ 重试耗尽 (连续失败)                                          │
│           ▼                                                             │
│  adminAuthDegraded                                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  state.adminApi = null                                            │ │
│  │  state.adminAuthDegraded = true                                   │ │
│  │  UI: 隐藏审核按钮                                                  │ │
│  │  Toast: "审核功能暂时不可用"                                       │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│           │                                                             │
│           │ 页面刷新 ──▶ adminAuthReady (重新尝试初始化)                │
│           │                                                             │
│           │ 不可重试失败 (B1-B5 / C1-C3)                                │
│           ▼                                                             │
│  adminAuthFailed                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  state.adminApi = null                                            │ │
│  │  state.adminAuthFailed = true                                     │ │
│  │  UI: 隐藏审核按钮                                                  │ │
│  │  Toast: 具体错误信息                                               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│           │                                                             │
│           │ 页面刷新 + 问题修复 ──▶ adminAuthReady                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 第四部分: 实施顺序

### 4.1 阶段划分

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          实施顺序决策                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 0: 准备 (0.5 天)                                                  │
│  ├── 任务清单:                                                          │
│  │   ├── 分析现有代码结构 (已完成)                                      │
│  │   ├── 确认测试框架和覆盖率要求                                        │
│  │   └── 准备开发环境                                                   │
│  │                                                                     │
│  └── 交付物:                                                           │
│      └── 开发环境就绪                                                   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 1: P0 - 安全/可用性修复 (4 天)                                     │
│  ├── 优先级: 必须 (Critical)                                            │
│  ├── 目标: 解决静默失败导致的 Promise 永久 pending                       │
│  │                                                                     │
│  ├── 任务 1.1: admin-api.ts 添加超时机制 (1 天)                          │
│  │   ├── callApi() 添加 10 秒默认超时                                   │
│  │   ├── 超时时清理 handlers[uid]                                       │
│  │   └── 支持自定义 timeout 参数                                        │
│  │                                                                     │
│  ├── 任务 1.2: message-handler.js 静默失败返回响应 (1 天)                │
│  │   ├── origin 不匹配: 返回错误响应                                    │
│  │   ├── JSON 解析失败: 返回错误响应                                    │
│  │   └── action 不匹配: 返回错误响应                                    │
│  │                                                                     │
│  ├── 任务 1.3: app.tsx initAdminAuth 超时 (1 天)                        │
│  │   ├── adminApi.getUser() 包装 15 秒超时                              │
│  │   └── 超时后标记 adminAuthFailed                                    │
│  │                                                                     │
│  └── 任务 1.4: P0 测试 (1 天)                                          │
│      ├── 单元测试: 超时逻辑、handlers 清理                              │
│      └── 集成测试: 各种错误场景                                         │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 2: P1 - 用户体验提升 (4 天)                                        │
│  ├── 优先级: 建议 (Important)                                           │
│  ├── 目标: 自动重试 + 降级状态管理                                       │
│  │                                                                     │
│  ├── 任务 2.1: admin-api.ts 可重试错误自动重试 (1 天)                    │
│  │   ├── 网络错误: 指数退避 3 次                                        │
│  │   ├── 5xx: 指数退避 2 次                                            │
│  │   └── 429: 按 Retry-After 等待 + Jitter                             │
│  │                                                                     │
│  ├── 任务 2.2: app.tsx 降级状态管理 (1 天)                               │
│  │   ├── 新增 state: adminAuthFailed, adminAuthDegraded                │
│  │   ├── 连续失败计数器                                                │
│  │   └── 降级后不再尝试 adminApi 调用                                   │
│  │                                                                     │
│  ├── 任务 2.3: 新增 i18n 翻译键 (0.5 天)                                │
│  │   ├── ghost/i18n/locales/en/comments.json 新增 8 个键               │
│  │   └── 运行 pnpm translate 同步其他语言                               │
│  │                                                                     │
│  ├── 任务 2.4: 错误提示 UI (1 天)                                       │
│  │   ├── Toast 组件集成                                                │
│  │   └── 降级状态提示条                                                │
│  │                                                                     │
│  └── 任务 2.5: P1 测试 (0.5 天)                                        │
│      ├── 单元测试: 重试逻辑、降级状态机                                 │
│      └── 集成测试: 用户提示验证                                         │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 3: P2 - 可选优化 (3 天, 可选)                                      │
│  ├── 优先级: 可选 (Nice-to-have)                                        │
│  ├── 目标: 跨会话一致性增强 + 可观测性                                   │
│  │                                                                     │
│  ├── 任务 3.1: 跨会话刷新机制 (2 天)                                    │
│  │   ├── visibilitychange 事件触发刷新                                  │
│  │   └── 可选: 30s 轮询 (配置开关)                                      │
│  │                                                                     │
│  └── 任务 3.2: 错误上报集成 (1 天)                                      │
│      └── 失败时上报到监控系统                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 详细任务清单

#### 阶段 1: P0 任务详情

**任务 1.1: admin-api.ts 添加超时机制**

| 项 | 内容 |
|---|------|
| 文件 | `apps/comments-ui/src/utils/admin-api.ts:33` |
| 修改 | `callApi()` 函数 |
| 改动点 | 1. 新增 `DEFAULT_TIMEOUT = 10000`<br>2. `callApi()` 支持 `options.timeout`<br>3. Promise 内设置 `setTimeout`<br>4. 超时时 `delete handlers[uid]` + reject<br>5. 正常响应时 `clearTimeout`<br>6. postMessage try/catch 包裹 |
| 测试用例 | - 超时后 Promise reject<br>- 超时后 handlers 已清理<br>- 正常响应不触发超时<br>- postMessage 异常正确处理 |

**任务 1.2: message-handler.js 静默失败返回响应**

| 项 | 内容 |
|---|------|
| 文件 | `ghost/core/core/frontend/src/admin-auth/message-handler.js` |
| 修改 | `window.addEventListener('message', ...)` 事件处理器 |
| 改动点 | 1. 调整执行顺序: 先 parse JSON，再校验 origin<br>2. origin 不匹配: `respond(new Error('Origin mismatch'), null)`<br>3. action 不匹配: `respond(new Error('Unknown action'), null)`<br>4. JSON parse 失败: 无法获取 uid，依赖前端超时兜底 |
| 测试用例 | - origin 不匹配返回错误响应<br>- action 不匹配返回错误响应<br>- 正常流程不受影响 |

**任务 1.3: app.tsx initAdminAuth 超时**

| 项 | 内容 |
|---|------|
| 文件 | `apps/comments-ui/src/app.tsx:122-178` |
| 修改 | `initAdminAuth()` 函数 |
| 改动点 | 1. `adminApi.getUser()` 包装在 `Promise.race`<br>2. 超时 Promise: 15 秒<br>3. 超时/异常后 `setState({adminAuthFailed: true})`<br>4. 新增 state 字段: `adminAuthFailed`, `adminAuthDegraded` |
| 测试用例 | - getUser 超时后触发降级<br>- 正常获取不受影响 |

#### 阶段 2: P1 任务详情

**任务 2.1: admin-api.ts 可重试错误自动重试**

| 项 | 内容 |
|---|------|
| 文件 | `apps/comments-ui/src/utils/admin-api.ts` |
| 新增 | 重试包装函数 |
| 改动点 | 1. 新增 `withRetry(fn, options)` 工具函数<br>2. 可重试错误判定: 网络错误、5xx、429<br>3. 指数退避: `delay = base * (2 ** attempt) + jitter`<br>4. 重试上限: 网络错误 3 次，5xx 2 次<br>5. 429 读取 Retry-After 响应头 |
| 设计 | `callApi()` 内部不处理重试，由上层 `browse()`/`hideComment()` 等包装 |

**任务 2.2: app.tsx 降级状态管理**

| 项 | 内容 |
|---|------|
| 文件 | `apps/comments-ui/src/app.tsx` |
| 新增 | 状态机逻辑 |
| 改动点 | 1. state 新增字段: `adminAuthRetrying: boolean`<br>2. 连续失败计数器: `consecutiveFailures: number`<br>3. 降级阈值: 连续 3 次失败 → degraded<br>4. 成功时重置计数器 |
| 组件集成 | 审核按钮渲染条件: `isAdmin && !adminAuthFailed && !adminAuthDegraded` |

**任务 2.3: 新增 i18n 翻译键**

| 项 | 内容 |
|---|------|
| 文件 | `ghost/i18n/locales/en/comments.json` |
| 新增键 | 8 个 (见 3.3 节) |
| 命令 | `pnpm --filter @tryghost/i18n translate` |
| 注意 | `context.json` 必须添加描述 |

#### 阶段 3: P2 任务详情 (可选)

**任务 3.1: 跨会话刷新机制**

| 项 | 内容 |
|---|------|
| 方案 A | `visibilitychange` 事件触发刷新<br>用户切回页面时自动刷新评论列表 |
| 方案 B | 配置开关控制的轮询<br>`pollingInterval: 30000` (默认关闭) |
| 注意 | 需避免与 firstCommentCreatedAt 锚点冲突 |

---

## 第五部分: 验收清单

### 5.1 功能验收

#### P0 验收 (必须)

```
□ P0-1: callApi 超时机制
   □ 调用 callApi 后 10 秒无响应，Promise 应 reject
   □ 超时后 handlers 对象中对应 uid 应被清理
   □ 正常响应 (5 秒内) 不触发超时
   □ postMessage 抛出异常时正确 reject

□ P0-2: message-handler 错误响应
   □ origin 不匹配时，前端应收到 {error: 'Origin mismatch'}
   □ action 不匹配时，前端应收到 {error: 'Unknown action'}
   □ 正常请求不受影响

□ P0-3: initAdminAuth 超时
   □ adminApi.getUser() 15 秒无响应，应标记 adminAuthFailed = true
   □ 正常获取用户时不受影响
   □ 失败后控制台有警告日志
```

#### P1 验收 (建议)

```
□ P1-1: 可重试错误自动重试
   □ 网络错误: 自动重试 3 次 (1s, 2s, 4s)
   □ 5xx 错误: 自动重试 2 次 (2s, 4s)
   □ 429 错误: 按 Retry-After 等待
   □ 重试成功后清除失败计数
   □ 重试耗尽后触发降级

□ P1-2: 降级状态机
   □ 连续 3 次失败: adminAuthDegraded = true
   □ 降级后: adminApi = null
   □ 降级后: 审核按钮隐藏
   □ 页面刷新后重新尝试初始化
   □ 401/403 立即进入 adminAuthFailed

□ P1-3: 用户提示
   □ 降级后显示 Toast: "审核功能暂时不可用"
   □ 401 显示 Toast: "请先登录后台以使用审核功能"
   □ 403 显示 Toast: "您没有审核评论的权限"
   □ 配置错误显示 Toast: "审核功能配置异常，请联系站长"
   □ 所有提示有对应的 i18n 翻译
```

#### P2 验收 (可选)

```
□ P2-1: visibilitychange 刷新
   □ 页面从后台切回前台时刷新评论
   □ 刷新时重置 firstCommentCreatedAt

□ P2-2: 错误上报
   □ 降级事件上报到监控系统
   □ 包含错误类型、次数、用户角色
```

### 5.2 回归测试

```
□ 同会话审核功能
   □ hideComment 成功路径
   □ showComment 成功路径
   □ 评论计数正确更新

□ 跨会话一致性
   □ 页面刷新后看到最新审核状态
   □ 切换排序后看到最新审核状态
   □ (已知限制) 加载更多不影响已加载页

□ 普通用户功能 (无影响)
   □ 发表评论
   □ 点赞/取消点赞
   □ 编辑/删除自己的评论
   □ 加载更多评论
   □ 切换排序
```

### 5.3 性能验收

```
□ 超时机制不影响正常请求
   □ 正常请求延迟增加 < 1ms

□ 重试机制不增加服务器负担
   □ 指数退避 + Jitter 生效
   □ 最大重试次数限制生效

□ 内存泄漏检查
   □ 1000 次超时后 handlers 数量稳定
   □ 无 EventListener 泄漏
```

### 5.4 安全验收

```
□ Origin 校验仍生效
   □ 恶意 origin 无法调用 admin API
   □ 但会返回错误响应 (不再静默)

□ 接收方 origin 校验
   □ 伪造响应仍被忽略
   □ 依赖超时机制兜底
```

---

## 第六部分: 风险与缓解措施

### 6.1 风险清单

| ID | 风险 | 等级 | 影响 | 缓解措施 |
|----|------|------|------|---------|
| R1 | 超时值设置不当 | 中 | 正常请求被误判为失败，或静默问题仍存在 | 10s 默认 + 可配置 + 监控超时发生率 |
| R2 | 重试风暴 (Retry Storm) | 高 | 服务器短暂故障时雪崩 | 指数退避 + 最大 3 次 + Jitter |
| R3 | 版本不兼容 | 中 | 旧前端 + 新后端，或反之 | 错误格式一致 + 超时兜底 |
| R4 | 管理员困惑 | 低 | 不知道审核按钮为何消失 | Toast 提示 + 控制台日志 |
| R5 | handlers 泄漏 | 中 | 内存泄漏 | 超时必清理 + 数量限制 + 监控 |
| R6 | i18n 同步 | 低 | 非英文用户看到英文 | fallback 到英文 + CI 检查 |

### 6.2 回滚方案

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            回滚方案                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景 1: P0 上线后发现问题                                                │
│  └── 回滚: 直接 revert 相关 commit                                       │
│      └── 影响: 回到静默失败状态 (当前生产状态)                            │
│                                                                         │
│  场景 2: P1 上线后发现问题                                                │
│  └── 回滚: revert P1 相关 commit                                         │
│      └── 保留 P0 的超时机制                                              │
│                                                                         │
│  场景 3: 重试风暴                                                         │
│  └── 临时关闭: 配置开关设置重试次数为 0                                   │
│      └── 保留超时机制                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 附录: 关键文件索引

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 超时机制改造 | `apps/comments-ui/src/utils/admin-api.ts` | 33 |
| 静默失败改造 | `ghost/core/core/frontend/src/admin-auth/message-handler.js` | 40-74 |
| 降级状态管理 | `apps/comments-ui/src/app.tsx` | 122-178 (initAdminAuth) |
| hideComment | `apps/comments-ui/src/actions.ts` | 145-177 |
| showComment | `apps/comments-ui/src/actions.ts` | 179-215 |
| setOrder | `apps/comments-ui/src/actions.ts` | 34-56 |
| loadMoreComments | `apps/comments-ui/src/actions.ts` | 6-26 |
| firstCommentCreatedAt | `apps/comments-ui/src/utils/admin-api.ts` | 7, 68-93 |
| 翻译文件 | `ghost/i18n/locales/en/comments.json` | - |
| 角色校验 | `apps/comments-ui/src/app.tsx` | 21, 137 |

---

**文档版本历史**

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-05-11 | 初始版本，基于 R1-R4 汇总整理 |
