# Portal 集成 - 书签列表页面

## Portal 路由扩展

**文件**: `apps/portal/src/App.js`

在 Portal 应用中添加书签路由:

```javascript
import BookmarksPage from './pages/BookmarksPage';

// 在路由配置中添加
const routes = [
    // ... 现有路由
    {
        path: '/bookmarks',
        component: BookmarksPage,
        requireAuth: true
    }
];
```

## 导航菜单扩展

**文件**: `apps/portal/src/components/AccountMenu.js`

在会员账户菜单中添加书签入口:

```javascript
const menuItems = [
    {
        label: '我的账户',
        path: '/account',
        icon: 'user'
    },
    {
        label: '我的书签',
        path: '/bookmarks',
        icon: 'bookmark'
    },
    {
        label: '订阅管理',
        path: '/subscription',
        icon: 'credit-card'
    }
];
```

## 书签列表页面组件

**文件**: `apps/portal/src/pages/BookmarksPage.js`

```javascript
import React from 'react';
import {useBookmarksList} from '../hooks/useBookmarksList';
import BookmarkCard from '../components/BookmarkCard';
import LoadingSpinner from '../components/LoadingSpinner';
import EmptyState from '../components/EmptyState';

const BookmarksPage = () => {
    const {
        bookmarks,
        totalCount,
        isLoading,
        isFetchingNextPage,
        hasNextPage,
        fetchNextPage
    } = useBookmarksList({limit: 15});

    if (isLoading) {
        return <LoadingSpinner />;
    }

    if (bookmarks.length === 0) {
        return (
            <EmptyState
                icon="bookmark"
                title="还没有保存的文章"
                description="浏览文章时点击书签按钮,即可保存到这里"
            />
        );
    }

    return (
        <div className="bookmarks-page">
            <header className="bookmarks-header">
                <h1>我的书签</h1>
                <p className="bookmarks-count">
                    共 {totalCount} 篇文章
                </p>
            </header>

            <div className="bookmarks-grid">
                {bookmarks.map(bookmark => (
                    <BookmarkCard
                        key={bookmark.id}
                        bookmark={bookmark}
                    />
                ))}
            </div>

            {hasNextPage && (
                <button
                    onClick={() => fetchNextPage()}
                    disabled={isFetchingNextPage}
                    className="load-more-button"
                >
                    {isFetchingNextPage ? '加载中...' : '加载更多'}
                </button>
            )}
        </div>
    );
};

export default BookmarksPage;
```
