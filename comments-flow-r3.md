# Ghost 评论系统链路深度分析 (R3)

> 本文档聚焦两个问题的失败恢复视角：
> 1. 审核状态变更回推：同会话 vs 跨会话的详细时序
> 2. 跨域认证失败：可重试与不可重试分类 + 统一兜底方案

---

## 一、审核状态变更回推时序分析

### 1.1 审核状态变更的两种场景

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          审核状态变更触发点                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  场景 A: 同会话审核 (Same-Session Moderation)                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  管理员 A 在浏览器中：                                               │   │
│  │  1. 打开帖子页面，评论区加载                                          │   │
│  │  2. admin-auth iframe 加载，验证管理员身份                            │   │
│  │  3. 管理员点击某条评论的 "Hide" 按钮                                  │   │
│  │  4. 状态变更应即时反映在同一页面                                        │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  场景 B: 跨会话审核 (Cross-Session Moderation)                             │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  管理员 A 在后台管理页面 (另一浏览器/标签页)                           │   │
│  │  1. 进入 /ghost/#/comments                                           │   │
│  │  2. 筛选、查看评论列表                                                │   │
│  │  3. 点击某条评论的 "Hide" 按钮                                        │   │
│  │                                                                            │
│  │  访客 B 在帖子页面：                                                    │   │
│  │  1. 页面已打开，评论区已加载                                            │   │
│  │  2. 评论内存中仍显示被隐藏的评论                                        │   │
│  │  3. 需要某个触发点才能拉取最新状态                                      │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、场景 A：同会话审核详细时序

### 2.1 初始化时序

```
时序: 管理员打开帖子页面

T0: 用户滚动到评论区（或页面有 permalink）
     │
     ▼
T1: IntersectionObserver 触发 (or permalink 直接加载)
     │
     ├──▶ initSetup()
     │     │
     │     ├──▶ api.init()
     │     │     ├──▶ api.member.sessionData() ── 验证会员身份 (Cookie)
     │     │     └──▶ api.site.settings() ── 加载实验室配置
     │     │
     │     ├──▶ fetchComments()
     │     │     ├──▶ api.comments.browse({page:1}) ── 第1页评论
     │     │     └──▶ api.comments.count() ── 评论总数
     │     │
     │     └──▶ (如果有 permalink)
     │           ├──▶ fetchScrollTarget(commentId)
     │           │     └──▶ api.comments.read(commentId)
     │           │          └── 检查 status === 'published'
     │           │
     │           └──▶ paginateToComment() / loadScrollTarget()
     │                 └──▶ 多页加载 + 展开回复
     │
     T2: initSetup 完成
     │   state.initStatus = 'success'
     │   state.comments = [...]
     │
     └──▶ AuthFrame 加载 (仅当 comments.length > 0)
           │
           ├──▶ <iframe data-frame="admin-auth" src={adminUrl + 'auth-frame/'}>
           │
           └──▶ onLoad → initAdminAuth()
                 │
                 ├──▶ adminApi = setupAdminAPI({adminUrl})
                 │     │
                 │     ├──▶ 注册 postMessage listener
                 │     └──▶ 等待 iframe 准备好
                 │
                 ├──▶ adminApi.getUser() ── postMessage 到 admin-auth iframe
                 │     │
                 │     ├── message-handler.js 验证 origin
                 │     ├── fetch /api/admin/users/me/?include=roles
                 │     └── 返回用户信息
                 │
                 ├──▶ 检查角色: ALLOWED_MODERATORS
                 │     ├── Owner / Administrator / Super Editor ✓
                 │     └── Author / Contributor / Editor ✗
                 │
                 └──▶ (如果是管理员)
                       ├──▶ adminApi.browse({page:1})
                       │     └── 通过 admin-auth 重新拉取评论
                       │         （因为管理员可以看到 hidden 状态）
                       │
                       └── state.admin = {...}, state.adminApi = {...}
```

### 2.2 隐藏评论时序（同会话）

```
时序: 管理员在站点评论区点击 "Hide comment"

T0: 管理员看到评论 C1，点击右侧菜单 → "Hide comment"
     │
     └──▶ dispatchAction('hideComment', {id: C1.id})
           │
           ▼
T1: actions.ts: hideComment()
     │
     ├──▶ state.adminApi.hideComment(C1.id)
     │     │
     │     ├── admin-api.ts: callApi('hideComment', {id: C1.id})
     │     │     │
     │     │     ├── uid = 2
     │     │     ├── handlers[2] = (error, result) => {resolve/reject}
     │     │     │
     │     │     └── postMessage to admin-auth iframe
     │     │         {uid: 2, action: 'hideComment', id: C1.id}
     │     │
     │     ├── message-handler.js (admin-auth iframe)
     │     │     │
     │     │     ├── 校验 origin: event.origin === siteOrigin ✓
     │     │     ├── JSON.parse 成功 ✓
     │     │     ├── id 校验: /^[a-f0-9]{24}$/i ✓
     │     │     │
     │     │     ├── fetch(`/api/admin/comments/${C1.id}/`, {
     │     │     │     method: 'PUT',
     │     │     │     body: {comments: [{id: C1.id, status: 'hidden'}]}
     │     │     │   })
     │     │     │   (同源，自动携带 Admin Cookie)
     │     │     │
     │     │     ├── 后端处理
     │     │     │   ├── comments-controller.js: edit()
     │     │     │   ├── models.Comment.edit({status: 'hidden'})
     │     │     │   ├── 数据库更新
     │     │     │   └── 设置 X-Cache-Invalidate 头
     │     │     │
     │     │     ├── 响应 JSON 解析
     │     │     │
     │     │     └── postMessage 回 comments-ui
     │     │         {uid: 2, error: null, result: {...}}
     │     │
     │     ├── admin-api.ts: message listener
     │     │     │
     │     │     ├── 校验 origin: event.origin === adminOrigin ✓
     │     │     ├── handlers[2](null, result)
     │     │     ├── delete handlers[2]
     │     │     └── Promise.resolve(result)
     │     │
     │     └── hideComment Promise 完成
     │
T2: API 调用成功，更新本地状态
     │
     └── return {
           comments: state.comments.map((c) => {
               if (c.id === C1.id) {
                   return {...c, status: 'hidden'}
               }
               return c;
           }),
           commentCount: state.commentCount - 1
       }
     │
T3: React 重新渲染
     │
     ├── C1 显示为 "Hidden for members" 灰色状态
     ├── 评论总数减 1
     └── 操作完成（用户感知：即时）
```

**一致性**: **强一致** (同一会话内，API 成功后立即更新本地状态)

### 2.3 显示评论时序（同会话）

```
时序: 管理员点击 "Show comment"

T0: 管理员点击被隐藏评论的 "Show comment"
     │
     └──▶ dispatchAction('showComment', {id: C1.id})
           │
           ▼
T1: actions.ts: showComment()
     │
     ├──▶ state.adminApi.showComment({id: C1.id})
     │     │
     │     └── [同上，PUT status: 'published']
     │
T2: 关键区别：重新拉取单条评论
     │
     └── (显示操作需要重新拉取，因为：
         1. 需要确保 HTML 内容是最新的
         2. 关系数据要以当前会员身份加载，而非管理员身份)
     │
     ├──▶ state.adminApi.read({commentId: C1.id, memberUuid: ...})
     │     │
     │     └── callApi('readComment', {commentId: C1.id, params: 'impersonate_member_uuid=...'})
     │
     └── const updatedComment = data.comments[0]
     │
T3: 用完整数据替换本地状态
     │
     └── return {
           comments: state.comments.map((c) => {
               if (c.id === C1.id) {
                   return updatedComment;  // 完整替换
               }
               return c;
           }),
           commentCount: state.commentCount + 1
       }
     │
T4: React 重新渲染
     │
     ├── C1 恢复正常显示
     ├── 评论总数加 1
     └── 操作完成
```

### 2.4 同会话触发点汇总

| 操作 | 触发方法 | 数据来源 | 一致性 |
|------|---------|---------|--------|
| 初始化加载 | `initSetup()` → `fetchComments()` | `api.comments.browse(page=1)` | 强一致 |
| 管理员身份加载 | `initAdminAuth()` → `adminApi.getUser()` | `admin-auth` 代理 | 强一致 |
| 管理员重新拉取 | `adminApi.browse(page=1)` | Admin API (可看 hidden) | 强一致 |
| 隐藏评论 | `hideComment()` | API 成功后本地更新 | 强一致 |
| 显示评论 | `showComment()` → `adminApi.read()` | 重新拉取单条 | 强一致 |
| 切换排序 | `setOrder()` → `api.comments.browse(order=...)` | 重新拉取第一页 | 强一致 |
| 加载更多 | `loadMoreComments()` | `api.comments.browse(page=N)` | 追加，强一致 |

---

## 三、场景 B：跨会话审核详细时序

### 3.1 问题核心

```
关键洞察: firstCommentCreatedAt 分页锚点

apps/comments-ui/src/utils/api.ts:36
┌─────────────────────────────────────────────────────────────────────────┐
│  // To fix pagination when we create new comments (or people post       │
│  // comments after you loaded the page), we need to only load          │
│  // comments created AFTER the page load                                │
│  let firstCommentCreatedAt: null | string = null;                      │
└─────────────────────────────────────────────────────────────────────────┘

影响: 
- 首次加载时记录 "最旧评论的 created_at" (实际是第一条，因为按时间倒序)
- 后续 loadMore 只加载 created_at <= firstCommentCreatedAt 的评论
- 新评论（包括审核通过的评论）不会出现在分页加载中
- 只有切换排序 order 时，firstCommentCreatedAt 才会重置
```

### 3.2 跨会话隐藏评论时序

```
时序: 管理员 A 在后台隐藏评论，访客 B 页面保持打开

T0 (访客 B): 打开帖子页面
     │
     ├──▶ 评论区加载: [C1, C2, C3, C4, C5] (page 1, 最新5条)
     ├──▶ firstCommentCreatedAt = C1.created_at (锚点)
     └──▶ 内存中: C1-C5 都是 published
     │
T1 (管理员 A): 在后台管理页面点击 "Hide" C2
     │
     ├──▶ useHideComment mutation
     │     ├── PUT /api/admin/comments/C2/
     │     │   body: {comments: [{id: C2, status: 'hidden'}]}
     │     │
     │     └── 后端:
     │         ├── 数据库更新 C2.status = 'hidden'
     │         ├── X-Cache-Invalidate 头
     │         └── invalidateQueries: {dataType: 'CommentsResponseType'}
     │
     └──▶ react-query 自动失效并重新拉取后台列表
         └── 管理员 A 的页面: C2 从列表中消失
     │
T2 (访客 B): 内存状态仍为 [C1, C2, C3, C4, C5] (无任何变化)
     │
     ├── 没有 WebSocket 通知
     ├── 没有 SSE 推送
     ├── 没有轮询
     └── 只有用户主动操作才会触发重新拉取
     │
T3 (访客 B): 点击 "Load more" 加载下一页
     │
     ├──▶ api.comments.browse({page: 2})
     │     ├── filter: created_at:<=firstCommentCreatedAt
     │     └── 返回 [C6, C7, C8, C9, C10]
     │
     └──▶ dedupedComments = [...state.comments, ...newComments]
         └── 内存: [C1, C2, C3, C4, C5, C6, C7, C8, C9, C10]
         └── C2 仍然在列表中！
     │
     问题: C2 虽然在数据库中是 hidden，但访客 B 的内存中仍显示
     │
T4 (访客 B): 点击 C2 的 "Like" 按钮
     │
     ├──▶ dispatchAction('likeComment', C2)
     │     │
     │     ├── 乐观更新本地: C2.liked = true, count.likes += 1
     │     │
     │     └──▶ api.comments.like({comment: C2})
     │           │
     │           ├── POST /api/members/comments/C2/like
     │           │
     │           └── 后端:
     │               ├── 查找评论 C2
     │               ├── 检查 status: 'hidden' 对普通会员不可见
     │               └── 返回 404 或 403
     │
     └──▶ catch 块触发
           ├── dispatchAction('updateCommentLikeState', {id: C2.id, liked: false})
           └── C2.liked = false, count.likes -= 1 (回滚)
     │
     此时 C2 仍在列表中，但点赞失败（用户困惑）
     │
T5 (访客 B): 刷新页面 (F5)
     │
     ├──▶ initSetup() 重新执行
     │     ├── fetchComments()
     │     │   └── api.comments.browse(page=1)
     │     │       └── 后端过滤: status NOT IN ('hidden', 'deleted')
     │     │
     │     └── 返回 [C1, C3, C4, C5, C6] (C2 消失)
     │
     └──▶ state.comments = [C1, C3, C4, C5, C6]
         └── 终于一致了
```

### 3.3 跨会话显示评论时序

```
时序: 管理员 A 在后台显示评论，访客 B 页面保持打开

T0 (访客 B): 打开帖子页面
     │
     ├──▶ 评论区加载: [C1, C3, C4, C5]
     │     (C2 已被隐藏，不在列表中)
     ├──▶ firstCommentCreatedAt = C1.created_at
     └──▶ 内存中没有 C2
     │
T1 (管理员 A): 在后台点击 "Show" C2
     │
     ├──▶ useShowComment mutation
     │     ├── PUT /api/admin/comments/C2/
     │     │   body: {comments: [{id: C2, status: 'published'}]}
     │     │
     │     └── 后端: C2.status = 'published'
     │
     └──▶ 管理员 A 的页面: C2 重新出现
     │
T2 (访客 B): 内存状态不变，没有 C2
     │
T3 (访客 B): 点击 "Load more"
     │
     ├──▶ api.comments.browse({page: 2})
     │     ├── filter: created_at:<=firstCommentCreatedAt
     │     │
     │     └── 假设 C2.created_at 在 C1 之前（因为 C2 是更早的评论）
     │         C2 应该出现在第 2 页或更后
     │
     └──▶ 如果 C2 在第 2 页，会正常显示
         如果 C2 在第 1 页（被隐藏后空缺），不会出现在新加载的数据中
     │
T4 (访客 B): 切换排序 (Newest → Best)
     │
     ├──▶ setOrder({order: 'count__likes desc, ...'})
     │     │
     │     ├── firstCommentCreatedAt 重置 (因为有 order 参数)
     │     └──▶ api.comments.browse({page: 1, order: '...'})
     │           │
     │           └── 重新拉取第一页，按点赞排序
     │
     └──▶ 如果 C2 有足够的点赞，会出现在第一页
         否则需要翻页才能看到
     │
T5 (访客 B): 刷新页面
     │
     └──▶ C2 出现在正确的位置
```

### 3.4 跨会话触发点与回拉条件

| 触发事件 | 回拉范围 | firstCommentCreatedAt 重置? | 能否发现审核变更? |
|---------|---------|---------------------------|-----------------|
| 页面刷新 (F5) | 完整重新加载 | 是 | ✓ 能 |
| 切换排序方式 | 第一页重新拉取 | 是 (order 参数存在) | ✓ 能 |
| 点击 "Load more" | 下一页追加 | 否 (使用锚点) | 部分 (仅新页数据) |
| 自己发表评论 | 本地 unshift + API 返回 | 否 | 仅新评论，不影响旧评论 |
| 自己点赞/取消点赞 | 乐观更新 | 否 | ✗ 不能 |
| 自己编辑评论 | 本地替换 | 否 | ✗ 不能 |
| IntersectionObserver 重新触发 | 否 (已 observe 过) | - | ✗ 不能 |

### 3.5 分页锚点机制的深层问题

```
代码分析: apps/comments-ui/src/utils/api.ts:129-133

browse({page, postId, order}) {
    let filter = null;
    if (firstCommentCreatedAt && !order) {
        filter = `created_at:<=${firstCommentCreatedAt}`;
    }
    // ...
}

关键:
1. 只有当 !order 时才使用锚点
2. 切换排序方式 (order 参数存在) 会绕过锚点
3. 锚点记录的是首次加载时"最旧评论"的时间（但实际上是第一条，因为倒序）

问题场景:
- 管理员显示了一条"古老"的评论 C_old
- C_old.created_at < firstCommentCreatedAt (因为 C_old 更早)
- 访客点击 "Load more" 时，filter = created_at:<=firstCommentCreatedAt
- C_old 会被包含在结果中 ✓
- 但如果 C_old 在第一页的空缺位置，不会被发现 ✗

另一个问题场景:
- 管理员隐藏了 C1 (第一页第一条)
- 访客点击 "Load more"
- 新评论 C6 被添加到列表末尾
- C1 仍在内存中，不会被移除
```

---

## 四、跨域认证失败分类与恢复策略

### 4.1 跨域认证架构回顾

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        跨域认证消息流                                      │
│                                                                         │
│  comments-ui iframe (site.com)                                          │
│       │                                                                 │
│       │ 1. postMessage({uid, action, ...}, adminOrigin)                 │
│       ▼                                                                 │
│  admin-auth iframe (admin.com)                                          │
│       │                                                                 │
│       │ 2. 校验 origin → 解析 JSON → 校验 action → 校验 id              │
│       │                                                                 │
│       │ 3. fetch(`/api/admin/...`, {credentials: 'include'})            │
│       │    (同源，自动携带 Admin Cookie)                                 │
│       │                                                                 │
│       │ 4. 响应处理 → JSON 解析                                         │
│       │                                                                 │
│       │ 5. postMessage({uid, error, result}, siteOrigin)                │
│       ▼                                                                 │
│  comments-ui iframe                                                     │
│       │                                                                 │
│       │ 6. 校验 origin → 查找 handlers[uid] → resolve/reject            │
│       └─────────────────────────────────────────────────────────────────┘
```

### 4.2 失败路径全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     admin-auth (接收方) 失败点                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  F1. origin 不匹配 (message-handler.js:41)                              │
│      ┌── 触发: event.origin !== siteOrigin                              │
│      ├── 行为: console.warn + return (静默忽略)                          │
│      ├── 响应: 无 postMessage 返回                                       │
│      └── 后果: 前端 Promise 永久 pending, handlers[uid] 泄漏            │
│                                                                         │
│  F2. 消息解析失败 (message-handler.js:46-52)                             │
│      ├── 触发: JSON.parse(event.data) 失败                               │
│      ├── 行为: console.error + return                                    │
│      ├── 响应: 无 postMessage 返回                                       │
│      └── 后果: 同 F1                                                     │
│                                                                         │
│  F3. action 不匹配 (message-handler.js:62-65)                           │
│      ├── 触发: actions[data.action] === undefined                        │
│      ├── 行为: return (静默忽略)                                          │
│      ├── 响应: 无 postMessage 返回                                       │
│      └── 后果: 同 F1                                                     │
│                                                                         │
│  F4. 标识符校验失败 (message-handler.js:6-11)                            │
│      ├── 触发: !/^[a-f0-9]{24}$/i.test(value)                           │
│      ├── 行为: throw new Error('Invalid identifier')                     │
│      ├── 响应: postMessage({uid, error: 'Invalid identifier', result: null}) │
│      └── 后果: 前端 Promise reject (正常错误路径)                        │
│                                                                         │
│  F5. fetch 网络失败 (message-handler.js:67-73)                           │
│      ├── 触发: 网络中断、DNS 失败、CORS 问题                              │
│      ├── 行为: catch → respond(err, null)                                │
│      ├── 响应: postMessage({uid, error: err.message, result: null})     │
│      └── 后果: 前端 Promise reject                                       │
│                                                                         │
│  F6. HTTP 非 2xx (message-handler.js:67-73)                              │
│      ├── 触发: res.ok === false (401, 403, 404, 500 等)                 │
│      ├── 行为: res.json() 可能成功或失败                                  │
│      ├── 响应: 取决于 res.json() 是否抛出                                 │
│      └── 后果: 前端 Promise reject (如果 json 解析成功)                  │
│                                                                         │
│  F7. JSON 解析失败 (message-handler.js:69)                               │
│      ├── 触发: res.json() 抛出 (非 JSON 响应)                            │
│      ├── 行为: catch → respond(err, null)                                │
│      ├── 响应: postMessage({uid, error: err.message, result: null})     │
│      └── 后果: 前端 Promise reject                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     comments-ui (发送方) 失败点                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  F8. origin 不匹配 (admin-api.ts:9-13)                                   │
│      ├── 触发: event.origin !== adminOrigin                             │
│      ├── 行为: return (静默忽略)                                          │
│      └── 后果: 如果是伪造响应，真实 handler 不会被调用 → 泄漏             │
│                                                                         │
│  F9. 消息解析失败 (admin-api.ts:15-20)                                   │
│      ├── 触发: JSON.parse(event.data) 失败                               │
│      ├── 行为: return                                                    │
│      └── 后果: 同 F1 (handler 泄漏)                                      │
│                                                                         │
│  F10. uid 不匹配 (admin-api.ts:22-26)                                    │
│      ├── 触发: handlers[data.uid] === undefined                          │
│      ├── 行为: return                                                    │
│      └── 后果: 可能是伪造消息，也可能是时序问题                          │
│                                                                         │
│  F11. iframe 未加载 (admin-api.ts:2)                                     │
│      ├── 触发: setupAdminAPI() 时 iframe 还不存在                        │
│      ├── 行为: frame.contentWindow 可能为 null                           │
│      └── 后果: postMessage 抛出异常，Promise reject                     │
│                                                                         │
│  F12. 管理员角色校验失败 (app.tsx:137-139)                               │
│      ├── 触发: admin.roles 不在 ALLOWED_MODERATORS                       │
│      ├── 行为: admin = null                                              │
│      └── 后果: 不显示审核按钮，但不影响普通评论功能                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 可重试 vs 不可重试失败分类

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        可重试失败 (Retryable)                             │
│                                                                         │
│  定义: 可能是临时问题，重试可能成功                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F5. fetch 网络失败                                              │   │
│  │  ├── 原因: 网络抖动、临时 DNS 失败、连接超时                      │   │
│  │  ├── 特征: 间歇性，可能下一次就成功                                │   │
│  │  └── 建议: 指数退避重试 (3次: 1s, 2s, 4s)                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F6. HTTP 5xx (服务器错误)                                       │   │
│  │  ├── 原因: 后端临时过载、数据库连接问题                            │   │
│  │  ├── 特征: 500, 502, 503, 504                                   │   │
│  │  └── 建议: 重试 (注意: 500 可能是代码 bug，需要谨慎)              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F11. iframe 未加载 (时序问题)                                    │   │
│  │  ├── 原因: setupAdminAPI() 调用早于 iframe onLoad                 │   │
│  │  ├── 特征: 仅发生在初始化阶段                                      │   │
│  │  └── 建议: 等待 iframe load 事件后再调用                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F6. HTTP 429 (Too Many Requests)                                │   │
│  │  ├── 原因: 速率限制                                               │   │
│  │  ├── 特征: Retry-After 头可能存在                                  │   │
│  │  └── 建议: 按 Retry-After 等待后重试                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                       不可重试失败 (Non-Retryable)                        │
│                                                                         │
│  定义: 配置问题或权限问题，重试也不会成功                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F1. origin 不匹配                                               │   │
│  │  ├── 原因: 站点配置错误、部署问题、潜在攻击                        │   │
│  │  ├── 特征: 永久性，直到配置修复                                    │   │
│  │  └── 建议: 降级为普通会员模式 (无审核能力)                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F2/F3/F8/F9/F10. 消息格式/路由问题                               │   │
│  │  ├── 原因: 版本不匹配、消息篡改                                    │   │
│  │  ├── 特征: 消息协议层问题                                          │   │
│  │  └── 建议: 降级 + 错误上报 (可能是 bug)                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F4. 标识符校验失败                                               │   │
│  │  ├── 原因: 无效的 ID 格式、潜在注入尝试                            │   │
│  │  ├── 特征: 输入参数问题                                            │   │
│  │  └── 建议: 不重试，记录错误 (可能是攻击)                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F6. HTTP 4xx (客户端错误)                                       │   │
│  │  ├── 401: 未登录 → 引导用户登录                                   │   │
│  │  ├── 403: 无权限 → 降级为普通会员                                 │   │
│  │  ├── 404: 资源不存在 → 不重试                                     │   │
│  │  └── 400: 请求格式错误 → 不重试 (可能是 bug)                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F7. JSON 解析失败                                               │   │
│  │  ├── 原因: 后端返回非 JSON (如 HTML 错误页)                       │   │
│  │  ├── 特征: 后端异常响应                                           │   │
│  │  └── 建议: 降级 + 上报 (后端 bug)                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  F12. 角色校验失败                                               │   │
│  │  ├── 原因: 管理员登录但角色不够 (Author, Editor 等)                │   │
│  │  ├── 特征: 业务权限问题                                            │   │
│  │  └── 建议: 降级为普通会员模式 (设计如此)                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.4 现有超时与降级策略分析

#### 4.4.1 现有实现检查

```
检查结果: 当前实现几乎没有容错机制

┌─────────────────────────────────────────────────────────────────────────┐
│  admin-api.ts: callApi()                                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  function callApi(action, args) {                                       │
│      return new Promise((resolve, reject) => {                          │
│          // 问题 1: 没有超时机制                                         │
│          // 如果 origin 不匹配，Promise 永远 pending                    │
│                                                                         │
│          function handler(error, result) {                              │
│              if (error) return reject(error);  // 问题 2: 没有重试逻辑  │
│              return resolve(result);                                    │
│          }                                                              │
│                                                                         │
│          uid += 1;                                                      │
│          handlers[uid] = handler;  // 问题 3: 失败时泄漏                │
│                                                                         │
│          frame.contentWindow!.postMessage(...);  // 问题 4: 没有 try/catch │
│      });                                                                │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  app.tsx: initAdminAuth()                                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  const initAdminAuth = async () => {                                    │
│      try {                                                              │
│          const adminApi = setupAdminAPI({adminUrl});                    │
│                                                                         │
│          try {                                                          │
│              admin = await adminApi.getUser();  // 问题: 没有超时        │
│              // 如果 origin 不匹配，这里永远卡住                         │
│          } catch (e) {                                                  │
│              // 只记录 warn，没有降级逻辑                                 │
│              console.warn(`[Comments] Failed to fetch admin endpoint:`, e);
│          }                                                              │
│                                                                         │
│      } catch (e) {                                                      │
│          console.error(`[Comments] Failed to initialize admin auth:`, e);
│          // 没有设置 fallback state                                      │
│      }                                                                  │
│  };                                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  actions.ts: hideComment()                                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  async function hideComment({state, data: comment}) {                  │
│      if (state.adminApi) {                                              │
│          await state.adminApi.hideComment(comment.id);  // 没有 try/catch │
│          // 如果 Promise 永久 pending，这里永远卡住                       │
│      }                                                                  │
│      return {...};  // 永远不会执行                                       │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.4.2 点赞操作的部分容错

```
点赞是唯一有回滚机制的操作:

// actions.ts: likeComment()
async function likeComment({api, data: comment, dispatchAction}) {
    // 先乐观更新
    dispatchAction('updateCommentLikeState', {id: comment.id, liked: true});
    
    try {
        await api.comments.like({comment});
        return {};
    } catch {
        // 失败回滚
        dispatchAction('updateCommentLikeState', {id: comment.id, liked: false});
    }
}

问题:
1. 回滚只针对点赞状态，不处理其他操作
2. 没有重试逻辑
3. 没有超时处理
```

### 4.5 统一兜底方案设计

#### 4.5.1 建议的 callApi 增强

```typescript
// 建议的增强版本

function callApi(action: string, args?: any, options: {
    timeout?: number;
    maxRetries?: number;
} = {}): Promise<any> {
    const timeout = options.timeout ?? 10000;  // 10秒默认超时
    const maxRetries = options.maxRetries ?? 0;  // 默认不重试
    
    return new Promise((resolve, reject) => {
        let attempt = 0;
        
        const execute = () => {
            const uid = ++uidCounter;
            
            // 超时定时器
            const timer = setTimeout(() => {
                delete handlers[uid];
                if (attempt < maxRetries && isRetryableError(new Error('Timeout'))) {
                    attempt++;
                    execute();  // 重试
                } else {
                    reject(new Error(`Admin API call timed out after ${timeout}ms`));
                }
            }, timeout);
            
            handlers[uid] = (error, result) => {
                clearTimeout(timer);
                delete handlers[uid];
                
                if (error) {
                    if (attempt < maxRetries && isRetryableError(error)) {
                        attempt++;
                        // 指数退避
                        setTimeout(execute, Math.pow(2, attempt) * 1000);
                    } else {
                        reject(error);
                    }
                } else {
                    resolve(result);
                }
            };
            
            try {
                frame.contentWindow!.postMessage(
                    JSON.stringify({uid, action, ...args}),
                    adminOrigin
                );
            } catch (e) {
                clearTimeout(timer);
                delete handlers[uid];
                reject(e);
            }
        };
        
        execute();
    });
}

// 可重试错误判断
function isRetryableError(error: any): boolean {
    const message = error?.message?.toLowerCase() || '';
    
    // 网络错误
    if (message.includes('network') || 
        message.includes('fetch') ||
        message.includes('timeout')) {
        return true;
    }
    
    return false;
}
```

#### 4.5.2 降级策略

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        降级策略矩阵                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  触发条件:                                                               │
│  - initAdminAuth() 超时或失败                                            │
│  - adminApi 调用连续失败 N 次                                            │
│  - origin 配置明显错误                                                   │
│                                                                         │
│  降级状态:                                                               │
│  state.admin = null                                                     │
│  state.adminApi = null                                                  │
│  state.isAdmin = false                                                  │
│  state.adminAuthFailed = true (新增)                                    │
│                                                                         │
│  UI 表现:                                                                │
│  ├── 不显示 "Hide" / "Show" 按钮                                        │
│  ├── 管理员仍可使用普通会员功能 (评论、点赞)                              │
│  ├── 可显示小提示: "审核功能暂不可用"                                    │
│  └── 不影响普通访客体验                                                  │
│                                                                         │
│  恢复机制:                                                               │
│  └── 页面刷新时重新尝试初始化                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.5.3 完整的错误处理流程图

```
用户点击 "Hide comment"
        │
        ▼
dispatchAction('hideComment', comment)
        │
        ▼
actions.ts: hideComment()
        │
        ├── state.adminApi 存在?
        │   ├── No ──▶ (不应发生，按钮不显示)
        │   └── Yes
        │           │
        │           ▼
        │   adminApi.hideComment(comment.id)
        │           │
        │           ├──┬── 成功 ──▶ 更新本地状态 ──▶ 完成 ✓
        │           │  │
        │           │  └── 失败 ──┬── 可重试?
        │           │            ├── Yes ──▶ 指数退避重试
        │           │            │         ├── 成功 ──▶ 完成 ✓
        │           │            │         └── 耗尽重试 ──▶ 降级?
        │           │            │                    ├── Yes ──▶ 降级状态
        │           │            │                    └── No ──▶ 显示错误提示
        │           │            └── No ──┬── 权限问题 (403) ──▶ 降级
        │           │                      ├── 未登录 (401) ──▶ 提示登录
        │           │                      └── 其他 ──▶ 显示错误提示
        │           │
        │           └── 超时 ──┬── 可重试?
        │                      ├── Yes ──▶ 指数退避重试
        │                      └── No ──▶ 降级 + 提示
        │
        └── (adminApi 为 null) ──▶ 静默忽略 (按钮本应隐藏)
```

---

## 五、总结与建议

### 5.1 审核状态变更回推关键发现

| 维度 | 发现 |
|------|------|
| 同会话 | 强一致，API 成功后立即更新本地状态 |
| 跨会话 | 最终一致，依赖页面刷新或排序切换 |
| 分页锚点 | `firstCommentCreatedAt` 导致"加载更多"不会发现审核变更 |
| 触发点 | 只有 F5 刷新、切换排序会完整重新拉取 |

### 5.2 跨域认证关键发现

| 类别 | 问题数量 | 严重程度 |
|------|---------|---------|
| 可重试失败 | 4 种 (F5, F6-5xx, F11, F6-429) | 中 |
| 不可重试失败 | 8 种 (F1-F4, F6-4xx, F7-F10, F12) | 高 |
| 静默失败导致泄漏 | 5 种 (F1-F3, F8-F9) | 高 |
| 无超时机制 | 全部 | 高 |
| 无降级策略 | 全部 | 中 |

### 5.3 改进建议优先级

#### P0 - 立即修复

1. **添加超时机制**: 所有 `callApi()` 调用添加 10 秒超时
2. **清理泄漏 handlers**: 超时时删除 `handlers[uid]`
3. **静默失败返回错误**: origin/action 不匹配时应发送错误响应

#### P1 - 高优先级

1. **可重试错误自动重试**: 网络错误、5xx 使用指数退避
2. **降级策略实现**: 连续失败 N 次后进入降级模式
3. **initAdminAuth 超时**: 管理员初始化设置超时 (15秒)

#### P2 - 中优先级

1. **跨会话刷新机制**: 添加可选的轮询或 SSE
2. **firstCommentCreatedAt 优化**: 审核变更时考虑锚点更新
3. **更友好的错误 UI**: 失败时显示可理解的提示

#### P3 - 低优先级

1. **消息协议版本化**: 避免前端/后端版本不匹配
2. **错误上报集成**: 失败时上报到监控系统
3. **操作级别的重试策略**: 不同操作设置不同的重试次数

---

## 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 初始化流程 | `apps/comments-ui/src/app.tsx:262` |
| 分页锚点 | `apps/comments-ui/src/utils/api.ts:36, 131-133` |
| 审核操作 | `apps/comments-ui/src/actions.ts:145` (hide), `apps/comments-ui/src/actions.ts:179` (show) |
| Admin API 调用 | `apps/comments-ui/src/utils/admin-api.ts:33` |
| message-handler | `ghost/core/core/frontend/src/admin-auth/message-handler.js` |
| Admin hooks | `apps/admin-x-framework/src/api/comments.ts:99` |
