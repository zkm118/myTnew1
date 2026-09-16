# 开发指南

## 环境要求

| 工具 | 版本建议 | 用途 |
|------|---------|------|
| Python | 3.11+ | 后端服务与 MQTT 消息处理 |
| Node.js | 18+ | Web 管理后台与 uni-app 构建 |
| Docker / Docker Compose | 24+ / v2 | 一键拉起 MySQL、Redis、MQTT Broker |
| MySQL | 8.0 | 元数据与遥测归档 |
| Redis | 7.x | 在线状态与会话缓存 |
| MQTT Broker | EMQX 5 / Mosquitto 2 | 设备接入 |

## 快速开始

1. 拉起基础依赖（MySQL、Redis、MQTT Broker）：

```bash
# 在仓库根目录启动基础组件
docker compose -f deploy/docker-compose.yml up -d mysql redis mqtt
```

2. 启动后端服务：

```bash
# 安装依赖（建议使用虚拟环境）
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# 执行数据库迁移
alembic upgrade head

# 启动开发服务，默认监听 8000
uvicorn app.main:app --reload --port 8000
```

3. 启动 Web 管理后台：

```bash
# 安装依赖并启动，默认监听 5173
cd web
npm install
npm run dev
```

4. 启动移动端（uni-app）：

```bash
# 通过 HBuilderX 或 CLI 运行到小程序 / App
cd mobile
npm install
npm run dev:mp-weixin
```

5. 依赖就绪后刷新接口文档：后端启动后访问 `http://localhost:8000/docs` 查看 OpenAPI 文档。

## 项目结构说明

```text
backend/app
├── api/v1        # 路由层，只做参数校验与调用服务
├── core          # 配置、JWT、安全、日志等基础设施
├── models        # SQLAlchemy 模型，对应数据库表
├── schemas       # Pydantic 模型，对应接口契约
├── services      # 业务逻辑，可被路由与 MQTT 网关复用
├── mqtt          # Broker 连接、主题订阅与消息分发
└── db            # 数据库会话与初始化

web/src
├── api           # 接口封装，统一处理鉴权与错误
├── views         # 页面：产品、设备、看板
├── components    # 复用组件
├── router        # 路由与权限守卫
└── stores        # Pinia 状态

mobile/src
├── pages         # 登录、配网、设备列表、设备详情
└── api           # 移动端接口封装
```

## 开发规范

### 命名规范

- Python：模块与函数使用 `snake_case`，类使用 `PascalCase`，常量使用 `UPPER_SNAKE_CASE`。
- JavaScript / TypeScript：变量与函数使用 `camelCase`，组件使用 `PascalCase`，文件名与组件名保持一致。
- 数据库表与字段使用 `snake_case`，主键统一 `id`，外键统一 `<entity>_id`。

### 代码风格

- 后端使用 `ruff` 做格式化与静态检查，提交前必须通过。
- 前端遵循 Element Plus 项目约定，使用 ESLint + Prettier。
- 业务逻辑集中在 `services`，路由层与 MQTT 网关只做编排，避免重复实现。

### 提交规范

采用 Conventional Commits：

```text
feat(device): 支持设备配网会话创建
fix(mqtt): 修复遗嘱消息导致的误判离线
docs: 补充物模型字段说明
```

## 常见任务

### 新增一个产品与物模型

1. 通过管理后台或 `POST /api/v1/products` 创建产品，获取 `product_key`。
2. 调用 `PUT /api/v1/products/{product_id}/thing-model` 提交物模型。
3. 在设备端按物模型标识符上报数据，平台自动完成校验与落库。

### 调试 MQTT 上行链路

```bash
# 订阅某台设备的全部上行主题
mosquitto_sub -h localhost -p 1883 -u <device_id> -P <device_secret> \
  -t "device/<product_key>/<device_id>/#" -v
```

### 运行测试

```bash
# 后端单元测试与覆盖率
cd backend && pytest --cov=app

# 前端单元测试
cd web && npm run test
```

## 构建与发布

### 前端构建与反向代理

Web 管理后台在开发环境通过 Vite 反向代理访问后端，避免跨域：

```typescript
// web/vite.config.ts
export default defineConfig({
  server: {
    allowedHosts: ['.monkeycode-ai.online'],
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
})
```

### Docker Compose 部署

```bash
# 构建并启动全部组件
docker compose -f deploy/docker-compose.yml up -d --build

# 查看服务状态
docker compose -f deploy/docker-compose.yml ps

# 查看后端日志
docker compose -f deploy/docker-compose.yml logs -f backend
```

发布前检查清单：

- 后端 `ruff check` 与 `pytest` 全部通过。
- 前端 `npm run build` 成功，产物目录为 `web/dist`。
- 数据库迁移脚本已提交并可在空库上完整执行。
- 环境变量通过 `.env` 提供，仓库中只保留 `.env.example` 占位符。

## 环境变量

| 变量 | 说明 |
|------|------|
| `DB_HOST` / `DB_PORT` | MySQL 连接地址 |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | MySQL 库名与账号 |
| `REDIS_HOST` / `REDIS_PORT` | Redis 连接地址 |
| `MQTT_HOST` / `MQTT_PORT` | MQTT Broker 地址 |
| `JWT_SECRET` | JWT 签名密钥，由部署方自行提供 |
| `JWT_EXPIRE_SECONDS` | 访问令牌有效期 |
