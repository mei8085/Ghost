# Ghost 站点级导入导出迁移机制分析

## 一、一键导出的快照范围

### 1.1 默认导出范围 (TABLES_ALLOWLIST)

默认的 "一键导出" 功能导出以下表的数据，位于 `ghost/core/core/server/data/exporter/table-lists.js:69-90`:

**核心内容表：**
- `posts` - 文章/页面
- `posts_authors` - 文章-作者关联
- `posts_meta` - 文章元数据
- `posts_tags` - 文章-标签关联
- `posts_products` - 文章-产品关联

**用户与权限：**
- `roles` - 角色定义
- `roles_users` - 用户-角色关联
- `users` - 用户数据

**标签与分类：**
- `tags` - 标签

**设置与配置：**
- `settings` - 站点设置 (部分敏感配置被过滤)
- `custom_theme_settings` - 主题自定义设置

**会员与产品：**
- `products` - 产品/套餐
- `stripe_products` - Stripe 产品映射
- `stripe_prices` - Stripe 价格配置
- `newsletters` - 通讯
- `benefits` - 权益
- `products_benefits` - 产品-权益关联

**营销相关：**
- `offers` - 优惠码
- `offer_redemptions` - 优惠码兑换记录
- `snippets` - 代码片段

### 1.2 可选备份表 (BACKUP_TABLES)

通过 API 的 `include` 参数可以额外导出以下表，位于 `ghost/core/core/server/data/exporter/table-lists.js:2-64`:

**安全与认证：**
- `api_keys` - API 密钥
- `integrations` - 集成配置
- `invites` - 邀请记录
- `tokens` - Token
- `sessions` - 会话
- `brute` - 暴力破解防护记录
- `actions` - 操作日志

**会员与订阅：**
- `members` - 会员数据
- `members_labels` - 会员标签
- `members_products` - 会员-产品关联
- `members_stripe_customers` - 会员-Stripe 映射
- `members_stripe_customers_subscriptions` - 会员订阅
- `subscriptions` - 订阅记录
- `members_cancel_events` - 会员取消事件
- `members_payment_events` - 支付事件
- `members_login_events` - 登录事件
- `members_email_change_events` - 邮箱变更事件
- `members_status_events` - 状态变更事件
- `members_paid_subscription_events` - 付费订阅事件
- `members_subscribe_events` - 订阅事件
- `members_product_events` - 产品事件
- `members_created_events` - 创建事件
- `members_subscription_created_events` - 订阅创建事件
- `members_newsletters` - 会员-通讯关联
- `members_click_events` - 点击事件
- `members_feedback` - 会员反馈

**邮件相关：**
- `emails` - 发送的邮件
- `email_batches` - 邮件批次
- `email_recipients` - 邮件接收者
- `email_recipient_failures` - 发送失败记录
- `email_design_settings` - 邮件设计设置
- `automated_email_recipients` - 自动化邮件接收者
- `welcome_email_automations` - 欢迎邮件自动化
- `welcome_email_automation_runs` - 欢迎邮件自动化运行记录
- `welcome_email_automated_emails` - 欢迎自动化邮件

**内容历史：**
- `mobiledoc_revisions` - Mobiledoc 修订
- `post_revisions` - 文章修订

**评论与互动：**
- `comments` - 评论
- `comment_likes` - 评论点赞
- `comment_reports` - 评论举报
- `mentions` - 提及

**社交与推荐：**
- `recommendations` - 推荐
- `recommendation_click_events` - 推荐点击事件
- `recommendation_subscribe_events` - 推荐订阅事件
- `outbox` - ActivityPub 发件箱

**其他：**
- `donation_payment_events` - 捐赠支付事件
- `suppressions` - 邮件抑制列表
- `email_spam_complaint_events` - 垃圾邮件投诉事件
- `milestones` - 里程碑
- `collections` - 集合
- `collections_posts` - 集合-文章关联
- `gifts` - 礼品
- `labels` - 标签 (会员用)
- `redirects` - 重定向
- `jobs` - 后台任务
- `migrations` - 迁移记录
- `migrations_lock` - 迁移锁
- `permissions` - 权限
- `permissions_roles` - 权限-角色关联
- `permissions_users` - 权限-用户关联
- `webhooks` - Webhook

### 1.3 敏感配置过滤

导出时会过滤以下设置项 (`SETTING_KEYS_BLOCKLIST`)，位于 `ghost/core/core/server/data/exporter/table-lists.js:93-104`:

- `stripe_connect_publishable_key`
- `stripe_connect_secret_key`
- `stripe_connect_account_id`
- `stripe_secret_key`
- `stripe_publishable_key`
- `stripe_billing_portal_configuration_id`
- `members_stripe_webhook_id`
- `members_stripe_webhook_secret`
- `email_verification_required`
- `indexnow_api_key`

### 1.4 导出格式

导出数据为 JSON 格式，结构如下：

```json
{
  "meta": {
    "exported_on": 1699999999999,
    "version": "5.0.0"
  },
  "data": {
    "posts": [...],
    "users": [...],
    "tags": [...],
    ...
  }
}
```

文件名格式: `{站点标题}.ghost.{YYYY-MM-DD-HH-mm-ss}.json`

---

## 二、跨实例导入时的标识重映射

### 2.1 整体流程

导入管理器 (`ImportManager`) 位于 `ghost/core/core/server/data/importer/import-manager.js`，处理流程为 6 个步骤：

1. **loadFile** - 加载文件 (支持 zip 或单个 json)
2. **preProcess** - 预处理 (替换资源路径等)
3. **doImport** - 实际导入数据
4. **generateReport** - 生成报告
5. **cleanUp** - 清理临时文件
6. **发送完成邮件**

### 2.2 数据导入器架构

数据导入由 `DataImporter` (`ghost/core/core/server/data/importer/importers/data/data-importer.js`) 协调，使用 8 个专用导入器按顺序处理：

1. `UsersImporter` - 用户
2. `RolesImporter` - 角色
3. `TagsImporter` - 标签
4. `NewslettersImporter` - 通讯
5. `SettingsImporter` - 设置
6. `ProductsImporter` - 产品
7. `StripeProductsImporter` - Stripe 产品
8. `StripePricesImporter` - Stripe 价格
9. `PostsImporter` - 文章
10. `CustomThemeSettingsImporter` - 主题设置
11. `RevueSubscriberImporter` - Revue 订阅者

### 2.3 ID 生成与映射

**新 ID 生成** (`base.js:89-99`):
```javascript
generateIdentifier() {
    _.each(this.dataToImport, (obj) => {
        const newId = ObjectId().toHexString();
        if (obj.id) {
            this.originalIdMap[newId] = obj.id;  // 保存原始ID映射
        }
        obj.id = newId;  // 替换为新ID
    });
}
```

每个导入对象都会生成新的 ObjectId，同时通过 `originalIdMap` 保存 `新ID → 原始ID` 的映射关系。

### 2.4 用户引用重映射

用户引用处理位于 `base.js:184-304`，`replaceIdentifiers()` 方法处理以下字段：
- `author_id`
- `published_by`

**解析策略（优先级从高到低）：**

1. **空值处理** - 如果引用为空，fallback 到当前站点的 Owner 用户
2. **文件内匹配** - 在导入文件的 users 数据中查找
3. **已导入用户匹配** - 通过 email 查找已导入的用户 (因为 slug 可能在插入时变化)
4. **数据库已有用户匹配** - 先按 slug 查找，再按 ID 查找
5. **最终 fallback** - 如果都找不到，使用 Owner 用户

### 2.5 文章关系重映射

`PostsImporter.replaceIdentifiers()` (`posts-importer.js:116-214`) 处理：

**标签关系 (posts_tags → tags):**
- 先在导入文件的 tags 中查找
- 再通过 `originalId` 在已导入数据中匹配
- 最后在数据库现有 tags 中按 slug 查找

**作者关系 (posts_authors → users):**
- 同样的多级匹配策略
- 特殊处理：如果所有作者都无法匹配，fallback 到 Owner 用户（因为文章必须至少有一个作者）

**产品关系 (posts_products → products):**
- 类似的匹配逻辑

**通讯关联 (newsletter_id):**
- 查找已导入的 newsletters
- 如果在导入文件中存在但未导入，删除该引用

### 2.6 Stripe 循环引用修复

Stripe 存在循环引用问题 (`posts-importer.js:155-182`):

```
stripe_prices → stripe_products → products → stripe_prices
```

修复策略：
1. 先导入所有数据
2. 在导入序列最后，专门处理 products 的 `monthly_price_id` 和 `yearly_price_id`
3. 通过 `originalId` 查找已导入的 stripe_prices，更新 products 表

### 2.7 角色处理

`UsersImporter` (`users-importer.js:31-88`) 的特殊处理：

- **Owner 角色不导入** - 导入时将 Owner 角色转换为 Administrator
- **员工限制处理** - 如果站点有员工数量限制，所有导入用户角色设为 Contributor
- **角色匹配** - 通过角色名称匹配，而非 ID

---

## 三、图片与附件资源迁移

### 3.1 资源导入器

`ImportManager` 初始化了三个内容文件处理器 (`import-manager.js:58-96`):

1. **ImageHandler** - 处理图片
   - 目录: `images/`, `content/`
   - 使用 imageStorage 适配器

2. **mediaHandler** (ImporterContentFileHandler) - 处理媒体文件
   - 目录: `media/`, `content/`
   - 使用 mediaStorage 适配器

3. **filesHandler** (ImporterContentFileHandler) - 处理通用文件
   - 目录: `files/`, `content/`
   - 使用 fileStorage 适配器

### 3.2 资源文件处理流程

**Step 1: 路径规范化** (`importer-content-file-handler.js:38-79`):

```javascript
// 1. 移除 zip 基础目录前缀
const noBaseDir = file.name.replace(baseDirRegex, '');

// 2. 移除 Ghost 特定目录前缀 (如 content/images/)
let noGhostDirs = noBaseDir;
_.each(contentFilesFolderRegexes, function (regex) {
    noGhostDirs = noGhostDirs.replace(regex, '');
});

// 3. 生成目标路径
file.originalPath = noBaseDir;  // 原始路径，用于内容替换
file.name = noGhostDirs;
file.targetDir = path.dirname(noGhostDirs);
```

**Step 2: 生成唯一文件名**

使用 storage 适配器的 `getUniqueFileName()` 方法避免冲突。

**Step 3: 生成新 URL**

```javascript
file.newPath = urlUtils.urlJoin(
    '/',
    urlUtils.getSubdir(),
    storage.staticFileURLPrefix,
    targetFilename
);
```

### 3.3 内容中的资源引用替换

`ContentFileImporter.preProcess()` (`content-file-importer.js:74-110`) 在导入数据前进行路径替换：

**替换范围：**

1. **Posts 中的图片引用**:
   - `post.markdown` - Markdown 内容
   - `post.html` - HTML 内容
   - `post.feature_image` - 特色图片

2. **Tags 中的图片引用**:
   - `tag.feature_image`

3. **Users 中的图片引用**:
   - `user.cover_image` - 封面图
   - `user.profile_image` - 头像

**替换逻辑** (`content-file-importer.js:7-16`):

```javascript
replaceImage = function (markdown, image) {
    if (!markdown) {
        return;
    }
    // 匹配原始路径，支持可选的前导斜杠
    const regex = new RegExp('(/)?' + image.originalPath, 'gm');
    return markdown.replace(regex, image.newPath);
};
```

### 3.4 资源文件存储

`ContentFileImporter.doImport()` (`content-file-importer.js:117-125`):

```javascript
doImport(contentFilesData) {
    return Promise.all(contentFilesData.map(function (contentFile) {
        return store.save(contentFile, contentFile.targetDir).then(function (result) {
            return {
                originalPath: contentFile.originalPath,
                newPath: contentFile.newPath,
                stored: result
            };
        });
    }));
}
```

### 3.5 支持的 Zip 结构

导入器支持以下 zip 结构：

1. **根目录直接放置** - JSON 文件和 images/ 等目录在 zip 根目录
2. **单层基础目录** - 所有内容在一个子目录内

zip 内的目录名可以是：
- `images/` - 图片
- `media/` - 媒体文件
- `files/` - 文件附件
- `content/` - 通用内容目录

---

## 四、失败时的回滚边界

### 4.1 数据库事务边界

**数据导入的事务范围** (`data-importer.js:124-197`):

```javascript
return models.Base.transaction(async function (transacting) {
    modelOptions.transacting = transacting;
    
    // 所有导入操作在同一个事务中
    await sequence(ops);  // 顺序执行所有 importer
    
    // 检查错误
    if (errors.length > 0) {
        debug(errors);
        throw errors;  // 抛出错误触发回滚
    }
    
    return { ... };
});
```

**事务包含的操作：**
- 所有 importer 的 `fetchExisting`, `beforeImport`, `replaceIdentifiers`, `doImport`
- Stripe 循环引用修复

**事务回滚触发条件：**
- 任何 importer 产生 `errors` (注意：不是 `problems`)
- 未捕获的异常

### 4.2 错误分类

**Errors 与 Problems 的区别** (`base.js:111-174`):

| 类型 | 处理方式 | 例子 |
|------|----------|------|
| **Errors** | 阻止导入，触发事务回滚 | 数据验证失败、重复数据（当 `allowDuplicates=false`） |
| **Problems** | 仅记录警告，不影响导入继续 | 日期格式错误（自动修复）、用户引用找不到（fallback 到 Owner）、重复条目（被忽略） |

**常见 Problems：**
- 日期格式错误 → 使用当前时间戳
- 用户引用找不到 → fallback 到 Owner
- 重复的 slug → 忽略重复项
- 找不到关联的 tag/author → 移除关联或 fallback

### 4.3 资源文件的回滚边界

**关键发现：图片/媒体文件导入不在数据库事务内**

导入执行顺序 (`import-manager.js:494-552`):

```javascript
async importFromFile(file, importOptions) {
    // Step 1: 加载文件
    importData = await this.loadFile(file);
    
    // Step 2: 预处理 (替换内容中的路径引用)
    importData = await this.preProcess(importData);
    
    // Step 3: 执行导入
    importResult = await this.doImport(importData, importOptions);
    
    // Step 5: 清理
    await this.cleanUp();
}
```

**doImport 中的 importer 执行顺序** (`import-manager.js:101`):
```javascript
this.importers = [
    imageImporter,        // 1. 先保存图片到存储
    mediaImporter,        // 2. 保存媒体文件
    contentFilesImporter, // 3. 保存文件附件
    RevueImporter,
    DataImporter          // 4. 最后才在数据库事务中导入结构数据
];
```

**回滚边界结论：**

| 组件 | 事务保护 | 失败后状态 |
|------|----------|-----------|
| 图片文件 (imageImporter) | ❌ 无 | 已保存的文件不会回滚 |
| 媒体文件 (mediaImporter) | ❌ 无 | 已保存的文件不会回滚 |
| 文件附件 (filesImporter) | ❌ 无 | 已保存的文件不会回滚 |
| 结构数据 (DataImporter) | ✅ 有 | 完全回滚 |

### 4.4 清理机制

**临时文件清理** (`import-manager.js:441-457`):

```javascript
async cleanUp() {
    if (this.fileToDelete === null) {
        return;
    }
    try {
        await fs.remove(this.fileToDelete);  // 删除 zip 解压的临时目录
    } catch (err) {
        // 清理失败只记录错误，不影响导入结果
        logging.error(...);
    }
    this.fileToDelete = null;
}
```

**清理始终执行** - 在 `finally` 块中调用，无论成功失败。

### 4.5 导入失败后的部分数据

如果导入过程中发生错误：

1. **已保存到存储的文件** - 保留在文件系统（孤立文件）
2. **数据库中的结构数据** - 完全回滚（通过事务）
3. **临时解压目录** - 被清理
4. **用户邮件通知** - 发送失败邮件

---

## 五、导入器依赖关系

### 5.1 导入顺序

`DataImporter` 中 importers 的初始化顺序 (`data-importer.js:33-47`) 决定了执行顺序：

1. UsersImporter → 2. RolesImporter → 3. TagsImporter → 4. NewslettersImporter → 5. SettingsImporter → 6. ProductsImporter → 7. StripeProductsImporter → 8. StripePricesImporter → 9. PostsImporter

### 5.2 依赖声明

每个 importer 通过 options 声明依赖：

**PostsImporter** (`posts-importer.js:15-26`):
```javascript
requiredFromFile: [
    'posts', 'tags', 'posts_tags', 'posts_authors', 
    'posts_meta', 'products', 'posts_products'
],
requiredImportedData: ['tags', 'products', 'newsletters'],
requiredExistingData: ['tags', 'products', 'newsletters']
```

**依赖解析** (`data-importer.js:132-142`):
```javascript
if (importer.options.requiredImportedData.length) {
    _.each(importer.options.requiredImportedData, (key) => {
        importer.requiredImportedData[key] = importers[key].importedData;
    });
}
```

### 5.3 默认用户依赖

所有 importer 都默认依赖 users (`base.js:34-54`):
```javascript
if (!this.options.requiredImportedData) {
    this.options.requiredImportedData = ['users'];
} else {
    this.options.requiredImportedData.push('users');
}
```

---

## 六、版本兼容性检查

### 6.1 导入文件版本验证

`DataImporter.doImport()` (`data-importer.js:99-120`):

```javascript
// 必须有 meta 字段
if (!importData.meta) {
    return Promise.reject(new IncorrectUsageError(...));
}

// 必须有 version 字段
if (!importData.meta.version) {
    return Promise.reject(new IncorrectUsageError(...));
}

// 必须是有效的 semver 版本 (拒绝 Ghost v0.x 的非 semver 格式)
if (!semver.valid(importData.meta.version)) {
    return Promise.reject(new IncorrectUsageError({
        message: 'Detected unsupported file structure.',
        help: 'Please install Ghost 1.0, import the file and then update your blog...'
    }));
}
```

### 6.2 支持的版本

- **Ghost 1.x+** - 支持直接导入
- **Ghost 0.x** - 必须先升级到 Ghost 1.0 再导入

---

## 七、代码位置索引

| 功能 | 文件路径 |
|------|----------|
| 导出核心 | `ghost/core/core/server/data/exporter/exporter.js` |
| 导出表定义 | `ghost/core/core/server/data/exporter/table-lists.js` |
| 导入管理器 | `ghost/core/core/server/data/importer/import-manager.js` |
| 数据导入器 | `ghost/core/core/server/data/importer/importers/data/data-importer.js` |
| 基础导入器 | `ghost/core/core/server/data/importer/importers/data/base.js` |
| 文章导入器 | `ghost/core/core/server/data/importer/importers/data/posts-importer.js` |
| 用户导入器 | `ghost/core/core/server/data/importer/importers/data/users-importer.js` |
| 标签导入器 | `ghost/core/core/server/data/importer/importers/data/tags-importer.js` |
| 内容文件处理器 | `ghost/core/core/server/data/importer/handlers/importer-content-file-handler.js` |
| 图片处理器 | `ghost/core/core/server/data/importer/handlers/image.js` |
| 内容文件导入器 | `ghost/core/core/server/data/importer/importers/content-file-importer.js` |
| DB API 端点 | `ghost/core/core/server/api/endpoints/db.js` |
