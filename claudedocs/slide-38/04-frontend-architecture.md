# 前端应用架构

## 应用结构

新建 React 应用: `apps/bookmarks-ui`

```
apps/bookmarks-ui/
├── src/
│   ├── components/
│   │   ├── BookmarkButton.tsx      # 书签按钮组件
│   │   ├── BookmarkIcon.tsx        # 图标组件
│   │   └── Toast.tsx               # 提示消息
│   ├── hooks/
│   │   └── useBookmark.ts          # 书签操作 Hook
│   ├── utils/
│   │   ├── api.ts                  # API 客户端
│   │   └── auth.ts                 # 会员认证
│   ├── App.tsx                     # 应用入口
│   └── index.ts                    # UMD 导出
├── public/
│   └── bookmarks.css               # 默认样式
├── package.json
├── vite.config.ts                  # Vite 配置
└── tsconfig.json
```

## 技术栈

- **框架**: React 18
- **构建**: Vite (UMD 输出)
- **状态管理**: React Query (数据缓存)
- **样式**: CSS Modules + 内联样式
- **类型**: TypeScript

## 构建配置

**文件**: `apps/bookmarks-ui/vite.config.ts`

```typescript
import {defineConfig} from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    build: {
        lib: {
            entry: 'src/index.ts',
            name: 'GhostBookmarks',
            formats: ['umd'],
            fileName: () => 'bookmarks.min.js'
        },
        rollupOptions: {
            external: ['react', 'react-dom'],
            output: {
                globals: {
                    react: 'React',
                    'react-dom': 'ReactDOM'
                }
            }
        },
        outDir: 'umd',
        minify: 'terser'
    }
});
```

## 依赖管理

**文件**: `apps/bookmarks-ui/package.json`

```json
{
    "name": "@tryghost/bookmarks-ui",
    "version": "0.0.1",
    "private": true,
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview"
    },
    "dependencies": {
        "react": "^18.2.0",
        "react-dom": "^18.2.0",
        "@tanstack/react-query": "^5.0.0"
    },
    "devDependencies": {
        "@vitejs/plugin-react": "^4.0.0",
        "typescript": "^5.0.0",
        "vite": "^5.0.0"
    }
}
```

## 自动注入机制

书签 UI 通过 `{{ghost_head}}` 自动注入到主题:

**文件**: `ghost/core/core/frontend/helpers/ghost_head.js`

```javascript
// 在现有 ghost_head helper 中添加
function getBookmarksScript() {
    const bookmarksEnabled = settingsCache.get('members_enabled');

    if (!bookmarksEnabled) {
        return '';
    }

    const scriptUrl = urlUtils.urlFor({
        relativeUrl: '/public/bookmarks.min.js',
        secure: true
    }, true);

    const styleUrl = urlUtils.urlFor({
        relativeUrl: '/public/bookmarks.css',
        secure: true
    }, true);

    return `
        <link rel="stylesheet" href="${styleUrl}">
        <script src="${scriptUrl}" defer></script>
        <script>
            window.addEventListener('DOMContentLoaded', function() {
                if (window.GhostBookmarks) {
                    window.GhostBookmarks.init({
                        siteUrl: '${urlUtils.getSiteUrl()}',
                        apiUrl: '${urlUtils.urlFor('api', {version: 'v4', versionType: 'members'}, true)}'
                    });
                }
            });
        </script>
    `;
}
```

## 初始化流程

1. **页面加载**: `{{ghost_head}}` 注入脚本和样式
2. **DOM 就绪**: `DOMContentLoaded` 触发初始化
3. **自动挂载**: 扫描页面中的文章元素,自动插入书签按钮
4. **会员检测**: 检查用户登录状态,未登录则隐藏按钮

## 下一步

继续生成核心组件实现?
