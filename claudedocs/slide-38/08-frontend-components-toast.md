# Toast 提示组件

## Toast 组件实现

**文件**: `apps/bookmarks-ui/src/components/Toast.tsx`

```typescript
import React, {useEffect, useState} from 'react';

interface ToastProps {
    message: string;
    type: 'success' | 'error';
    duration?: number;
}

export const Toast: React.FC<ToastProps> = ({
    message,
    type,
    duration = 3000
}) => {
    const [visible, setVisible] = useState(true);

    useEffect(() => {
        const timer = setTimeout(() => {
            setVisible(false);
        }, duration);

        return () => clearTimeout(timer);
    }, [duration]);

    if (!visible) return null;

    const bgColor = type === 'success' ? '#10b981' : '#ef4444';

    return (
        <div
            className="ghost-bookmark-toast"
            style={{
                position: 'fixed',
                bottom: '24px',
                right: '24px',
                padding: '12px 20px',
                background: bgColor,
                color: 'white',
                borderRadius: '8px',
                boxShadow: '0 4px 12px rgba(0, 0, 0, 0.15)',
                fontSize: '14px',
                fontWeight: '500',
                zIndex: 9999,
                animation: 'slideIn 0.3s ease-out'
            }}
        >
            {message}
        </div>
    );
};
```

## Toast 动画样式

**文件**: `apps/bookmarks-ui/public/bookmarks.css`

```css
@keyframes slideIn {
    from {
        transform: translateY(100%);
        opacity: 0;
    }
    to {
        transform: translateY(0);
        opacity: 1;
    }
}

.ghost-bookmark-button:hover {
    border-color: #d1d5db;
    background: #f9fafb;
}

.ghost-bookmark-button:active {
    transform: scale(0.98);
}
```
