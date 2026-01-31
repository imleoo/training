# Issue #25979 分析报告：Post URLs in UTF-8 not supported

## 🔍 问题概述

**Issue**: [#25979 - Post URLs in UTF-8 not supported](https://github.com/TryGhost/Ghost/issues/25979)

**症状**: 用户在发布非英文（如中文、日文、阿拉伯文等）文章时，URL 中的 UTF-8 字符被错误处理或拒绝，导致国际化支持受到负面影响。

**影响范围**:
- 所有非英语语言用户
- 国际化部署场景
- 多语言内容发布

---

## 🎯 根本原因分析

### 1. 调用链追溯

```
用户创建/编辑文章标题（包含 UTF-8 字符）
    ↓
Admin API: POST/PUT /ghost/api/admin/posts/
    ↓
ghost/core/core/server/api/endpoints/posts.js
    ↓
ghost/core/core/server/models/post.js (Model 层)
    ↓
ghost/core/core/server/models/base/plugins/generate-slug.js
    ↓
security.string.safe(base, options)  ← 【核心问题点】
    ↓
生成的 slug 仅包含 ASCII 字符，UTF-8 字符被移除或转换
```

### 2. 核心问题代码位置

**文件**: `ghost/core/core/server/models/base/plugins/generate-slug.js`

**关键代码** (第 73 行):
```javascript
slug = security.string.safe(base, options);
```

**问题分析**:
1. `security.string.safe()` 函数来自 `@tryghost/security` 包
2. 该函数的默认行为是将字符串转换为 **URL-safe ASCII 字符串**
3. 转换过程中，非 ASCII 字符（包括 UTF-8 字符）会被：
   - 移除
   - 或转换为 ASCII 近似字符（如 `中文` → 空字符串或 `zhong-wen`）

### 3. 具体问题细节

#### 问题 1: Slug 生成过程中的字符丢失

**场景**:
```
标题: "如何使用 Ghost CMS"
预期 slug: "如何使用-ghost-cms" 或 "%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8-ghost-cms"
实际 slug: "ghost-cms" (UTF-8 部分被移除)
```

**代码位置**: `generate-slug.js:73`
```javascript
// 当前实现
slug = security.string.safe(base, options);
// "如何使用 Ghost CMS" → "ghost-cms"
```

#### 问题 2: 185 字符限制对 UTF-8 URL 的影响

**代码位置**: `generate-slug.js:78-83`
```javascript
if (slug.length > 185) {
    if (!_.has(options, 'importing') || !options.importing) {
        slug = slug.slice(0, 185);
    }
}
```

**问题**:
- 使用字符串长度 (`slug.length`) 而非字节长度
- UTF-8 编码后的 URL 可能远超数据库字段限制（191 字节）
- 示例：中文字符在 UTF-8 编码中占 3 字节，日文占 3-4 字节

#### 问题 3: 数据库字段限制

**数据库 Schema**: `posts` 表的 `slug` 字段
```sql
-- 当前定义 (推测)
slug VARCHAR(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
```

**问题**:
- MySQL 的 `VARCHAR(191)` 限制是**字节数**而非字符数
- UTF-8 字符占用更多字节
- 当前代码按字符计数，可能导致数据库插入失败

---

## 🔧 修复方案

### 方案 A: 支持 UTF-8 URL（推荐）

**优点**:
- 真正的国际化支持
- SEO 友好（Google 支持 UTF-8 URL）
- 用户体验最佳

**实施步骤**:

#### 步骤 1: 修改 `generate-slug.js` 支持 UTF-8

```javascript
// ghost/core/core/server/models/base/plugins/generate-slug.js

const _ = require('lodash');
const security = require('@tryghost/security');
const urlUtils = require('../../../../shared/url-utils');

// 新增：UTF-8 安全的 slug 生成函数
const generateUtf8Slug = (str) => {
    // 1. 转换为小写
    let slug = str.toLowerCase();

    // 2. 移除特殊字符，但保留 UTF-8 字符、数字、连字符
    slug = slug.replace(/[^\u4e00-\u9fa5\u3040-\u309f\u30a0-\u30ff\u0600-\u06ff\w\s-]/g, '');

    // 3. 将空格替换为连字符
    slug = slug.replace(/\s+/g, '-');

    // 4. 移除多余的连字符
    slug = slug.replace(/-+/g, '-').replace(/^-|-$/g, '');

    return slug;
};

// 新增：计算 UTF-8 字节长度
const getUtf8ByteLength = (str) => {
    return Buffer.byteLength(str, 'utf8');
};

module.exports = function (Bookshelf) {
    Bookshelf.Model = Bookshelf.Model.extend({}, {
        generateSlug: function generateSlug(Model, base, options) {
            let slug;
            let slugTryCount = 1;
            const baseName = Model.prototype.tableName.replace(/s$/, '');
            let longSlug;

            const checkIfSlugExists = function checkIfSlugExists(slugToFind) {
                const args = {slug: slugToFind};

                if (options && options.status) {
                    args.status = options.status;
                }

                return Model.findOne(args, options).then(function then(found) {
                    let trimSpace;

                    if (!found) {
                        return slugToFind;
                    }

                    if (options.modelId) {
                        if (found.id === options.modelId) {
                            return slugToFind;
                        }
                    }

                    slugTryCount += 1;

                    if (slugTryCount === 2 && longSlug) {
                        slugToFind = longSlug;
                        longSlug = null;
                        slugTryCount = 1;
                        return checkIfSlugExists(slugToFind);
                    }

                    if (slugTryCount === 2) {
                        slugToFind += '-';
                    } else {
                        trimSpace = -(String(slugTryCount - 1).length);
                        slugToFind = slugToFind.slice(0, trimSpace);
                    }

                    slugToFind += slugTryCount;

                    return checkIfSlugExists(slugToFind);
                });
            };

            // ✅ 修改：支持 UTF-8 slug 生成
            const useUtf8Slugs = options?.utf8Slugs ?? true; // 默认开启

            if (useUtf8Slugs) {
                slug = generateUtf8Slug(base);
            } else {
                // 向后兼容：使用原有的 ASCII-only 方案
                slug = security.string.safe(base, options);
            }

            // ✅ 修改：使用字节长度而非字符长度限制
            const maxByteLength = 185; // 预留空间给后缀数字

            if (getUtf8ByteLength(slug) > maxByteLength) {
                if (!_.has(options, 'importing') || !options.importing) {
                    // 按字节截断，确保不超过限制
                    while (getUtf8ByteLength(slug) > maxByteLength) {
                        // 每次减少一个字符，直到字节长度符合要求
                        slug = slug.slice(0, -1);
                    }
                    // 移除末尾可能的不完整字符或连字符
                    slug = slug.replace(/-+$/, '');
                }
            }

            // User slug shortening logic
            if (baseName === 'user' && options && options.shortSlug && slugTryCount === 1 && slug !== 'ghost-owner') {
                longSlug = slug;
                slug = (slug.indexOf('-') > -1) ? slug.slice(0, slug.indexOf('-')) : slug;
            }

            // Internal tag handling
            if (!_.has(options, 'importing') || !options.importing) {
                if (baseName === 'tag' && /^#/.test(base)) {
                    slug = 'hash-' + slug;
                }
            }

            // Protected slugs
            slug = _.includes(urlUtils.getProtectedSlugs(), slug) ? slug + '-' + baseName : slug;

            // Fallback to model name if empty
            if (!slug) {
                slug = baseName;
            }

            if (options && options.skipDuplicateChecks === true) {
                return slug;
            }

            return checkIfSlugExists(slug);
        }
    });
};
```

#### 步骤 2: 数据库 Migration（确保兼容性）

```javascript
// ghost/core/core/server/data/migrations/versions/5.x/yyyy-mm-dd-hh-mm-ensure-slug-utf8mb4.js

const {createNonTransactionalMigration} = require('../../utils');

module.exports = createNonTransactionalMigration(
    async function up(connection) {
        // 确保 slug 字段使用 utf8mb4 字符集
        await connection.raw(`
            ALTER TABLE posts
            MODIFY slug VARCHAR(191)
            CHARACTER SET utf8mb4
            COLLATE utf8mb4_unicode_ci
        `);

        await connection.raw(`
            ALTER TABLE tags
            MODIFY slug VARCHAR(191)
            CHARACTER SET utf8mb4
            COLLATE utf8mb4_unicode_ci
        `);

        await connection.raw(`
            ALTER TABLE users
            MODIFY slug VARCHAR(191)
            CHARACTER SET utf8mb4
            COLLATE utf8mb4_unicode_ci
        `);
    },

    async function down(connection) {
        // Rollback: 恢复为 utf8 (不推荐，可能丢失数据)
        // 通常不实现 down，因为这会破坏已有数据
    }
);
```

#### 步骤 3: 配置选项（可选）

```javascript
// ghost/core/core/shared/config/defaults.json

{
    "url": {
        "utf8Slugs": true,  // 启用 UTF-8 slug 支持
        "urlEncoding": "auto"  // 可选: "auto" | "percent-encoded" | "punycode"
    }
}
```

#### 步骤 4: URL 输出时的编码处理

```javascript
// ghost/core/core/frontend/meta/url.js

const encodeSlug = (slug, encoding = 'auto') => {
    if (encoding === 'percent-encoded') {
        // 对 UTF-8 字符进行百分号编码
        return encodeURIComponent(slug);
    } else if (encoding === 'punycode') {
        // 使用 Punycode 编码（较少见）
        const punycode = require('punycode/');
        return punycode.toASCII(slug);
    } else {
        // auto: 保持原样，让浏览器处理
        return slug;
    }
};

// 在生成 URL 时使用
const getUrl = (resourceType, data) => {
    // ... 现有逻辑
    const slug = encodeSlug(data.slug, config.get('url:urlEncoding'));
    // ... 构建完整 URL
};
```

---

### 方案 B: URL 编码方案（兼容性方案）

**优点**:
- 兼容性最好
- 不改变现有 ASCII slug 逻辑

**缺点**:
- URL 可读性差
- SEO 效果不如方案 A

**实施示例**:

```javascript
// 在 slug 存储时保留 UTF-8，但在 URL 输出时进行 percent-encoding

// 存储: "如何使用-ghost-cms"
// 输出 URL: "/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8-ghost-cms"

const getPostUrl = (post) => {
    const encodedSlug = encodeURIComponent(post.slug);
    return `${siteUrl}/${encodedSlug}/`;
};
```

---

### 方案 C: 音译方案（折中方案）

**优点**:
- URL 可读性好
- 兼容现有系统

**缺点**:
- 音译不准确
- 需要额外依赖库

**实施示例**:

```javascript
const slugify = require('slugify');

const generateTransliteratedSlug = (title) => {
    return slugify(title, {
        lower: true,
        strict: true,
        locale: 'en', // 或根据内容语言自动检测
        // "如何使用 Ghost CMS" → "ru-he-shi-yong-ghost-cms"
    });
};
```

---

## 📊 方案对比

| 方案 | 实施难度 | 兼容性 | SEO | 用户体验 | 推荐度 |
|------|----------|--------|-----|----------|--------|
| A: UTF-8 支持 | 中等 | 需要测试 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| B: Percent-Encoding | 简单 | 高 | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| C: 音译 | 简单 | 高 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

**推荐**: **方案 A** (UTF-8 支持) + **方案 B** (作为降级选项)

---

## 🧪 测试用例

```javascript
// test/unit/server/models/base/plugins/generate-slug.test.js

describe('UTF-8 Slug Generation', function () {
    it('should generate UTF-8 slugs for Chinese titles', async function () {
        const slug = await models.Post.generateSlug(
            models.Post,
            '如何使用 Ghost CMS',
            {utf8Slugs: true}
        );

        slug.should.equal('如何使用-ghost-cms');
    });

    it('should handle Japanese characters', async function () {
        const slug = await models.Post.generateSlug(
            models.Post,
            'ゴーストCMSの使い方',
            {utf8Slugs: true}
        );

        slug.should.equal('ゴーストcmsの使い方');
    });

    it('should respect byte length limits', async function () {
        const longTitle = '这是一个非常长的中文标题'.repeat(20);
        const slug = await models.Post.generateSlug(
            models.Post,
            longTitle,
            {utf8Slugs: true}
        );

        Buffer.byteLength(slug, 'utf8').should.be.below(191);
    });

    it('should handle duplicate UTF-8 slugs', async function () {
        // 创建第一个文章
        await models.Post.add({
            title: '测试文章',
            slug: '测试文章'
        });

        // 第二个同名文章应该得到 '测试文章-2'
        const slug = await models.Post.generateSlug(
            models.Post,
            '测试文章',
            {utf8Slugs: true}
        );

        slug.should.equal('测试文章-2');
    });
});
```

---

## 🚀 实施步骤建议

### Phase 1: 准备阶段（1-2 天）
1. ✅ 审查现有代码和依赖
2. ✅ 确认数据库字符集配置
3. ✅ 制定测试计划

### Phase 2: 开发阶段（3-5 天）
1. ✅ 实现 UTF-8 slug 生成逻辑
2. ✅ 添加配置选项
3. ✅ 编写单元测试
4. ✅ 编写集成测试

### Phase 3: 测试阶段（2-3 天）
1. ✅ 多语言环境测试（中文、日文、阿拉伯文等）
2. ✅ 性能测试
3. ✅ 向后兼容性测试
4. ✅ 数据库迁移测试

### Phase 4: 发布阶段（1-2 天）
1. ✅ Beta 版本发布
2. ✅ 收集用户反馈
3. ✅ 正式版本发布
4. ✅ 文档更新

---

## ⚠️ 注意事项

### 1. 浏览器兼容性
- 现代浏览器都支持 UTF-8 URL
- 旧浏览器可能需要 percent-encoding 降级

### 2. Web 服务器配置
- Nginx/Apache 需要正确配置 UTF-8 支持
```nginx
# Nginx 配置示例
location / {
    charset utf-8;
    charset_types text/html text/plain text/css application/json;
}
```

### 3. SEO 影响
- Google 完全支持 UTF-8 URL
- 确保 `sitemap.xml` 正确编码

### 4. 缓存失效
- 修改 slug 生成逻辑后，需要清理相关缓存
- 考虑提供数据迁移工具

---

## 📚 参考资料

1. **RFC 3986** - Uniform Resource Identifier (URI): Generic Syntax
2. **UTF-8 URL 标准**: https://www.w3.org/International/articles/idn-and-iri/
3. **Google SEO 指南**: UTF-8 URLs in Search
4. **MySQL UTF8MB4**: https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb4.html

---

## 🎯 总结

**根本原因**: Ghost CMS 的 slug 生成机制使用 `security.string.safe()` 函数强制转换为 ASCII-only 字符串，导致 UTF-8 字符丢失。

**推荐修复方案**:
1. 修改 `generate-slug.js` 支持 UTF-8 字符保留
2. 使用字节长度而非字符长度进行限制
3. 提供配置选项允许向后兼容
4. 完善测试覆盖多语言场景

**预期效果**:
- ✅ 完整的国际化支持
- ✅ SEO 友好的多语言 URL
- ✅ 向后兼容现有 ASCII slug
- ✅ 符合 Web 标准和最佳实践

---

**文档版本**: 1.0
**创建日期**: 2026-01-26
**分析者**: Claude Code Assistant
**相关 Issue**: [#25979](https://github.com/TryGhost/Ghost/issues/25979)
