# 书签按钮组件

## 书签图标组件

**文件**: `apps/bookmarks-ui/src/components/BookmarkIcon.tsx`

```typescript
import React from 'react';

interface BookmarkIconProps {
    filled: boolean;
    size?: number;
}

export const BookmarkIcon: React.FC<BookmarkIconProps> = ({
    filled,
    size = 20
}) => {
    return (
        <svg
            width={size}
            height={size}
            viewBox="0 0 24 24"
            fill={filled ? 'currentColor' : 'none'}
            stroke="currentColor"
            strokeWidth="2"
            strokeLinecap="round"
            strokeLinejoin="round"
        >
            <path d="M19 21l-7-5-7 5V5a2 2 0 0 1 2-2h10a2 2 0 0 1 2 2z" />
        </svg>
    );
};
```

## 书签按钮组件

**文件**: `apps/bookmarks-ui/src/components/BookmarkButton.tsx`

```typescript
import React, {useState} from 'react';
import {useBookmark} from '../hooks/useBookmark';
import {BookmarkIcon} from './BookmarkIcon';
import {Toast} from './Toast';
import {BookmarksAPI} from '../utils/api';

interface BookmarkButtonProps {
    postId: string;
    api: BookmarksAPI;
    showLabel?: boolean;
    size?: 'small' | 'medium' | 'large';
}

export const BookmarkButton: React.FC<BookmarkButtonProps> = ({
    postId,
    api,
    showLabel = true,
    size = 'medium'
}) => {
    const [toast, setToast] = useState<{
        message: string;
        type: 'success' | 'error';
    } | null>(null);

    const {isBookmarked, isLoading, toggleBookmark} = useBookmark({
        postId,
        api,
        onSuccess: (bookmarked) => {
            setToast({
                message: bookmarked ? '已保存到书签' : '已从书签移除',
                type: 'success'
            });
            setTimeout(() => setToast(null), 3000);
        },
        onError: (error) => {
            setToast({
                message: error.message || '操作失败,请重试',
                type: 'error'
            });
            setTimeout(() => setToast(null), 3000);
        }
    });

    const sizeMap = {
        small: {icon: 16, padding: '6px 10px', fontSize: '13px'},
        medium: {icon: 20, padding: '8px 14px', fontSize: '14px'},
        large: {icon: 24, padding: '10px 18px', fontSize: '16px'}
    };

    const currentSize = sizeMap[size];

    return (
        <>
            <button
                onClick={toggleBookmark}
                disabled={isLoading}
                className="ghost-bookmark-button"
                style={{
                    display: 'inline-flex',
                    alignItems: 'center',
                    gap: '6px',
                    padding: currentSize.padding,
                    fontSize: currentSize.fontSize,
                    border: '1px solid #e5e7eb',
                    borderRadius: '6px',
                    background: isBookmarked ? '#f3f4f6' : 'white',
                    color: isBookmarked ? '#1f2937' : '#6b7280',
                    cursor: isLoading ? 'not-allowed' : 'pointer',
                    transition: 'all 0.2s ease',
                    opacity: isLoading ? 0.6 : 1
                }}
                aria-label={isBookmarked ? '移除书签' : '保存到书签'}
            >
                <BookmarkIcon filled={isBookmarked} size={currentSize.icon} />
                {showLabel && (
                    <span>{isBookmarked ? '已保存' : '保存'}</span>
                )}
            </button>

            {toast && <Toast message={toast.message} type={toast.type} />}
        </>
    );
};
```

## 下一步

继续生成 Toast 组件和应用入口。
