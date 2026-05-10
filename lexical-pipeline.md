# Lexical 编辑器工作流程分析报告

## 1. 概述

Ghost 使用 Lexical 编辑器作为新一代内容编辑工具，替代了之前的 Mobiledoc/Koenig 编辑器。本文档详细分析 Lexical 编辑器从后台编辑到服务端渲染的完整协作过程。

## 2. 编辑器与后台主框架的嵌套集成

### 2.1 技术栈架构

Ghost 后台采用 **Ember.js + React** 混合架构：

- **主框架**: Ember.js（传统的 Ghost Admin）
- **编辑器**: React（使用 Lexical 库）

### 2.2 嵌套集成方式

编辑器通过以下层次嵌套集成到 Ember 后台中：

#### 第一层：Ember 控制器与路由

```
ghost/admin/app/routes/lexical-editor.js
ghost/admin/app/controllers/lexical-editor.js
```

这些文件定义了编辑器的路由配置和用户交互处理。

#### 第二层：Ember 组件包装

**`gh-koenig-editor-lexical.js`** (`ghost/admin/app/components/gh-koenig-editor-lexical.js`)

这是最外层的 Ember 组件，负责：
- 管理编辑器的整体布局（标题、摘要、正文区域）
- 处理标题和摘要字段的输入
- 管理焦点切换（标题 → 摘要 → 编辑器）
- 注册编辑器 API 实例

#### 第三层：Ember-React 桥接组件

**`koenig-lexical-editor.js`** (`ghost/admin/app/components/koenig-lexical-editor.js`)

这是核心的桥接组件，实现了 Ember 与 React 的集成：

##### 关键特性：

1. **React 组件嵌入**
   - 使用 `React` 和 `Suspense` 动态加载 React 组件
   - 通过 `ErrorHandler` 组件捕获错误并上报 Sentry

2. **编辑器资源管理**
   ```javascript
   editorResource = this.koenig.resource;
   ```
   使用 Ember 的资源管理机制延迟加载 Koenig Lexical 编辑器

3. **双重编辑器实例**
   - 主编辑器实例（用于实际编辑）
   - 辅助编辑器实例（隐藏，用于初始化和同步）
   ```javascript
   <KGEditorComponent />
   <KGEditorComponent isInitInstance={true} />
   ```

4. **配置传递**
   - 向 React 编辑器传递 Ember 服务和配置
   - 包括：文件上传、嵌入获取、搜索链接、Stripe 配置等

5. **事件回调**
   - `onChange`: 内容变化时触发
   - `onError`: 错误处理
   - `registerAPI`: 注册编辑器 API

#### 第四层：React 编辑器核心

**`@tryghost/koenig-lexical`**（外部包）

实际的 Lexical 编辑器实现，包含：
- KoenigComposer: 编辑器容器
- KoenigEditor: 编辑器主体
- WordCountPlugin: 字数统计插件
- TKCountPlugin: TK 标记统计插件

### 2.3 数据流

```
Ember 控制器
    ↓
gh-koenig-editor-lexical (Ember 组件)
    ↓ (配置 + 回调)
koenig-lexical-editor (Ember-React 桥接)
    ↓ (React 组件)
KoenigComposer / KoenigEditor (React)
    ↓
Lexical 编辑引擎
```

## 3. 编辑结果的数据结构与存储

### 3.1 Lexical 数据结构

Lexical 使用 JSON 格式存储编辑器状态，核心结构如下：

```json
{
  "root": {
    "children": [
      {
        "type": "paragraph",
        "children": [
          {
            "type": "text",
            "text": "Hello ",
            "format": 0,
            "version": 1
          },
          {
            "type": "text",
            "text": "World",
            "format": 1,
            "version": 1
          }
        ],
        "direction": "ltr",
        "format": "",
        "indent": 0,
        "version": 1
      },
      {
        "type": "image",
        "src": "/content/images/2024/01/example.jpg",
        "width": 1920,
        "height": 1080,
        "caption": "Example image",
        "cardWidth": "wide"
      }
    ],
    "type": "root",
    "direction": "ltr",
    "format": "",
    "indent": 0,
    "version": 1
  }
}
```

### 3.2 数据库存储

#### 数据库表结构

**`posts` 表** (`ghost/core/core/server/data/schema/schema.js:62-100`)

```javascript
posts: {
    id: {type: 'string', maxlength: 24, nullable: false, primary: true},
    uuid: {type: 'string', maxlength: 36, nullable: false, index: true},
    title: {type: 'string', maxlength: 2000, nullable: false},
    slug: {type: 'string', maxlength: 191, nullable: false},
    mobiledoc: {type: 'text', maxlength: 1000000000, fieldtype: 'long', nullable: true},
    lexical: {type: 'text', maxlength: 1000000000, fieldtype: 'long', nullable: true},
    html: {type: 'text', maxlength: 1000000000, fieldtype: 'long', nullable: true},
    plaintext: {type: 'text', maxlength: 1000000000, fieldtype: 'long', nullable: true},
    // ... 其他字段
}
```

关键字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `lexical` | LONG TEXT | 存储 Lexical JSON 字符串（编辑器原始格式） |
| `mobiledoc` | LONG TEXT | 旧版编辑器格式（向后兼容） |
| `html` | LONG TEXT | 预渲染的 HTML（用于快速访问） |
| `plaintext` | LONG TEXT | 纯文本版本（用于搜索等） |

#### 存储策略

**双格式存储**：
- `lexical`: 原始编辑器数据（可编辑）
- `html`: 预渲染的 HTML（只读，服务端生成）

**互斥原则** (`ghost/core/core/server/models/post.js:560-564`):
```javascript
if (this.previous('mobiledoc') && this.get('lexical')) {
    this.set('lexical', null);
} else if (this.get('mobiledoc') && this.get('lexical')) {
    this.set('mobiledoc', null);
}
```

同一篇文章只能使用一种格式（Lexical 或 Mobiledoc）。

### 3.3 URL 转换机制

#### 存储时的 URL 转换

**写入数据库前** (`ghost/core/core/server/models/post.js:178-239`):

```javascript
formatOnWrite(attrs) {
    const urlTransformMap = {
        lexical: {
            method: 'lexicalToTransformReady',
            options: {
                nodes: lexicalLib.nodes,
                transformMap: lexicalLib.urlTransformMap
            }
        },
        // ...
    };
    // 将绝对 URL 转换为 __GHOST_URL__ 占位符
}
```

#### 读取时的 URL 转换

**从数据库读取后** (`ghost/core/core/server/models/post.js:152-175`):

```javascript
parse() {
    // 将 __GHOST_URL__ 占位符转换回绝对 URL
    ['mobiledoc', 'lexical', 'html', 'plaintext', ...].forEach((attr) => {
        if (attrs[attr]) {
            attrs[attr] = urlUtils.transformReadyToAbsolute(attrs[attr]);
        }
    });
}
```

**目的**：支持站点 URL 变更时的无缝迁移。

## 4. 服务端渲染流程

### 4.1 HTML 生成时机

在 Post 模型的 `onSaving` 钩子中触发 (`ghost/core/core/server/models/post.js:714-735`):

```javascript
// CASE: lexical 已变更，生成 html
// CASE: ?force_rerender=true 通过 Admin API 传递
// CASE: html 为 null 但 lexical 存在（迁移和导入时）
if (
    !this.get('mobiledoc') &&
    (
        this.hasChanged('lexical')
        || options.force_rerender
        || (!this.get('html') && (options.migrating || options.importing))
    )
) {
    try {
        this.set('html', await lexicalLib.render(this.get('lexical'), {transacting: options.transacting}));
    } catch (err) {
        throw new errors.ValidationError({...});
    }
}
```

### 4.2 渲染核心库

**`ghost/core/core/server/lib/lexical.js`**

这是 Lexical 渲染的核心入口，主要功能：

#### 空白文档模板

```javascript
get blankDocument() {
    return {
        root: {
            children: [{
                children: [],
                direction: null,
                format: '',
                indent: 0,
                type: 'paragraph',
                version: 1
            }],
            // ...
        }
    };
}
```

#### 渲染器初始化

```javascript
get lexicalHtmlRenderer() {
    if (!lexicalHtmlRenderer) {
        if (!nodes) {
            populateNodes();  // 加载 @tryghost/kg-default-nodes
        }
        const {LexicalHTMLRenderer} = require('@tryghost/kg-lexical-html-renderer');
        lexicalHtmlRenderer = new LexicalHTMLRenderer({nodes});
    }
    return lexicalHtmlRenderer;
}
```

#### 渲染方法

```javascript
async render(lexical, userOptions = {}) {
    const options = Object.assign({
        siteUuid: settingsCache.get('site_uuid'),
        siteUrl: config.get('url'),
        imageBaseUrl: config.get('urls:image') || '',
        imageOptimization: config.get('imageOptimization'),
        canTransformImage(storagePath) { /* ... */ },
        feature: {
            contentVisibility: true,
            emailCustomization: true,
            emailUniqueid: labs.isSet('emailUniqueid'),
            pictureImageFormats: labs.isSet('pictureImageFormats')
        },
        nodeRenderers: this.customNodeRenderers  // 自定义节点渲染器
    }, userOptions);

    return await this.lexicalHtmlRenderer.render(lexical, options);
}
```

### 4.3 自定义节点渲染器

**位置**: `ghost/core/core/server/services/koenig/node-renderers/`

**已注册的渲染器** (`index.js`):

| 节点类型 | 渲染器文件 | 说明 |
|---------|-----------|------|
| `audio` | audio-renderer.js | 音频卡片 |
| `bookmark` | bookmark-renderer.js | 书签卡片 |
| `button` | button-renderer.js | 按钮卡片 |
| `callout` | callout-renderer.js | 提示卡片 |
| `call-to-action` | call-to-action-renderer.js | CTA 卡片 |
| `codeblock` | codeblock-renderer.js | 代码块 |
| `email-cta` | email-cta-renderer.js | 邮件 CTA |
| `email` | email-renderer.js | 自定义 HTML 邮件卡片 |
| `embed` | embed-renderer.js | 嵌入内容 |
| `file` | file-renderer.js | 文件下载 |
| `gallery` | gallery-renderer.js | 图片画廊 |
| `header.1`, `header.2` | header-v1/v2-renderer.js | 头部卡片 |
| `horizontalrule` | horizontalrule-renderer.js | 分隔线 |
| `html` | html-renderer.js | 自定义 HTML |
| `image` | image-renderer.js | 图片卡片 |
| `markdown` | markdown-renderer.js | Markdown |
| `paywall` | paywall-renderer.js | 付费墙 |
| `product` | product-renderer.js | 产品卡片 |
| `signup` | signup-renderer.js | 注册表单 |
| `toggle` | toggle-renderer.js | 折叠卡片 |
| `transistor` | transistor-renderer.js | Transistor 音频 |
| `video` | video-renderer.js | 视频卡片 |

### 4.4 站点渲染 vs 邮件渲染

#### 站点渲染（Web）

**触发时机**: 文章保存时预渲染到 `html` 字段

**特点**:
- 完整的 HTML 结构
- 响应式图片（srcset）
- 现代图片格式（WebP, AVIF）
- 交互元素（视频播放器等）
- Lazy loading

#### 邮件渲染（Email）

**触发时机**: 发送邮件时实时渲染

**入口**: `ghost/core/core/server/services/email-service/email-renderer.js:363-383`

```javascript
async renderPostBaseHtml(post, newsletter) {
    const postUrl = this.#getPostUrl(post);

    let html;
    if (post.get('lexical')) {
        html = await this.#renderers.lexical.render(
            post.get('lexical'),
            {
                target: 'email',
                postUrl,
                design: this.#getEmailDesign(newsletter)
            }
        );
    } else {
        html = this.#renderers.mobiledoc.render(
            JSON.parse(post.get('mobiledoc')), 
            {target: 'email', postUrl}
        );
    }
    return html;
}
```

**关键差异**: 通过 `target: 'email'` 选项区分渲染目标。

## 5. 媒体处理机制

### 5.1 图片渲染

**渲染器**: `ghost/core/core/server/services/koenig/node-renderers/image-renderer.js`

#### 站点渲染特性

1. **响应式图片**
   - 生成 `srcset` 属性，提供多种尺寸
   - 设置 `sizes` 属性适配不同布局

2. **现代图片格式**
   - 使用 `<picture>` 元素
   - 优先提供 AVIF、WebP 格式
   - 保留原始格式作为回退

3. **Lazy Loading**
   ```javascript
   img.setAttribute('loading', 'lazy');
   ```

4. **尺寸优化**
   - 根据 `cardWidth` 调整显示尺寸
   - 超过最大宽度时自动缩放

#### 邮件渲染特性

1. **简化结构**
   - 不使用 `<picture>` 元素
   - 不使用 `srcset`（邮件客户端支持有限）

2. **Retina 优化**
   ```javascript
   if (options.target === 'email' && node.width && node.height) {
       // 使用 2x 分辨率图片确保在 Retina 屏幕清晰
       const srcWidth = availableImageWidths.find(width => width >= 1200);
       if (srcWidth) {
           img.setAttribute('src', `${imagesPath}/size/w${srcWidth}/${filename}`);
       }
   }
   ```

3. **Outlook 兼容**
   - 设置明确的 `width` 和 `height` 属性
   - 最大宽度限制为 600px

### 5.2 视频渲染

**渲染器**: `ghost/core/core/server/services/koenig/node-renderers/video-renderer.js`

#### 站点渲染

- 完整的 HTML5 `<video>` 元素
- 自定义播放器 UI（播放按钮、进度条、音量控制等）
- 自动播放循环视频（静音）

#### 邮件渲染

由于邮件客户端对视频支持有限，采用**图片预览 + 链接**策略：

```javascript
const emailCardTemplate = ({node, options, cardClasses}) => {
    // 使用缩略图作为背景
    // 点击跳转到文章页面
    // Outlook 使用 VML 绘制播放按钮
};
```

- 显示视频缩略图作为预览
- 点击跳转到文章 URL
- Outlook 兼容性处理（VML 语法）

### 5.3 外部媒体内联

**服务**: `ghost/core/core/server/services/media-inliner/external-media-inliner.js`

用于导入时将外部媒体下载到本地存储。

#### 核心流程

1. **查找匹配**
   ```javascript
   static findMatches(content, domain) {
       const regex = new RegExp(`(${domain}.*?)(${srcTerminationSymbols})`, 'igm');
       // 从 Lexical/Mobiledoc JSON 中查找指定域名的媒体 URL
   }
   ```

2. **下载媒体**
   ```javascript
   async getRemoteMedia(requestURL) {
       const response = await request(requestURL, {
           followRedirect: true,
           responseType: 'buffer'
       });
   }
   ```

3. **处理文件**
   - 检测文件类型
   - HEIC/HEIF 自动转换为 JPEG
   - 生成唯一文件名

4. **本地存储**
   ```javascript
   async storeMediaLocally(media) {
       const storage = this.getMediaStorage(media.extension);
       const filePath = await storage.saveRaw(media.fileBuffer, targetPath);
       return urlUtils.toTransformReady(filePath);
   }
   ```

5. **替换引用**
   - 更新 `lexical` 字段中的 URL
   - 更新 `mobiledoc` 字段中的 URL
   - 更新特征图、OG 图片等字段

## 6. 完整工作流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         后台编辑阶段                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Ember Admin (gh-koenig-editor-lexical)                             │
│         │                                                           │
│         ├─→ 标题/摘要输入（Ember 组件）                               │
│         │                                                           │
│         └─→ koenig-lexical-editor (Ember-React 桥接)                │
│                   │                                                 │
│                   └─→ React (KoenigComposer + KoenigEditor)         │
│                             │                                       │
│                             └─→ Lexical 编辑引擎                     │
│                                       │                             │
│                                       └─→ JSON 状态                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         数据保存阶段                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. API 接收 POST 请求                                               │
│     └─→ serializers/input/posts.js                                  │
│                                                                     │
│  2. Post 模型 onSaving 钩子                                          │
│     ├─→ URL 转换（absolute → __GHOST_URL__）                         │
│     ├─→ Lexical → HTML 渲染                                          │
│     │    └─→ lexicalLib.render()                                    │
│     │         ├─→ @tryghost/kg-lexical-html-renderer                │
│     │         └─→ 自定义节点渲染器                                    │
│     ├─→ HTML → Plaintext                                            │
│     └─→ 保存到数据库                                                 │
│                                                                     │
│  3. 数据库存储                                                       │
│     ├─→ lexical: JSON 字符串（可编辑源）                              │
│     ├─→ html: 预渲染 HTML（快速访问）                                 │
│     └─→ plaintext: 纯文本（搜索用）                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          ▼                                       ▼
┌─────────────────────┐                ┌───────────────────────────────┐
│    站点渲染          │                │         邮件渲染               │
├─────────────────────┤                ├───────────────────────────────┤
│                     │                │                               │
│  1. 读取 posts.html  │                │  1. 实时渲染 lexical          │
│     (预渲染内容)      │                │     target: 'email'          │
│                     │                │                               │
│  2. 直接输出到主题   │                │  2. 邮件特定处理               │
│                     │                │     ├─→ 图片简化 (无 srcset)    │
│  3. 图片特性         │                │     ├─→ Retina 优化           │
│     ├─→ srcset       │                │     ├─→ Outlook 兼容         │
│     ├─→ WebP/AVIF    │                │     └─→ 视频 → 预览图+链接    │
│     └─→ lazy loading │                │                               │
│                     │                │  3. 链接跟踪与替换            │
└─────────────────────┘                │     ├─→ 点击追踪              │
                                       │     ├─→ 会员归因              │
                                       │     └─→ 邮件个性化 (%%{uuid}%%)│
                                       │                               │
                                       │  4. Handlebars 模板渲染       │
                                       │     └─→ 邮件设计系统           │
                                       │                               │
                                       └───────────────────────────────┘
```

## 7. 关键文件索引

### 7.1 编辑器相关

| 文件路径 | 说明 |
|---------|------|
| `ghost/admin/app/components/gh-koenig-editor-lexical.js` | Ember 外层编辑器组件 |
| `ghost/admin/app/components/koenig-lexical-editor.js` | Ember-React 桥接组件 |
| `@tryghost/koenig-lexical` | Lexical 编辑器核心包（外部依赖） |

### 7.2 后端处理

| 文件路径 | 说明 |
|---------|------|
| `ghost/core/core/server/models/post.js` | Post 模型，数据存储与渲染触发 |
| `ghost/core/core/server/lib/lexical.js` | Lexical 渲染核心库 |
| `ghost/core/core/server/api/endpoints/utils/serializers/input/posts.js` | API 输入序列化 |

### 7.3 节点渲染器

| 文件路径 | 说明 |
|---------|------|
| `ghost/core/core/server/services/koenig/node-renderers/` | 自定义节点渲染器目录 |
| `ghost/core/core/server/services/koenig/node-renderers/image-renderer.js` | 图片渲染器 |
| `ghost/core/core/server/services/koenig/node-renderers/video-renderer.js` | 视频渲染器 |
| `ghost/core/core/server/services/koenig/node-renderers/email-renderer.js` | 自定义 HTML 邮件卡片渲染器 |

### 7.4 邮件服务

| 文件路径 | 说明 |
|---------|------|
| `ghost/core/core/server/services/email-service/email-renderer.js` | 邮件内容渲染器 |
| `ghost/core/core/server/services/media-inliner/external-media-inliner.js` | 外部媒体内联服务 |

## 8. 设计亮点

1. **双格式存储**: 同时保留可编辑的 Lexical JSON 和预渲染的 HTML，平衡编辑灵活性和渲染性能

2. **URL 占位符**: 使用 `__GHOST_URL__` 实现站点 URL 变更时的无缝迁移

3. **目标感知渲染**: 同一内容可根据 `target: 'web' | 'email'` 选项生成不同的 HTML，适配不同场景

4. **渐进式图片**: Web 端使用 `<picture>` + `srcset` 提供现代格式和响应式尺寸

5. **邮件兼容性**: 针对邮件客户端（特别是 Outlook）做了专门的兼容性处理

6. **插件化渲染器**: 自定义节点渲染器采用注册机制，易于扩展新的卡片类型
