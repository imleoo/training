# 测试策略

## 测试金字塔

```
        E2E Tests (10%)
       /              \
      /   Integration   \
     /    Tests (30%)    \
    /____________________\
    |                    |
    |  Unit Tests (60%)  |
    |____________________|
```

## 单元测试 (60%)

### 后端单元测试

**数据模型测试** (`ghost/core/test/unit/server/models/member-bookmark.test.js`)
```javascript
describe('MemberBookmark Model', function () {
    it('should create bookmark with valid data', async function () {
        const bookmark = await models.MemberBookmark.add({
            member_id: 'member_123',
            post_id: 'post_456'
        });

        should.exist(bookmark.id);
        bookmark.get('member_id').should.equal('member_123');
    });

    it('should enforce unique constraint', async function () {
        await models.MemberBookmark.add({
            member_id: 'member_123',
            post_id: 'post_456'
        });

        await models.MemberBookmark.add({
            member_id: 'member_123',
            post_id: 'post_456'
        }).should.be.rejected();
    });

    it('should cascade delete when member is deleted', async function () {
        const bookmark = await models.MemberBookmark.add({
            member_id: 'member_123',
            post_id: 'post_456'
        });

        await models.Member.destroy({id: 'member_123'});

        const deleted = await models.MemberBookmark.findOne({id: bookmark.id});
        should.not.exist(deleted);
    });
});
```

**服务层测试** (`ghost/core/test/unit/server/services/members-bookmarks/BookmarksService.test.js`)
```javascript
describe('BookmarksService', function () {
    let service;

    beforeEach(function () {
        service = new BookmarksService({
            models: models,
            urlUtils: urlUtils
        });
    });

    describe('add', function () {
        it('should add bookmark for authenticated member', async function () {
            const result = await service.add(
                {post_id: 'post_123'},
                {context: {member: {id: 'member_456'}}}
            );

            result.should.have.property('id');
            result.post_id.should.equal('post_123');
        });

        it('should throw UnauthorizedError for unauthenticated request', async function () {
            await service.add(
                {post_id: 'post_123'},
                {context: {}}
            ).should.be.rejectedWith(/must be signed in/);
        });

        it('should throw NotFoundError for non-existent post', async function () {
            await service.add(
                {post_id: 'invalid_post'},
                {context: {member: {id: 'member_456'}}}
            ).should.be.rejectedWith(/Post not found/);
        });

        it('should throw ValidationError for duplicate bookmark', async function () {
            await service.add(
                {post_id: 'post_123'},
                {context: {member: {id: 'member_456'}}}
            );

            await service.add(
                {post_id: 'post_123'},
                {context: {member: {id: 'member_456'}}}
            ).should.be.rejectedWith(/already bookmarked/);
        });
    });

    describe('browse', function () {
        it('should return paginated bookmarks', async function () {
            const result = await service.browse({
                context: {member: {id: 'member_456'}},
                page: 1,
                limit: 15
            });

            result.should.have.property('bookmarks');
            result.should.have.property('meta');
            result.meta.should.have.property('pagination');
        });

        it('should include post data in response', async function () {
            const result = await service.browse({
                context: {member: {id: 'member_456'}}
            });

            result.bookmarks[0].should.have.property('post');
            result.bookmarks[0].post.should.have.properties([
                'id', 'title', 'slug', 'url'
            ]);
        });
    });
});
```

### 前端单元测试

**Hook 测试** (`apps/bookmarks-ui/src/hooks/useBookmark.test.ts`)
```typescript
import {renderHook, waitFor} from '@testing-library/react';
import {QueryClient, QueryClientProvider} from '@tanstack/react-query';
import {useBookmark} from './useBookmark';

describe('useBookmark', () => {
    let queryClient: QueryClient;
    let mockApi: any;

    beforeEach(() => {
        queryClient = new QueryClient();
        mockApi = {
            checkBookmarked: jest.fn(),
            addBookmark: jest.fn(),
            removeBookmark: jest.fn()
        };
    });

    it('should fetch bookmark status on mount', async () => {
        mockApi.checkBookmarked.mockResolvedValue({bookmarked: true});

        const {result} = renderHook(
            () => useBookmark({postId: 'post_123', api: mockApi}),
            {wrapper: ({children}) => (
                <QueryClientProvider client={queryClient}>
                    {children}
                </QueryClientProvider>
            )}
        );

        await waitFor(() => {
            expect(result.current.isBookmarked).toBe(true);
        });
    });

    it('should toggle bookmark state', async () => {
        mockApi.checkBookmarked.mockResolvedValue({bookmarked: false});
        mockApi.addBookmark.mockResolvedValue({id: 'bookmark_123'});

        const {result} = renderHook(
            () => useBookmark({postId: 'post_123', api: mockApi}),
            {wrapper: ({children}) => (
                <QueryClientProvider client={queryClient}>
                    {children}
                </QueryClientProvider>
            )}
        );

        await waitFor(() => {
            expect(result.current.isBookmarked).toBe(false);
        });

        result.current.toggleBookmark();

        await waitFor(() => {
            expect(mockApi.addBookmark).toHaveBeenCalledWith('post_123');
            expect(result.current.isBookmarked).toBe(true);
        });
    });
});
```

## 集成测试 (30%)

### API 集成测试

**文件**: `ghost/core/test/e2e-api/members/bookmarks.test.js`

```javascript
const {agentProvider, mockManager} = require('../../utils/e2e-framework');

describe('Members Bookmarks API', function () {
    let agent;
    let membersAgent;

    before(async function () {
        agent = await agentProvider.getAdminAPIAgent();
        membersAgent = await agentProvider.getMembersAPIAgent();
    });

    describe('POST /bookmarks', function () {
        it('should add bookmark for authenticated member', async function () {
            const post = await agent.get('/posts/').expect(200);
            const postId = post.body.posts[0].id;

            await membersAgent
                .post('/bookmarks/')
                .send({post_id: postId})
                .expect(201)
                .expect((res) => {
                    res.body.bookmark.should.have.property('id');
                    res.body.bookmark.post_id.should.equal(postId);
                });
        });

        it('should return 401 for unauthenticated request', async function () {
            await agent
                .post('/members/api/bookmarks/')
                .send({post_id: 'post_123'})
                .expect(401);
        });
    });

    describe('GET /bookmarks', function () {
        it('should return member bookmarks with pagination', async function () {
            await membersAgent
                .get('/bookmarks/?page=1&limit=15')
                .expect(200)
                .expect((res) => {
                    res.body.should.have.property('bookmarks');
                    res.body.should.have.property('meta');
                    res.body.meta.pagination.should.have.properties([
                        'page', 'limit', 'pages', 'total'
                    ]);
                });
        });
    });

    describe('DELETE /bookmarks/:id', function () {
        it('should remove bookmark', async function () {
            const post = await agent.get('/posts/').expect(200);
            const postId = post.body.posts[0].id;

            await membersAgent.post('/bookmarks/').send({post_id: postId});

            await membersAgent
                .delete(`/bookmarks/${postId}`)
                .expect(204);

            const check = await membersAgent.get(`/posts/${postId}/bookmarked`);
            check.body.bookmarked.should.be.false();
        });
    });
});
```

## E2E 测试 (10%)

### Playwright 浏览器测试

**文件**: `e2e/tests/bookmarks.spec.ts`

```typescript
import {test, expect} from '@playwright/test';

test.describe('Bookmarks Feature', () => {
    test.beforeEach(async ({page}) => {
        // 登录会员
        await page.goto('/');
        await page.click('[data-portal="signin"]');
        await page.fill('input[type="email"]', 'member@example.com');
        await page.click('button[type="submit"]');
        // 等待登录完成
        await page.waitForSelector('[data-member-authenticated]');
    });

    test('should add bookmark from article page', async ({page}) => {
        // 访问文章页面
        await page.goto('/welcome/');

        // 点击书签按钮
        await page.click('.ghost-bookmark-button');

        // 验证提示消息
        await expect(page.locator('.ghost-bookmark-toast'))
            .toContainText('已保存到书签');

        // 验证按钮状态变化
        await expect(page.locator('.ghost-bookmark-button'))
            .toContainText('已保存');
    });

    test('should view bookmarks in Portal', async ({page}) => {
        // 打开 Portal
        await page.click('[data-portal="account"]');

        // 点击"我的书签"
        await page.click('text=我的书签');

        // 验证书签列表
        await expect(page.locator('.bookmarks-page')).toBeVisible();
        await expect(page.locator('.bookmark-card')).toHaveCount(1);
    });

    test('should remove bookmark from list', async ({page}) => {
        // 访问书签页面
        await page.goto('/portal/bookmarks');

        // 点击移除按钮
        await page.click('.bookmark-remove');

        // 验证卡片消失
        await expect(page.locator('.bookmark-card')).toHaveCount(0);

        // 验证空状态
        await expect(page.locator('text=还没有保存的文章')).toBeVisible();
    });

    test('should persist bookmarks across sessions', async ({page, context}) => {
        // 添加书签
        await page.goto('/welcome/');
        await page.click('.ghost-bookmark-button');

        // 创建新标签页(模拟新会话)
        const newPage = await context.newPage();
        await newPage.goto('/portal/bookmarks');

        // 验证书签仍然存在
        await expect(newPage.locator('.bookmark-card')).toHaveCount(1);
    });
});
```

## 性能测试

### 负载测试脚本

```javascript
// 使用 k6 进行负载测试
import http from 'k6/http';
import {check} from 'k6';

export let options = {
    stages: [
        {duration: '1m', target: 50},  // 50 并发用户
        {duration: '3m', target: 100}, // 100 并发用户
        {duration: '1m', target: 0}    // 降至 0
    ]
};

export default function () {
    // 添加书签
    let addRes = http.post(
        'http://localhost:2368/members/api/bookmarks/',
        JSON.stringify({post_id: 'post_123'}),
        {headers: {'Content-Type': 'application/json'}}
    );

    check(addRes, {
        'add bookmark status 201': (r) => r.status === 201,
        'add bookmark duration < 200ms': (r) => r.timings.duration < 200
    });

    // 获取书签列表
    let listRes = http.get('http://localhost:2368/members/api/bookmarks/');

    check(listRes, {
        'list bookmarks status 200': (r) => r.status === 200,
        'list bookmarks duration < 300ms': (r) => r.timings.duration < 300
    });
}
```

## 测试覆盖率目标

- **后端代码**: ≥ 80%
- **前端组件**: ≥ 75%
- **API 端点**: 100%
- **关键用户流程**: 100%
