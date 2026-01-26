# 📦 Ghost Core 模块深度分析

## 模块概述

**ghost/core** 是 Ghost CMS 的核心应用模块，负责整个发布平台的后端逻辑、API 服务、数据管理和前端渲染。

- **版本**: 6.14.0
- **运行时**: Node.js ^22.13.1
- **主框架**: Express.js
- **ORM**: Bookshelf.js (基于 Knex)
- **模板引擎**: Handlebars (express-hbs)

---

## 🏗️ 核心架构

### 目录结构
```
ghost/core/
├── index.js                 # 应用入口点
├── ghost.js                 # 启动模式路由器
├── core/
│   ├── boot.js             # 启动序列
│   ├── server/             # 后端核心
│   │   ├── ghost-server.js # HTTP 服务器类
│   │   ├── api/            # RESTful API
│   │   ├── models/         # 数据模型 (Bookshelf)
│   │   ├── services/       # 业务服务层
│   │   ├── data/           # 数据库与迁移
│   │   ├── lib/            # 工具库
│   │   └── web/            # Web 层 (路由/中间件)
│   ├── frontend/           # 前端渲染
│   │   ├── services/       # 前端服务
│   │   ├── helpers/        # Handlebars 辅助函数
│   │   └── views/          # 模板视图
│   └── shared/             # 共享模块
│       ├── config/         # 配置管理
│       ├── settings-cache/ # 设置缓存
│       └── url-utils/      # URL 工具
├── content/                # 用户内容目录
│   ├── themes/             # 主题
│   ├── images/             # 图片
│   └── data/               # 数据库文件
└── test/                   # 测试套件
    ├── unit/               # 单元测试
    ├── integration/        # 集成测试
    └── e2e/                # E2E 测试
```

---

## 🎯 核心职责

### 1. **应用启动与生命周期管理**
- 初始化应用环境
- 配置加载与验证
- 数据库连接与迁移
- 服务启动编排
- 优雅关闭处理

### 2. **RESTful API 服务**
- Content API (公开内容访问)
- Admin API (管理后台接口)
- Members API (会员系统)
- Webhooks (事件通知)

### 3. **数据持久化**
- 数据模型定义 (Bookshelf ORM)
- 数据库迁移管理 (Knex Migrator)
- 关系映射与查询优化
- 事务处理

### 4. **内容管理**
- 文章发布工作流 (草稿 → 发布 → 定时发布)
- 媒体文件管理
- 标签与分类
- 主题系统

### 5. **会员与订阅**
- 会员注册与认证
- Stripe 支付集成
- 邮件营销 (Newsletter)
- 订阅管理

### 6. **前端渲染**
- 主题引擎 (Handlebars)
- 动态路由系统
- RSS/Sitemap 生成
- SEO 元数据

### 7. **安全与权限**
- 基于角色的访问控制 (RBAC)
- JWT 认证
- API 密钥管理
- 速率限制

---

## 🔑 关键类详解

### 1. **GhostServer** (`core/server/ghost-server.js`)

**职责**: HTTP 服务器生命周期管理

```javascript
class GhostServer {
    constructor({url, env, serverConfig}) {
        this.url = url;              // 站点 URL
        this.env = env;              // 环境 (dev/prod/test)
        this.serverConfig = serverConfig;
        this.rootApp = null;         // Express 应用实例
        this.httpServer = null;      // HTTP 服务器
        this.cleanupTasks = [];      // 清理任务队列
    }

    // 启动服务器
    async start(rootApp) {
        this.rootApp = rootApp;
        return new Promise((resolve, reject) => {
            this.httpServer = rootApp.listen(port, host);

            this.httpServer.on('error', (error) => {
                // 错误处理 (EADDRINUSE 等)
            });

            this.httpServer.on('listening', () => {
                this._logStartMessages();
                notify.notifyServerStarted();
                resolve(this);
            });

            // 使服务器可优雅关闭
            stoppable(this.httpServer, shutdownTimeout);

            // 绑定信号处理
            process.on('SIGINT', () => this.shutdown());
            process.on('SIGTERM', () => this.shutdown());
        });
    }

    // 优雅关闭
    async shutdown(code = 0) {
        if (this.isShuttingDown) return;

        this.isShuttingDown = true;
        await this.stop();
        setTimeout(() => process.exit(code), 100);
    }

    // 停止服务器
    async stop() {
        if (this.httpServer?.listening) {
            await this._stopServer();  // 关闭连接
        }
        await this._cleanup();          // 执行清理任务
        this.httpServer = null;
    }

    // 注册清理任务
    registerCleanupTask(task) {
        this.cleanupTasks.push(task);
    }
}
```

**关键功能**:
- ✅ 优雅启动/关闭
- ✅ 信号处理 (SIGINT/SIGTERM)
- ✅ 可配置超时
- ✅ 清理任务编排
- ✅ 测试模式支持

---

### 2. **Post Model** (`core/server/models/post.js`)

**职责**: 文章数据模型与业务逻辑

```javascript
const Post = ghostBookshelf.Model.extend({
    tableName: 'posts',

    // 启用 CRUD 操作审计
    actionsCollectCRUD: true,
    actionsResourceType: 'post',

    // 默认值
    defaults() {
        return {
            uuid: crypto.randomUUID(),
            status: 'draft',              // 状态: draft/published/scheduled/sent
            featured: false,
            type: 'post',                 // 类型: post/page
            visibility: 'public',         // 可见性: public/members/paid/tiers
            email_recipient_filter: 'all',
            show_title_and_feature_image: true
        };
    },

    // 关系定义
    author() {
        return this.belongsTo('User', 'author_id');
    },

    tags() {
        return this.belongsToMany('Tag', 'posts_tags', 'post_id', 'tag_id');
    },

    newsletter() {
        return this.belongsTo('Newsletter', 'newsletter_id');
    },

    tiers() {
        return this.belongsToMany('Product', 'posts_products', 'post_id', 'product_id');
    },

    // 发布工作流
    async changeStatus(newStatus, options) {
        const currentStatus = this.get('status');

        // 状态转换验证
        if (currentStatus === 'published' && newStatus !== 'published') {
            throw new errors.ValidationError({
                message: messages.isAlreadyPublished
            });
        }

        // 定时发布验证
        if (newStatus === 'scheduled') {
            const publishedAt = this.get('published_at');
            const minScheduleTime = moment().add(
                config.get('times:cannotScheduleAPostBeforeInMinutes'),
                'minutes'
            );

            if (moment(publishedAt).isBefore(minScheduleTime)) {
                throw new errors.ValidationError({
                    message: messages.expectedPublishedAtInFuture
                });
            }
        }

        this.set('status', newStatus);
    }
});
```

**关键功能**:
- ✅ 状态机管理 (草稿/发布/定时/已发送)
- ✅ 关系映射 (作者/标签/会员层级)
- ✅ 内容格式支持 (Mobiledoc/Lexical)
- ✅ 可见性控制
- ✅ 发布时间验证

---

### 3. **User Model** (`core/server/models/user.js`)

**职责**: 用户账户管理

```javascript
const User = ghostBookshelf.Model.extend({
    tableName: 'users',

    actionsCollectCRUD: true,
    actionsResourceType: 'user',

    defaults() {
        return {
            password: security.identifier.uid(50),
            visibility: 'public',
            status: 'active',           // active/inactive/locked
            comment_notifications: true,
            free_member_signup_notification: true,
            paid_subscription_started_notification: true,
            mention_notifications: true
        };
    },

    // 角色关系
    roles() {
        return this.belongsToMany('Role', 'roles_users', 'user_id', 'role_id');
    },

    // 文章关系
    posts() {
        return this.hasMany('Post', 'author_id');
    },

    // 密码哈希处理
    async hashPassword(password) {
        const hashedPassword = await security.password.hash(password);
        this.set('password', hashedPassword);
    },

    // 密码验证
    async validatePassword(password) {
        return security.password.compare(password, this.get('password'));
    }
});

// 静态方法
Users = ghostBookshelf.Collection.extend({
    model: User,

    // 按角色查找用户
    async findByRole(roleName, options) {
        return this.query((qb) => {
            qb.innerJoin('roles_users', 'users.id', 'roles_users.user_id');
            qb.innerJoin('roles', 'roles_users.role_id', 'roles.id');
            qb.where('roles.name', roleName);
        }).fetch(options);
    }
});
```

**关键功能**:
- ✅ 密码安全处理
- ✅ 角色权限绑定
- ✅ 用户状态管理
- ✅ 通知偏好设置
- ✅ 个人资料管理

---

## 🛠️ 主要功能模块

### API 层 (`core/server/api/`)

**结构**:
```
api/
├── endpoints/              # API 端点定义
│   ├── posts.js           # 文章 API
│   ├── users.js           # 用户 API
│   ├── tags.js            # 标签 API
│   ├── members.js         # 会员 API
│   ├── newsletters.js     # 邮件通讯 API
│   ├── emails.js          # 邮件 API
│   ├── authentication.js  # 认证 API
│   └── ...                # 60+ 端点
└── index.js               # API 路由汇总
```

**示例: Posts API 端点**
```javascript
// core/server/api/endpoints/posts.js
module.exports = {
    docName: 'posts',

    browse: {
        options: [
            'include', 'filter', 'fields', 'formats',
            'limit', 'order', 'page', 'debug'
        ],
        validation: {
            options: {
                include: ['authors', 'tags', 'tiers']
            }
        },
        permissions: true,
        query(frame) {
            return models.Post.findPage(frame.options);
        }
    },

    read: {
        options: ['include', 'fields', 'formats', 'debug'],
        data: ['id', 'slug', 'uuid'],
        validation: {
            options: {
                include: ['authors', 'tags', 'tiers']
            }
        },
        permissions: true,
        query(frame) {
            return models.Post.findOne(frame.data, frame.options);
        }
    },

    add: {
        statusCode: 201,
        headers: {},
        options: ['source', 'formats'],
        validation: {
            options: {
                source: ['html']
            }
        },
        permissions: {
            method: 'add'
        },
        async query(frame) {
            return models.Post.add(frame.data.posts[0], frame.options);
        }
    },

    edit: {
        headers: {},
        options: ['id', 'formats', 'source', 'force_rerender', 'save_revision'],
        validation: {
            options: {
                id: {required: true},
                source: ['html']
            }
        },
        permissions: {
            method: 'edit'
        },
        async query(frame) {
            return models.Post.edit(frame.data.posts[0], frame.options);
        }
    },

    destroy: {
        statusCode: 204,
        headers: {},
        options: ['id'],
        validation: {
            options: {
                id: {required: true}
            }
        },
        permissions: true,
        async query(frame) {
            return models.Post.destroy({...frame.options, require: true});
        }
    }
};
```

**API 特性**:
- ✅ 标准化 CRUD 操作
- ✅ 权限验证
- ✅ 参数验证
- ✅ 关系加载 (`include`)
- ✅ 过滤与排序 (NQL 查询语言)
- ✅ 分页支持
- ✅ 版本控制

---

### 服务层 (`core/server/services/`)

**核心服务**:
```
services/
├── auth/                   # 认证服务
│   ├── authorize.js       # 授权逻辑
│   ├── session.js         # 会话管理
│   └── api-key.js         # API 密钥
├── members/                # 会员系统
│   ├── MembersAPI.js      # 会员 API
│   ├── stripe.js          # Stripe 集成
│   └── email.js           # 会员邮件
├── email-service/          # 邮件服务
│   ├── EmailService.js    # 邮件发送
│   ├── batch-sending/     # 批量发送
│   └── analytics/         # 邮件分析
├── comments/               # 评论系统
├── link-tracking/          # 链接追踪
├── activitypub/            # ActivityPub 联邦
├── themes/                 # 主题管理
└── ...                     # 60+ 服务
```

**示例: 认证服务**
```javascript
// core/server/services/auth/session.js
class SessionService {
    constructor({sessionStore, sessionSecret}) {
        this.sessionStore = sessionStore;
        this.sessionSecret = sessionSecret;
    }

    // 创建会话中间件
    createSessionMiddleware() {
        return expressSession({
            store: this.sessionStore,
            secret: this.sessionSecret,
            resave: false,
            saveUninitialized: false,
            name: 'ghost-admin-api-session',
            cookie: {
                maxAge: 12 * 60 * 60 * 1000, // 12 hours
                httpOnly: true,
                sameSite: 'lax',
                secure: config.get('url').startsWith('https')
            }
        });
    }

    // 销毁会话
    async destroySession(sessionID) {
        return new Promise((resolve, reject) => {
            this.sessionStore.destroy(sessionID, (err) => {
                if (err) return reject(err);
                resolve();
            });
        });
    }
}
```

**示例: 邮件服务**
```javascript
// core/server/services/email-service/EmailService.js
class EmailService {
    constructor({config, memberRepository, emailRenderer}) {
        this.config = config;
        this.memberRepository = memberRepository;
        this.emailRenderer = emailRenderer;
        this.mailer = null;
    }

    async init() {
        // 初始化邮件传输 (Mailgun/SMTP)
        this.mailer = await this._createMailer();
    }

    // 发送邮件通讯
    async sendNewsletter({newsletter, post, members}) {
        // 1. 渲染邮件内容
        const emailContent = await this.emailRenderer.render(post, {
            newsletter: newsletter
        });

        // 2. 批量发送
        const batches = this._createBatches(members, 1000);

        for (const batch of batches) {
            await this._sendBatch({
                newsletter,
                post,
                members: batch,
                content: emailContent
            });
        }
    }

    // 追踪邮件事件
    async trackEmailEvent({emailId, memberId, event}) {
        return models.EmailRecipient.edit({
            [`${event}_at`]: new Date()
        }, {
            id: emailId,
            member_id: memberId
        });
    }
}
```

---

### 数据层 (`core/server/data/`)

**结构**:
```
data/
├── schema/                 # 数据库架构定义
│   ├── schema.js          # 表结构
│   └── commands.js        # DDL 命令
├── migrations/             # 数据库迁移
│   ├── init/              # 初始迁移
│   └── versions/          # 版本迁移
├── db/                     # 数据库连接
│   ├── connection.js      # Knex 连接
│   └── backup.js          # 备份工具
└── importer/               # 数据导入
```

**示例: Schema 定义**
```javascript
// core/server/data/schema/schema.js
module.exports = {
    posts: {
        id: {type: 'string', maxlength: 24, nullable: false, primary: true},
        uuid: {type: 'string', maxlength: 36, nullable: false, unique: true},
        title: {type: 'string', maxlength: 2000, nullable: false},
        slug: {type: 'string', maxlength: 191, nullable: false, unique: true},
        mobiledoc: {type: 'text', maxlength: 1000000000, nullable: true},
        lexical: {type: 'text', maxlength: 1000000000, nullable: true},
        html: {type: 'text', maxlength: 1000000000, nullable: true},
        plaintext: {type: 'text', maxlength: 1000000000, nullable: true},
        feature_image: {type: 'string', maxlength: 2000, nullable: true},
        featured: {type: 'bool', nullable: false, defaultTo: false},
        type: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'post'},
        status: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'draft'},
        visibility: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'public'},
        author_id: {type: 'string', maxlength: 24, nullable: false},
        newsletter_id: {type: 'string', maxlength: 24, nullable: true},
        created_at: {type: 'dateTime', nullable: false},
        created_by: {type: 'string', maxlength: 24, nullable: false},
        updated_at: {type: 'dateTime', nullable: true},
        updated_by: {type: 'string', maxlength: 24, nullable: true},
        published_at: {type: 'dateTime', nullable: true},
        published_by: {type: 'string', maxlength: 24, nullable: true}
    },

    users: {
        id: {type: 'string', maxlength: 24, nullable: false, primary: true},
        name: {type: 'string', maxlength: 191, nullable: false},
        slug: {type: 'string', maxlength: 191, nullable: false, unique: true},
        email: {type: 'string', maxlength: 191, nullable: false, unique: true},
        password: {type: 'string', maxlength: 60, nullable: false},
        profile_image: {type: 'string', maxlength: 2000, nullable: true},
        cover_image: {type: 'string', maxlength: 2000, nullable: true},
        bio: {type: 'text', maxlength: 65535, nullable: true},
        website: {type: 'string', maxlength: 2000, nullable: true},
        location: {type: 'string', maxlength: 65535, nullable: true},
        status: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'active'},
        visibility: {type: 'string', maxlength: 50, nullable: false, defaultTo: 'public'},
        created_at: {type: 'dateTime', nullable: false},
        updated_at: {type: 'dateTime', nullable: true}
    }
};
```

**示例: 迁移文件**
```javascript
// core/server/data/migrations/versions/5.97/2024-10-08-add-body-font-settings.js
const {addSetting} = require('../../utils');

module.exports = addSetting({
    key: 'body_font',
    value: 'system-ui',
    type: 'string',
    group: 'site'
});
```

---

### 前端层 (`core/frontend/`)

**结构**:
```
frontend/
├── services/
│   ├── routing/           # 动态路由系统
│   ├── theme-engine/      # 主题引擎
│   ├── rendering/         # 渲染服务
│   ├── rss/               # RSS 生成
│   └── sitemap/           # Sitemap 生成
├── helpers/               # Handlebars 辅助函数
│   ├── get.js            # 数据获取
│   ├── foreach.js        # 循环
│   ├── img_url.js        # 图片 URL
│   └── ...               # 100+ 辅助函数
└── web/                   # Web 层
    ├── middleware/        # 中间件
    └── site/              # 站点路由
```

**示例: Routing Service**
```javascript
// core/frontend/services/routing/CollectionRouter.js
class CollectionRouter {
    constructor(name, route, config) {
        this.name = name;           // 例如: 'index', 'tag', 'author'
        this.route = route;         // 例如: '/', '/tag/:slug/', '/author/:slug/'
        this.config = config;
        this.router = express.Router();

        this._registerRoutes();
    }

    _registerRoutes() {
        // 集合页路由
        this.router.get(
            this.route,
            this._createMiddleware()
        );

        // 分页路由
        this.router.get(
            `${this.route}page/:page/`,
            this._createMiddleware()
        );
    }

    _createMiddleware() {
        return [
            // 1. 解析查询参数
            queryMiddleware,

            // 2. 获取数据
            async (req, res, next) => {
                const data = await this._fetchData(req);
                res.locals.data = data;
                next();
            },

            // 3. 渲染模板
            renderMiddleware
        ];
    }

    async _fetchData(req) {
        return models.Post.findPage({
            filter: this.config.filter,
            limit: this.config.limit || 15,
            page: req.params.page || 1,
            order: this.config.order || 'published_at DESC'
        });
    }
}
```

**示例: Handlebars Helper**
```javascript
// core/frontend/helpers/get.js
module.exports = async function get(resource, options) {
    // 使用示例: {{#get "posts" filter="featured:true" limit="3"}}
    const data = await api[resource].browse(options.hash);

    if (data) {
        return options.fn(data);
    }
    return options.inverse(this);
};
```

---

## 🔌 集成点

### 1. **与 Ghost Admin 集成**

**方式**: Admin 作为客户端通过 Admin API 与 Core 通信

```javascript
// Core 提供 Admin API
// core/server/web/admin/app.js
module.exports = function setupAdminApp() {
    const adminApp = express();

    // 静态资源服务 (Admin Ember App)
    adminApp.use('/ghost/assets', express.static(
        path.join(__dirname, '../../built/admin')
    ));

    // API 路由
    adminApp.use('/ghost/api/admin', require('../api/admin'));

    return adminApp;
};
```

**数据流**:
```
Admin Ember App (ghost/admin)
    ↓ HTTP Request (Admin API)
Core API Layer (ghost/core/api)
    ↓ 权限验证
Service Layer (ghost/core/services)
    ↓ 业务逻辑
Model Layer (ghost/core/models)
    ↓ ORM 查询
Database (MySQL/SQLite)
```

---

### 2. **与 React Apps 集成**

**方式**: React apps 构建后复制到 Core，由 Core 提供服务

```javascript
// Admin 构建流程
// ghost/admin/lib/asset-delivery/index.js
async function copyReactApps() {
    const apps = [
        'admin-x-settings',
        'posts',
        'stats',
        'activitypub'
    ];

    for (const app of apps) {
        await fs.copy(
            `apps/${app}/dist`,
            `ghost/core/core/built/admin/assets/${app}`
        );
    }
}
```

**运行时加载**:
```javascript
// Ember Admin 动态导入 React
// ghost/admin/app/components/admin-x.js
export default Component.extend({
    async loadReactApp(appName) {
        const script = document.createElement('script');
        script.src = `/ghost/assets/${appName}/${appName}.js`;
        document.body.appendChild(script);
    }
});
```

---

### 3. **与主题集成**

**方式**: 主题通过 Handlebars 模板和 API 集成

```handlebars
<!-- 主题模板: index.hbs -->
{{!-- 使用 get helper 获取数据 --}}
{{#get "posts" limit="5" filter="featured:true"}}
    {{#foreach posts}}
        <article>
            <h2>{{title}}</h2>
            <p>{{excerpt}}</p>
            {{#if feature_image}}
                <img src="{{img_url feature_image size="m"}}" alt="{{title}}">
            {{/if}}
        </article>
    {{/foreach}}
{{/get}}

{{!-- 内置辅助函数 --}}
<meta name="description" content="{{meta_description}}">
{{{ghost_head}}}  {{!-- 注入脚本/样式 --}}
```

**主题 API**:
```javascript
// 主题可以通过 Content API 获取数据
fetch('/ghost/api/content/posts/?key=xxx&filter=tag:news')
    .then(res => res.json())
    .then(data => console.log(data.posts));
```

---

### 4. **与外部服务集成**

#### **Stripe 集成** (支付)
```javascript
// core/server/services/members/stripe.js
class StripeService {
    constructor({config}) {
        this.stripe = require('stripe')(config.secretKey);
    }

    async createCheckoutSession({priceId, customerId, metadata}) {
        return this.stripe.checkout.sessions.create({
            mode: 'subscription',
            customer: customerId,
            line_items: [{price: priceId, quantity: 1}],
            success_url: `${config.url}/success`,
            cancel_url: `${config.url}/cancel`,
            metadata
        });
    }

    async handleWebhook(event) {
        switch (event.type) {
            case 'checkout.session.completed':
                await this._handleCheckoutCompleted(event.data.object);
                break;
            case 'customer.subscription.updated':
                await this._handleSubscriptionUpdated(event.data.object);
                break;
        }
    }
}
```

#### **Mailgun 集成** (邮件)
```javascript
// core/server/services/email-service/mailgun.js
class MailgunService {
    async send({to, subject, html, from}) {
        const mailgun = require('mailgun.js');
        const client = mailgun.client({
            username: 'api',
            key: this.config.apiKey
        });

        return client.messages.create(this.config.domain, {
            from,
            to,
            subject,
            html,
            'o:tracking': true,
            'o:tracking-clicks': true,
            'o:tracking-opens': true
        });
    }
}
```

#### **Tinybird 集成** (分析)
```javascript
// core/server/services/analytics/TinybirdService.js
class TinybirdService {
    async trackPageView({url, referrer, memberId}) {
        return fetch(`${this.endpoint}/v0/events`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${this.token}`
            },
            body: JSON.stringify({
                timestamp: Date.now(),
                url,
                referrer,
                member_id: memberId
            })
        });
    }
}
```

---

### 5. **事件系统集成**

**内部事件总线**:
```javascript
// core/shared/events/index.js
const EventEmitter = require('events');
const DomainEvents = new EventEmitter();

// 发布事件
DomainEvents.emit('post.published', {
    id: post.id,
    title: post.get('title'),
    published_at: post.get('published_at')
});

// 订阅事件
DomainEvents.on('post.published', async (data) => {
    // 发送通知邮件
    await emailService.sendPublicationNotification(data);

    // 触发 Webhooks
    await webhookService.trigger('post.published', data);

    // 清除缓存
    await cacheService.invalidate(`post:${data.id}`);
});
```

**Webhook 集成**:
```javascript
// core/server/services/webhooks/WebhookService.js
class WebhookService {
    async trigger(event, data) {
        const webhooks = await models.Webhook.findAll({
            filter: `event:${event}`
        });

        for (const webhook of webhooks) {
            await this._send(webhook, data);
        }
    }

    async _send(webhook, data) {
        return fetch(webhook.get('target_url'), {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-Ghost-Signature': this._generateSignature(data, webhook.get('secret'))
            },
            body: JSON.stringify(data)
        });
    }
}
```

---

## 💡 实战示例

### 示例 1: 创建自定义 API 端点

**场景**: 添加文章阅读统计 API

```javascript
// 1. 创建端点文件: core/server/api/endpoints/post-analytics.js
module.exports = {
    docName: 'post_analytics',

    // GET /ghost/api/admin/post-analytics/:id/
    read: {
        options: ['id'],
        permissions: true,
        async query(frame) {
            const postId = frame.options.id;

            // 从 Tinybird 获取统计数据
            const analytics = await services.analytics.getPostStats(postId);

            return {
                post_id: postId,
                views: analytics.views,
                unique_visitors: analytics.unique_visitors,
                avg_reading_time: analytics.avg_reading_time,
                clicks: analytics.clicks
            };
        }
    }
};

// 2. 注册路由: core/server/api/index.js
module.exports = {
    get http() {
        return [
            // ... 其他路由
            require('./endpoints/post-analytics')
        ];
    }
};
```

---

### 示例 2: 创建自定义服务

**场景**: 添加 Slack 通知服务

```javascript
// core/server/services/slack-notifications/SlackService.js
const {IncomingWebhook} = require('@slack/webhook');

class SlackNotificationService {
    constructor({webhookUrl}) {
        this.webhook = new IncomingWebhook(webhookUrl);
        this._init();
    }

    _init() {
        // 监听文章发布事件
        events.on('post.published', async (post) => {
            await this.notifyPostPublished(post);
        });

        // 监听新会员事件
        events.on('member.added', async (member) => {
            await this.notifyNewMember(member);
        });
    }

    async notifyPostPublished(post) {
        return this.webhook.send({
            text: `📝 New post published: ${post.title}`,
            blocks: [
                {
                    type: 'section',
                    text: {
                        type: 'mrkdwn',
                        text: `*${post.title}*\n${post.excerpt}`
                    }
                },
                {
                    type: 'actions',
                    elements: [
                        {
                            type: 'button',
                            text: {type: 'plain_text', text: 'View Post'},
                            url: post.url
                        }
                    ]
                }
            ]
        });
    }

    async notifyNewMember(member) {
        return this.webhook.send({
            text: `🎉 New member: ${member.email}`
        });
    }
}

module.exports = SlackNotificationService;
```

---

### 示例 3: 添加数据库迁移

**场景**: 为文章添加阅读时间字段

```javascript
// core/server/data/migrations/versions/6.0/2025-01-26-add-reading-time.js
const {addColumn} = require('../../utils');

module.exports = addColumn('posts', 'reading_time', {
    type: 'integer',
    nullable: true,
    unsigned: true,
    defaultTo: null
});

// 使用迁移
// yarn knex-migrator migrate
```

**在 Model 中使用**:
```javascript
// core/server/models/post.js
Post = ghostBookshelf.Model.extend({
    // ... 现有代码

    // 自动计算阅读时间
    async calculateReadingTime() {
        const plaintext = this.get('plaintext') || '';
        const wordsPerMinute = 200;
        const wordCount = plaintext.split(/\s+/).length;
        const readingTime = Math.ceil(wordCount / wordsPerMinute);

        this.set('reading_time', readingTime);
    },

    // 保存前钩子
    async saving(model, attrs, options) {
        ghostBookshelf.Model.prototype.saving.apply(this, arguments);

        // 当内容改变时重新计算
        if (this.hasChanged('plaintext')) {
            await this.calculateReadingTime();
        }
    }
});
```

---

### 示例 4: 创建 Handlebars Helper

**场景**: 添加阅读时间显示辅助函数

```javascript
// core/frontend/helpers/reading_time.js
const {SafeString} = require('express-hbs');

module.exports = function readingTime(options) {
    const readingTime = this.reading_time;

    if (!readingTime) {
        return new SafeString('');
    }

    const text = readingTime === 1
        ? '1 minute read'
        : `${readingTime} minutes read`;

    return new SafeString(`<span class="reading-time">${text}</span>`);
};

// 在主题中使用
// {{reading_time}}
```

---

## 🧪 测试架构

### 测试类型

```javascript
// 1. 单元测试 (test/unit/)
// test/unit/server/models/post.test.js
describe('Post Model', function () {
    it('should calculate reading time correctly', async function () {
        const post = models.Post.forge({
            plaintext: 'word '.repeat(200) // 200 words
        });

        await post.calculateReadingTime();

        post.get('reading_time').should.equal(1);
    });
});

// 2. 集成测试 (test/integration/)
// test/integration/api/posts.test.js
describe('Posts API', function () {
    it('can create a post', async function () {
        const res = await request.post(localUtils.API.getApiQuery('posts/'))
            .set('Authorization', `Bearer ${token}`)
            .send({posts: [{
                title: 'Test Post',
                status: 'draft'
            }]})
            .expect(201);

        res.body.posts[0].title.should.equal('Test Post');
    });
});

// 3. E2E 测试 (test/e2e/)
// test/e2e/api/admin/posts.test.js
describe('Posts Admin API E2E', function () {
    it('complete publishing workflow', async function () {
        // 创建草稿
        const draft = await createPost({status: 'draft'});

        // 发布文章
        const published = await publishPost(draft.id);
        published.status.should.equal('published');

        // 验证前端可见
        const frontendRes = await request.get(`/${published.slug}/`);
        frontendRes.statusCode.should.equal(200);
    });
});

// 4. 浏览器测试 (test/e2e-browser/)
// test/e2e-browser/admin/posts.test.js
const {test, expect} = require('@playwright/test');

test('create and publish post in admin', async ({page}) => {
    await page.goto('http://localhost:2368/ghost');
    await page.fill('[name="identification"]', 'test@example.com');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');

    await page.click('text=New post');
    await page.fill('[placeholder="Post title"]', 'Test Post');
    await page.click('text=Publish');

    await expect(page.locator('text=Published')).toBeVisible();
});
```

---

## 🎯 最佳实践

### 1. **错误处理**
```javascript
const errors = require('@tryghost/errors');

// 业务逻辑错误
throw new errors.ValidationError({
    message: 'Invalid email format',
    context: 'User email validation failed',
    help: 'Please provide a valid email address'
});

// 资源未找到
throw new errors.NotFoundError({
    message: 'Post not found',
    id: postId
});

// 权限错误
throw new errors.NoPermissionError({
    message: 'You do not have permission to edit this post'
});
```

### 2. **日志记录**
```javascript
const logging = require('@tryghost/logging');

// 不同级别日志
logging.info('Post published', {postId, title});
logging.warn('Slow query detected', {duration, query});
logging.error('Email sending failed', {error, recipient});

// 调试日志 (DEBUG=ghost:* 时显示)
const debug = require('@tryghost/debug')('posts');
debug('Fetching post', {id: postId});
```

### 3. **性能优化**
```javascript
// 使用 eager loading 避免 N+1 查询
const posts = await models.Post.findPage({
    withRelated: ['authors', 'tags', 'tiers']  // 一次性加载关系
});

// 使用缓存
const settingsCache = require('../../shared/settings-cache');
const siteTitle = settingsCache.get('title');  // 从缓存读取

// 批量操作
await models.Post.bulkEdit(
    postIds.map(id => ({id, status: 'published'}))
);
```

---

## 📚 总结

### Ghost Core 的核心价值

1. **模块化架构** - 清晰的分层设计 (API → Service → Model → DB)
2. **可扩展性** - 插件化的服务系统，易于添加新功能
3. **安全性** - 完善的权限系统和安全措施
4. **性能** - 优化的查询、缓存和批处理机制
5. **开发体验** - 丰富的测试工具和清晰的代码组织

### 关键技术栈

- **后端**: Node.js + Express + Bookshelf/Knex
- **数据库**: MySQL/SQLite
- **模板**: Handlebars
- **认证**: JWT + Session
- **支付**: Stripe
- **邮件**: Mailgun/SMTP
- **测试**: Mocha + Playwright

### 学习路径建议

1. **入门**: 从 `ghost-server.js` 和简单 Model 开始
2. **进阶**: 理解 API 层和 Service 层的设计
3. **深入**: 学习事件系统、权限系统和数据库迁移
4. **实战**: 创建自定义端点、服务和 Handlebars helper

---

**文档版本**: 1.0.0
**最后更新**: 2026-01-26
**对应 Ghost 版本**: 6.14.0
