# Ghost 书签功能 - 需求规格说明书

## 功能概述

为 Ghost 添加书签功能,允许已登录会员保存文章以便稍后阅读。

## 核心需求

### 用户角色
- **目标用户**: 所有已登录会员(免费 + 付费)
- **访问限制**: 未登录用户不可见书签功能

### 功能范围 (MVP)
1. **添加/移除书签**: 在文章页面保存或取消保存文章
2. **查看书签列表**: 在 Portal 会员区查看已保存的文章列表
3. **基础交互**: 实时状态更新、简单的列表展示

### 不包含的功能 (未来迭代)
- 书签分类/标签
- 阅读进度追踪
- 书签笔记
- 书签分享
- 书签搜索/过滤

## 技术实现策略

### 主题集成方式
**自动注入模式** (类似 Portal/Comments UI):
- 通过 `{{ghost_head}}` 自动注入书签 JavaScript 和 CSS
- 主题开发者无需修改代码即可支持
- 提供可选的配置项用于自定义样式和位置

### 前端架构
- **新建应用**: `apps/bookmarks-ui` (React + Vite)
- **构建产物**: UMD bundle 注入到主题
- **API 通信**: 使用 Members API 进行身份验证和数据操作

### 后端架构
- **数据库**: 新建 `members_bookmarks` 表
- **API**: 扩展 Members API 添加书签端点
- **权限**: 基于现有 Members 会话验证

## 用户体验设计

### 书签按钮位置
**文章详情页**:
- 默认位置: 文章标题下方,与作者信息并列
- 备选位置: 文章底部工具栏(可通过配置调整)
- 视觉样式: 书签图标 + "保存" 文字(已保存时高亮)

### 书签列表页面
**Portal 会员区集成**:
- 路径: Portal 会员区新增 "我的书签" 标签
- 位置: 与 "账户设置"、"订阅管理" 并列
- 列表项显示:
  - 文章封面图(如有)
  - 文章标题
  - 文章摘要(前 150 字符)
  - 保存时间
  - 移除书签按钮

### 交互反馈
- **添加书签**: 图标填充 + Toast 提示 "已保存到书签"
- **移除书签**: 图标清空 + Toast 提示 "已从书签移除"
- **加载状态**: 按钮显示 loading 状态防止重复点击

## 数据设计

### 数据库表结构
```sql
CREATE TABLE members_bookmarks (
    id VARCHAR(24) PRIMARY KEY,
    member_id VARCHAR(24) NOT NULL,
    post_id VARCHAR(24) NOT NULL,
    created_at DATETIME NOT NULL,
    FOREIGN KEY (member_id) REFERENCES members(id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    UNIQUE KEY unique_member_post (member_id, post_id)
);

CREATE INDEX idx_member_bookmarks_member_id ON members_bookmarks(member_id);
CREATE INDEX idx_member_bookmarks_created_at ON members_bookmarks(created_at);
```

### API 端点
```
POST   /members/api/bookmarks/
       Body: { post_id: string }
       Response: { bookmark: { id, post_id, created_at } }

DELETE /members/api/bookmarks/:post_id
       Response: { success: true }

GET    /members/api/bookmarks/?page=1&limit=15
       Response: {
         bookmarks: [{ id, post, created_at }],
         meta: { pagination }
       }

GET    /members/api/posts/:id/bookmarked
       Response: { bookmarked: boolean, bookmark_id?: string }
```

## 实施优先级

### Phase 1: 核心功能 (MVP)
1. 数据库迁移
2. Members API 端点实现
3. Bookmarks UI 应用开发
4. Portal 集成书签列表
5. 主题自动注入机制

### Phase 2: 优化增强 (未来)
- 书签分类和标签
- 阅读进度追踪
- 书签导出功能
- Admin 端书签数据分析

## 成功指标

- 书签功能在所有主题中自动可用
- 已登录会员可以无缝添加/移除书签
- Portal 会员区正确显示书签列表
- API 响应时间 < 200ms
- 支持至少 1000 个书签/用户无性能问题
