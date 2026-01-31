# 应用入口和自动挂载

## 应用主入口

**文件**: `apps/bookmarks-ui/src/App.tsx`

```typescript
import React from 'react';
import {QueryClient, QueryClientProvider} from '@tanstack/react-query';
import {BookmarkButton} from './components/BookmarkButton';
import {BookmarksAPI} from './utils/api';

const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            retry: 1,
            refetchOnWindowFocus: false
        }
    }
});

interface AppProps {
    postId: string;
    api: BookmarksAPI;
    showLabel?: boolean;
    size?: 'small' | 'medium' | 'large';
}

export const App: React.FC<AppProps> = (props) => {
    return (
        <QueryClientProvider client={queryClient}>
            <BookmarkButton {...props} />
        </QueryClientProvider>
    );
};
```

## UMD 导出和自动挂载

**文件**: `apps/bookmarks-ui/src/index.ts`

```typescript
import React from 'react';
import {createRoot} from 'react-dom/client';
import {App} from './App';
import {BookmarksAPI} from './utils/api';
import {MemberAuth} from './utils/auth';

interface BookmarksConfig {
    siteUrl: string;
    apiUrl: string;
}

class GhostBookmarks {
    private api: BookmarksAPI | null = null;
    private auth: MemberAuth | null = null;
    private config: BookmarksConfig | null = null;

    async init(config: BookmarksConfig) {
        this.config = config;
        this.api = new BookmarksAPI(config);
        this.auth = new MemberAuth();

        // 检查会员登录状态
        const isAuthenticated = await this.auth.checkAuth();

        if (!isAuthenticated) {
            console.log('Ghost Bookmarks: User not authenticated');
            return;
        }

        // 自动挂载书签按钮
        this.autoMount();
    }

    private autoMount() {
        // 查找文章容器
        const articleSelectors = [
            'article[data-post-id]',
            '.post[data-post-id]',
            '.gh-article[data-post-id]'
        ];

        let articles: Element[] = [];
        for (const selector of articleSelectors) {
            articles = Array.from(document.querySelectorAll(selector));
            if (articles.length > 0) break;
        }

        if (articles.length === 0) {
            console.warn('Ghost Bookmarks: No articles found for auto-mount');
            return;
        }

        articles.forEach(article => {
            const postId = article.getAttribute('data-post-id');
            if (!postId) return;

            // 查找插入位置
            const insertTarget = this.findInsertTarget(article);
            if (!insertTarget) return;

            // 创建容器
            const container = document.createElement('div');
            container.className = 'ghost-bookmark-container';
            insertTarget.appendChild(container);

            // 挂载 React 组件
            const root = createRoot(container);
            root.render(
                React.createElement(App, {
                    postId,
                    api: this.api!,
                    showLabel: true,
                    size: 'medium'
                })
            );
        });
    }

    private findInsertTarget(article: Element): Element | null {
        // 尝试多个可能的插入位置
        const selectors = [
            '.post-meta',
            '.article-meta',
            '.gh-article-meta',
            'header'
        ];

        for (const selector of selectors) {
            const target = article.querySelector(selector);
            if (target) return target;
        }

        // 如果没有找到,返回文章本身
        return article;
    }

    // 手动挂载方法(供主题开发者使用)
    mount(containerId: string, postId: string, options = {}) {
        const container = document.getElementById(containerId);
        if (!container || !this.api) return;

        const root = createRoot(container);
        root.render(
            React.createElement(App, {
                postId,
                api: this.api,
                showLabel: true,
                size: 'medium',
                ...options
            })
        );
    }
}

// 导出到全局
if (typeof window !== 'undefined') {
    (window as any).GhostBookmarks = new GhostBookmarks();
}

export default GhostBookmarks;
```

## 主题数据属性约定

主题需要在文章元素上添加 `data-post-id` 属性:

```handlebars
{{!-- 主题模板示例 --}}
<article class="post" data-post-id="{{id}}">
    <header class="post-header">
        <h1>{{title}}</h1>
        <div class="post-meta">
            {{!-- 书签按钮会自动插入到这里 --}}
        </div>
    </header>
    <div class="post-content">
        {{content}}
    </div>
</article>
```

## 下一步

继续生成 Portal 集成方案。
