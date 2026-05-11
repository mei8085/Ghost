# Ghost 自定义集成技术分析报告

## 目录
1. [概述](#概述)
2. [集成模型架构](#集成模型架构)
3. [API Key 的 Scope 与生成流程](#api-key-的-scope-与生成流程)
4. [第三方调用与 Webhook 协作机制](#第三方调用与-webhook-协作机制)
5. [多集成隔离与凭证轮换](#多集成隔离与凭证轮换)
6. [安全机制](#安全机制)

---

## 概述

Ghost 平台的自定义集成系统允许第三方应用通过标准的 API 和 Webhook 机制与 Ghost 进行双向通信。该系统采用了现代化的安全设计，包括基于角色的权限控制、JWT 认证、HMAC 签名验证等。

集成系统的核心组件包括：
- **Integration Model**: 管理集成实例的元数据
- **API Key Model**: 管理 API 密钥及其权限
- **Webhook Model**: 管理事件订阅和回调配置
- **认证中间件**: 处理 API 请求的身份验证
- **Webhook Trigger**: 处理事件触发和回调发送

---

## 集成模型架构

### 数据库模型

#### 1. 集成表 (`integrations`)

**schema 定义位置**: `ghost/core/core/server/data/schema/schema.js:339-354`

```javascript
integrations: {
    id: {type: 'string', maxlength: 24, nullable: false, primary: true},
    type: {
        type: 'string',
        maxlength: 50,
        nullable: false,
        defaultTo: 'custom',
        validations: {isIn: [['internal', 'builtin', 'custom', 'core']]}
    },
    name: {type: 'string', maxlength: 191, nullable: false},
    slug: {type: 'string', maxlength: 191, nullable: false, unique: true},
    icon_image: {type: 'string', maxlength: 2000, nullable: true},
    description: {type: 'string', maxlength: 2000, nullable: true},
    created_at: {type: 'dateTime', nullable: false},
    updated_at: {type: 'dateTime', nullable: true}
}
```

**集成类型说明**:
| 类型 | 说明 |
|------|------|
| `internal` | Ghost 内部使用的集成，不显示在 UI 中 |
| `builtin` | Ghost 内置的官方集成 |
| `custom` | 用户创建的自定义集成 |
| `core` | Ghost 核心功能集成 |

#### 2. API 密钥表 (`api_keys`)

**schema 定义位置**: `ghost/core/core/server/data/schema/schema.js:373-396`

```javascript
api_keys: {
    id: {type: 'string', maxlength: 24, nullable: false, primary: true},
    type: {
        type: 'string',
        maxlength: 50,
        nullable: false,
        validations: {isIn: [['content', 'admin']]}
    },
    secret: {
        type: 'string',
        maxlength: 191,
        nullable: false,
        unique: true,
        validations: {isLength: {min: 26, max: 128}}
    },
    role_id: {type: 'string', maxlength: 24, nullable: true},
    // integration_id is nullable to allow "internal" API keys that don't show in the UI
    integration_id: {type: 'string', maxlength: 24, nullable: true},
    user_id: {type: 'string', maxlength: 24, nullable: true},
    last_seen_at: {type: 'dateTime', nullable: true},
    last_seen_version: {type: 'string', maxlength: 50, nullable: true},
    created_at: {type: 'dateTime', nullable: false},
    updated_at: {type: 'dateTime', nullable: true}
}
```

**API Key 类型**:
| 类型 | 说明 | 权限模型 |
|------|------|----------|
| `content` | Content API 密钥，用于读取公开内容 | 无角色关联（只读公开数据） |
| `admin` | Admin API 密钥，用于管理操作 | 关联 "Admin Integration" 角色 |

#### 3. Webhook 表 (`webhooks`)

**schema 定义位置**: `ghost/core/core/server/data/schema/schema.js:355-372`

```javascript
webhooks: {
    id: {type: 'string', maxlength: 24, nullable: false, primary: true},
    event: {type: 'string', maxlength: 50, nullable: false, validations: {isLowercase: true}},
    target_url: {type: 'string', maxlength: 2000, nullable: false},
    name: {type: 'string', maxlength: 191, nullable: true},
    secret: {type: 'string', maxlength: 191, nullable: true},
    api_version: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'v2'},
    // NOTE: integration_id column needs "nullable: true" -> "nullable: false" migration
    integration_id: {type: 'string', maxlength: 24, nullable: false, references: 'integrations.id', cascadeDelete: true},
    last_triggered_at: {type: 'dateTime', nullable: true},
    last_triggered_status: {type: 'string', maxlength: 50, nullable: true},
    last_triggered_error: {type: 'string', maxlength: 50, nullable: true},
    created_at: {type: 'dateTime', nullable: false},
    updated_at: {type: 'dateTime', nullable: true}
}
```

### 模型关系与关联类型

**Integration 模型** (`ghost/core/core/server/models/integration.js:6-112`)

```javascript
relationships: ['api_keys', 'webhooks'],

api_keys: function apiKeys() {
    return this.hasMany('ApiKey', 'integration_id');
},

webhooks: function webhooks() {
    return this.hasMany('Webhook', 'integration_id');
}
```

**关键差异：强关联 vs 软关联**

| 关联对象 | 外键约束 | 可空性 | 级联删除 | 关联类型 |
|----------|----------|--------|----------|----------|
| **Webhook** | `integration_id` | `nullable: false` | `cascadeDelete: true` | **强关联** |
| **API Key** | `integration_id` | `nullable: true` | 无明确级联配置 | **软关联** |

**关系结构**:
```
Integration (1) ──┬──> (N) API Key  (软关联：integration_id 可空，无强制级联)
                  └──> (N) Webhook  (强关联：integration_id 非空，cascadeDelete: true)
```

**设计意图分析**:

1. **Webhook 强关联** (`schema.js:366`):
   ```javascript
   integration_id: {type: 'string', maxlength: 24, nullable: false, references: 'integrations.id', cascadeDelete: true}
   ```
   - Webhook 必须隶属于某个集成，不能独立存在
   - 删除集成时，所有关联的 Webhook 会被**自动级联删除**
   - 这是数据库层面的强制约束

2. **API Key 软关联** (`schema.js:389-390`):
   ```javascript
   // integration_id is nullable to allow "internal" API keys that don't show in the UI
   integration_id: {type: 'string', maxlength: 24, nullable: true}
   ```
   - API Key 可以不隶属于任何集成（如内部 API Key）
   - 注释明确说明："integration_id is nullable to allow 'internal' API keys that don't show in the UI"
   - 删除集成时，API Key **不会被自动级联删除**（需要应用层处理）

---

## API Key 的 Scope 与生成流程

### API Key 类型与权限范围 (Scope)

#### 1. Content API Key

**认证方式**: Query 参数 `?key=<secret>`

**权限范围**:
- 只能访问 Content API（公开内容读取）
- 无角色关联 (`role_id: null`)
- 用于读取文章、页面、标签等公开数据

**认证流程** (`ghost/core/core/server/services/auth/api-key/content.js:12-63`):

```javascript
const authenticateContentApiKey = async function authenticateContentApiKey(req, res, next) {
    let key = req.query.key;
    const apiKey = await models.ApiKey.findOne({secret: key}, {withRelated: ['integration']});
    
    if (apiKey.get('type') !== 'content') {
        return next(new errors.UnauthorizedError({...}));
    }
    
    req.api_key = apiKey;
    next();
};
```

#### 2. Admin API Key

**认证方式**: JWT Token 在 `Authorization: Ghost <token>` 头中

**权限范围**:
- 关联 **"Admin Integration" 角色**（所有自定义集成共享此角色）
- 拥有完整的管理权限（根据角色权限配置）
- 可访问 Admin API 的所有端点

**认证流程** (`ghost/core/core/server/services/auth/api-key/admin.js:48-198`):

```javascript
const authenticateWithToken = async function apiKeyAuthenticateWithToken(originalUrl, token, ignoreMaxAge) {
    const decoded = jwt.decode(token, {complete: true});
    const apiKeyId = decoded.header.kid;
    
    const apiKey = await models.ApiKey.findOne({id: apiKeyId}, {withRelated: ['integration']});
    
    if (apiKey.get('type') !== 'admin') {
        throw new errors.UnauthorizedError({...});
    }
    
    const secret = Buffer.from(apiKey.get('secret'), 'hex');
    jwt.verify(token, secret, {
        audience: new RegExp(`/?${api}/?$`),
        algorithms: ['HS256'],
        maxAge: '5m'
    });
    
    return {apiKey, user: null};
};
```

### 权限模型：共享角色边界

**关键设计：多个自定义集成共享同一角色**

从 `ghost/core/core/server/models/api-key.js:39-47` 可以看到：

```javascript
onSaving(model, attrs, options) {
    // enforce roles which are currently hardcoded
    // - admin key = Administrator role
    // - content key = no role
    if (this.hasChanged('type') || this.hasChanged('role_id')) {
        if (this.get('type') === 'admin') {
            return Role.findOne(
                {name: attrs.role || 'Admin Integration'},  // 硬编码默认角色
                Object.assign({}, options, {columns: ['id']})
            ).then((role) => {
                this.set('role_id', role.get('id'));
            });
        }

        if (this.get('type') === 'content') {
            this.set('role_id', null);
        }
    }
}
```

**权限加载机制** (`ghost/core/core/server/services/permissions/providers.js:58-74`):

```javascript
apiKey(id) {
    return models.ApiKey.findOne({id}, {withRelated: ['role', 'role.permissions']})
        .then((foundApiKey) => {
            // api keys have a belongs_to relationship to a role and no individual permissions
            // so there's no need for permission deduplication
            const permissions = foundApiKey.related('role').related('permissions').models;
            const roles = [foundApiKey.toJSON().role];

            return {permissions, roles};
        });
}
```

**权限边界分析**:

1. **所有自定义集成共享 "Admin Integration" 角色**:
   - 每个 Admin API Key 创建时自动绑定 `role_id` = "Admin Integration" 角色 ID
   - 权限通过 `api_key.role_id` → `roles` → `roles_permissions` → `permissions` 加载
   - **没有**按集成 ID 进行资源隔离的机制

2. **"Admin Integration" 角色权限定义** (`ghost/core/core/server/data/schema/fixtures/fixtures.json:989-1021`):
   ```javascript
   "Admin Integration": {
       "mail": "all",
       "notification": "all",
       "post": "all",
       "setting": "all",
       "slug": "all",
       "tag": "all",
       "theme": "all",
       "user": "all",
       "role": "all",
       "invite": "all",
       "redirect": "all",
       "webhook": "all",
       "action": "all",
       "member": "all",
       "label": "all",
       "automated_email": "all",
       "email_design_setting": "all",
       "email_preview": "all",
       "email": "all",
       "snippet": "all",
       "product": ["browse", "read", "add", "edit"],
       "offer": ["browse", "read", "add", "edit"],
       "newsletter": ["browse", "read", "add", "edit"],
       "explore": "read",
       "comment": "all",
       "link": "all",
       "mention": "browse",
       "collection": "all",
       "recommendation": "all",
       "automation": ["browse", "read", "edit"],
       "member_signin_url": "read"
   }
   ```

3. **权限检查示例** (`ghost/core/core/server/models/post.js:1414`):
   ```javascript
   isIntegration = loadedPermissions.apiKey && _.some(loadedPermissions.apiKey.roles, {name: 'Admin Integration'});
   ```

**重要结论**：
- **多个自定义集成之间不是按资源隔离的**
- 所有自定义集成的 Admin API Key 都共享 "Admin Integration" 角色的权限边界
- 任何一个自定义集成都可以：
  - 读取/修改所有文章、页面、标签
  - 管理所有成员数据
  - 修改站点设置
  - 管理其他集成的 webhook（通过 `webhook: all` 权限）
- 这是**有意的设计选择**，而非遗漏

### API Key 生成流程

#### 1. 创建集成时自动生成

**API 端点**: `POST /ghost/api/admin/integrations/`

**控制器位置**: `ghost/core/core/server/api/endpoints/integrations.js:103-139`

创建集成时自动生成两种 API Key：

```javascript
add: {
    query({data, options}) {
        const dataWithApiKeys = Object.assign({
            api_keys: [
                {type: 'content'},  // 自动生成 Content API Key
                {type: 'admin'}     // 自动生成 Admin API Key
            ]
        }, data);
        return models.Integration.add(dataWithApiKeys, options);
    }
}
```

#### 2. 密钥生成算法

**模型位置**: `ghost/core/core/server/models/api-key.js:6-74`

密钥通过 `@tryghost/security` 模块生成：

```javascript
const ApiKey = ghostBookshelf.Model.extend({
    defaults() {
        const secret = security.secret.create(this.get('type'));
        return {secret};
    }
}, {
    refreshSecret(data, options) {
        const secret = security.secret.create(data.type);
        return this.edit(Object.assign({}, data, {secret}), options);
    }
});
```

#### 3. Admin API Key 角色绑定

**模型逻辑**: `ghost/core/core/server/models/api-key.js:36-59`

```javascript
onSaving(model, attrs, options) {
    if (this.hasChanged('type') || this.hasChanged('role_id')) {
        if (this.get('type') === 'admin') {
            return Role.findOne(
                {name: attrs.role || 'Admin Integration'}, 
                Object.assign({}, options, {columns: ['id']})
            ).then((role) => {
                this.set('role_id', role.get('id'));
            });
        }

        if (this.get('type') === 'content') {
            this.set('role_id', null);
        }
    }
}
```

### JWT Token 格式

Admin API 使用 JWT 进行请求签名，Token 格式如下：

**Header**:
```json
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "<api_key_id>"  // API Key 的 ID，用于查找密钥
}
```

**Payload (Claims)**:
```json
{
  "aud": "/ghost/api/admin/",  // 受众（API 路径）
  "iat": 1234567890,           // 签发时间
  "exp": 1234568190            // 过期时间（默认 5 分钟）
}
```

**签名**:
- 使用 API Key 的 `secret`（从 hex 转换为字节）
- 算法: HS256

---

## 第三方调用与 Webhook 协作机制

### 双向通信架构

```
┌─────────────────┐          ┌─────────────────────────────────────────────┐
│   第三方应用     │          │                Ghost 平台                    │
└────────┬────────┘          └─────────────────────────────────────────────┘
         │                                    │
         │  1. API 调用 (JWT/Key)            │
         │ ────────────────────────────────> │
         │                                    │
         │  2. 模型层写入                      │
         │                                    │  ┌─────────────────────────┐
         │                                    │  │   onSaved/onUpdated    │
         │                                    │  │   emitChange()         │
         │                                    │  └──────────┬──────────────┘
         │                                    │             │
         │                                    │             ▼
         │                                    │  ┌─────────────────────────┐
         │                                    │  │   事件总线 (events)     │
         │                                    │  │   'post.published'      │
         │                                    │  └──────────┬──────────────┘
         │                                    │             │
         │                                    │             ▼
         │                                    │  ┌─────────────────────────┐
         │                                    │  │  WebhookTrigger.trigger │
         │                                    │  │  getAll(event)          │
         │                                    │  └──────────┬──────────────┘
         │                                    │             │
         │                                    │             ▼
         │                                    │  ┌─────────────────────────┐
         │                                    │  │  HTTP POST to target_url│
         │ <─────────────────────────────────│  │  (含 X-Ghost-Signature) │
         │     Webhook 回调                   │  └──────────┬──────────────┘
         │                                    │             │
         │  3. 响应 (2xx/4xx/5xx)             │             │
         │ ────────────────────────────────> │             ▼
         │                                    │  ┌─────────────────────────┐
         │                                    │  │  状态回写 (update)      │
         │                                    │  │  last_triggered_at      │
         │                                    │  │  last_triggered_status  │
         │                                    │  │  last_triggered_error   │
         │                                    │  └─────────────────────────┘
```

### 完整时序流程

以文章发布为例，完整协作链如下：

#### Step 1: 第三方调用 API 创建/更新文章

```http
POST /ghost/api/admin/posts/
Authorization: Ghost eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjVmNGUzZDJjMWIwYTlmOGU3ZDZjNWI0YSJ9...
Content-Type: application/json

{
  "posts": [{
    "title": "Hello World",
    "status": "published"
  }]
}
```

#### Step 2: 模型层事件触发

**Post 模型事件逻辑** (`ghost/core/core/server/models/post.js:379-474`):

```javascript
// 新建文章时
onSaved: function onSaved(model, options) {
    const status = model.get('status');

    model.emitChange('added', options);

    if (['published', 'scheduled'].indexOf(status) !== -1) {
        model.emitChange(status, options);  // 触发 'post.published' 事件
    }
},

// 更新文章时
onUpdated: function onUpdated(model, options) {
    model.statusChanging = model.get('status') !== model.previous('status');
    model.isPublished = model.get('status') === 'published';
    model.wasPublished = model.previous('status') === 'published';

    if (model.statusChanging) {
        if (model.wasPublished) {
            model.emitChange('unpublished', options);
        }

        if (model.isPublished) {
            model.emitChange('published', options);  // 触发 'post.published'
        }
    } else {
        if (model.isPublished) {
            model.emitChange('published.edited', options);  // 触发 'post.published.edited'
        }
    }

    model.emitChange('edited', options);
}
```

**事件发射机制** (`ghost/core/core/server/models/post.js:357-368`):

```javascript
emitChange: function emitChange(event, options = {}) {
    let eventToTrigger;
    let resourceType = this.get('type');

    if (options.usePreviousAttribute) {
        resourceType = this.previous('type');
    }

    eventToTrigger = resourceType + '.' + event;  // 如 'post.published'

    ghostBookshelf.Model.prototype.emitChange.bind(this)(this, eventToTrigger, options);
}
```

#### Step 3: Webhook 监听注册

**Webhook 监听器** (`ghost/core/core/server/services/webhooks/listen.js:10-68`):

```javascript
const WEBHOOKS = [
    'site.changed',
    
    // 文章事件
    'post.added', 'post.deleted', 'post.edited',
    'post.published', 'post.published.edited',
    'post.unpublished',
    'post.scheduled', 'post.unscheduled', 'post.rescheduled',
    
    // 页面事件
    'page.added', 'page.deleted', 'page.edited',
    'page.published', 'page.published.edited',
    'page.unpublished',
    'page.scheduled', 'page.unscheduled', 'page.rescheduled',
    
    // 标签事件
    'tag.added', 'tag.edited', 'tag.deleted',
    
    // 成员事件
    'member.added', 'member.deleted', 'member.edited',
    
    // 关联事件
    'post.tag.attached', 'post.tag.detached',
    'page.tag.attached', 'page.tag.detached'
];

const listen = async () => {
    const webhookTrigger = new WebhookTrigger({models, payload, limitService});
    _.each(WEBHOOKS, (event) => {
        // @NOTE: The early exit makes sure the listeners are only registered once.
        if (events.hasRegisteredListener(event, 'processWebhookTrigger')) {
            return;
        }

        events.on(event, function processWebhookTrigger(model, options) {
            // CASE: avoid triggering webhooks when importing
            if (options && options.importing) {
                return;
            }

            webhookTrigger.trigger(event, model);
        });
    });
};
```

#### Step 4: Webhook 触发投递

**触发器实现** (`ghost/core/core/server/services/webhooks/webhook-trigger.js:105-151`):

```javascript
async trigger(event, model) {
    const response = {
        onSuccess: this.onSuccess.bind(this),
        onError: this.onError.bind(this)
    };

    // 1. 获取所有订阅该事件的 webhook
    const hooks = await this.getAll(event);

    debug(`${hooks.models.length} webhooks found for ${event}.`);

    for (const webhook of hooks.models) {
        // 2. 生成 payload
        const hookPayload = await this.payload(webhook.get('event'), model);
        const reqPayload = JSON.stringify(hookPayload);
        
        const url = webhook.get('target_url');
        const secret = webhook.get('secret') || '';
        const ts = Date.now();

        // 3. 构建请求头
        const headers = {
            'Content-Length': Buffer.byteLength(reqPayload),
            'Content-Type': 'application/json',
            'Content-Version': `v${ghostVersion.safe}`
        };

        // 4. 如果配置了 secret，生成签名
        if (secret !== '') {
            const signature = crypto.createHmac('sha256', secret)
                .update(`${reqPayload}${ts}`)
                .digest('hex');
            headers['X-Ghost-Signature'] = `sha256=${signature}, t=${ts}`;
        }

        // 5. 发送请求
        const opts = {
            method: 'POST',
            body: reqPayload,
            headers,
            timeout: {
                request: 2 * 1000  // 2 秒超时
            },
            retry: {
                limit: process.env.NODE_ENV?.startsWith('test') ? 0 : 5  // 重试 5 次
            }
        };

        logging.info(`Triggering webhook for "${webhook.get('event')}" with url "${url}"`);

        await this.request(url, opts)
            .then(response.onSuccess(webhook))
            .catch(response.onError(webhook));
    }
}
```

#### Step 5: 状态回写

**成功处理** (`ghost/core/core/server/services/webhooks/webhook-trigger.js:58-86`):

```javascript
update(webhook, data) {
    this.models
        .Webhook
        .edit({
            last_triggered_at: Date.now(),
            last_triggered_status: data.statusCode,
            last_triggered_error: data.error || null
        }, {id: webhook.id, autoRefresh: false})
        .catch(() => {
            logging.warn(`Unable to update "last_triggered" for webhook: ${webhook.id}`);
        });
},

onSuccess(webhook) {
    return (res) => {
        this.update(webhook, {
            statusCode: res.statusCode  // 如 200, 201, 204
        });
    };
}
```

**错误处理** (`ghost/core/core/server/services/webhooks/webhook-trigger.js:88-103`):

```javascript
onError(webhook) {
    return (err) => {
        // 410 Gone: 目标资源已永久删除，自动销毁 webhook
        if (err.statusCode === 410) {
            logging.info(`Webhook destroyed (410 response) for "${webhook.get('event')}" with url "${webhook.get('target_url')}".`);

            return this.destroy(webhook);
        }

        // 记录错误信息
        this.update(webhook, {
            statusCode: err.statusCode,
            error: `Request failed: ${err.code || 'unknown'}`
        });

        logging.error(`[WEBHOOK_DELIVERY_FAILURE] url=${webhook.get('target_url') || 'unknown'} status=${err.statusCode || 'none'} error_code=${err.code || 'unknown'} message=${err.message || ''}`, err);
    };
}
```

### 1. 第三方调用 API

#### Content API 调用示例

```http
GET /ghost/api/content/posts/?key=9c8e78b7d6f5a4s3d2f1g0h9j8k7l6m5
Accept: application/json
```

#### Admin API 调用示例

**生成 JWT Token** (Node.js):
```javascript
const jwt = require('jsonwebtoken');

// API Key 信息
const apiKeyId = '5f4e3d2c1b0a9f8e7d6c5b4a';  // kid
const apiSecret = Buffer.from('abcdef123456...', 'hex');  // secret (hex)

// 生成 Token
const token = jwt.sign(
    {
        // 自定义 claims
    },
    apiSecret,
    {
        algorithm: 'HS256',
        audience: '/ghost/api/admin/',
        expiresIn: '5m',
        header: {
            kid: apiKeyId
        }
    }
);
```

**API 调用**:
```http
POST /ghost/api/admin/posts/
Authorization: Ghost eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjVmNGUzZDJjMWIwYTlmOGU3ZDZjNWI0YSJ9...
Content-Type: application/json

{
  "posts": [{
    "title": "Hello World",
    "status": "published"
  }]
}
```

### 2. Webhook 事件机制

#### 支持的事件类型

**定义位置**: `ghost/core/core/server/services/webhooks/listen.js:10-45`

| 事件类别 | 事件类型 |
|----------|----------|
| **站点** | `site.changed` |
| **文章** | `post.added`, `post.deleted`, `post.edited`, `post.published`, `post.published.edited`, `post.unpublished`, `post.scheduled`, `post.unscheduled`, `post.rescheduled` |
| **页面** | `page.added`, `page.deleted`, `page.edited`, `page.published`, `page.published.edited`, `page.unpublished`, `page.scheduled`, `page.unscheduled`, `page.rescheduled` |
| **标签** | `tag.added`, `tag.edited`, `tag.deleted` |
| **成员** | `member.added`, `member.deleted`, `member.edited` |
| **关联** | `post.tag.attached`, `post.tag.detached`, `page.tag.attached`, `page.tag.detached` |

### 3. Webhook 签名验证

当 Webhook 配置了 `secret` 时，Ghost 会在请求头中包含签名：

```
X-Ghost-Signature: sha256=<hmac_signature>, t=<timestamp>
```

**签名算法**:
```javascript
signature = HMAC-SHA256(secret, payload + timestamp)
```

**第三方验证示例** (Node.js):
```javascript
const crypto = require('crypto');

function verifyWebhook(req, secret) {
    const signatureHeader = req.headers['x-ghost-signature'];
    const [signaturePart, timestampPart] = signatureHeader.split(', ');
    
    const signature = signaturePart.split('=')[1];  // sha256=xxx
    const timestamp = timestampPart.split('=')[1];  // t=xxx
    
    // 可选：检查时间戳是否在合理范围内（防止重放攻击）
    const now = Date.now();
    if (now - parseInt(timestamp) > 5 * 60 * 1000) {
        throw new Error('Webhook timestamp expired');
    }
    
    // 计算期望的签名
    const expectedSignature = crypto.createHmac('sha256', secret)
        .update(req.rawBody + timestamp)
        .digest('hex');
    
    // 比较签名（使用恒定时间比较防止时序攻击）
    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedSignature)
    );
}
```

---

## 多集成隔离与凭证轮换

### 1. 多集成隔离机制

#### Webhook 强关联与 API Key 软关联的关键差异

| 维度 | Webhook | API Key |
|------|---------|---------|
| **外键可空性** | `nullable: false` | `nullable: true` |
| **级联删除** | `cascadeDelete: true` | 无明确配置 |
| **是否必须隶属集成** | 是 | 否（内部 API Key 可独立存在） |
| **删除集成时的行为** | 自动级联删除所有关联 webhook | 不会自动删除，需应用层处理 |

**Schema 定义对比**:

```javascript
// Webhook - 强关联 (schema.js:364-366)
integration_id: {
    type: 'string', 
    maxlength: 24, 
    nullable: false,                          // 必须有值
    references: 'integrations.id', 
    cascadeDelete: true                       // 级联删除
}

// API Key - 软关联 (schema.js:389-390)
// integration_id is nullable to allow "internal" API keys that don't show in the UI
integration_id: {
    type: 'string', 
    maxlength: 24, 
    nullable: true                            // 可以为空
}
```

#### 数据隔离：删除行为差异

**删除集成的级联行为**:

```
删除 Integration (id=123)
        │
        ├──> Webhook (强关联)
        │     ├── webhook_1 (integration_id=123) → 自动级联删除 ✅
        │     ├── webhook_2 (integration_id=123) → 自动级联删除 ✅
        │     └── webhook_3 (integration_id=123) → 自动级联删除 ✅
        │
        └──> API Key (软关联)
              ├── api_key_1 (integration_id=123) → 不会自动删除 ⚠️
              └── api_key_2 (integration_id=123) → 不会自动删除 ⚠️
```

**实际影响**:
- Webhook 与集成是**生命周期绑定**的
- API Key 可以**独立存在**，即使集成被删除
- 这意味着：
  - 删除集成时，webhook 回调会立即停止（因为 webhook 记录已被删除）
  - 但 API Key 可能仍然有效（如果没有被显式清理）

#### 权限隔离：共享角色边界

**关键设计：多个自定义集成共享 "Admin Integration" 角色**

```
┌─────────────────────────────────────────────────────────────────┐
│                        "Admin Integration" Role                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Permissions: post:all, member:all, setting:all, ...    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          ▲                                      │
│                          │ belongs_to                           │
│                          │                                      │
│    ┌─────────────────────┼─────────────────────┐               │
│    │                     │                     │               │
│    ▼                     ▼                     ▼               │
│  ┌───────┐             ┌───────┐             ┌───────┐         │
│  │ Key A │             │ Key B │             │ Key C │         │
│  │(Int A)│             │(Int B)│             │(Int C)│         │
│  └───┬───┘             └───┬───┘             └───┬───┘         │
│      │                     │                     │               │
│      ▼                     ▼                     ▼               │
│  ┌───────┐             ┌───────┐             ┌───────┐         │
│  │ Int A │             │ Int B │             │ Int C │         │
│  │(Custom)│            │(Custom)│            │(Custom)│         │
│  └───────┘             └───────┘             └───────┘         │
└─────────────────────────────────────────────────────────────────┘
```

**权限加载流程** (`ghost/core/core/server/services/permissions/providers.js:58-74`):

```javascript
apiKey(id) {
    return models.ApiKey.findOne({id}, {withRelated: ['role', 'role.permissions']})
        .then((foundApiKey) => {
            // api keys have a belongs_to relationship to a role and no individual permissions
            const permissions = foundApiKey.related('role').related('permissions').models;
            const roles = [foundApiKey.toJSON().role];

            return {permissions, roles};
        });
}
```

**实际影响**:

1. **没有按集成 ID 的资源隔离**:
   - 集成 A 的 API Key 可以读取/修改集成 B 创建的文章
   - 集成 A 的 API Key 可以修改集成 B 的 webhook 配置
   - 所有集成共享相同的权限边界

2. **权限检查逻辑** (`ghost/core/core/server/models/post.js:1414`):
   ```javascript
   isIntegration = loadedPermissions.apiKey && _.some(loadedPermissions.apiKey.roles, {name: 'Admin Integration'});
   ```
   - 只检查角色名称，不检查集成 ID

3. **内部/核心集成有独立角色**:
   - `Ghost Explore Integration`: 只有 `explore: read` 权限
   - `Self-Serve Migration Integration`: `db: importContent`, `member: add`, `tag: read`
   - `Scheduler Integration`: 定时发布相关权限
   - `DB Backup Integration`: 备份相关权限
   - 这些是 `type: internal` 或 `type: core` 的集成，不是自定义集成

#### 集成类型限制

**内部/核心集成豁免限制**: `ghost/core/core/server/services/auth/api-key/admin.js:140-146`

```javascript
// 只对 custom 和 builtin 类型的集成应用限制
if (limitService.isLimited('customIntegrations')
        && !['internal', 'core'].includes(apiKey.relations.integration.get('type'))) {
    await limitService.errorIfWouldGoOverLimit('customIntegrations');
}
```

#### Webhook 触发隔离

**按集成类型过滤 webhook**: `ghost/core/core/server/services/webhooks/webhook-trigger.js:30-56`

```javascript
async getAll(event) {
    if (this.limitService.isLimited('customIntegrations')) {
        const overLimit = await this.limitService.checkWouldGoOverLimit('customIntegrations');
        
        if (overLimit) {
            logging.info(`Skipping all non-internal webhooks for event ${event}...`);
            
            // 只返回 internal 类型集成的 webhook
            const result = await this.models.Webhook.findAllByEvent(event, {
                context: {internal: true},
                withRelated: ['integration']
            });
            
            return {
                models: result?.models?.filter((model) => {
                    return model.related('integration')?.get('type') === 'internal';
                }) || []
            };
        }
    }
    
    return this.models.Webhook.findAllByEvent(event, {context: {internal: true}});
}
```

### 2. 凭证轮换机制

#### API Key 轮换流程

**服务层实现**: `ghost/core/core/server/services/integrations/integrations-service.js:14-51`

```javascript
async edit(data, options) {
    // 如果请求中包含 keyid，则执行密钥轮换
    if (options.keyid) {
        const model = await this.ApiKeyModel.findOne({id: options.keyid});
        
        if (!model) {
            throw new NotFoundError({resource: 'ApiKey'});
        }
        
        try {
            // 刷新密钥（生成新的 secret）
            await this.ApiKeyModel.refreshSecret(
                model.toJSON(), 
                Object.assign({}, options, {id: options.keyid})
            );
            
            // 返回更新后的集成信息（包含新的 API Key）
            return await this.IntegrationModel.findOne(
                {id: options.id}, 
                {withRelated: ['api_keys', 'webhooks']}
            );
        } catch (err) {
            throw new InternalServerError({err});
        }
    }
    
    // 普通的集成编辑
    return await this.IntegrationModel.edit(data, Object.assign(options, {require: true}));
}
```

**模型层实现**: `ghost/core/core/server/models/api-key.js:60-64`

```javascript
{
    refreshSecret(data, options) {
        // 生成新的 secret
        const secret = security.secret.create(data.type);
        // 更新数据库中的 secret
        return this.edit(Object.assign({}, data, {secret}), options);
    }
}
```

**审计日志**: `ghost/core/core/server/models/api-key.js:55-59`

```javascript
onUpdated(model, options) {
    // 如果 secret 发生变化，记录 "refreshed" 动作
    if (this.previous('secret') !== this.get('secret')) {
        this.addAction(model, 'refreshed', options);
    }
}
```

#### 轮换与删除的差异

| 操作 | API Key 行为 | Webhook 行为 |
|------|--------------|--------------|
| **轮换 API Key** | 生成新的 `secret`，`id` 和 `role_id` 不变 | 无影响，继续正常工作 |
| **删除集成** | 不会自动删除（软关联） | 自动级联删除（强关联） |
| **手动删除 API Key** | 从 `api_keys` 表删除记录 | 无影响 |

**轮换时序**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Key 轮换流程                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 客户端请求轮换                                                   │
│     PUT /integrations/{int_id}/?keyid={key_id}                      │
│                                                                     │
│  2. 服务层验证                                                       │
│     ┌─> ApiKeyModel.findOne({id: keyid})                            │
│     └─> 验证 key 属于该集成                                          │
│                                                                     │
│  3. 生成新 secret                                                    │
│     ┌─> security.secret.create('admin' | 'content')                 │
│     └─> ApiKeyModel.edit({secret: new_secret})                      │
│                                                                     │
│  4. 记录审计日志                                                     │
│     ┌─> onUpdated 触发                                               │
│     └─> this.addAction(model, 'refreshed', options)                 │
│                                                                     │
│  5. 返回更新后的集成信息                                             │
│     ┌─> IntegrationModel.findOne(...)                                │
│     └─> 包含新的 api_keys (但 secret 只在创建时返回一次？)            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### API 调用方式

**端点**: `PUT /ghost/api/admin/integrations/{integration_id}/?keyid={api_key_id}`

**控制器**: `ghost/core/core/server/api/endpoints/integrations.js:73-102`

```javascript
edit: {
    options: [
        'id',
        'keyid',  // 可选参数，用于指定要轮换的 API Key
        'include'
    ],
    query({data, options}) {
        return integrationsService.edit(data, options);
    }
}
```

#### 轮换最佳实践

**建议流程**:
1. 调用轮换接口生成新的 API Key
2. 更新第三方应用使用新密钥
3. 验证新密钥正常工作
4. 确认旧密钥已失效（因为轮换是原地替换，旧 secret 立即失效）

**注意**: 
- 由于 Ghost 当前实现是**直接替换 secret**（原地更新），不是创建新 Key
- 轮换后旧密钥立即失效，没有过渡期
- 建议在低峰期执行轮换
- 确保第三方应用能快速更新配置

---

## 安全机制

### 1. API Key 安全

**密钥强度**:
- `secret` 字段: `validations: {isLength: {min: 26, max: 128}}`
- 由 `@tryghost/security` 模块生成，确保随机性

**传输安全**:
- Admin API: JWT 签名，密钥不直接传输
- Content API: 建议使用 HTTPS 传输

**过期机制**:
- Admin API JWT: `maxAge: '5m'` (5 分钟过期)
- 支持 `ignoreMaxAge` 用于特殊场景（如定时发布）

### 2. Webhook 安全

**签名验证**:
- 支持 HMAC-SHA256 签名
- 包含时间戳防止重放攻击
- 签名格式: `sha256=<signature>, t=<timestamp>`

**错误处理**:
- 410 响应自动销毁 webhook
- 记录错误状态码和错误信息
- 最多重试 5 次

**网络安全**:
```javascript
// webhook-trigger.js:20-26
if (config.get('security:allowWebhookInternalIPs')) {
    this.request = require('@tryghost/request');
} else {
    // 默认阻止访问内部 IP，防止 SSRF
    this.request = require('../../lib/request-external');
}
```

### 3. 授权机制

**认证流程**: `ghost/core/core/server/services/auth/authenticate.js:5-10`

```javascript
const authenticate = {
    // Admin API: 先尝试 API Key 认证，再尝试 Session 认证
    authenticateAdminApi: [apiKeyAuth.admin.authenticate, session.authenticate],
    
    // 支持 URL 中的 token（用于定时发布等场景）
    authenticateAdminApiWithUrl: [apiKeyAuth.admin.authenticateWithUrl],
    
    // Content API: 先尝试 API Key 认证，再尝试 Member Token 认证
    authenticateContentApi: [apiKeyAuth.content.authenticateContentApiKey, members.authenticateMembersToken]
};
```

**请求上下文**:
- 认证成功后，`req.api_key` 包含 API Key 信息
- 可用于后续的权限检查和审计日志

### 4. 限制机制

**自定义集成限制**:
- 基于 `limitService.isLimited('customIntegrations')`
- 可限制自定义集成的数量
- 内部和核心集成不受限制

**检查时机**:
- API 认证时
- Webhook 触发时
- 创建新集成时

---

## 总结

Ghost 的自定义集成系统提供了完善的第三方应用接入能力：

### 关键设计要点

1. **API Key 机制**: 支持两种类型（Content/Admin），分别用于只读和管理操作
   
2. **共享角色权限边界**: 
   - 所有自定义集成的 Admin API Key 共享 "Admin Integration" 角色
   - **不按集成 ID 进行资源隔离**
   - 任何自定义集成都可以访问所有文章、成员、设置等资源
   - 内部/核心集成有独立的受限角色

3. **强关联 vs 软关联**:
   - **Webhook**: 强关联（`integration_id` 非空 + `cascadeDelete: true`），删除集成时自动级联删除
   - **API Key**: 软关联（`integration_id` 可空），删除集成时不会自动删除
   - API Key 可以独立存在（如内部 API Key）

4. **JWT 认证**: Admin API 使用 JWT 签名请求，密钥不直接传输

5. **Webhook 完整协作链**: 
   - 模型写入 → `emitChange()` → 事件总线 → `WebhookTrigger.trigger()` → HTTP POST → 状态回写（`last_triggered_at/status/error`）
   - 支持 30+ 种事件类型，双向通信能力完善

6. **签名验证**: Webhook 支持 HMAC-SHA256 签名，包含时间戳防重放攻击

7. **凭证轮换**: 
   - 支持 API Key 原地替换（生成新 secret）
   - 记录 `refreshed` 审计日志
   - 轮换后旧密钥立即失效，无过渡期

8. **安全防护**: SSRF 防护、内部 IP 限制、超时和重试机制

该设计遵循了现代 API 安全最佳实践，同时保持了良好的扩展性和易用性。
