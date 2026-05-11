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
    integration_id: {type: 'string', maxlength: 24, nullable: false, references: 'integrations.id', cascadeDelete: true},
    last_triggered_at: {type: 'dateTime', nullable: true},
    last_triggered_status: {type: 'string', maxlength: 50, nullable: true},
    last_triggered_error: {type: 'string', maxlength: 50, nullable: true},
    created_at: {type: 'dateTime', nullable: false},
    updated_at: {type: 'dateTime', nullable: true}
}
```

### 模型关系

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

**关系结构**:
```
Integration (1) ──┬──> (N) API Key
                  └──> (N) Webhook
```

每个集成可以有多个 API Key 和多个 Webhook，通过外键 `integration_id` 关联。

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
    // 1. 从 query 参数中提取 key
    let key = req.query.key;
    
    // 2. 查找匹配的 API Key
    const apiKey = await models.ApiKey.findOne({secret: key}, {withRelated: ['integration']});
    
    // 3. 验证 key 类型
    if (apiKey.get('type') !== 'content') {
        return next(new errors.UnauthorizedError({...}));
    }
    
    // 4. 检查集成限制
    if (limitService.isLimited('customIntegrations')
        && !['internal', 'core'].includes(apiKey.relations.integration.get('type'))) {
        await limitService.errorIfWouldGoOverLimit('customIntegrations');
    }
    
    // 5. 认证通过
    req.api_key = apiKey;
    next();
};
```

#### 2. Admin API Key

**认证方式**: JWT Token 在 `Authorization: Ghost <token>` 头中

**权限范围**:
- 关联 "Admin Integration" 角色
- 拥有完整的管理权限（根据角色权限配置）
- 可访问 Admin API 的所有端点

**认证流程** (`ghost/core/core/server/services/auth/api-key/admin.js:48-198`):

```javascript
const authenticateWithToken = async function apiKeyAuthenticateWithToken(originalUrl, token, ignoreMaxAge) {
    // 1. 解码 JWT 提取 kid (Key ID)
    const decoded = jwt.decode(token, {complete: true});
    const apiKeyId = decoded.header.kid;
    
    // 2. 通过 kid 查找 API Key
    const apiKey = await models.ApiKey.findOne({id: apiKeyId}, {withRelated: ['integration']});
    
    // 3. 验证 key 类型
    if (apiKey.get('type') !== 'admin') {
        throw new errors.UnauthorizedError({...});
    }
    
    // 4. 检查集成限制
    if (limitService.isLimited('customIntegrations')
        && !['internal', 'core'].includes(apiKey.relations.integration.get('type'))) {
        await limitService.errorIfWouldGoOverLimit('customIntegrations');
    }
    
    // 5. 验证 JWT 签名和 claims
    const secret = Buffer.from(apiKey.get('secret'), 'hex');
    jwt.verify(token, secret, {
        audience: new RegExp(`/?${api}/?$`),
        algorithms: ['HS256'],
        maxAge: '5m'
    });
    
    // 6. 认证通过
    return {apiKey, user: null};
};
```

### 权限模型 (Role-based Scope)

**Admin Integration 角色**

从测试文件 `ghost/core/test/integration/migrations/migration.test.js` 中可以看到，"Admin Integration" 角色拥有以下权限：

| 权限类别 | 具体权限 |
|----------|----------|
| **文章管理** | Browse, Read, Edit, Add, Delete, Publish posts |
| **标签管理** | Browse, Read, Edit, Add, Delete tags |
| **设置管理** | Browse, Read, Edit settings |
| **主题管理** | Browse, Edit, Activate, Upload, Download, Delete themes |
| **用户管理** | Browse, Read, Edit, Add, Delete users, Assign roles |
| **Webhook 管理** | Add, Edit, Delete webhooks |
| **成员管理** | Add Members |
| **产品管理** | Browse, Read, Edit, Add Products |
| **通讯管理** | Browse, Read, Edit, Add newsletters |
| **评论管理** | Browse, Read, Edit, Add, Delete, Moderate comments |

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
    // enforce roles which are currently hardcoded
    // - admin key = Administrator role
    // - content key = no role
    if (this.hasChanged('type') || this.hasChanged('role_id')) {
        if (this.get('type') === 'admin') {
            // 自动绑定 "Admin Integration" 角色
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
┌─────────────────┐          ┌─────────────────┐
│   第三方应用     │          │   Ghost 平台    │
└────────┬────────┘          └────────┬────────┘
         │                           │
         │  1. API 调用 (JWT/Key)    │
         │ ────────────────────────> │
         │                           │
         │  2. 操作执行              │
         │                           │
         │  3. 事件触发              │
         │ <──────────────────────── │
         │     Webhook (HTTP POST)   │
         │                           │
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
```

#### Webhook 触发流程

**监听注册**: `ghost/core/core/server/services/webhooks/listen.js:47-68`

```javascript
const listen = async () => {
    const webhookTrigger = new WebhookTrigger({models, payload, limitService});
    _.each(WEBHOOKS, (event) => {
        events.on(event, function processWebhookTrigger(model, options) {
            if (options && options.importing) {
                return;  // 导入时不触发 webhook
            }
            webhookTrigger.trigger(event, model);
        });
    });
};
```

**触发器实现**: `ghost/core/core/server/services/webhooks/webhook-trigger.js:105-151`

```javascript
async trigger(event, model) {
    // 1. 获取所有订阅该事件的 webhook
    const hooks = await this.getAll(event);
    
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
            timeout: { request: 2 * 1000 },  // 2 秒超时
            retry: { limit: 5 }  // 重试 5 次（测试环境为 0）
        };
        
        await this.request(url, opts)
            .then(this.onSuccess(webhook))
            .catch(this.onError(webhook));
    }
}
```

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

### 4. Webhook 响应处理

**成功处理**: `ghost/core/core/server/services/webhooks/webhook-trigger.js:80-86`

```javascript
onSuccess(webhook) {
    return (res) => {
        this.update(webhook, {
            statusCode: res.statusCode
        });
    };
}
```

**错误处理**: `ghost/core/core/server/services/webhooks/webhook-trigger.js:88-103`

```javascript
onError(webhook) {
    return (err) => {
        // 410 Gone: 目标资源已永久删除，自动销毁 webhook
        if (err.statusCode === 410) {
            logging.info(`Webhook destroyed (410 response)...`);
            return this.destroy(webhook);
        }
        
        // 记录错误信息
        this.update(webhook, {
            statusCode: err.statusCode,
            error: `Request failed: ${err.code || 'unknown'}`
        });
        
        logging.error(`[WEBHOOK_DELIVERY_FAILURE] url=${url} status=${statusCode}...`);
    };
}
```

---

## 多集成隔离与凭证轮换

### 1. 多集成隔离机制

#### 数据隔离

**通过外键关联实现隔离**:
- 每个 `api_keys` 记录都有 `integration_id`
- 每个 `webhooks` 记录都有 `integration_id`
- 删除集成时级联删除相关的 API Key 和 Webhook

```sql
-- webhooks 表定义
integration_id: {
    type: 'string', 
    maxlength: 24, 
    nullable: false, 
    references: 'integrations.id', 
    cascadeDelete: true
}
```

#### 权限隔离

**通过角色和 API Key 类型隔离**:
- Admin API Key 绑定 "Admin Integration" 角色
- Content API Key 无角色（只读公开数据）
- 认证中间件验证 `type` 字段

```javascript
// 只允许 admin 类型的 key 访问 Admin API
if (apiKey.get('type') !== 'admin') {
    throw new errors.UnauthorizedError({
        message: 'Invalid API Key type',
        code: 'INVALID_API_KEY_TYPE'
    });
}
```

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
1. 生成新的 API Key (或轮换现有 Key)
2. 更新第三方应用使用新密钥
3. 验证新密钥正常工作
4. （可选）删除旧密钥

**注意**: 由于 Ghost 当前实现是直接替换 secret，建议：
- 在低峰期执行轮换
- 确保第三方应用能快速更新配置
- 考虑实现平滑过渡（支持新旧密钥并行一段时间）

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

1. **API Key 机制**: 支持两种类型（Content/Admin），分别用于只读和管理操作
2. **基于角色的权限**: Admin API Key 绑定 "Admin Integration" 角色，实现细粒度权限控制
3. **JWT 认证**: Admin API 使用 JWT 签名请求，密钥不直接传输
4. **Webhook 事件**: 支持 30+ 种事件类型，双向通信能力完善
5. **签名验证**: Webhook 支持 HMAC-SHA256 签名，确保请求真实性
6. **多集成隔离**: 通过外键和类型字段实现数据和权限隔离
7. **凭证轮换**: 支持 API Key 刷新，并记录审计日志
8. **安全防护**: SSRF 防护、内部 IP 限制、超时和重试机制

该设计遵循了现代 API 安全最佳实践，同时保持了良好的扩展性和易用性。
