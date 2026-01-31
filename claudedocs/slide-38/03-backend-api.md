# 后端 API 实现方案

## 文件结构

```
ghost/core/core/server/
├── api/endpoints/
│   └── members-bookmarks.js          # API 端点定义
├── services/
│   └── members-bookmarks/
│       ├── index.js                  # 服务入口
│       ├── BookmarksService.js       # 核心业务逻辑
│       └── BookmarksService.test.js  # 单元测试
└── models/
    └── member-bookmark.js            # 数据模型
```

## 数据模型 (Model)

**文件**: `ghost/core/core/server/models/member-bookmark.js`

```javascript
const ghostBookshelf = require('./base');

const MemberBookmark = ghostBookshelf.Model.extend({
    tableName: 'members_bookmarks',

    defaults() {
        return {
            id: ghostBookshelf.Model.generateId(),
            created_at: new Date()
        };
    },

    member() {
        return this.belongsTo('Member', 'member_id');
    },

    post() {
        return this.belongsTo('Post', 'post_id');
    }
}, {
    // 静态方法
    async add(data, options) {
        return this.forge(data).save(null, options);
    },

    async findOne(data, options) {
        return this.forge(data).fetch(options);
    },

    async destroy(data, options) {
        return this.forge({id: data.id}).destroy(options);
    }
});

module.exports = {
    MemberBookmark: ghostBookshelf.model('MemberBookmark', MemberBookmark)
};
```

## 业务逻辑服务 (Service)

**文件**: `ghost/core/core/server/services/members-bookmarks/BookmarksService.js`

```javascript
const errors = require('@tryghost/errors');
const tpl = require('@tryghost/tpl');

const messages = {
    bookmarkNotFound: 'Bookmark not found.',
    postNotFound: 'Post not found.',
    bookmarkAlreadyExists: 'Post already bookmarked.',
    unauthorized: 'You must be signed in to bookmark posts.'
};

class BookmarksService {
    constructor({models, urlUtils}) {
        this.models = models;
        this.urlUtils = urlUtils;
    }

    /**
     * 添加书签
     * @param {Object} data - { post_id: string }
     * @param {Object} options - { context: { member: { id } } }
     */
    async add(data, options) {
        const memberId = options?.context?.member?.id;

        if (!memberId) {
            throw new errors.UnauthorizedError({
                message: tpl(messages.unauthorized)
            });
        }

        // 验证文章是否存在
        const post = await this.models.Post.findOne(
            {id: data.post_id, status: 'published'},
            {require: false}
        );

        if (!post) {
            throw new errors.NotFoundError({
                message: tpl(messages.postNotFound)
            });
        }

        // 检查是否已存在
        const existing = await this.models.MemberBookmark.findOne(
            {member_id: memberId, post_id: data.post_id},
            {require: false}
        );

        if (existing) {
            throw new errors.ValidationError({
                message: tpl(messages.bookmarkAlreadyExists)
            });
        }

        // 创建书签
        const bookmark = await this.models.MemberBookmark.add({
            member_id: memberId,
            post_id: data.post_id
        });

        return this._formatBookmark(bookmark);
    }

    /**
     * 移除书签
     * @param {string} postId - 文章 ID
     * @param {Object} options - { context: { member: { id } } }
     */
    async remove(postId, options) {
        const memberId = options?.context?.member?.id;

        if (!memberId) {
            throw new errors.UnauthorizedError({
                message: tpl(messages.unauthorized)
            });
        }

        const bookmark = await this.models.MemberBookmark.findOne(
            {member_id: memberId, post_id: postId},
            {require: false}
        );

        if (!bookmark) {
            throw new errors.NotFoundError({
                message: tpl(messages.bookmarkNotFound)
            });
        }

        await this.models.MemberBookmark.destroy({id: bookmark.id});

        return {success: true};
    }

    /**
     * 获取用户书签列表
     * @param {Object} options - { context: { member: { id } }, page, limit }
     */
    async browse(options) {
        const memberId = options?.context?.member?.id;

        if (!memberId) {
            throw new errors.UnauthorizedError({
                message: tpl(messages.unauthorized)
            });
        }

        const page = options.page || 1;
        const limit = options.limit || 15;

        const bookmarks = await this.models.MemberBookmark.findPage({
            filter: `member_id:${memberId}`,
            withRelated: ['post'],
            order: 'created_at DESC',
            page,
            limit
        });

        return {
            bookmarks: bookmarks.data.map(b => this._formatBookmark(b)),
            meta: bookmarks.meta
        };
    }

    /**
     * 检查文章是否已书签
     * @param {string} postId - 文章 ID
     * @param {Object} options - { context: { member: { id } } }
     */
    async check(postId, options) {
        const memberId = options?.context?.member?.id;

        if (!memberId) {
            return {bookmarked: false};
        }

        const bookmark = await this.models.MemberBookmark.findOne(
            {member_id: memberId, post_id: postId},
            {require: false}
        );

        return {
            bookmarked: !!bookmark,
            bookmark_id: bookmark?.id
        };
    }

    _formatBookmark(bookmark) {
        const json = bookmark.toJSON();

        return {
            id: json.id,
            post_id: json.post_id,
            created_at: json.created_at,
            post: json.post ? {
                id: json.post.id,
                title: json.post.title,
                slug: json.post.slug,
                excerpt: json.post.excerpt,
                feature_image: json.post.feature_image,
                url: this.urlUtils.urlFor('post', {post: json.post}, true)
            } : null
        };
    }
}

module.exports = BookmarksService;
```

## API 端点定义

**文件**: `ghost/core/core/server/api/endpoints/members-bookmarks.js`

```javascript
const bookmarksService = require('../../services/members-bookmarks');

module.exports = {
    docName: 'members_bookmarks',

    add: {
        headers: {
            cacheInvalidate: false
        },
        permissions: true,
        validation: {
            data: {
                post_id: {required: true}
            }
        },
        query(frame) {
            return bookmarksService.add(frame.data, frame.options);
        }
    },

    destroy: {
        headers: {
            cacheInvalidate: false
        },
        permissions: true,
        validation: {
            options: {
                id: {required: true}
            }
        },
        query(frame) {
            return bookmarksService.remove(frame.options.id, frame.options);
        }
    },

    browse: {
        headers: {
            cacheInvalidate: false
        },
        permissions: true,
        query(frame) {
            return bookmarksService.browse(frame.options);
        }
    },

    check: {
        headers: {
            cacheInvalidate: false
        },
        permissions: false, // 允许未登录用户调用(返回 false)
        validation: {
            options: {
                id: {required: true}
            }
        },
        query(frame) {
            return bookmarksService.check(frame.options.id, frame.options);
        }
    }
};
```

## 路由注册

**文件**: `ghost/core/core/server/api/endpoints/index.js`

```javascript
// 添加到现有路由配置
module.exports = {
    // ... 现有路由
    membersBookmarks: require('./members-bookmarks')
};
```

**文件**: `ghost/core/core/server/web/api/middleware/routes.js`

```javascript
// 在 Members API 路由组中添加
router.post('/bookmarks', mw.authMember, api.http(api.membersBookmarks.add));
router.del('/bookmarks/:id', mw.authMember, api.http(api.membersBookmarks.destroy));
router.get('/bookmarks', mw.authMember, api.http(api.membersBookmarks.browse));
router.get('/posts/:id/bookmarked', api.http(api.membersBookmarks.check));
```

## 下一步

继续生成前端实现方案?
