# 实施步骤清单

## Phase 1: 数据库和后端 API (2-3天)

### 1.1 数据库迁移
- [ ] 创建迁移文件 `add-members-bookmarks-table.js`
- [ ] 定义表结构(id, member_id, post_id, created_at)
- [ ] 添加外键约束和索引
- [ ] 测试迁移执行和回滚
- [ ] 验证级联删除行为

### 1.2 数据模型
- [ ] 创建 `ghost/core/core/server/models/member-bookmark.js`
- [ ] 定义 Bookshelf 模型和关系
- [ ] 添加静态方法(add, findOne, destroy)
- [ ] 编写模型单元测试

### 1.3 业务逻辑服务
- [ ] 创建 `services/members-bookmarks/BookmarksService.js`
- [ ] 实现 `add()` 方法(添加书签)
- [ ] 实现 `remove()` 方法(移除书签)
- [ ] 实现 `browse()` 方法(获取列表)
- [ ] 实现 `check()` 方法(检查状态)
- [ ] 添加错误处理和验证
- [ ] 编写服务单元测试

### 1.4 API 端点
- [ ] 创建 `api/endpoints/members-bookmarks.js`
- [ ] 定义 4 个端点(add, destroy, browse, check)
- [ ] 配置权限和验证规则
- [ ] 注册路由到 Members API
- [ ] 编写 API 集成测试

### 1.5 后端测试
- [ ] 单元测试覆盖率 >80%
- [ ] 集成测试覆盖所有 API 端点
- [ ] 测试并发场景(重复添加书签)
- [ ] 测试级联删除(会员/文章删除)
- [ ] 性能测试(1000+ 书签查询)

---

## Phase 2: 前端 Bookmarks UI 应用 (3-4天)

### 2.1 项目初始化
- [ ] 创建 `apps/bookmarks-ui` 目录
- [ ] 配置 `package.json` 和依赖
- [ ] 配置 Vite 构建(UMD 输出)
- [ ] 配置 TypeScript

### 2.2 API 客户端
- [ ] 实现 `utils/api.ts` (BookmarksAPI 类)
- [ ] 实现 `utils/auth.ts` (MemberAuth 类)
- [ ] 添加错误处理和重试逻辑
- [ ] 编写 API 客户端测试

### 2.3 React Hooks
- [ ] 实现 `hooks/useBookmark.ts`
- [ ] 实现 `hooks/useBookmarksList.ts`
- [ ] 配置 React Query 缓存策略
- [ ] 编写 Hook 单元测试

### 2.4 UI 组件
- [ ] 实现 `components/BookmarkIcon.tsx`
- [ ] 实现 `components/BookmarkButton.tsx`
- [ ] 实现 `components/Toast.tsx`
- [ ] 添加组件样式和动画
- [ ] 编写组件测试

### 2.5 应用入口
- [ ] 实现 `App.tsx` (React 应用)
- [ ] 实现 `index.ts` (UMD 导出和自动挂载)
- [ ] 实现自动挂载逻辑(查找文章元素)
- [ ] 添加手动挂载 API
- [ ] 编写集成测试

### 2.6 构建和部署
- [ ] 构建 UMD bundle (`yarn build`)
- [ ] 验证产物大小(<50KB gzipped)
- [ ] 配置资源复制到 `ghost/core/core/built/admin/`
- [ ] 测试在不同主题中的兼容性

---

## Phase 3: Portal 集成 (2-3天)

### 3.1 路由和导航
- [ ] 在 Portal 添加 `/bookmarks` 路由
- [ ] 在账户菜单添加"我的书签"入口
- [ ] 配置路由权限(requireAuth)

### 3.2 书签列表页面
- [ ] 实现 `pages/BookmarksPage.js`
- [ ] 实现 `components/BookmarkCard.js`
- [ ] 实现空状态组件
- [ ] 实现加载状态和错误处理
- [ ] 添加无限滚动/分页

### 3.3 样式和交互
- [ ] 实现响应式布局(移动端适配)
- [ ] 添加卡片悬停效果
- [ ] 添加移除书签动画
- [ ] 优化加载性能

### 3.4 Portal 测试
- [ ] 单元测试覆盖核心组件
- [ ] E2E 测试书签列表页面
- [ ] 测试移动端体验
- [ ] 测试无障碍访问(a11y)

---

## Phase 4: 主题集成和自动注入 (1-2天)

### 4.1 Ghost Head 注入
- [ ] 修改 `ghost/core/core/frontend/helpers/ghost_head.js`
- [ ] 添加书签脚本和样式注入逻辑
- [ ] 配置条件加载(仅会员启用时)
- [ ] 添加配置选项(位置、样式等)

### 4.2 主题约定
- [ ] 定义 `data-post-id` 属性约定
- [ ] 编写主题集成文档
- [ ] 提供可选的 Handlebars helper
- [ ] 测试官方主题兼容性

### 4.3 自动挂载测试
- [ ] 测试在 Casper 主题中的表现
- [ ] 测试在自定义主题中的表现
- [ ] 测试多文章列表页面
- [ ] 测试单文章详情页面

---

## Phase 5: 测试和优化 (2-3天)

### 5.1 端到端测试
- [ ] 编写 Playwright E2E 测试
- [ ] 测试完整用户流程(添加→查看→移除)
- [ ] 测试未登录用户体验
- [ ] 测试错误场景(网络失败等)

### 5.2 性能优化
- [ ] 优化 API 查询性能
- [ ] 优化前端 bundle 大小
- [ ] 添加 React Query 缓存策略
- [ ] 测试大量书签场景(1000+)

### 5.3 安全审计
- [ ] 验证 API 权限控制
- [ ] 测试 CSRF 保护
- [ ] 测试 SQL 注入防护
- [ ] 测试 XSS 防护

### 5.4 文档编写
- [ ] 用户使用文档
- [ ] 主题开发者集成指南
- [ ] API 文档
- [ ] 故障排查指南

---

## Phase 6: 发布准备 (1天)

### 6.1 代码审查
- [ ] 后端代码审查
- [ ] 前端代码审查
- [ ] 测试覆盖率检查
- [ ] 性能基准测试

### 6.2 迁移准备
- [ ] 验证迁移脚本
- [ ] 准备回滚方案
- [ ] 测试生产环境部署

### 6.3 发布
- [ ] 合并到主分支
- [ ] 创建发布标签
- [ ] 更新 CHANGELOG
- [ ] 发布公告

---

## 总计时间估算

- **Phase 1**: 2-3 天(后端)
- **Phase 2**: 3-4 天(前端应用)
- **Phase 3**: 2-3 天(Portal)
- **Phase 4**: 1-2 天(主题集成)
- **Phase 5**: 2-3 天(测试优化)
- **Phase 6**: 1 天(发布)

**总计**: 11-16 天(约 2-3 周)
