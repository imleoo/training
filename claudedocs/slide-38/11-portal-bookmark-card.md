# 书签卡片组件

## 书签卡片实现

**文件**: `apps/portal/src/components/BookmarkCard.js`

```javascript
import React, {useState} from 'react';
import {useBookmark} from '../hooks/useBookmark';

const BookmarkCard = ({bookmark}) => {
    const [isRemoving, setIsRemoving] = useState(false);
    const {toggleBookmark} = useBookmark({
        postId: bookmark.post_id,
        onSuccess: (bookmarked) => {
            if (!bookmarked) {
                setIsRemoving(true);
                // 延迟移除以显示动画
                setTimeout(() => {
                    // 组件会被 React Query 自动移除
                }, 300);
            }
        }
    });

    const formatDate = (dateString) => {
        const date = new Date(dateString);
        const now = new Date();
        const diffDays = Math.floor((now - date) / (1000 * 60 * 60 * 24));

        if (diffDays === 0) return '今天';
        if (diffDays === 1) return '昨天';
        if (diffDays < 7) return `${diffDays} 天前`;
        return date.toLocaleDateString('zh-CN');
    };

    return (
        <article
            className={`bookmark-card ${isRemoving ? 'removing' : ''}`}
            style={{
                opacity: isRemoving ? 0 : 1,
                transform: isRemoving ? 'scale(0.95)' : 'scale(1)',
                transition: 'all 0.3s ease'
            }}
        >
            {bookmark.post.feature_image && (
                <a href={bookmark.post.url} className="bookmark-image">
                    <img
                        src={bookmark.post.feature_image}
                        alt={bookmark.post.title}
                        loading="lazy"
                    />
                </a>
            )}

            <div className="bookmark-content">
                <h3 className="bookmark-title">
                    <a href={bookmark.post.url}>
                        {bookmark.post.title}
                    </a>
                </h3>

                {bookmark.post.excerpt && (
                    <p className="bookmark-excerpt">
                        {bookmark.post.excerpt}
                    </p>
                )}

                <div className="bookmark-meta">
                    <span className="bookmark-date">
                        保存于 {formatDate(bookmark.created_at)}
                    </span>

                    <button
                        onClick={toggleBookmark}
                        className="bookmark-remove"
                        aria-label="移除书签"
                    >
                        移除
                    </button>
                </div>
            </div>
        </article>
    );
};

export default BookmarkCard;
```

## 书签卡片样式

**文件**: `apps/portal/src/styles/bookmarks.css`

```css
.bookmarks-page {
    max-width: 1200px;
    margin: 0 auto;
    padding: 40px 20px;
}

.bookmarks-header {
    margin-bottom: 32px;
}

.bookmarks-header h1 {
    font-size: 32px;
    font-weight: 700;
    margin-bottom: 8px;
}

.bookmarks-count {
    color: #6b7280;
    font-size: 14px;
}

.bookmarks-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 24px;
    margin-bottom: 32px;
}

.bookmark-card {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    overflow: hidden;
    transition: all 0.2s ease;
}

.bookmark-card:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    transform: translateY(-2px);
}

.bookmark-card.removing {
    pointer-events: none;
}

.bookmark-image {
    display: block;
    aspect-ratio: 16 / 9;
    overflow: hidden;
}

.bookmark-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 0.3s ease;
}

.bookmark-card:hover .bookmark-image img {
    transform: scale(1.05);
}

.bookmark-content {
    padding: 20px;
}

.bookmark-title {
    font-size: 18px;
    font-weight: 600;
    margin-bottom: 8px;
    line-height: 1.4;
}

.bookmark-title a {
    color: #1f2937;
    text-decoration: none;
}

.bookmark-title a:hover {
    color: #3b82f6;
}

.bookmark-excerpt {
    color: #6b7280;
    font-size: 14px;
    line-height: 1.6;
    margin-bottom: 16px;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
}

.bookmark-meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-top: 16px;
    border-top: 1px solid #f3f4f6;
}

.bookmark-date {
    color: #9ca3af;
    font-size: 13px;
}

.bookmark-remove {
    color: #ef4444;
    font-size: 13px;
    font-weight: 500;
    background: none;
    border: none;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 4px;
    transition: background 0.2s ease;
}

.bookmark-remove:hover {
    background: #fee2e2;
}

.load-more-button {
    display: block;
    margin: 0 auto;
    padding: 12px 32px;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.2s ease;
}

.load-more-button:hover {
    background: #2563eb;
}

.load-more-button:disabled {
    background: #9ca3af;
    cursor: not-allowed;
}
```
