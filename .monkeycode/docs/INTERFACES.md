# 接口定义

## 通用约定

- 基础路径：`/api/v1`
- 数据格式：`application/json; charset=utf-8`
- 认证方式：`Authorization: Bearer <JWT>`
- 时间格式：ISO 8601（如 `2026-09-16T10:00:00Z`）

统一响应结构：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

分页响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [],
    "total": 0,
    "page": 1,
    "page_size": 20
  }
}
```

错误码约定：

| code | 含义 |
|------|------|
| 0 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未认证或令牌失效 |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 409 | 资源冲突（如设备 ID 重复） |
| 422 | 物模型校验失败 |
| 500 | 服务内部错误 |

## 认证与用户

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/auth/login` | 账号密码登录，返回 JWT |
| POST | `/api/v1/auth/refresh` | 刷新访问令牌 |
| POST | `/api/v1/auth/logout` | 退出登录，失效当前令牌 |
| GET | `/api/v1/auth/me` | 获取当前登录用户信息与角色 |
| GET | `/api/v1/users` | 管理员查询用户列表 |
| POST | `/api/v1/users` | 管理员创建用户 |

登录请求 / 响应：

```json
{
  "username": "admin",
  "password": "<PASSWORD>"
}
```

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "access_token": "<JWT>",
    "refresh_token": "<JWT>",
    "expires_in": 7200,
    "role": "admin"
  }
}
```

角色约定：

| 角色 | 标识 | 权限范围 |
|------|------|---------|
| 平台管理员 | `admin` | 产品与物模型、设备台账、看板与用户管理 |
| 终端用户 | `user` | 配网、绑定设备、查看自己名下的设备与数据 |

## 产品与物模型

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/products` | 产品列表 |
| POST | `/api/v1/products` | 创建产品 |
| GET | `/api/v1/products/{product_id}` | 产品详情 |
| PUT | `/api/v1/products/{product_id}` | 更新产品 |
| DELETE | `/api/v1/products/{product_id}` | 删除产品（无关联设备时） |
| GET | `/api/v1/products/{product_id}/thing-model` | 查询物模型 |
| PUT | `/api/v1/products/{product_id}/thing-model` | 保存物模型（含版本号） |

物模型结构：

```json
{
  "version": "1.0.0",
  "properties": [
    {
      "identifier": "temperature",
      "name": "温度",
      "data_type": "float",
      "access_mode": "r",
      "unit": "℃",
      "min": -20,
      "max": 60
    }
  ],
  "events": [
    {
      "identifier": "low_battery",
      "name": "低电量告警",
      "type": "alert",
      "output": [{ "identifier": "battery", "data_type": "int" }]
    }
  ],
  "services": [
    {
      "identifier": "reboot",
      "name": "重启设备",
      "input": [],
      "output": []
    }
  ]
}
```

字段说明：

| 字段 | 说明 |
|------|------|
| `identifier` | 物模型标识符，产品内唯一，由字母、数字、下划线组成 |
| `data_type` | 支持 `int` / `float` / `bool` / `string` / `enum` / `struct` |
| `access_mode` | 属性读写权限：`r` 只读、`rw` 读写，`w` 只写 |
| `version` | 物模型版本号，采用语义化版本 |

## 设备管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/devices` | 设备列表，支持按产品、状态筛选 |
| POST | `/api/v1/devices` | 管理员登记设备（预注册） |
| GET | `/api/v1/devices/{device_id}` | 设备详情 |
| PUT | `/api/v1/devices/{device_id}` | 更新设备名称、备注等 |
| DELETE | `/api/v1/devices/{device_id}` | 删除设备 |
| POST | `/api/v1/devices/{device_id}/bind` | 终端用户绑定设备到家庭空间 |
| POST | `/api/v1/devices/{device_id}/unbind` | 解绑设备 |
| GET | `/api/v1/devices/{device_id}/status` | 查询设备在线状态（读 Redis） |

设备对象：

```json
{
  "device_id": "dev_0001",
  "product_id": "prod_1001",
  "device_name": "客厅温湿度计",
  "status": "online",
  "activated_at": "2026-09-16T10:00:00Z",
  "last_online_at": "2026-09-16T12:00:00Z",
  "owner_id": "usr_2001"
}
```

设备状态：

| 状态 | 说明 |
|------|------|
| `inactive` | 已登记未激活 |
| `online` | 已连接 MQTT |
| `offline` | 已激活但当前离线 |
| `disabled` | 被管理员禁用 |

## 配网

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/provisioning/sessions` | 创建配网会话，返回配网令牌 |
| GET | `/api/v1/provisioning/sessions/{session_id}` | 查询配网会话状态 |
| POST | `/api/v1/activation` | 设备侧激活接口，凭配网令牌换取设备凭证 |

创建配网会话：

```json
{
  "product_id": "prod_1001",
  "channel": "softap"
}
```

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "session_id": "ps_3001",
    "provision_token": "<PROVISION_TOKEN>",
    "expires_in": 300,
    "ssid_prefix": "LinkEquip-"
  }
}
```

设备激活：

```json
{
  "product_key": "<PRODUCT_KEY>",
  "device_sn": "<DEVICE_SN>",
  "provision_token": "<PROVISION_TOKEN>"
}
```

## 数据看板与报表

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/dashboard/overview` | 概览指标：设备总数、在线数、在线率、活跃数 |
| GET | `/api/v1/dashboard/trends` | 在线趋势 / 数据上报趋势 |
| GET | `/api/v1/devices/{device_id}/telemetry` | 查询设备历史遥测数据 |
| POST | `/api/v1/reports/device-summary` | 生成设备汇总报表 |
| GET | `/api/v1/reports/{report_id}/download` | 下载报表文件 |

历史遥测查询参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `identifier` | string | 物模型属性标识符，可多个 |
| `start` | string | 起始时间（ISO 8601） |
| `end` | string | 结束时间（ISO 8601） |
| `page` / `page_size` | int | 分页参数 |

遥测数据点：

```json
{
  "device_id": "dev_0001",
  "identifier": "temperature",
  "value": 24.5,
  "timestamp": "2026-09-16T12:00:00Z"
}
```

## MQTT 主题与消息

设备端使用设备凭证连接 Broker：`username = device_id`，`password = <DEVICE_SECRET>`。

| 方向 | 主题 | 说明 |
|------|------|------|
| 上行 | `device/{product_key}/{device_id}/property/post` | 属性上报 |
| 上行 | `device/{product_key}/{device_id}/event/post` | 事件上报 |
| 上行 | `device/{product_key}/{device_id}/reply` | 指令执行结果回执 |
| 下行 | `device/{product_key}/{device_id}/command` | 平台下发指令 |
| 状态 | `device/{product_key}/{device_id}/status` | 在线状态（LWT 遗嘱消息） |

属性上报消息体：

```json
{
  "id": "msg_0001",
  "timestamp": 1789000000,
  "params": [
    { "identifier": "temperature", "value": 24.5 }
  ]
}
```

指令下发消息体：

```json
{
  "id": "cmd_0001",
  "identifier": "reboot",
  "input": {},
  "timestamp": 1789000000
}
```

处理约定：

- 网关收到上行消息后按 `product_id` 加载物模型，校验标识符与取值范围，非法数据按 `422` 语义丢弃并记录日志。
- 上行消息使用 QoS 1，状态类消息使用 LWT 实现离线检测。
- `id` 用于上行与回执的关联，平台按该字段做去重。
