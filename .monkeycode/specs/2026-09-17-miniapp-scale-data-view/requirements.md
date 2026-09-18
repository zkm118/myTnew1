# Requirements Document

## Introduction

终端用户在小程序中完成体脂秤绑定后，需要在小程序内查看该秤产生的测量与使用数据。本功能覆盖「已绑定体脂秤」场景下的数据查询、展示与权限隔离，复用平台已有的设备绑定关系与遥测数据链路。

## Glossary

- **终端用户**：通过账号密码登录小程序的 `user` 角色用户。
- **体脂秤**：接入平台的消费级体重 / 体成分测量设备，产品类型为体脂秤。
- **绑定关系**：终端用户与某台设备之间的有效归属记录，由设备接入模块维护。
- **测量记录**：一次称重产生的遥测数据集合，至少包含体重，可包含体脂率、肌肉量、水分率等物模型属性。
- **核心指标**：详情页默认展示的四项属性：体重 `weight`、体脂率 `body_fat`、肌肉量 `muscle_mass`、水分率 `body_water`。
- **小程序**：uni-app 编译的微信小程序端。
- **系统**：link_equipment 平台，含 FastAPI 后端与小程序客户端。

## Requirements

### Requirement 1: 已绑定设备入口

**User Story:** AS 终端用户, I want 在小程序中看到自己已绑定的体脂秤, so that 我能进入该设备的数据页面。

#### Acceptance Criteria

1. WHEN 终端用户完成登录且存在至少一条有效绑定关系, THE 系统 SHALL 在小程序设备列表中展示该用户已绑定的体脂秤，包含设备名称与在线状态。
2. WHEN 终端用户点击列表中的一台已绑定体脂秤, THE 系统 SHALL 进入该设备的数据详情页。
3. IF 终端用户当前没有有效绑定关系, THE 系统 SHALL 展示空状态并提供「添加设备」入口。

### Requirement 2: 最近一次测量展示

**User Story:** AS 终端用户, I want 打开体脂秤详情后立刻看到最近一次测量结果, so that 我能快速了解当前身体数据。

#### Acceptance Criteria

1. WHEN 终端用户打开已绑定体脂秤的数据详情页, THE 系统 SHALL 展示该设备最近一次测量记录的时间与物模型属性值。
2. WHILE 最近一次测量记录存在, THE 系统 SHALL 展示核心指标中该次测量已上报的属性值，未上报的核心指标展示为「--」。
3. IF 该设备尚无任何测量记录, THE 系统 SHALL 展示「暂无测量数据」提示。

### Requirement 3: 历史测量记录查询

**User Story:** AS 终端用户, I want 按时间查看体脂秤的历史测量, so that 我能追踪自己的使用与身体变化。

#### Acceptance Criteria

1. WHEN 终端用户进入数据详情页且未更改时间范围, THE 系统 SHALL 以近 30 天作为默认查询区间。
2. WHEN 终端用户在数据详情页选择时间范围, THE 系统 SHALL 返回该范围内、属于该绑定设备的全部测量记录列表，按测量时间倒序排列。
3. WHEN 历史记录超过单页容量, THE 系统 SHALL 以分页方式返回，默认每页 20 条。
4. WHILE 终端用户查看历史记录, THE 系统 SHALL 展示每条记录的测量时间与核心指标值。
5. IF 所选时间范围内没有测量记录, THE 系统 SHALL 展示「该时间段暂无数据」提示。

### Requirement 4: 趋势图

**User Story:** AS 终端用户, I want 看到体重与体脂率随时间变化的曲线, so that 我能直观观察趋势。

#### Acceptance Criteria

1. WHEN 终端用户打开数据详情页且所选时间范围内存在不少于 2 条测量记录, THE 系统 SHALL 展示体重趋势图。
2. WHEN 所选时间范围内存在体脂率数据且不少于 2 条, THE 系统 SHALL 展示体脂率趋势图。
3. IF 所选时间范围内可绘图的数据点少于 2 个, THE 系统 SHALL 隐藏对应趋势图并保留列表展示。

### Requirement 5: 数据归属与权限

**User Story:** AS 终端用户, I want 只能看到自己绑定设备的数据, so that 我的健康数据不被他人查看。

#### Acceptance Criteria

1. WHEN 终端用户请求某台设备的测量数据, THE 系统 SHALL 校验当前登录用户与该设备存在有效绑定关系后，返回该设备上的全部测量记录。
2. IF 当前登录用户与目标设备不存在有效绑定关系, THE 系统 SHALL 返回 403 并拒绝返回任何测量数据。
3. IF 访问令牌缺失或失效, THE 系统 SHALL 返回 401 并引导小程序重新登录。

### Requirement 6: 数据时效

**User Story:** AS 终端用户, I want 称重完成后尽快在小程序看到新数据, so that 我确认本次测量已同步。

#### Acceptance Criteria

1. WHEN 已绑定体脂秤成功上报一次测量记录, THE 系统 SHALL 在 10 秒内使该记录可通过小程序数据接口查询。
2. WHEN 终端用户在数据详情页执行下拉刷新, THE 系统 SHALL 重新拉取最近一次测量记录与当前时间范围的历史列表。
