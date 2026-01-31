# Ghost 架构深度解析

本文档详细解释 Ghost CMS 的架构设计，重点说明请求如何流经各个层次，以及各层的职责和交互方式。

---

## 📊 架构层次总览

Ghost 采用经典的分层架构设计，从上到下分为六个主要层次：

![Ghost 架构层次图](../diagrams/ghost-architecture-layers.svg)

### 架构层次说明

#### 1. 客户端层 (Client Layer)

**职责**: 提供用户界面和用户交互入口

**主要组件**:
- **Admin UI** - Ember.js 管理后台（正在迁移到 React）
- **React Apps** - 现代化独立功能模块
  - `admin-x-settings` - 设置界面
  - `posts` - 文章分析
  - `stats` - 站点统计
- **Public Apps** - 面向访客的公共应用
  - `portal` - 会员门户
  - `comments-ui` - 评论系统
  - `signup-form` - 注册表单
  - `sodo-search` - 站点搜索
- **Theme Engine** - 主题引擎，处理前端渲染

**技术特点**:
- 微前端架构，各应用独立部署
- React 应用构建后复制到 Core
- Public Apps 使用 UMD 格式
- Theme Engine 使用 Handlebars 模板

---

#### 2. Web 层 (Express Routing)

**职责**: HTTP 请求路由、中间件链处理

**主要组件**:
- **Admin Web** - 处理 `/ghost/` 路径
  - Admin UI 静态资源服务
  - Admin API 路由分发
  - Session 管理
- **API Web** - 处理 API 路径
  - Admin API: `/ghost/api/admin/`
  - Content API: `/ghost/api/content/`
- **Members Web** - 处理会员相关路由
  - 路径: `/members/`
  - Magic Link 登录
  - Stripe Webhook
- **Frontend Web** - 处理前端路由
  - 路径: `/` (根路径)
  - 主题渲染
  - RSS/Sitemap 生成

**中间件链**:
```
HTTP Request
    ↓
Body Parser (JSON 解析，50MB 限制)
    ↓
CORS (跨域处理)
    ↓
JWT/Session Auth (认证)
    ↓
Permission Check (权限验证)
    ↓
Version Match (API 版本控制)
    ↓
Route Handler
```

---

#### 3. API 层 (RESTful Endpoints)

**职责**: 定义 API 端点、参数验证、响应格式化

**主要端点**:
- **Posts API** - 文章管理
  - `GET /posts/` - 浏览列表
  - `GET /posts/:id` - 读取单个
  - `POST /posts/` - 创建文章
  - `PUT /posts/:id` - 更新文章
  - `DELETE /posts/:id` - 删除文章

- **Users API** - 用户管理
  - `GET /users/` - 用户列表
  - `PUT /users/:id` - 更新用户
  - `PUT /users/password` - 修改密码

- **Members API** - 会员管理
  - `GET /members/` - 会员列表
  - `POST /members/` - 创建会员
  - `POST /members/:id/subscriptions/` - 创建订阅

- **Settings API** - 系统设置
  - `GET /settings/` - 获取设置
  - `PUT /settings/` - 更新设置

**端点配置**:
```javascript
{
  docName: 'posts',              // 资源名称
  statusCode: 201,                // HTTP 状态码
  options: ['include', 'formats'], // 允许的查询参数
  validation: {                    // 参数验证
    options: {
      include: {values: ['tags', 'authors']},
      source: {values: ['html']}
    }
  },
  permissions: {                   // 权限配置
    unsafeAttrs: ['status', 'authors']
  },
  async query(frame) {             // 处理函数
    const model = await models.Post.add(
      frame.data.posts[0],
      frame.options
    );
    return model;
  }
}
```

---

#### 4. 服务层 (Business Logic)

**职责**: 实现复杂业务逻辑、协调多个模型、处理跨领域关注点

**核心服务**:
- **Posts Service** - 文章业务逻辑
  - 邮件发送协调
  - 发布工作流管理
  - 定时发布调度

- **Email Service** - 邮件发送管理
  - Newsletter 批量发送
  - 事务性邮件（欢迎邮件、密码重置）
  - 邮件追踪（打开率、点击率）

- **Members Service** - 会员系统核心逻辑
  - 会员认证 (Magic Link)
  - 订阅管理
  - Stripe 集成
  - 会员事件追踪

- **Auth Service** - 认证与授权
  - JWT Token 生成/验证
  - Session 管理
  - 权限检查 (RBAC)

**注意**: `add` 操作直接调用模型层，跳过服务层以提高效率。`edit`、`publish` 等复杂操作需要经过服务层。

---

#### 5. 模型层 (ORM - Bookshelf/Knex)

**职责**: 数据模型定义、ORM 操作、生命周期钩子、数据验证

**核心模型**:
- **Post Model** - 文章数据模型
  ```javascript
  const Post = ghostBookshelf.Model.extend({
    tableName: 'posts',
    
    // 默认值
    defaults() {
      return {
        uuid: crypto.randomUUID(),
        status: 'draft',
        featured: false,
        type: 'post',
        visibility: 'public'
      };
    },
    
    // 关系定义
    author() {
      return this.belongsTo('User', 'author_id');
    },
    tags() {
      return this.belongsToMany('Tag', 'posts_tags');
    },
    
    // 生命周期钩子
    onCreating(model, options) {
      // 生成 UUID, slug
    },
    onSaving(model, options) {
      // 验证内容，生成 HTML
    },
    onCreated(model, options) {
      // 触发事件
    },
    onSaved(model, options) {
      // 处理关联
    }
  });
  ```

- **User Model** - 用户数据模型
  - 密码哈希处理
  - 角色关系
  - 状态管理

- **Member Model** - 会员数据模型
  - 订阅关系
  - Stripe 关联
  - 事件追踪

**生命周期钩子执行顺序**:
```
onCreating
    ↓
onSaving (验证 → 生成 HTML → URL 转换)
    ↓
[数据库插入]
    ↓
onCreated
    ↓
onSaved (处理关联)
    ↓
触发领域事件
```

---

#### 6. 数据层 (Database & Storage)

**职责**: 数据持久化、缓存、文件存储

**主要组件**:
- **MySQL Database** - 主数据库
  - 核心表: posts, users, members, tags
  - 关联表: posts_tags, posts_authors, posts_products
  - 迁移管理: Knex Migrator

- **Redis Cache** - 缓存系统
  - Session 存储 (Admin + Members)
  - 设置缓存 (settings-cache)
  - 速率限制计数器

- **File Storage** - 文件存储
  - 本地存储: `content/images/`, `content/media/`
  - S3 存储: AWS S3 兼容存储
  - URL 转换: `__GHOST_URL__` ↔ 绝对 URL

---

## 🔄 请求完整流程追踪

### 文章创建请求详细流程

![文章创建请求完整流程](../diagrams/post-creation-flow.svg)

### 步骤详解

#### 步骤 1: HTTP Request

**客户端发起请求**:
```http
POST /ghost/api/admin/posts/ HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "posts": [{
    "title": "My New Post",
    "status": "draft",
    "mobiledoc": "{\"version\":\"0.3.1\",...}",
    "tags": [{"name": "News"}]
  }]
}
```

**关键信息**:
- HTTP 方法: `POST`
- 路径: `/ghost/api/admin/posts/`
- 认证: JWT Bearer Token
- Content-Type: `application/json`
- 请求体: 符合 Ghost API 格式的 JSON

---

#### 步骤 2: Middleware Chain

**2.1 Body Parser**
```javascript
apiApp.use(bodyParser.json({limit: '50mb'}));
```
- 解析 JSON 请求体
- 限制大小: 50 MB
- 输出: `req.body = {posts: [...]}`

**2.2 CORS**
```javascript
cors({
  origin: true,
  credentials: true,
  maxAge: 86400
});
```
- 处理跨域请求
- 允许任何来源
- 支持凭证传输
- 预检缓存 24 小时

**2.3 JWT 认证**
```javascript
async function authAdminApi(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const decoded = jwt.verify(token, secret);
  const user = await models.User.findOne({id: decoded.sub});
  
  if (!user || user.get('status') !== 'active') {
    throw new UnauthorizedError();
  }
  
  req.user = user;
  next();
}
```
- 提取 Bearer Token
- 验证 JWT 签名和过期时间
- 加载用户信息
- 检查用户状态
- 附加到 `req.user`

**2.4 权限检查**
```javascript
const hasPermission = await permissions.canThis({
  user: req.user.id,
  context: req.context
}).add.post();

if (!hasPermission) {
  throw new NoPermissionError();
}
```
- 检查用户是否有 `post:add` 权限
- 基于角色的访问控制 (RBAC)
- 权限不足返回 403

**2.5 版本匹配**
```javascript
versionMatch(req, res, next);
```
- 验证 API 版本
- 支持 v3, v4, v5
- 默认使用最新版本

---

#### 步骤 3: API Endpoint

**端点处理器**:
```javascript
add: {
  statusCode: 201,
  options: ['include', 'formats', 'source'],
  validation: {
    options: {
      include: {values: allowedIncludes},
      source: {values: ['html']}
    }
  },
  permissions: {
    unsafeAttrs: ['status', 'authors', 'visibility']
  },
  async query(frame) {
    // 核心处理逻辑
    const model = await models.Post.add(
      frame.data.posts[0],
      frame.options
    );
    
    if (model.get('status') === 'published') {
      frame.setHeader('X-Cache-Invalidate', '/*');
    }
    
    return model;
  }
}
```

**处理流程**:
1. 参数验证 (`validation`)
2. 权限验证 (`permissions`)
3. 调用模型层 (`models.Post.add()`)
4. 缓存控制 (`setHeader`)

---

#### 步骤 4: Model Layer

**Post.add() 方法**:
```javascript
add: function add(data, unfilteredOptions) {
  let options = this.filterOptions(unfilteredOptions, 'add', {
    extraAllowedProperties: ['id']
  });
  
  const addPost = (() => {
    return limitService.errorIfWouldGoOverLimit('posts')
      .then(() => {
        return ghostBookshelf.Model.add.call(this, data, options);
      })
      .then((post) => {
        return this.findOne({
          status: 'all',
          id: post.id
        }, _.merge({transacting: options.transacting}, unfilteredOptions));
      });
  });
  
  if (!options.transacting) {
    return ghostBookshelf.transaction((transacting) => {
      options.transacting = transacting;
      return addPost();
    });
  }
  
  return addPost();
}
```

**执行流程**:
1. 过滤选项
2. 检查资源限制
3. 调用基类 `add()` 方法
4. 重新获取完整数据（包含关联）
5. 事务管理

---

#### 步骤 5: Lifecycle Hooks

**onCreating 钩子**:
```javascript
onCreating: function onCreating(model, options) {
  if (!model.get('created_by')) {
    model.set('created_by', options.context.user);
  }
  
  if (!model.get('slug')) {
    return this.generateSlug(Post, model.get('title'), options)
      .then((slug) => {
        model.set('slug', slug);
      });
  }
}
```
- 设置创建者
- 生成 slug
- 生成 UUID

**onSaving 钩子**:
```javascript
onSaving: async function onSaving(model, options) {
  // 验证 mobiledoc 结构
  if (model.hasChanged('mobiledoc')) {
    this.validateMobiledoc(model.get('mobiledoc'));
  }
  
  // 生成 HTML
  if (model.hasChanged('mobiledoc') || model.hasChanged('lexical')) {
    const html = this.generateHTML(model);
    model.set('html', html);
    const plaintext = htmlToPlaintext(html);
    model.set('plaintext', plaintext);
  }
  
  // URL 转换
  model.set('mobiledoc', urlUtils.mobiledocToTransformReady(model.get('mobiledoc')));
  
  // 验证发布时间
  if (model.get('status') === 'scheduled') {
    this.validatePublishedAt(model);
  }
}
```
- 验证内容格式
- 生成 HTML 和纯文本
- URL 转换（`__GHOST_URL__`）
- 验证发布时间

---

#### 步骤 6: Database

**SQL 查询生成** (由 Bookshelf/Knex 生成):
```sql
-- 1. 插入主记录
INSERT INTO `posts` (
    `id`, `uuid`, `title`, `slug`, 
    `mobiledoc`, `html`, `plaintext`,
    `status`, `visibility`, `type`,
    `created_at`, `created_by`
) VALUES (
    '507f1f77bcf86cd799439011',
    'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    'My New Post', 'my-new-post',
    '{"version":"0.3.1",...}',
    '<p>Content</p>', 'Content',
    'draft', 'public', 'post',
    '2026-01-26 12:00:00', '1'
);

-- 2. 插入标签关联
INSERT INTO `posts_tags` (`post_id`, `tag_id`, `sort_order`)
VALUES ('507f1f77bcf86cd799439011', '1', 0);

-- 3. 插入作者关联
INSERT INTO `posts_authors` (`post_id`, `author_id`, `sort_order`)
VALUES ('507f1f77bcf86cd799439011', '1', 0);

-- 4. 重新查询（带关联）
SELECT
    posts.*,
    GROUP_CONCAT(DISTINCT tags.id) as tag_ids,
    GROUP_CONCAT(DISTINCT authors.id) as author_ids
FROM posts
LEFT JOIN posts_tags ON posts.id = posts_tags.post_id
LEFT JOIN tags ON posts_tags.tag_id = tags.id
LEFT JOIN posts_authors ON posts.id = posts_authors.author_id
LEFT JOIN users as authors ON posts_authors.author_id = authors.id
WHERE posts.id = '507f1f77bcf86cd799439011'
GROUP BY posts.id;
```

**事务保证**:
```sql
BEGIN;
  INSERT INTO posts ...;
  INSERT INTO posts_tags ...;
  INSERT INTO posts_authors ...;
  -- 如果任何步骤失败
  ROLLBACK;
  -- 如果全部成功
  COMMIT;
```

---

#### 步骤 7: Event Trigger

**事件发射**:
```javascript
model.emitChange('added', options);

// 内部实现
DomainEvents.emit('post.added', {
  id: model.id,
  title: model.get('title'),
  status: model.get('status'),
  published_at: model.get('published_at')
});
```

**事件监听器**:
```javascript
// Webhook 处理器
DomainEvents.on('post.added', async (data) => {
  await webhookService.trigger('post.added', data);
});

// 缓存处理器
DomainEvents.on('post.added', async (data) => {
  if (data.status === 'published') {
    await cacheService.invalidate('/*');
  }
});

// 审计日志
DomainEvents.on('post.added', async (data) => {
  await auditService.log('post.added', {
    resource_id: data.id,
    resource_type: 'post',
    user_id: options.context.user
  });
});
```

---

#### 步骤 8: Response

**响应格式化**:
```javascript
const json = model.toJSON(options);
const response = {posts: [json]};
```

**HTTP 响应**:
```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /ghost/api/admin/posts/507f1f77bcf86cd799439011
X-Cache-Invalidate: /* (如果 status=published)

{
  "posts": [{
    "id": "507f1f77bcf86cd799439011",
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "My New Post",
    "slug": "my-new-post",
    "status": "draft",
    "created_at": "2026-01-26T12:00:00.000Z",
    "tags": [{"id": "1", "name": "News"}],
    "authors": [{"id": "1", "name": "Ghost Admin"}]
  }]
}
```

**状态码**: `201 Created`

---

## 📋 完整请求-响应生命周期

![请求-响应完整生命周期](../diagrams/request-response-lifecycle.svg)

### 阶段 1: 请求入口

**1.1 HTTP Request**
- 客户端发送 POST 请求
- 包含请求头和请求体
- 携带 JWT Token

**1.2 Express Server**
- GhostServer 监听端口 2368
- 接收 HTTP 连接
- 解析 HTTP 请求

**1.3 路由匹配**
- Express Router 匹配路由
- `/ghost/api/admin/posts/`
- 找到对应的处理函数

---

### 阶段 2: 中间件链

**2.1 Body Parser**
- 解析 JSON 请求体
- 限制大小: 50 MB
- 输出: `req.body`

**2.2 CORS**
- 处理跨域请求
- 设置 CORS 响应头

**2.3 JWT Auth**
- 验证 Bearer Token
- 加载用户信息
- 附加到 `req.user`

**2.4 权限检查**
- 检查操作权限
- 基于 RBAC
- 权限不足返回 403

**2.5 Version Match**
- 验证 API 版本
- 支持多版本并存

---

### 阶段 3: 业务处理

**3.1 API Endpoint**
- 端点处理器调用
- 参数验证
- 调用模型层

**3.2 Service Layer**
- 可选的业务逻辑层
- 复杂操作经过服务层
- 简单操作直接调用模型层

**3.3 Model Layer**
- 数据模型处理
- 生命周期钩子
- ORM 操作

**3.4 Database**
- SQL 执行
- 事务管理
- 关联加载

**3.5 Event Trigger**
- 触发领域事件
- 异步处理
- Webhook 通知

---

### 阶段 4: 响应返回

**4.1 Response**
- 序列化模型为 JSON
- 包装响应格式

**4.2 HTTP 201**
- 返回 201 状态码
- 设置 Location 头
- 设置缓存控制头

**4.3 Webhook**
- 异步触发 Webhook
- 发送 POST 到目标 URL
- 签名验证

**4.4 Cache**
- 清除相关缓存
- Redis 缓存失效
- CDN 缓存失效

**4.5 Audit**
- 记录审计日志
- 用户ID、操作类型
- 时间戳、IP地址

---

## 🔑 关键设计决策

### 为什么需要事务？

**原因**:
- 主记录 + 关联记录必须原子性操作
- 要么全部成功，要么全部失败
- 保证数据一致性

**示例**:
```sql
BEGIN;
  INSERT INTO posts ...;
  INSERT INTO posts_tags ...;  -- 失败
  INSERT INTO posts_authors ...;
ROLLBACK;  -- 全部回滚
```

---

### 为什么需要生命周期钩子？

**原因**:
- 自动处理默认值
- 数据验证和转换
- 解耦业务逻辑
- 提高代码复用性

**钩子顺序**:
```
onCreating (默认值)
    ↓
onSaving (验证、转换)
    ↓
[数据库操作]
    ↓
onCreated (事件)
    ↓
onSaved (关联)
```

---

### 为什么需要事件系统？

**原因**:
- 解耦业务逻辑
- 支持异步处理
- 便于扩展
- 实现发布-订阅模式

**事件监听**:
- Webhook 通知
- 缓存清除
- 审计日志
- 邮件发送

---

### 为什么需要两次查询？

**第一次**: 插入数据
```javascript
ghostBookshelf.Model.add.call(this, data, options)
```

**第二次**: 获取完整数据
```javascript
this.findOne({id: post.id}, {withRelated: ['tags', 'authors']})
```

**原因**:
- Knex `INSERT` 只返回 ID
- 需要加载所有关联关系
- 需要获取计算字段和默认值
- 确保返回数据完整

---

### 性能优化点

**Eager Loading**:
```javascript
// ✅ 好: 一次查询 + 关联查询
const posts = await Post.findAll({
  withRelated: ['author', 'tags']
});

// ❌ 差: N+1 查询
for (const post of posts) {
  await post.author().fetch();
}
```

**设置缓存**:
```javascript
// 从缓存读取
const defaultVisibility = settingsCache.get('default_content_visibility');
```

**批量操作**:
```javascript
// 批量更新
await Post.bulkEdit(postIds, {featured: true});
```

---

### 安全检查

**JWT 认证**:
- Token 有效期: 12 小时
- 签名验证
- 用户状态检查

**权限验证**:
- 基于 RBAC
- 资源级权限检查
- API Key 权限隔离

**输入验证**:
- JSON Schema 验证
- SQL 注入防护
- XSS 防护

**速率限制**:
- 登录: 5 次/小时
- API: 1000 次/小时
- 邮件发送: 按会员数量

---

## 🎯 架构优势

✅ **分层清晰** - 每层职责明确，易于维护
✅ **事件驱动** - 解耦系统组件，易于扩展
✅ **事务保证** - 数据一致性
✅ **性能优化** - 缓存、批量操作、Eager Loading
✅ **错误处理** - 完善的错误处理和回滚机制
✅ **可扩展性** - 插件化的服务系统
✅ **安全性** - 完善的权限系统和安全措施
✅ **开发体验** - 丰富的测试工具和清晰的代码组织

---

## 📚 技术栈总结

### 后端技术
- **运行时**: Node.js ^22.13.1
- **框架**: Express.js 4.21.2
- **ORM**: Bookshelf.js 1.2.0 + Knex.js 2.4.2
- **数据库**: MySQL 8.0+ / MariaDB 10.3+
- **缓存**: Redis 6.0+
- **模板**: Handlebars 4.7.8
- **认证**: JWT 8.5.1

### 前端技术
- **管理后台**: Ember.js 4.x (legacy) + React 18.x
- **构建工具**: Vite 5.x
- **状态管理**: @tanstack/react-query 5.x
- **设计系统**: shadcn/ui + Radix UI
- **表单验证**: react-hook-form + zod

### 外部服务
- **支付**: Stripe
- **邮件**: Mailgun / SMTP
- **分析**: Tinybird
- **存储**: AWS S3 / S3 兼容
- **错误追踪**: Sentry

---

**文档版本**: 1.0.0
**最后更新**: 2026-01-31
**对应 Ghost 版本**: 6.14.0
