# Ghost 评论系统链路分析报告

## 1. 系统架构概览

Ghost 评论系统采用**前端 iframe 跨域嵌入 + 后端服务分层**的架构设计：

### 1.1 组件层次

```
┌─────────────────────────────────────────────────────────────────┐
│                      访客浏览器 (站点域名)                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  主题页面 (https://site.com/posts/slug)                  │    │
│  │    ┌──────────────────────────────────────────────┐     │    │
│  │    │  <iframe title="comments-frame">             │     │    │
│  │    │  ┌─────────────────────────────────────┐    │     │    │
│  │    │  │  comments-ui (React 应用)            │    │     │    │
│  │    │  │  - 状态管理 (useState/useReducer)    │    │     │    │
│  │    │  │  - API 封装 (api.ts, admin-api.ts)   │    │     │    │
│  │    │  └─────────────────────────────────────┘    │     │    │
│  │    └──────────────────────────────────────────────┘     │    │
│  │    ┌──────────────────────────────────────────────┐     │    │
│  │    │  <iframe data-frame="admin-auth"> (隐藏)     │     │    │
│  │    │  ┌─────────────────────────────────────┐    │     │    │
│  │    │  │  admin-auth message-handler.js       │    │     │    │
│  │    │  │  (跨域代理 Admin API 请求)            │    │     │    │
│  │    │  └─────────────────────────────────────┘    │     │    │
│  │    └──────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Ghost 后端服务                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  API 层                                                   │    │
│  │  - Members API: /api/members/comments/*                  │    │
│  │  - Admin API:   /api/admin/comments/*                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Controller 层 (comments-controller.js)                  │    │
│  │  - browse/read/add/edit/like/unlike/report               │    │
│  │  - adminBrowse/adminBrowseAll/adminReplies               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Service 层 (comments-service.js)                        │    │
│  │  - 业务逻辑：权限检查、状态流转、通知发送                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Model 层 (comment.js)                                   │    │
│  │  - 数据持久化、HTML 清理、关联查询                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  数据库: comments / comment_likes / comment_reports       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 评论提交流程

### 2.1 完整链路

```
访客在 iframe 中输入评论
        │
        ▼
comments-ui Form 组件 (main-form.tsx / reply-form.tsx)
        │
        │ dispatchAction('addComment', {html, parent_id?, in_reply_to_id?})
        ▼
actions.ts: addComment handler
        │
        │ api.comments.add({comment})
        ▼
api.ts: comments.add()
        │
        │ POST /api/members/comments/
        │ credentials: 'same-origin' (携带站点 Cookie)
        ▼
Members API 中间件 (Cookie 认证会员身份)
        │
        ▼
comments-controller.js: add(frame)
        │
        ├─> #checkMember(frame) - 验证会员身份
        │    检查 frame.options.context.member.id
        │
        ├─> 区分顶级评论 / 回复
        │
        └─> comments-service.js: commentOnPost() / replyToComment()
                 │
                 ├─> checkEnabled() - 检查评论功能是否开启
                 │
                 ├─> checkCommentAccess() - 检查会员权限
                 │    (免费会员在 paid-only 模式下被拒绝)
                 │
                 ├─> checkPostAccess() - 检查帖子访问权限
                 │    (内容门控检查)
                 │
                 ├─> models.Comment.add()
                 │    │
                 │    └─> comment.js: onSaving()
                 │         ├─> sanitize-html 清理 (仅允许 <p>, <br>, <a>, <blockquote>)
                 │         ├─> 链接自动添加 rel="ugc noopener noreferrer nofollow"
                 │         └─> trimParagraphs() 清理空段落
                 │
                 ├─> sendNewCommentNotifications()
                 │    ├─> 通知帖子作者
                 │    └─> 通知被回复的评论作者
                 │
                 ├─> DomainEvents.dispatch(MemberCommentEvent)
                 │
                 └─> 返回结果，设置 X-Cache-Invalidate 头
                      ├─> /api/members/comments/post/{postId}/
                      └─> /api/members/comments/{parentId}/replies/
```

### 2.2 关键代码位置

- 前端表单提交: `apps/comments-ui/src/actions.ts`
- 前端 API 封装: `apps/comments-ui/src/utils/api.ts`
- 后端控制器: `ghost/core/core/server/services/comments/comments-controller.js:236`
- 后端服务: `ghost/core/core/server/services/comments/comments-service.js:309`
- 模型层: `ghost/core/core/server/models/comment.js:24`

---

## 3. 跨域评论组件的会员身份认证

### 3.1 架构挑战

comments-ui 通过 iframe 嵌入在主题页面中，面临两个跨域问题：

1. **iframe 与父页面跨域**: iframe 通常从 Ghost 核心域名加载，但可能与主题域名不同
2. **Admin API 跨域**: Admin 接口在独立域名/路径下，CORS 限制导致无法直接传递 Cookie

### 3.2 会员认证机制 (Same-Origin Cookie)

对于普通会员操作（浏览、评论、点赞、举报），使用**同源 Cookie 认证**：

```
┌──────────────────────────────────────────────────────────────┐
│  浏览器                                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  主题页面 (https://site.com/posts/slug)                  │  │
│  │    ┌────────────────────────────────────────────────┐  │  │
│  │    │  iframe: https://site.com/ghost/comments-ui/   │  │  │
│  │    │  (同源，可访问站点 Cookie)                        │  │  │
│  │    │                                                │  │  │
│  │    │  api.ts: makeRequest({credentials: 'same-origin'})│  │
│  │    │        │                                         │  │  │
│  │    │        ▼                                         │  │  │
│  │    │  GET /api/members/session/                       │  │  │
│  │    │  (携带 ghost-members-ssr Cookie)                 │  │  │
│  │    │        │                                         │  │  │
│  │    │        ▼                                         │  │  │
│  │    │  Members API 中间件                              │  │  │
│  │    │  -> 解析 Cookie -> 获取会员 ID                   │  │  │
│  │    │  -> frame.options.context.member.id             │  │  │
│  │    └────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**关键实现**:

```typescript
// apps/comments-ui/src/utils/api.ts:58
member: {
    identity() {
        const url = endpointFor({type: 'members', resource: 'session'});
        return makeRequest({
            url,
            credentials: 'same-origin'  // 关键：携带同源 Cookie
        }).then(function (res) {
            // ...
        });
    },
    sessionData() {
        const url = endpointFor({type: 'members', resource: 'member'});
        return makeRequest({
            url,
            credentials: 'same-origin'  // 关键：携带同源 Cookie
        });
    }
}
```

### 3.3 管理员认证机制 (postMessage 代理)

对于管理员操作（隐藏/显示评论、浏览隐藏评论），使用**隐藏 iframe + postMessage 消息代理**：

```
┌──────────────────────────────────────────────────────────────────┐
│  浏览器                                                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  comments-ui iframe (站点域名)                              │ │
│  │                                                            │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  auth-frame.tsx:                                    │  │ │
│  │  │  <iframe data-frame="admin-auth"                     │  │ │
│  │  │   src="{adminUrl}auth-frame/">                      │  │ │
│  │  │  (display: none, 隐藏的 admin 域名 iframe)          │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │                           │                                │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  admin-api.ts: callApi('hideComment', {id})          │  │ │
│  │  │                                                       │  │ │
│  │  │  window.postMessage({                                 │  │ │
│  │  │    uid: 123,                                          │  │ │
│  │  │    action: 'hideComment',                             │  │ │
│  │  │    id: 'comment-id'                                   │  │ │
│  │  │  }, adminOrigin)                                      │  │ │
│  │  └───────────────────┬─────────────────────────────────┘  │ │
│  │                      │ postMessage                         │ │
│  │                      ▼                                     │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  admin-auth iframe (admin 域名同源)                   │  │ │
│  │  │                                                       │  │ │
│  │  │  message-handler.js:                                  │  │ │
│  │  │  window.addEventListener('message', handler)          │  │ │
│  │  │                                                       │  │ │
│  │  │  // 同源，可访问 Admin Cookie                         │  │ │
│  │  │  fetch(`${adminUrl}/comments/${id}/`, {               │  │ │
│  │  │    method: 'PUT',                                     │  │ │
│  │  │    credentials: 'include'  // 携带 Admin Cookie       │  │ │
│  │  │  })                                                   │  │ │
│  │  │       │                                               │  │ │
│  │  │       │ response                                      │  │ │
│  │  │       ▼                                               │  │ │
│  │  │  event.source.postMessage({                           │  │ │
│  │  │    uid: 123,                                          │  │ │
│  │  │    result: {...}                                      │  │ │
│  │  │  }, siteOrigin)                                       │  │ │
│  │  └───────────────────┬─────────────────────────────────┘  │ │
│  │                      │ postMessage (响应)                   │ │
│  │                      ▼                                     │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  admin-api.ts: handlers[uid](error, result)          │  │ │
│  │  │  -> Promise resolve/reject                           │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

**关键实现**:

1. **前端代理**: `apps/comments-ui/src/utils/admin-api.ts:33`
   - 生成唯一 uid 标识请求
   - 通过 `postMessage` 发送到 admin-auth iframe
   - 等待响应消息，通过 handlers 回调映射

2. **后端代理**: `ghost/core/core/frontend/src/admin-auth/message-handler.js:40`
   - 验证消息来源 (origin 白名单)
   - 执行实际的 Admin API 请求 (同源，自动携带 Admin Cookie)
   - 通过 `postMessage` 返回结果

3. **管理员角色检查**: `apps/comments-ui/src/app.tsx:137`
   ```typescript
   const ALLOWED_MODERATORS = ['Owner', 'Administrator', 'Super Editor'];
   
   // 只有特定角色可以审核评论
   if (!admin || !(admin.roles.some(role => ALLOWED_MODERATORS.includes(role.name)))) {
       admin = null;
   }
   ```

### 3.4 模拟会员上下文 (Impersonation)

管理员在浏览评论时，需要知道"当前登录的会员"是否已经点赞等状态。通过 `impersonate_member_uuid` 参数：

```typescript
// apps/comments-ui/src/utils/admin-api.ts:83
if (memberUuid) {
    params.set('impersonate_member_uuid', memberUuid);
}

// ghost/core/core/server/services/comments/comments-controller.js:28
async #setImpersonationContext(options) {
    if (options.impersonate_member_uuid) {
        options.context = options.context || {};
        options.context.member = options.context.member || {};
        options.context.member.id = await this.service.getMemberIdByUUID(options.impersonate_member_uuid);
    }
}
```

---

## 4. 后台审核流程

### 4.1 审核入口

审核有两个入口：

1. **独立评论管理页面**: `apps/posts/src/views/comments/comments.tsx`
   - 路由: `/ghost/#/comments`
   - 支持筛选: 状态、日期、内容、帖子、作者、是否被举报

2. **评论区实时审核**: comments-ui 中的管理员上下文菜单
   - 仅对 `Owner` / `Administrator` / `Super Editor` 显示
   - 可直接隐藏/显示评论

### 4.2 状态流转

评论有三种状态：

| 状态 | 值 | 说明 |
|------|-----|------|
| 已发布 | `published` | 对所有访客可见 |
| 已隐藏 | `hidden` | 仅管理员可见，会员看到 "Hidden for members" |
| 已删除 | `deleted` | 软删除，仅数据库保留 |

**模型层过滤逻辑**: `ghost/core/core/server/models/comment.js:59`

```javascript
applyCustomQuery(options) {
    const excludedStatuses = options.isAdmin 
        ? ['deleted']           // 管理员: 能看到 hidden，看不到 deleted
        : ['hidden', 'deleted']; // 普通会员: hidden 和 deleted 都看不到
    // ...
}
```

### 4.3 审核操作链路

```
管理员点击 "Hide" / "Show" 按钮
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  路径 1: 站点评论区 (comments-ui)                             │
│                                                             │
│  admin-api.ts: hideComment(id) / showComment({id})          │
│         │                                                   │
│         ▼ postMessage                                       │
│  admin-auth iframe message-handler.js                       │
│         │                                                   │
│         ▼                                                   │
│  PUT /api/admin/comments/{id}/                              │
│  { comments: [{ id, status: 'hidden'|'published' }] }       │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  路径 2: 后台管理页面 (apps/posts)                           │
│                                                             │
│  useHideComment / useShowComment mutation                   │
│  (admin-x-framework/src/api/comments.ts:98)                 │
│         │                                                   │
│         ▼                                                   │
│  PUT /api/admin/comments/{id}/                              │
│  { comments: [{ id, status: 'hidden'|'published' }] }       │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
ghost/core/core/server/api/endpoints/comments.js:47 (edit)
        │
        │ models.Comment.edit({ id, status }, options)
        │
        │ 设置 X-Cache-Invalidate 响应头
        │ - /api/members/comments/post/{postId}/
        │ - /api/members/comments/{parentId}/replies/
        ▼
前端缓存失效，重新拉取评论列表
```

---

## 5. 审核状态变更推回前端的实时刷新机制

### 5.1 核心机制: HTTP 响应头 + 缓存失效

Ghost 评论系统**不使用 WebSocket**进行实时推送，而是采用**缓存失效 + 前端主动刷新**策略：

### 5.2 缓存失效链路

```
后端修改评论 (add/edit/like/unlike)
        │
        ▼
comments-controller.js 设置响应头
        │
        │ frame.setHeader('X-Cache-Invalidate', paths.join(', '))
        │
        │ 示例:
        │ X-Cache-Invalidate: /api/members/comments/post/post_123/, 
        │                     /api/members/comments/comment_456/replies/
        ▼
┌─────────────────────────────────────────────────────────────┐
│  场景 1: 同一会话操作 (如会员自己评论)                         │
│                                                             │
│  actions.ts: 操作完成后直接更新本地状态                       │
│  - 新增评论: unshift 到 comments 数组                       │
│  - 删除评论: filter 移除                                     │
│  - 点赞: 更新 liked 状态和 count.likes                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  场景 2: 跨会话操作 (如管理员在后台审核)                       │
│                                                             │
│  前端没有实时连接，依赖:                                      │
│  1. 页面刷新时重新拉取                                        │
│  2. 评论区组件重新挂载时重新拉取                              │
│  3. 分页加载时获取最新数据                                    │
│                                                             │
│  注意: 这是"最终一致性"模型，不是强实时                        │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 前端状态管理

comments-ui 使用 React Context + useReducer 风格的 dispatch 模式：

```typescript
// apps/comments-ui/src/app-context.ts
export type EditableAppContext = {
    comments: Comment[],           // 评论列表 (本地状态)
    pagination: {...},             // 分页信息
    commentCount: number,          // 评论总数
    // ...
}

// apps/comments-ui/src/actions.ts
// 每个操作后同步更新本地状态，避免网络请求
async function addComment(...) {
    // 1. 发送 API 请求
    const result = await api.comments.add({comment});
    
    // 2. 乐观更新本地状态
    return {
        comments: [newComment, ...state.comments],
        commentCount: state.commentCount + 1
    };
}
```

### 5.4 后台审核页面的实时性

后台评论管理页面使用 `@tanstack/react-query`：

```typescript
// apps/admin-x-framework/src/api/comments.ts:98
export const useHideComment = createMutation({
    method: 'PUT',
    path: ({id}) => `/comments/${id}/`,
    invalidateQueries: {
        dataType: 'CommentsResponseType'  // 自动失效相关查询
    }
});
```

当管理员修改评论状态后，react-query 会：
1. 使 `CommentsResponseType` 类型的所有查询失效
2. 自动重新拉取评论列表
3. UI 无缝更新

---

## 6. 垃圾过滤与内容安全

### 6.1 多层防护机制

```
访客提交评论
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  第一层: 会员准入控制                                         │
│                                                             │
│  comments-service.js:                                       │
│  ├─> checkEnabled() - 评论功能全局开关                       │
│  ├─> checkCommentAccess() - 付费会员检查                     │
│  │   (comments_enabled === 'paid' && member.status === 'free')│
│  └─> checkPostAccess() - 内容门控检查                        │
│      (contentGating.checkPostAccess)                        │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  第二层: HTML 清理 (保存时)                                   │
│                                                             │
│  comment.js: onSaving()                                     │
│  ├─> sanitize-html 白名单过滤                                │
│  │   允许标签: <p>, <br>, <a>, <blockquote>                  │
│  │   允许属性: a[href, target, rel]                          │
│  │                                                           │
│  ├─> 链接安全改造                                            │
│  │   transformTags: a -> 自动添加                            │
│  │     - target="_blank"                                    │
│  │     - rel="ugc noopener noreferrer nofollow"             │
│  │       (ugc = 用户生成内容标记，SEO 提示)                   │
│  │                                                           │
│  └─> trimParagraphs() - 清理空段落                          │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  第三层: 会员举报机制                                         │
│                                                             │
│  comments-service.js: reportComment()                       │
│  ├─> 检查是否已举报 (防止重复)                                │
│  ├─> 保存 comment_reports 记录                               │
│  └─> 发送邮件通知站点管理员                                   │
│                                                             │
│  后台筛选: count.reports:>0                                 │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  第四层: 管理员人工审核                                       │
│                                                             │
│  - 隐藏/显示评论                                             │
│  - 禁用特定会员的评论权限 (member.can_comment = false)        │
│  - 垃圾邮箱域名黑名单 (blocked_email_domains)               │
│    └─> apps/admin-x-settings/src/components/settings/       │
│        advanced/spam-filters.tsx                            │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 垃圾邮箱域名黑名单

```typescript
// apps/admin-x-settings/src/components/settings/advanced/spam-filters.tsx:28
const updateBlockedEmailDomainsSetting = (e) => {
    const input = e.target.value;
    const validEmailDomains = input
        .split(/[\s,]+/)                    // 按空格/逗号/换行分割
        .map(domain => 
            domain.trim()
                  .toLowerCase()
                  .split('@').pop()        // 提取域名部分
        )
        .filter(domain => 
            domain && domain.includes('.')  // 验证格式
        );
    
    updateSetting('blocked_email_domains', JSON.stringify(validEmailDomains));
};
```

**注意**: 黑名单阻止的是会员**注册**，不是评论提交。被封锁域名的邮箱无法注册成为会员，自然无法评论。

---

## 7. 多语言场景处理

### 7.1 国际化架构

评论组件使用 Ghost 集中式 i18n 系统：

```
┌─────────────────────────────────────────────────────────────┐
│  i18n 文件结构                                               │
│                                                             │
│  ghost/i18n/locales/                                        │
│  ├── en/                                                    │
│  │   └── comments.json           (英文源文件)               │
│  ├── zh/                                                    │
│  │   └── comments.json           (中文翻译)                 │
│  ├── zh-Hant/                                               │
│  │   └── comments.json           (繁体中文)                 │
│  └── ... (60+ 其他语言)                                     │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 评论组件初始化

```typescript
// apps/comments-ui/src/app.tsx:110
const i18n = useMemo(() => {
    return i18nLib(options.locale, 'comments');  // namespace = 'comments'
}, [options.locale]);

// 使用
context.t('Add comment');
context.t('{amount} comments', {amount: 5});
```

### 7.3 语言来源

评论组件的 `locale` 从脚本标签的 data 属性获取：

```html
<!-- 主题页面注入的脚本 -->
<script 
    src="https://site.com/ghost/comments-ui/comments-ui.min.js"
    data-locale="zh"
    data-post-id="..."
    data-site-url="..."
    data-admin-url="..."
></script>
```

### 7.4 翻译键管理

**规则** (来自 AGENTS.md):
1. **不拆分句子**: 永远不要把一个句子拆到多个 `t()` 调用中
2. **提供上下文描述**: 每个新键必须在 `context.json` 中有描述
3. **使用插值**: 动态值用 `{variable}` 语法
4. **内联元素用 `<tag>`**: 配合 `@doist/react-interpolate`

**评论组件翻译键示例** (`ghost/i18n/locales/en/comments.json`):

```json
{
    "{amount} comments": "",
    "Add comment": "",
    "Add reply": "",
    "Hidden for members": "",
    "Hide comment": "",
    "Show comment": "",
    "Report this comment?": "",
    "You can't post comments in this publication. <a>Contact support</a> for more information.": ""
}
```

### 7.5 翻译流程

```bash
# 提取翻译键并更新所有语言文件
pnpm --filter @tryghost/i18n translate

# 验证翻译
pnpm --filter @tryghost/i18n lint:translations
```

---

## 8. 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 评论前端 UI | `apps/comments-ui/src/` |
| 前端 API 封装 (会员) | `apps/comments-ui/src/utils/api.ts` |
| 前端 API 封装 (管理员) | `apps/comments-ui/src/utils/admin-api.ts` |
| 前端状态管理 | `apps/comments-ui/src/actions.ts` |
| Admin 认证代理 | `ghost/core/core/frontend/src/admin-auth/message-handler.js` |
| 评论控制器 | `ghost/core/core/server/services/comments/comments-controller.js` |
| 评论服务 | `ghost/core/core/server/services/comments/comments-service.js` |
| 评论模型 | `ghost/core/core/server/models/comment.js` |
| Members API 路由 | `ghost/core/core/server/api/endpoints/comments-members.js` |
| Admin API 路由 | `ghost/core/core/server/api/endpoints/comments.js` |
| 后台评论管理 | `apps/posts/src/views/comments/` |
| 后台评论 API hooks | `apps/admin-x-framework/src/api/comments.ts` |
| 垃圾过滤设置 | `apps/admin-x-settings/src/components/settings/advanced/spam-filters.tsx` |
| 评论翻译文件 | `ghost/i18n/locales/*/comments.json` |

---

## 9. 总结

### 9.1 设计亮点

1. **跨域认证优雅**: 通过隐藏 iframe + postMessage 代理解决 Admin API 跨域问题，无需复杂的 OAuth 流程
2. **分层清晰**: Controller → Service → Model 三层架构，职责明确
3. **内容安全**: sanitize-html 白名单 + rel 安全属性 + 会员举报 + 人工审核多层防护
4. **最终一致性**: 乐观更新 + 缓存失效头，在没有 WebSocket 的情况下实现良好的用户体验
5. **权限模型精细**: 支持全局开关 / 付费会员限制 / 内容门控 / 单会员禁用多级控制

### 9.2 注意事项

1. **非强实时**: 审核状态变更依赖页面刷新或重新拉取，不是 WebSocket 级别的实时推送
2. **状态软删除**: `deleted` 状态的数据仍保留在数据库中
3. **回复层级限制**: 只允许两级结构 (评论 → 回复)，不允许回复的回复 (`replyToReply` 错误)
4. **管理员角色硬编码**: `ALLOWED_MODERATORS` 数组在代码中写死，不可配置
