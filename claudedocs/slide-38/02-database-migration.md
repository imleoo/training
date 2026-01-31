# 数据库迁移方案

## 迁移文件位置

`ghost/core/core/server/data/migrations/versions/5.x/YYYY-MM-DD-HH-mm-add-members-bookmarks-table.js`

## 迁移脚本

```javascript
const {createAddColumnMigration} = require('../../utils');

module.exports = createAddColumnMigration('members_bookmarks', 'members_bookmarks', [
    {
        table: 'members_bookmarks',
        column: 'id',
        dbIsInCorrectState(columnExists) {
            return columnExists === true;
        },
        operation: createTable,
        operationVerb: 'Creating'
    }
]);

async function createTable(connection, tableSpec) {
    const tableName = tableSpec.table;

    return connection.schema.createTable(tableName, function (table) {
        table.string('id', 24).primary();
        table.string('member_id', 24).notNullable();
        table.string('post_id', 24).notNullable();
        table.dateTime('created_at').notNullable();

        // Foreign keys
        table.foreign('member_id')
            .references('members.id')
            .onDelete('CASCADE');

        table.foreign('post_id')
            .references('posts.id')
            .onDelete('CASCADE');

        // Unique constraint
        table.unique(['member_id', 'post_id'], 'unique_member_post_bookmark');

        // Indexes
        table.index('member_id', 'idx_member_bookmarks_member_id');
        table.index('created_at', 'idx_member_bookmarks_created_at');
    });
}
```

## 回滚脚本

```javascript
async function down(connection) {
    return connection.schema.dropTableIfExists('members_bookmarks');
}
```

## 数据完整性考虑

### 级联删除策略
- **会员删除**: 自动删除该会员的所有书签 (`ON DELETE CASCADE`)
- **文章删除**: 自动删除指向该文章的所有书签 (`ON DELETE CASCADE`)

### 唯一性约束
- 防止同一会员重复保存同一文章
- 数据库层面强制执行,避免应用层竞态条件

### 索引优化
- `member_id` 索引: 优化"获取用户所有书签"查询
- `created_at` 索引: 支持按时间排序和分页

## 测试数据

```sql
-- 测试插入
INSERT INTO members_bookmarks (id, member_id, post_id, created_at)
VALUES
    ('bookmark_001', 'member_123', 'post_456', NOW()),
    ('bookmark_002', 'member_123', 'post_789', NOW());

-- 测试唯一性约束(应该失败)
INSERT INTO members_bookmarks (id, member_id, post_id, created_at)
VALUES ('bookmark_003', 'member_123', 'post_456', NOW());
-- Error: Duplicate entry for unique constraint

-- 测试级联删除
DELETE FROM members WHERE id = 'member_123';
-- 应该自动删除 bookmark_001 和 bookmark_002
```

## 性能预估

### 存储空间
- 每条记录约 100 bytes
- 1000 个书签/用户 ≈ 100 KB
- 10,000 用户 × 平均 50 书签 = 500,000 条记录 ≈ 50 MB

### 查询性能
- **添加书签**: 单行插入,< 5ms
- **移除书签**: 索引查找 + 删除,< 10ms
- **获取列表**: 索引扫描 + JOIN,15 条/页 < 50ms
- **检查状态**: 索引查找,< 5ms

## 迁移执行

```bash
# 开发环境
yarn knex-migrator migrate

# 生产环境(自动执行)
# Ghost 启动时会自动检测并执行未运行的迁移
```
