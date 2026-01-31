# React Hooks 实现

## 书签操作 Hook

**文件**: `apps/bookmarks-ui/src/hooks/useBookmark.ts`

```typescript
import {useMutation, useQuery, useQueryClient} from '@tanstack/react-query';
import {BookmarksAPI} from '../utils/api';

interface UseBookmarkOptions {
    postId: string;
    api: BookmarksAPI;
    onSuccess?: (bookmarked: boolean) => void;
    onError?: (error: Error) => void;
}

export function useBookmark({postId, api, onSuccess, onError}: UseBookmarkOptions) {
    const queryClient = useQueryClient();

    // 查询书签状态
    const {data: bookmarkStatus, isLoading} = useQuery({
        queryKey: ['bookmark', postId],
        queryFn: () => api.checkBookmarked(postId),
        staleTime: 5 * 60 * 1000 // 5分钟缓存
    });

    // 添加书签
    const addMutation = useMutation({
        mutationFn: () => api.addBookmark(postId),
        onSuccess: () => {
            queryClient.setQueryData(['bookmark', postId], {
                bookmarked: true
            });
            queryClient.invalidateQueries({queryKey: ['bookmarks']});
            onSuccess?.(true);
        },
        onError: (error: Error) => {
            onError?.(error);
        }
    });

    // 移除书签
    const removeMutation = useMutation({
        mutationFn: () => api.removeBookmark(postId),
        onSuccess: () => {
            queryClient.setQueryData(['bookmark', postId], {
                bookmarked: false
            });
            queryClient.invalidateQueries({queryKey: ['bookmarks']});
            onSuccess?.(false);
        },
        onError: (error: Error) => {
            onError?.(error);
        }
    });

    const toggleBookmark = () => {
        if (bookmarkStatus?.bookmarked) {
            removeMutation.mutate();
        } else {
            addMutation.mutate();
        }
    };

    return {
        isBookmarked: bookmarkStatus?.bookmarked ?? false,
        isLoading: isLoading || addMutation.isPending || removeMutation.isPending,
        toggleBookmark,
        error: addMutation.error || removeMutation.error
    };
}
```

## 书签列表 Hook

**文件**: `apps/bookmarks-ui/src/hooks/useBookmarksList.ts`

```typescript
import {useInfiniteQuery} from '@tanstack/react-query';
import {BookmarksAPI} from '../utils/api';

interface UseBookmarksListOptions {
    api: BookmarksAPI;
    limit?: number;
}

export function useBookmarksList({api, limit = 15}: UseBookmarksListOptions) {
    const {
        data,
        fetchNextPage,
        hasNextPage,
        isFetchingNextPage,
        isLoading,
        error
    } = useInfiniteQuery({
        queryKey: ['bookmarks'],
        queryFn: ({pageParam = 1}) => api.getBookmarks(pageParam, limit),
        getNextPageParam: (lastPage) => {
            const {page, pages} = lastPage.meta.pagination;
            return page < pages ? page + 1 : undefined;
        },
        staleTime: 2 * 60 * 1000 // 2分钟缓存
    });

    const bookmarks = data?.pages.flatMap(page => page.bookmarks) ?? [];
    const totalCount = data?.pages[0]?.meta.pagination.total ?? 0;

    return {
        bookmarks,
        totalCount,
        isLoading,
        isFetchingNextPage,
        hasNextPage,
        fetchNextPage,
        error
    };
}
```

## 下一步

继续生成 UI 组件实现。
