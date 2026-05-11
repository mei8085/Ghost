# Ghost 站点级导入导出迁移机制分析

## 一、一键导出的快照范围

### 1.1 导出功能定位

Ghost 的站点级导出是 **纯数据库快照导出**，**不包含任何图片、附件或媒体文件**。

**代码证据：**

- 导出核心 `ghost/core/core/server/data/exporter/exporter.js:33-68` 仅执行数据库表查询
- API 端点 `ghost/core/core/server/api/endpoints/db.js:39-77` 的 `exportContent` 直接返回 JSON 数据
- 前端调用 `apps/admin-x-framework/src/api/db.ts:19` 通过 `downloadFromEndpoint('/db/')` 下载 JSON 文件

### 1.2 何时需要 Zip 资源包

| 场景 | 导出格式 | 是否包含资源 |
|------|----------|-------------|
| 一键导出（Admin UI "Content & settings" 按钮） | 单个 `.json` 文件 | ❌ 仅数据库数据 |
| 跨实例迁移（包含图片） | 需要 **手动打包** `images/` + `media/` + `files/` + `.json` 为 `.zip` | ✅ 需用户手动准备 |
| 导入支持 | `.json` 或 `.zip` | ✅ zip 可包含资源目录 |

**关键结论：**

1. **导出侧** - Ghost 官方导出功能 **不提供 zip 资源包导出**，只输出 JSON 数据库快照
2. **导入侧** - 支持两种格式：
   - 纯 JSON 文件（仅结构数据迁移）
   - ZIP 压缩包（JSON + `images/` + `media/` + `files/` 目录）

### 1.3 默认导出的数据库表 (TABLES_ALLOWLIST)

位于 `ghost/core/core/server/data/exporter/table-lists.js:69-90`，共 21 张表：

**内容核心：**
- `posts` - 文章/页面（含 html、mobiledoc、feature_image 等字段）
- `posts_authors` - 文章-作者关联
- `posts_meta` - 文章元数据
- `posts_tags` - 文章-标签关联
- `posts_products` - 文章-产品关联
- `tags` - 标签定义

**用户与权限：**
- `users` - 用户数据（含 profile_image、cover_image 字段）
- `roles` - 角色定义
- `roles_users` - 用户-角色关联

**站点配置：**
- `settings` - 站点设置（敏感项已过滤）
- `custom_theme_settings` - 主题自定义设置

**会员与商业化：**
- `products` - 产品/套餐
- `stripe_products` - Stripe 产品映射
- `stripe_prices` - Stripe 价格配置
- `newsletters` - 通讯
- `benefits` - 权益
- `products_benefits` - 产品-权益关联
- `offers` - 优惠码
- `offer_redemptions` - 优惠码兑换记录
- `snippets` - 代码片段

### 1.4 可选备份表 (BACKUP_TABLES)

通过 API 的 `include` 参数可额外导出 64 张表，位于 `table-lists.js:2-64`：

**安全与认证类：**
- `api_keys`, `integrations`, `invites`, `tokens`, `sessions`, `brute`, `actions`

**会员与订阅类：**
- `members`, `members_labels`, `members_products`, `members_stripe_customers`, `subscriptions`
- 各类会员事件表（取消、支付、登录、邮箱变更、状态变更等）
- `members_newsletters`, `members_click_events`, `members_feedback`

**邮件与通讯类：**
- `emails`, `email_batches`, `email_recipients`, `email_recipient_failures`
- `email_design_settings`, `automated_email_recipients`
- 欢迎邮件自动化相关表

**内容历史与互动：**
- `mobiledoc_revisions`, `post_revisions`
- `comments`, `comment_likes`, `comment_reports`, `mentions`

**社交与推荐：**
- `recommendations`, `recommendation_click_events`, `recommendation_subscribe_events`, `outbox`

**其他：**
- `donation_payment_events`, `suppressions`, `email_spam_complaint_events`
- `milestones`, `collections`, `collections_posts`
- `gifts`, `labels`, `redirects`, `jobs`
- `migrations`, `migrations_lock`, `permissions`, `webhooks`

### 1.5 敏感配置过滤 (SETTING_KEYS_BLOCKLIST)

导出时自动过滤以下设置项，位于 `table-lists.js:93-104`：

| 设置键 | 过滤原因 |
|--------|----------|
| `stripe_connect_publishable_key` | Stripe 凭证 |
| `stripe_connect_secret_key` | Stripe 凭证 |
| `stripe_connect_account_id` | Stripe 凭证 |
| `stripe_secret_key` | Stripe 凭证 |
| `stripe_publishable_key` | Stripe 凭证 |
| `stripe_billing_portal_configuration_id` | Stripe 配置 |
| `members_stripe_webhook_id` | Webhook 配置 |
| `members_stripe_webhook_secret` | Webhook 密钥 |
| `email_verification_required` | 运行时状态 |
| `indexnow_api_key` | 第三方服务密钥 |

### 1.6 导出格式

**API 响应结构：**
```json
{
  "db": [{
    "meta": {
      "exported_on": 1699999999999,
      "version": "5.75.0"
    },
    "data": {
      "posts": [...],
      "users": [...],
      "tags": [...],
      "settings": [...],
      ...
    }
  }]
}
```

**文件名格式：** `{站点标题}.ghost.{YYYY-MM-DD-HH-mm-ss}.json`

---

## 二、导入系统架构与主流程

### 2.1 核心组件：Handlers vs Importers

导入管理器 `ImportManager` 位于 `ghost/core/core/server/data/importer/import-manager.js`，包含两套组件：

**Handlers（文件加载器）** - 负责从 zip 或文件中读取原始数据，输出结构化的 `importData`：

| Handler | type | 处理内容 | 输出 |
|---------|------|----------|------|
| `JSONHandler` | `data` | `.json` 文件 | 解析后的数据库结构 `{meta, data}` |
| `MarkdownHandler` | `data` | `.md` 文件 | 转换为 `{meta: {}, data: {posts: []}}` |
| `RevueHandler` | `revue` | Revue 导出 csv/json | `{meta: {revue: true}, revue: {...}}` |
| `ImageHandler` | `images` | 图片文件 | 文件元数据数组 `[{name, path, originalPath, newPath, targetDir}]` |
| `mediaHandler` | `media` | 媒体文件 | 同上格式 |
| `filesHandler` | `files` | 文件附件 | 同上格式 |

**Importers（数据导入器）** - 负责将 `importData` 中的数据实际写入 Ghost：

| Importer | type | 职责 |
|----------|------|------|
| `ContentFileImporter` (x3) | `images`/`media`/`files` | 保存资源文件到存储，预处理路径替换 |
| `RevueImporter` | `revue` | 将 Revue 数据转换为 Ghost 格式 |
| `DataImporter` | `data` | 协调 11 个子 importer 导入数据库结构 |

### 2.2 完整导入主流程

`import-manager.js:494-552` 的 `importFromFile()` 主流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Step 1: loadFile()                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Zip 解压 → 遍历所有 Handlers → 生成 importData          │   │
│  │ importData = {                                          │   │
│  │   data: {meta: {...}, data: {posts: [...], ...}},       │   │
│  │   images: [{name, path, originalPath, newPath, ...}],   │   │
│  │   media: [...],                                         │   │
│  │   files: [...]                                          │   │
│  │ }                                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Step 2: preProcess()                        │
│  按顺序遍历所有 Importers:                                       │
│  ① imageImporter.preProcess → 替换 posts/tags/users 中的图片引用│
│  ② mediaImporter.preProcess → 替换 posts 中的媒体引用           │
│  ③ filesImporter.preProcess → 替换 posts 中的文件引用           │
│  ④ RevueImporter.preProcess → 将 revue 数据转为 Ghost 格式      │
│  ⑤ DataImporter.preProcess → 仅标记 preProcessedByData=true     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Step 3: doImport()                          │
│  按顺序遍历所有 Importers:                                       │
│  ① imageImporter.doImport → 保存图片到存储 ❌ 无事务保护         │
│  ② mediaImporter.doImport → 保存媒体到存储 ❌ 无事务保护         │
│  ③ filesImporter.doImport → 保存附件到存储 ❌ 无事务保护         │
│  ④ RevueImporter.doImport → 空操作 (数据已在 preProcess 转换)   │
│  ⑤ DataImporter.doImport → 数据库事务内导入结构数据 ✅ 事务保护  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Step 4-6: 收尾                               │
│  generateReport() → cleanUp() → 发送邮件通知                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Step 1: loadFile() 详解

**Zip 处理流程** (`import-manager.js:304-340`):

1. 解压到临时目录 `os.tmpdir()/randomUUID/`
2. 按 Handlers 数组顺序遍历：
   - `ImageHandler` 匹配 `images/` 或 `content/` 目录下的图片
   - `mediaHandler` 匹配 `media/` 或 `content/` 目录下的媒体
   - `filesHandler` 匹配 `files/` 或 `content/` 目录下的附件
   - `RevueHandler` 检测是否有 `issues*.csv`
   - `JSONHandler` 匹配 `.json` 文件
   - `MarkdownHandler` 匹配 `.md/.markdown` 文件
3. 每种 type 只能有一个 handler 匹配（否则抛 `zipContainsMultipleDataFormats` 错误）

**Handler 的 loadFile 做什么？**

以 `ImageHandler` (`handlers/image.js:14-45`) 为例：
- **不保存文件**，只做路径规划
- 调用 `storage.getUniqueFileName()` 生成不冲突的目标文件名
- 计算 `newPath`（最终 URL 路径）和 `targetDir`（存储目录）
- 返回文件元数据数组供后续使用

### 2.4 Step 2: preProcess() 详解

**关键：资源路径替换发生在此阶段**

`ContentFileImporter.preProcess()` (`importers/content-file-importer.js:74-110`):

```javascript
// images 类型的处理逻辑
if (importData.images && importData.data && importData.data.data) {
    _.each(importData.images, function (image) {
        preProcessPosts(importData.data.data, image);  // 替换 posts 中的引用
        preProcessTags(importData.data.data, image);   // 替换 tags 中的引用
        preProcessUsers(importData.data.data, image);  // 替换 users 中的引用
    });
}
```

**替换逻辑** (`content-file-importer.js:7-16`):

```javascript
replaceImage = function (markdown, image) {
    const regex = new RegExp('(/)?' + image.originalPath, 'gm');
    return markdown.replace(regex, image.newPath);
};
```

**替换范围：**

| 数据类型 | 字段 |
|----------|------|
| posts | `markdown`, `html`, `feature_image` |
| tags | `feature_image` |
| users | `profile_image`, `cover_image` |

### 2.5 Step 3: doImport() 详解

**串行执行机制** (`import-manager.js:411-424`):

```javascript
async doImport(importData, importOptions) {
    for (const importer of this.importers) {
        if (Object.prototype.hasOwnProperty.call(importData, importer.type)) {
            importResults[importer.type] = await importer.doImport(
                importData[importer.type], importOptions);
        }
    }
    return importResults;
}
```

**关键事实：**
- `for...of` + `await` 串行执行
- 前序 importer 抛错 → 循环立即停止 → 后续 importer 不会执行
- 错误抛出到外层 try-catch

**执行顺序** (`import-manager.js:101`):

```javascript
this.importers = [
    imageImporter,        // ① 保存图片
    mediaImporter,        // ② 保存媒体
    contentFilesImporter, // ③ 保存附件
    RevueImporter,        // ④ 空操作
    DataImporter          // ⑤ 数据库导入
];
```

**失败中断逻辑：**

```
① imageImporter.doImport()
    ├─ 成功 → 继续 ②
    └─ 失败 → throw → ②③④⑤ 全部跳过 → DataImporter 不执行

② mediaImporter.doImport()
    ├─ 成功 → 继续 ③
    └─ 失败 → throw → ③④⑤ 全部跳过 → DataImporter 不执行

③ filesImporter.doImport()
    ├─ 成功 → 继续 ④
    └─ 失败 → throw → ④⑤ 全部跳过 → DataImporter 不执行

④ RevueImporter.doImport() → 空操作，继续 ⑤

⑤ DataImporter.doImport() → 只有 ①②③④ 都成功才执行
```

**结论：**资源阶段任意一步失败，数据库导入根本不会开始**

**①-③ 资源文件保存** (`content-file-importer.js:117-125`):

```javascript
doImport(contentFilesData) {
    return Promise.all(contentFilesData.map(function (contentFile) {
        return store.save(contentFile, contentFile.targetDir);  // 直接写入存储
    }));
}
```

- 无事务保护
- 无回滚机制
- 失败后已保存的文件保留

**⑤ DataImporter 数据库导入** - 见下一节详细分析

---

## 三、DataImporter 的标识重映射机制

### 3.1 DataImporter 内部架构

`DataImporter.doImport()` (`importers/data/data-importer.js:50-198`) 协调 11 个子 importer：

```
执行顺序（由初始化顺序决定）:
1. UsersImporter       → 导入用户
2. RolesImporter       → 导入角色
3. TagsImporter        → 导入标签
4. NewslettersImporter → 导入通讯
5. SettingsImporter    → 导入设置
6. ProductsImporter    → 导入产品
7. StripeProductsImporter → Stripe 产品映射
8. StripePricesImporter   → Stripe 价格
9. PostsImporter       → 导入文章（最复杂的关联处理）
10. CustomThemeSettingsImporter → 主题设置
11. RevueSubscriberImporter → Revue 订阅者
```

### 3.2 数据库事务边界

**所有子 importer 的操作都在同一个事务内** (`data-importer.js:124-197`):

```javascript
return models.Base.transaction(async function (transacting) {
    modelOptions.transacting = transacting;

    // 所有操作共享同一个 transacting
    await sequence(ops);  // 顺序执行所有子 importer

    if (errors.length > 0) {
        throw errors;  // 触发事务回滚
    }

    return { ... };
});
```

### 3.3 每个子 importer 的执行流程

在事务内，每个子 importer 按以下顺序执行 (`data-importer.js:127-152`):

```
1. fetchExisting()    → 从数据库读取已有数据（用于冲突检测和匹配）
2. beforeImport()     → 数据清洗、ID 重新生成、关联构建
3. 依赖注入           → requiredImportedData / requiredExistingData
4. replaceIdentifiers() → 外键引用重映射
5. doImport()         → 实际写入数据库
```

### 3.4 ID 重新生成

**在 `beforeImport()` 阶段**，所有对象获得新的 ObjectId。

`BaseImporter.beforeImport()` → `generateIdentifier()` (`importers/data/base.js:89-99`):

```javascript
generateIdentifier() {
    _.each(this.dataToImport, (obj) => {
        const newId = ObjectId().toHexString();

        if (obj.id) {
            // 保存 新ID → 原始ID 的映射
            this.originalIdMap[newId] = obj.id;
        }

        obj.id = newId;  // 替换为新 ID
    });
}
```

**导入后的数据映射** (`base.js:309-317`):
```javascript
mapImportedData(originalObject, importedObject) {
    return {
        id: importedObject.id,              // 新 ID
        originalId: this.originalIdMap[importedObject.id],  // 原始 ID
        slug: importedObject.get('slug'),
        originalSlug: originalObject.slug,
        email: importedObject.get('email')  // 用户特有
    };
}
```

`importedData` 数组保存这些映射，供后续 importer 做关联匹配。

### 3.5 用户引用重映射

**在 `replaceIdentifiers()` 阶段**处理用户引用。

`BaseImporter.replaceIdentifiers()` (`base.js:184-304`) 处理字段：
- `author_id`
- `published_by`

**匹配策略（优先级从高到低）：**

| 优先级 | 匹配方式 | 说明 |
|--------|----------|------|
| 1 | 空值检查 | 引用为空 → fallback 到 Owner |
| 2 | 文件内匹配 | `requiredFromFile.users` 中按 ID 查找 |
| 3 | 已导入匹配 | `requiredImportedData.users` 中按 **email** 查找（slug 可能变化） |
| 4 | 数据库匹配 | `requiredExistingData.users` 中先按 slug、再按 ID 查找 |
| 5 | 最终 fallback | 都找不到 → 使用 Owner 用户 ID |

### 3.6 文章关系重映射

`PostsImporter.replaceIdentifiers()` (`posts-importer.js:116-214`) 处理三类关联：

**1. 标签关联 (tags):**
```
posts_tags[tag_id] → 原始 tag ID
                      ↓
         ① 匹配 requiredFromFile.tags (导入文件中的 tags)
         ② 匹配 requiredImportedData.tags (已导入的 tags，按 originalId)
         ③ 匹配 requiredExistingData.tags (数据库已有 tags，按 slug)
         ④ 都找不到 → 移除此关联
```

**2. 作者关联 (authors):**
```
posts_authors[author_id] → 原始 user ID
                           ↓
         ① 匹配 requiredFromFile.users
         ② 匹配 requiredImportedData.users (按 originalId)
         ③ 匹配 requiredExistingData.users (按 slug)
         ④ 都找不到 → 特殊处理：
            - 如果所有作者都丢失，fallback 到 Owner
            - 因为文章必须至少有一个作者
```

**3. 产品关联 (tiers):**
```
posts_products[product_id] → 原始 product ID
                             ↓
         ① 匹配 requiredFromFile.products
         ② 匹配 requiredImportedData.products
         ③ 匹配 requiredExistingData.products
         ④ 都找不到 → 移除此关联
```

**4. 通讯关联 (newsletter_id):**
```
post.newsletter_id → 原始 newsletter ID
                      ↓
         ① 匹配 requiredImportedData.newsletters (按 originalId)
         ② 匹配 requiredExistingData.newsletters (按 ID)
         ③ 都找不到 → 删除 newsletter_id 字段
```

### 3.7 Stripe 循环引用修复

**问题** (`data-importer.js:155-182`):
```
stripe_prices → stripe_products → products → stripe_prices
     ↑_________________________________________|
```

products 表有 `monthly_price_id` 和 `yearly_price_id` 字段引用 stripe_prices，但 stripe_prices 导入在 products 之后。

**修复策略：**
1. 所有数据导入完成后
2. 遍历已导入的 products
3. 通过 `originalId` 查找已导入的 stripe_prices
4. 更新 products 表的价格字段
5. 此修复也在同一事务内

### 3.8 用户角色特殊处理

`UsersImporter.beforeImport()` (`importers/data/users-importer.js:31-88`):

| 场景 | 处理方式 |
|------|----------|
| Owner 角色 | 不允许导入，自动转为 Administrator |
| 员工数量限制 | 如果站点有 staff limit，所有用户设为 Contributor |
| 角色匹配 | 通过角色 **名称** 匹配，而非 ID |
| 导入用户状态 | 默认锁定，需通过找回密码流程重新激活 |

---

## 四、回滚边界分析

### 4.1 两种失败场景

由于 `doImport()` 是 `for...of` + `await` 串行执行，存在两种本质不同的失败场景：

| 场景 | 失败位置 | DataImporter 是否执行 |
|------|----------|----------------------|
| **A. 资源阶段失败** | ①/②/③ 抛错 | ❌ 不执行 |
| **B. 数据阶段失败** | ⑤ 内部抛错 | ✅ 已执行，事务回滚 |

### 4.2 场景 A：资源阶段失败

**发生时机：** `imageImporter`、`mediaImporter` 或 `filesImporter` 的 `doImport()` 抛错

**执行流程：**

```
用户上传 zip
    ↓
Step 1: loadFile()          ✅ 完成
Step 2: preProcess()        ✅ 完成
Step 3: doImport()
    ├─ ① imageImporter.doImport()
    │   ├─ store.save() 逐个保存图片
    │   ├─ ...
    │   └─ 第 N 个图片保存失败 → throw error
    ├─ ② mediaImporter        ← 不会执行
    ├─ ③ filesImporter        ← 不会执行
    ├─ ④ RevueImporter        ← 不会执行
    └─ ⑤ DataImporter         ← 不会执行
    ↓
catch 捕获错误
    ↓
cleanUp() → 删除临时目录
    ↓
发送失败邮件
```

**最终状态（场景 A）：**

| 数据类型 | 状态 | 说明 |
|----------|------|------|
| 已保存的图片/媒体/附件 | ⚠️ 部分保留 | `Promise.all` 中已完成的 save 不会回滚 |
| 数据库结构数据 | ✅ 未修改 | DataImporter 根本没有执行 |
| 临时解压目录 | ✅ 已清理 | cleanUp() 在 finally 中执行 |
| 导入标记标签 | ✅ 不存在 | DataImporter 未执行 |

**关键事实：场景 A 下数据库完全干净，只有部分资源文件成为孤立文件**

### 4.3 场景 B：数据阶段失败

**发生时机：** `DataImporter` 内部执行过程中产生 `errors` 或抛出异常

**前提条件：** ①②③④ 全部成功执行

**执行流程：**

```
用户上传 zip
    ↓
Step 1: loadFile()          ✅ 完成
Step 2: preProcess()        ✅ 完成
Step 3: doImport()
    ├─ ① imageImporter.doImport()    ✅ 所有图片保存完毕
    ├─ ② mediaImporter.doImport()    ✅ 所有媒体保存完毕
    ├─ ③ filesImporter.doImport()    ✅ 所有附件保存完毕
    ├─ ④ RevueImporter.doImport()    ✅ 空操作
    └─ ⑤ DataImporter.doImport()
        └─ 数据库事务开始
           ├─ UsersImporter           ✅
           ├─ RolesImporter           ✅
           ├─ TagsImporter            ✅
           ├─ ...
           ├─ PostsImporter
           │   ├─ ...
           │   └─ 某篇文章验证失败 → error 加入 errors 数组
           ├─ ...
           ├─ 检测 errors.length > 0
           │   └─ Yes → throw errors → 事务回滚
           └─ 数据库事务结束
    ↓
catch 捕获错误
    ↓
cleanUp() → 删除临时目录
    ↓
发送失败邮件
```

**最终状态（场景 B）：**

| 数据类型 | 状态 | 说明 |
|----------|------|------|
| 已保存的图片/媒体/附件 | ⚠️ 全部保留 | 在 DataImporter 之前已完成，无事务保护 |
| 数据库结构数据 | ❌ 已回滚 | 事务保证原子性 |
| 临时解压目录 | ✅ 已清理 | cleanUp() 在 finally 中执行 |
| 导入标记标签 | ❌ 已回滚 | 在事务内创建，随事务回滚 |

### 4.4 回滚边界总结表

| 阶段 | 操作内容 | 事务保护 | 场景 A 失败后 | 场景 B 失败后 |
|------|----------|----------|--------------|--------------|
| loadFile | Zip 解压、Handlers 计算路径 | ❌ 无 | 临时目录已清理 | 临时目录已清理 |
| preProcess | 路径替换、格式转换 | - | 无持久化 | 无持久化 |
| imageImporter.doImport | 保存图片到存储 | ❌ 无 | **部分保留** | **全部保留** |
| mediaImporter.doImport | 保存媒体到存储 | ❌ 无 | **未执行** | **全部保留** |
| filesImporter.doImport | 保存附件到存储 | ❌ 无 | **未执行** | **全部保留** |
| DataImporter.doImport | 所有数据库操作 | ✅ 有 | **未执行** | **完全回滚** |

### 4.5 DataImporter 内部事务边界

所有子 importer 的操作都在同一个事务内 (`data-importer.js:124-197`)：

```javascript
return models.Base.transaction(async function (transacting) {
    modelOptions.transacting = transacting;

    await sequence(ops);  // 顺序执行所有子 importer

    if (errors.length > 0) {
        throw errors;  // 触发事务回滚
    }

    return { ... };
});
```

**事务内执行顺序：**
1. UsersImporter → RolesImporter → TagsImporter → NewslettersImporter
2. SettingsImporter → ProductsImporter → StripeProductsImporter → StripePricesImporter
3. PostsImporter → CustomThemeSettingsImporter → RevueSubscriberImporter
4. Stripe 循环引用修复

**任何一步产生 errors 都会导致整个事务回滚**

### 4.6 错误分类：Errors vs Problems

`BaseImporter.handleError()` (`base.js:111-174`) 区分两种错误：

| 类型 | 触发回滚 | 示例 |
|------|----------|------|
| **Errors** | ✅ 是 | 数据验证失败、唯一约束冲突（`allowDuplicates=false`） |
| **Problems** | ❌ 否 | 日期格式错误、用户引用找不到、重复条目被忽略 |

**常见 Problems（自动处理，不回滚）：**
- 日期格式无效 → 使用当前时间戳
- 用户引用找不到 → fallback 到 Owner
- 重复 slug → 忽略重复项
- 关联的 tag/author 找不到 → 移除关联或 fallback
- 文章必须有作者 → 所有作者丢失时 fallback 到 Owner

### 4.7 清理机制

`cleanUp()` (`import-manager.js:441-457`):
- 在 `finally` 块中执行，无论成功失败
- 只删除 zip 解压的临时目录
- **不清理** 已保存到存储的资源文件
- 清理失败只记录日志，不影响导入结果

---

## 五、跨实例迁移完整流程（可复查）

### 5.1 迁移准备

**源实例操作：**
1. Admin UI → Settings → Labs → Migration tools → "Content & settings"
2. 下载得到 `.json` 文件（仅数据库快照）
3. **手动** 从源实例存储复制以下目录：
   - `content/images/`
   - `content/media/`
   - `content/files/`
4. 将 `.json` 和上述目录打包为 `.zip`

**Zip 结构要求（任一即可）：**
```
# 结构 A：根目录直接放置
export.zip
├── ghost.2024-01-01-12-00-00.json
├── images/
│   └── 2024/
│       └── 01/
│           └── photo.jpg
├── media/
└── files/

# 结构 B：单层基础目录
export.zip
└── my-site/
    ├── ghost.2024-01-01-12-00-00.json
    ├── images/
    ├── media/
    └── files/
```

### 5.2 导入执行流程复查

```
用户上传 zip
    ↓
POST /db/
    ↓
ImportManager.importFromFile()
    ├─ 1. loadFile()
    │   ├─ 解压 zip 到临时目录
    │   ├─ ImageHandler 扫描 images/ → 生成 [{originalPath, newPath, targetDir}, ...]
    │   ├─ mediaHandler 扫描 media/ → 同上
    │   ├─ filesHandler 扫描 files/ → 同上
    │   └─ JSONHandler 解析 .json → {meta, data: {posts, users, tags, ...}}
    │
    ├─ 2. preProcess()
    │   ├─ imageImporter: 将 posts/tags/users 中的图片路径替换为 newPath
    │   ├─ mediaImporter: 将 posts 中的媒体路径替换为 newPath
    │   └─ filesImporter: 将 posts 中的文件路径替换为 newPath
    │
    ├─ 3. doImport()
    │   ├─ imageImporter.doImport()
    │   │   └─ store.save() → 图片写入存储 ← ⚠️ 无事务
    │   ├─ mediaImporter.doImport()
    │   │   └─ store.save() → 媒体写入存储 ← ⚠️ 无事务
    │   ├─ filesImporter.doImport()
    │   │   └─ store.save() → 附件写入存储 ← ⚠️ 无事务
    │   └─ DataImporter.doImport()
    │       └─ 数据库事务开始
    │           ├─ 版本检查 (meta.version 必须是有效 semver)
    │           ├─ UsersImporter:
    │           │   ├─ fetchExisting: 读取现有用户
    │           │   ├─ beforeImport: 生成新 ID，Owner 角色转 Admin
    │           │   ├─ replaceIdentifiers: 用户引用映射
    │           │   └─ doImport: 写入 users 表
    │           ├─ TagsImporter:
    │           │   ├─ fetchExisting: 读取现有标签
    │           │   ├─ beforeImport: 生成新 ID
    │           │   └─ doImport: slug 冲突则跳过
    │           ├─ ... (其他 importers)
    │           ├─ PostsImporter:
    │           │   ├─ fetchExisting: 读取现有 tags/products/newsletters
    │           │   ├─ beforeImport: 生成新 ID，构建关联数组
    │           │   ├─ replaceIdentifiers: tags/authors/tiers 重映射
    │           │   └─ doImport: 写入 posts 及关联表
    │           ├─ Stripe 循环引用修复
    │           ├─ errors 检查
    │           │   ├─ 有 errors → throw → 事务回滚 ← ✅ 结构数据回滚
    │           │   └─ 无 errors → commit ← ✅ 结构数据提交
    │           └─ 数据库事务结束
    │
    └─ 4-6. 收尾
        ├─ cleanUp(): 删除临时目录
        └─ 发送邮件（成功/失败）
```

### 5.3 迁移失败场景分析

**场景 A：资源阶段失败（串行中断）**

**子场景 A-1：imageImporter.doImport 抛错**
- 执行到 `imageImporter.doImport()` 时失败
- `for...of` 循环立即停止
- `mediaImporter`、`filesImporter`、`DataImporter` 都不会执行
- 结果：
  - 已保存的部分图片：保留（孤立）
  - 媒体、附件：未保存
  - 数据库：完全干净，未做任何修改

**子场景 A-2：mediaImporter.doImport 抛错**
- `imageImporter` 已全部完成
- 执行到 `mediaImporter.doImport()` 时失败
- `filesImporter`、`DataImporter` 不会执行
- 结果：
  - 图片：全部保留（孤立）
  - 已保存的部分媒体：保留（孤立）
  - 附件：未保存
  - 数据库：完全干净，未做任何修改

**子场景 A-3：filesImporter.doImport 抛错**
- `imageImporter`、`mediaImporter` 已全部完成
- 执行到 `filesImporter.doImport()` 时失败
- `DataImporter` 不会执行
- 结果：
  - 图片、媒体：全部保留（孤立）
  - 已保存的部分附件：保留（孤立）
  - 数据库：完全干净，未做任何修改

**场景 B：数据阶段失败**

**前提条件：** `imageImporter`、`mediaImporter`、`filesImporter` 全部成功完成

**子场景 B-1：DataImporter 中 users 导入产生 errors**
- 图片、媒体、附件：全部已保存
- 数据库事务回滚
- 结果：
  - 资源文件：全部保留（孤立）
  - 数据库：完全干净，无任何导入数据

**子场景 B-2：DataImporter 中 posts 导入产生 errors**
- 图片、媒体、附件：全部已保存
- 数据库事务回滚
- 结果：
  - 资源文件：全部保留（孤立）
  - 数据库：完全干净，无任何导入数据

**场景 C：数据阶段产生 problems（非错误）**

**子场景 C-1：DataImporter 中 posts 导入产生 problems**
- problems 被记录但不触发回滚
- 事务提交
- 结果：
  - 资源文件：全部保留且有引用
  - 数据库：成功导入，但部分数据被自动修复
  - 用户需检查邮件中的 problems 列表

---

## 六、关键设计决策与风险点

### 6.1 导出不包含资源文件

**设计意图：**
- 资源文件可能很大（GB 级别），JSON 导出保持轻量
- 资源存储可能在外部（S3 等），导出逻辑复杂
- 用户可手动管理资源文件迁移

**风险：**
- 用户容易遗漏资源文件，导致迁移后图片 404
- 需要文档明确指导手动打包流程

### 6.2 资源导入先于数据库导入（串行无事务）

**设计意图：**
- 存储适配器（本地文件系统、S3 等）不支持事务
- 跨系统事务（数据库 + 存储）实现复杂

**串行执行的真实边界：**

```
资源保存阶段（无事务）              数据库导入阶段（有事务）
────────────────────────────        ────────────────────────────────
① imageImporter.doImport()          ⑤ DataImporter.doImport()
② mediaImporter.doImport()            └─ 单一事务内完成
③ filesImporter.doImport()
    │
    └─ 任一阶段失败 → 立即中断 → ⑤ 不会执行
```

**两种失败场景的真实风险：**

| 场景 | 资源文件状态 | 数据库状态 | 风险 |
|------|-------------|-----------|------|
| **资源阶段失败** | 部分保存（孤立） | ✅ 完全干净 | 孤立文件需手动清理 |
| **数据阶段失败** | 全部保存（孤立） | ❌ 完全回滚 | 所有资源文件成为孤立文件 |

**不可能路径：** 资源保存失败但数据库提交成功 → 不存在，因为串行中断导致 DataImporter 根本不会执行

### 6.3 Problems 不触发回滚

**设计意图：**
- 尽量让导入完成，而不是因小问题完全失败
- 可恢复的问题（如用户引用找不到）自动修复

**风险：**
- 静默的数据丢失（如关联被移除）
- 用户需检查邮件中的 problems 列表

---

## 七、代码位置索引

| 功能模块 | 文件路径 |
|----------|----------|
| 导出核心逻辑 | `ghost/core/core/server/data/exporter/exporter.js` |
| 导出表定义 | `ghost/core/core/server/data/exporter/table-lists.js` |
| 导出文件名 | `ghost/core/core/server/data/exporter/export-filename.js` |
| 导入管理器 | `ghost/core/core/server/data/importer/import-manager.js` |
| 数据导入器协调 | `ghost/core/core/server/data/importer/importers/data/data-importer.js` |
| 基础导入器（ID生成、错误处理） | `ghost/core/core/server/data/importer/importers/data/base.js` |
| 文章导入器（关联重映射） | `ghost/core/core/server/data/importer/importers/data/posts-importer.js` |
| 用户导入器（角色处理） | `ghost/core/core/server/data/importer/importers/data/users-importer.js` |
| 标签导入器 | `ghost/core/core/server/data/importer/importers/data/tags-importer.js` |
| 内容文件导入器（资源保存） | `ghost/core/core/server/data/importer/importers/content-file-importer.js` |
| JSON Handler | `ghost/core/core/server/data/importer/handlers/json.js` |
| 图片 Handler | `ghost/core/core/server/data/importer/handlers/image.js` |
| 内容文件 Handler | `ghost/core/core/server/data/importer/handlers/importer-content-file-handler.js` |
| DB API 端点 | `ghost/core/core/server/api/endpoints/db.js` |
| 本地备份 | `ghost/core/core/server/data/db/backup.js` |
| Admin 前端导出按钮 | `apps/admin-x-settings/src/components/settings/advanced/migration-tools/migration-tools-export.tsx` |
| Admin 前端导入模态框 | `apps/admin-x-settings/src/components/settings/advanced/migration-tools/universal-import-modal.tsx` |
| 前端 API 封装 | `apps/admin-x-framework/src/api/db.ts` |
