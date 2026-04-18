# Ghost 与视频字幕服务集成方案

## 项目概况

### Ghost 项目
- **类型**: Node.js 内容发布平台
- **架构**: Monorepo (Yarn + Nx)
- **技术栈**: Node.js 22, Express, React 18.3, MySQL 8.4.5
- **位置**: `/Users/leoobai/leoobai/training-project/training`

### 视频字幕服务
- **类型**: Go 视频字幕翻译系统
- **架构**: 微服务 (API + Worker)
- **技术栈**: Go 1.25, Gin, PostgreSQL 16, Redis 7
- **位置**: `/Users/leoobai/jiwu-project/translate-video/subtitle-service`

---

## 集成方案

### 方案 1: Ghost 插件集成（推荐）

**适用场景**: 在 Ghost 中嵌入视频字幕功能

#### 架构设计
```
Ghost Admin (React)
    ↓
Ghost Plugin/Integration
    ↓ HTTP API
视频字幕服务 (Go)
```

#### 实施步骤

**1. 创建 Ghost 集成应用**
```
apps/video-subtitle/
├── src/
│   ├── main.tsx              # React 入口
│   ├── api/
│   │   └── subtitle.ts       # 调用 Go 服务 API
│   ├── components/
│   │   ├── VideoUpload.tsx   # 视频上传
│   │   ├── TaskList.tsx      # 任务列表
│   │   └── SubtitleEditor.tsx # 字幕编辑
│   └── hooks/
│       └── useSubtitle.ts    # API hooks
```

**2. Go 服务提供 REST API**
```go
// 已有接口
POST   /api/v1/auth/register
POST   /api/v1/auth/login
GET    /api/v1/tasks
POST   /api/v1/tasks
GET    /api/v1/tasks/:id
```

**3. Ghost 配置代理**
```javascript
// ghost/core/core/server/web/parent/app.js
app.use('/api/subtitle', proxy({
  target: 'http://localhost:8080',
  changeOrigin: true
}));
```

#### 优点
- ✅ 无缝集成到 Ghost Admin
- ✅ 统一认证体系
- ✅ 复用 Ghost 设计系统
- ✅ 独立部署 Go 服务

#### 缺点
- ⚠️ 需要开发 React 前端
- ⚠️ 跨语言认证同步

---

### 方案 2: 独立服务 + Webhook

**适用场景**: 两个系统独立运行，通过事件通信

#### 架构设计
```
Ghost CMS
    ↓ Webhook
视频字幕服务
    ↓ Callback
Ghost CMS (更新内容)
```

#### 实施步骤

**1. Ghost 发送 Webhook**
```javascript
// ghost/core/core/server/services/webhooks/
// 视频上传后触发
webhooks.trigger('video.uploaded', {
  videoUrl: url,
  postId: post.id
});
```

**2. Go 服务接收 Webhook**
```go
// internal/handler/webhook.go
func (h *Handler) HandleGhostWebhook(c *gin.Context) {
    // 接收 Ghost 事件
    // 创建字幕任务
    // 处理完成后回调 Ghost
}
```

#### 优点
- ✅ 完全解耦
- ✅ 各自独立部署
- ✅ 易于扩展

#### 缺点
- ⚠️ 需要实现 Webhook 机制
- ⚠️ 异步处理复杂度高

---

### 方案 3: Ghost 自定义 API 端点

**适用场景**: 在 Ghost 中直接调用 Go 服务

#### 架构设计
```
Ghost Admin UI
    ↓
Ghost Custom API Endpoint
    ↓ HTTP Client
Go 视频字幕服务
```

#### 实施步骤

**1. 创建 Ghost API 端点**
```javascript
// ghost/core/core/server/api/endpoints/subtitle.js
module.exports = {
    browse: {
        permissions: true,
        async query(frame) {
            // 调用 Go 服务
            const response = await fetch('http://localhost:8080/api/v1/tasks');
            return response.json();
        }
    }
};
```

**2. 注册端点**
```javascript
// ghost/core/core/server/api/endpoints.js
subtitle: require('./subtitle')
```

#### 优点
- ✅ 实现简单
- ✅ 复用 Ghost 权限系统

#### 缺点
- ⚠️ Ghost 成为代理层
- ⚠️ 增加 Ghost 复杂度

---

## 推荐方案对比

| 维度 | 方案1: 插件集成 | 方案2: Webhook | 方案3: API端点 |
|------|----------------|---------------|---------------|
| 开发复杂度 | 中 | 高 | 低 |
| 用户体验 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 系统解耦 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| 维护成本 | 中 | 低 | 中 |
| 扩展性 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

**推荐**: 方案 1（插件集成）- 最佳用户体验和功能完整性

---

## 详细实施：方案 1

### 第一阶段：基础集成

#### 1. 创建 React 应用

```bash
cd apps/
mkdir video-subtitle
cd video-subtitle
```

**package.json**
```json
{
  "name": "@tryghost/video-subtitle",
  "version": "0.0.1",
  "dependencies": {
    "react": "^18.3.0",
    "@tanstack/react-query": "^5.0.0",
    "@tryghost/admin-x-framework": "workspace:*"
  }
}
```

#### 2. API 客户端

**src/api/client.ts**
```typescript
const API_BASE = '/api/subtitle/v1';

export const subtitleApi = {
  getTasks: () => fetch(`${API_BASE}/tasks`).then(r => r.json()),
  createTask: (data) => fetch(`${API_BASE}/tasks`, {
    method: 'POST',
    body: JSON.stringify(data)
  }).then(r => r.json())
};
```

#### 3. React 组件

**src/components/VideoUpload.tsx**
```typescript
import { useMutation } from '@tanstack/react-query';
import { subtitleApi } from '../api/client';

export const VideoUpload = () => {
  const upload = useMutation({
    mutationFn: subtitleApi.createTask
  });

  return (
    <div>
      <input type="file" accept="video/*" />
      <button onClick={() => upload.mutate()}>上传</button>
    </div>
  );
};
```

### 第二阶段：认证集成

#### Ghost 端

**ghost/core/core/server/api/endpoints/subtitle-auth.js**
```javascript
const jwt = require('jsonwebtoken');

module.exports = {
    generateToken: {
        permissions: true,
        async query(frame) {
            const user = frame.user;
            // 生成 Go 服务可验证的 token
            const token = jwt.sign(
                { userId: user.id, email: user.email },
                process.env.SUBTITLE_JWT_SECRET,
                { expiresIn: '24h' }
            );
            return { token };
        }
    }
};
```

#### Go 端

**internal/middleware/ghost_auth.go**
```go
func GhostAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        // 验证 Ghost 签发的 JWT
        claims, err := validateGhostToken(token)
        if err != nil {
            c.JSON(401, gin.H{"error": "unauthorized"})
            c.Abort()
            return
        }
        c.Set("userId", claims.UserID)
        c.Next()
    }
}
```

### 第三阶段：数据同步

#### 选项 A: 共享数据库（不推荐）
- Ghost 和 Go 服务访问同一数据库
- 复杂度高，耦合严重

#### 选项 B: API 同步（推荐）
```javascript
// Ghost 保存视频元数据
// Go 服务处理字幕
// 通过 API 互相查询
```

---

## 部署架构

### 开发环境
```yaml
# docker-compose.yml
services:
  ghost:
    image: ghost:latest
    ports: ["2368:2368"]

  subtitle-api:
    build: ./subtitle-service
    ports: ["8080:8080"]

  subtitle-worker:
    build: ./subtitle-service
    command: worker

  postgres:
    image: postgres:16

  redis:
    image: redis:7
```

### 生产环境
```
[Nginx/Caddy]
    ↓
[Ghost] ← → [Go API] ← → [Go Worker]
    ↓           ↓              ↓
[MySQL]    [PostgreSQL]    [Redis]
```

---

## 快速开始

### 1. 启动 Go 服务
```bash
cd /Users/leoobai/jiwu-project/translate-video/subtitle-service
docker-compose up -d
go run cmd/api/main.go
```

### 2. 配置 Ghost 代理
```javascript
// ghost/core/config.development.json
{
  "subtitle": {
    "apiUrl": "http://localhost:8080",
    "jwtSecret": "shared-secret"
  }
}
```

### 3. 创建 React 应用
```bash
cd /Users/leoobai/leoobai/training-project/training/apps
# 创建 video-subtitle 应用
```

---

## 关键技术点

### 1. 跨域处理
```javascript
// Ghost CORS 配置
app.use(cors({
  origin: ['http://localhost:2368'],
  credentials: true
}));
```

### 2. 文件上传
- Ghost: 使用现有的媒体上传
- Go: tus 协议大文件上传

### 3. 实时进度
- Go: SSE (Server-Sent Events)
- React: EventSource API

---

## 下一步行动

### 立即可做
1. ✅ 在 Ghost `apps/` 创建 `video-subtitle` 应用
2. ✅ 配置 Ghost API 代理到 Go 服务
3. ✅ 实现基础的任务列表 UI

### 短期目标
1. 🔄 统一认证机制
2. 🔄 实现视频上传流程
3. 🔄 集成字幕编辑器

### 长期规划
1. 📋 多语言字幕支持
2. 📋 AI 翻译优化
3. 📋 批量处理功能

---

## 参考资料

- Ghost Admin Apps: `apps/admin-x-settings/`
- Ghost API: `ghost/core/core/server/api/`
- Go 服务: `/Users/leoobai/jiwu-project/translate-video/subtitle-service`
