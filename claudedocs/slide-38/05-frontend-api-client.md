# 前端 API 客户端

## API 客户端实现

**文件**: `apps/bookmarks-ui/src/utils/api.ts`

```typescript
interface BookmarkConfig {
    siteUrl: string;
    apiUrl: string;
}

interface Bookmark {
    id: string;
    post_id: string;
    created_at: string;
    post: {
        id: string;
        title: string;
        slug: string;
        excerpt: string;
        feature_image: string | null;
        url: string;
    };
}

interface BookmarksResponse {
    bookmarks: Bookmark[];
    meta: {
        pagination: {
            page: number;
            limit: number;
            pages: number;
            total: number;
        };
    };
}

class BookmarksAPI {
    private config: BookmarkConfig;

    constructor(config: BookmarkConfig) {
        this.config = config;
    }

    private async request<T>(
        endpoint: string,
        options: RequestInit = {}
    ): Promise<T> {
        const url = `${this.config.apiUrl}${endpoint}`;

        const response = await fetch(url, {
            ...options,
            credentials: 'include', // 发送会员 cookie
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            }
        });

        if (!response.ok) {
            const error = await response.json().catch(() => ({}));
            throw new Error(error.errors?.[0]?.message || 'Request failed');
        }

        return response.json();
    }

    async addBookmark(postId: string): Promise<Bookmark> {
        const response = await this.request<{bookmark: Bookmark}>(
            '/bookmarks/',
            {
                method: 'POST',
                body: JSON.stringify({post_id: postId})
            }
        );
        return response.bookmark;
    }

    async removeBookmark(postId: string): Promise<void> {
        await this.request(`/bookmarks/${postId}`, {
            method: 'DELETE'
        });
    }

    async getBookmarks(page = 1, limit = 15): Promise<BookmarksResponse> {
        return this.request<BookmarksResponse>(
            `/bookmarks/?page=${page}&limit=${limit}`
        );
    }

    async checkBookmarked(postId: string): Promise<{
        bookmarked: boolean;
        bookmark_id?: string;
    }> {
        return this.request(`/posts/${postId}/bookmarked`);
    }
}

export {BookmarksAPI, type Bookmark, type BookmarksResponse};
```

## 会员认证工具

**文件**: `apps/bookmarks-ui/src/utils/auth.ts`

```typescript
interface Member {
    id: string;
    email: string;
    name: string;
    avatar_image: string | null;
}

class MemberAuth {
    private member: Member | null = null;

    async checkAuth(): Promise<boolean> {
        try {
            // 使用 Ghost Members API 检查会员状态
            const response = await fetch('/members/api/member/', {
                credentials: 'include'
            });

            if (response.ok) {
                this.member = await response.json();
                return true;
            }

            return false;
        } catch (error) {
            console.error('Auth check failed:', error);
            return false;
        }
    }

    getMember(): Member | null {
        return this.member;
    }

    isAuthenticated(): boolean {
        return this.member !== null;
    }
}

export {MemberAuth, type Member};
```

## 下一步

继续生成 React Hook 和组件实现。
