# DMP 后台管理系统 API 接口文档

**版本：** v1.0.0  
**Base URL：** `http://localhost:8080/api/v1`  
**认证方式：** JWT Token（通过 Authorization Header 传递）

---

## 目录

1. [用户管理](#用户管理)
2. [权限管理](#权限管理)
3. [角色管理](#角色管理)
4. [广告主管理](#广告主管理)
5. [广告系列管理](#广告系列管理)
6. [人群包管理](#人群包管理)
7. [数据来源管理](#数据来源管理)

---

## 通用说明

### 请求头

| Header | 值 | 说明 |
|--------|-----|------|
| `Authorization` | `Bearer {token}` | JWT 认证令牌 |
| `Content-Type` | `application/json` | 请求体格式 |

### 响应格式

```json
{
  "code": 0,
  "message": "成功",
  "data": {},
  "page": {
    "page": 1,
    "page_size": 10,
    "total": 100
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | Integer | 错误码，0 表示成功 |
| `message` | String | 提示信息 |
| `data` | Object | 响应数据 |
| `page` | Object | 分页信息（列表接口使用） |

---

## 用户管理

### 1.1 用户登录

**接口：** `POST /api/v1/login`

**描述：** 使用用户名和密码登录，返回 JWT Token

**请求体：**

```json
{
  "username": "string",  // 必填
  "password": "string"   // 必填
}
```

**响应：**

```json
{
  "code": 0,
  "message": "登录成功",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": 1,
      "username": "admin",
      "real_name": "管理员",
      "email": "admin@example.com",
      "phone": "13800138000",
      "is_superuser": 1,
      "status": 1
    }
  }
}
```

---

### 1.2 获取当前用户信息

**接口：** `GET /api/v1/users/profile`

**描述：** 直接返回 JWT Auth 中间件注入的当前用户信息（无需查库）

**响应：** 同登录接口中的 `user` 对象

---

### 1.3 用户列表

**接口：** `GET /api/v1/users`

**描述：** 分页查询用户列表，支持用户名模糊搜索、状态和超级管理员筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `username` | String | - | 用户名（模糊搜索） |
| `status` | Integer | - | 状态: 1-Active 2-InActive |
| `is_superuser` | Integer | - | 超级管理员: 0-否 1-是 |

---

### 1.4 创建用户

**接口：** `POST /api/v1/users`

**描述：** 添加新用户，用户名不能重复。仅超级管理员可创建其他超级管理员

**请求体：**

```json
{
  "username": "string",     // 必填，3-32字符
  "password": "string",     // 必填，6-32字符
  "real_name": "string",   // 可选，真实姓名
  "email": "string",        // 可选，邮箱
  "phone": "string",        // 可选，手机号
  "is_superuser": 0,       // 可选，是否超级管理员
  "status": 1              // 可选，状态
}
```

---

### 1.5 获取用户详情

**接口：** `GET /api/v1/users/{id}`

**描述：** 根据用户 ID 获取用户信息

---

### 1.6 更新用户信息

**接口：** `PUT /api/v1/users/{id}`

**描述：** 更新用户的姓名、邮箱、手机号、状态。`is_superuser` 仅超级管理员可修改

---

### 1.7 修改密码

**接口：** `PUT /api/v1/users/{id}/password`

**描述：** 验证原密码后修改为新密码

**请求体：**

```json
{
  "old_password": "string",  // 必填
  "new_password": "string"   // 必填，6-32字符
}
```

---

### 1.8 删除用户

**接口：** `DELETE /api/v1/users/{id}`

**描述：** 软删除用户，不会物理删除数据

---

### 1.9 获取用户角色

**接口：** `GET /api/v1/users/{id}/roles`

**描述：** 获取指定用户已分配的角色列表（含完整角色信息）

---

### 1.10 分配用户角色

**接口：** `POST /api/v1/users/{id}/roles`

**描述：** 为指定用户批量分配角色（全量覆盖，传空数组可清空）

**业务规则：** 用户和角色必须均为 Active 状态

**请求体：**

```json
{
  "role_ids": [1, 2, 3]  // 角色ID数组
}
```

---

## 权限管理

### 2.1 权限列表

**接口：** `GET /api/v1/permissions`

**描述：** 分页查询权限列表，支持名称模糊搜索、编码模糊搜索、状态筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 权限名称（模糊搜索） |
| `code` | String | - | 权限编码（模糊搜索） |
| `status` | Integer | - | 状态: 1-Active 2-InActive |

---

### 2.2 创建权限

**接口：** `POST /api/v1/permissions`

**描述：** 添加新权限，权限编码不能重复

**请求体：**

```json
{
  "name": "string",     // 必填，权限名称
  "code": "string",     // 必填，权限编码（如 user:create）
  "description": "string",  // 可选，权限描述
  "status": 1              // 可选，状态
}
```

---

### 2.3 权限下拉选项

**接口：** `GET /api/v1/permissions/selector`

**描述：** 返回所有 Active 状态的权限，用于前端下拉选择

---

### 2.4 获取权限详情

**接口：** `GET /api/v1/permissions/{id}`

---

### 2.5 更新权限信息

**接口：** `PUT /api/v1/permissions/{id}`

---

### 2.6 删除权限

**接口：** `DELETE /api/v1/permissions/{id}`

**描述：** 物理删除权限，不可恢复

---

## 角色管理

### 3.1 角色列表

**接口：** `GET /api/v1/roles`

**描述：** 分页查询角色列表，支持名称模糊搜索、编码模糊搜索、状态筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 角色名称（模糊搜索） |
| `code` | String | - | 角色编码（模糊搜索） |
| `status` | Integer | - | 状态: 1-Active 2-InActive |

---

### 3.2 创建角色

**接口：** `POST /api/v1/roles`

**描述：** 添加新角色，角色编码不能重复

**请求体：**

```json
{
  "name": "string",     // 必填，角色名称
  "code": "string",     // 必填，角色编码（如 admin、editor）
  "description": "string",  // 可选，角色描述
  "status": 1              // 可选，状态
}
```

---

### 3.3 角色下拉选项

**接口：** `GET /api/v1/roles/selector`

**描述：** 返回所有 Active 状态的角色，用于前端下拉选择

---

### 3.4 获取角色详情

**接口：** `GET /api/v1/roles/{id}`

---

### 3.5 更新角色信息

**接口：** `PUT /api/v1/roles/{id}`

---

### 3.6 删除角色

**接口：** `DELETE /api/v1/roles/{id}`

**描述：** 物理删除角色，不可恢复

---

### 3.7 获取角色权限

**接口：** `GET /api/v1/roles/{id}/permissions`

**描述：** 获取指定角色已分配的权限列表（含完整权限信息）

---

### 3.8 分配角色权限

**接口：** `POST /api/v1/roles/{id}/permissions`

**描述：** 为指定角色批量分配权限（全量覆盖，传空数组可清空）

**业务规则：** 角色和权限必须均为 Active 状态

**请求体：**

```json
{
  "permission_ids": [1, 2, 3]  // 权限ID数组
}
```

---

## 广告主管理

### 4.1 广告主列表

**接口：** `GET /api/v1/advertisers`

**描述：** 分页查询广告主列表，支持名称模糊搜索、状态/结算模式/账单周期筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 名称（模糊搜索） |
| `status` | Integer | - | 状态: 0-Disabled 1-Enabled 2-Suspended 3-Frozen |
| `billing_type` | Integer | - | 结算模式: 1-prepay 2-postpay |
| `billing_cycle` | Integer | - | 账单周期: 1-5 |

---

### 4.2 创建广告主

**接口：** `POST /api/v1/advertisers`

**描述：** 添加新的广告主账户，名称不能重复

**请求体：**

```json
{
  "name": "string",             // 必填，广告主名称
  "company_name": "string",     // 必填，公司法人主体名称
  "billing_type": 1,           // 必填，结算模式: 1-prepay 2-postpay
  "billing_cycle": 1,          // 必填，账单周期: 1-pre 2-net7 3-net15 4-net30 5-net60
  "contact_email": "string",    // 可选，联系邮箱
  "contact_phone": "string",    // 可选，联系电话
  "margin_threshold": 0.05,    // 可选，毛利底线
  "status": 1                  // 可选，状态
}
```

---

### 4.3 广告主下拉选项

**接口：** `GET /api/v1/advertisers/selector`

**描述：** 返回所有 Enabled 状态的广告主，用于前端下拉选择

---

### 4.4 获取广告主详情

**接口：** `GET /api/v1/advertisers/{id}`

---

### 4.5 更新广告主

**接口：** `PUT /api/v1/advertisers/{id}`

---

### 4.6 删除广告主

**接口：** `DELETE /api/v1/advertisers/{id}`

**描述：** 物理删除广告主账户，不可恢复

---

## 广告系列管理

### 5.1 广告系列列表

**接口：** `GET /api/v1/campaigns`

**描述：** 分页查询广告系列列表，支持多维筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 名称（模糊搜索） |
| `advertiser_id` | Integer | - | 广告主ID |
| `status` | Integer | - | 状态: 0-暂停 1-运行中 2-已结束 3-待审核 4-熔断 |
| `target_platform` | Integer | - | 平台: 1-Android 2-iOS 3-Web |
| `payout_type` | Integer | - | 结算模式: 1-CPI 2-CPE |

---

### 5.2 创建广告系列

**接口：** `POST /api/v1/campaigns`

**描述：** 创建新的广告系列，同一广告主下 Campaign ID 不能重复

**请求体：**

```json
{
  "name": "string",                  // 必填，广告系列名称
  "advertiser_id": 1,               // 必填，广告主ID
  "advertiser_campaign_id": "string", // 必填，广告主侧Campaign ID
  "target_platform": 1,              // 必填，平台: 1-Android 2-iOS 3-Web
  "payout_type": 1,                // 必填，结算模式: 1-CPI 2-CPE
  "payout_event": "string",          // 必填，结算事件名称
  "trigger_source": ["string"],      // 必填，触发源限制
  "payout_amount": 0.0,            // 可选，单次转化结算单价
  "app_name": "string",             // 可选，关联APP名称
  "target_regions": ["US","JP"],   // 可选，目标国家
  "target_segment_ids": [1,2],     // 可选，目标人群包ID数组
  "blacklist_segment_ids": [3,4],  // 可选，黑名单人群包ID数组
  "daily_cap": 1000,               // 可选，每日转化上限
  "total_cap": 10000,              // 可选，生命周期转化上限
  "attribution_window": 24,         // 可选，归因时效窗口(小时)，默认24
  "mmp_type": 1,                  // 可选，归因平台: 1-AppsFlyer 2-Adjust
  "click_url": "string",            // 可选，点击追踪链接
  "margin_control_threshold": 0.05, // 可选，毛利控制底线
  "delay_strategy": {},             // 可选，AI反欺诈延迟策略
  "macro_mapping": {},              // 可选，宏参数映射
  "status": 3                     // 可选，状态，默认3-待审核
}
```

---

### 5.3 获取广告系列详情

**接口：** `GET /api/v1/campaigns/{id}`

---

### 5.4 更新广告系列

**接口：** `PUT /api/v1/campaigns/{id}`

**描述：** 更新广告系列的配置、状态、结算信息等。修改 `advertiser_id` 或 `campaign_id` 时会检查组合唯一性

---

### 5.5 删除广告系列

**接口：** `DELETE /api/v1/campaigns/{id}`

**描述：** 物理删除广告系列，不可恢复

---

## 人群包管理

### 6.1 人群包列表

**接口：** `GET /api/v1/audiences`

**描述：** 分页查询人群包列表，支持名称模糊搜索、状态/更新策略筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 名称（模糊搜索） |
| `status` | Integer | - | 状态: 0-draft 1-calculating 2-ready 3-failed 4-invalid |
| `refresh_type` | Integer | - | 更新策略: 1-static 2-hourly 3-daily |

---

### 6.2 创建人群包

**接口：** `POST /api/v1/audiences`

**描述：** 创建新的人群包配置，`audience_id` 自动生成

**注意：** `device_count` / `last_computed_at` 为只读字段，由计算系统填充，不在请求中提交

**请求体：**

```json
{
  "name": "string",             // 必填，人群包名称
  "description": "string",      // 可选，功能/业务场景描述
  "rule_expression": {},        // 必填，核心规则（JSON）
  "refresh_type": 1,           // 必填，更新策略: 1-static 2-hourly 3-daily
  "refresh_time": "string",     // 可选，更新时间规则
  "ttl_days": 0,              // 可选，有效期天数（0=永久有效）
  "status": 0                  // 可选，状态，默认0-draft
}
```

---

### 6.3 获取人群包详情

**接口：** `GET /api/v1/audiences/{id}`

**描述：** 根据 ID 获取人群包完整信息（含只读字段 `audience_id`、`device_count`、`last_computed_at`）

---

### 6.4 更新人群包

**接口：** `PUT /api/v1/audiences/{id}`

**描述：** 更新人群包的可编辑字段（`name`、`description`、`status`、`rule_expression`、`refresh_type` 等）

**注意：** `audience_id` / `device_count` / `last_computed_at` 不可修改

---

### 6.5 删除人群包

**接口：** `DELETE /api/v1/audiences/{id}`

**描述：** 物理删除人群包配置，不可恢复

---

## 数据来源管理

### 7.1 数据来源列表

**接口：** `GET /api/v1/data-sources`

**描述：** 分页查询数据来源列表，支持名称模糊搜索、类型/接入方式/状态筛选

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | Integer | 1 | 页码 |
| `page_size` | Integer | 10 | 每页条数 |
| `name` | String | - | 名称（模糊搜索） |
| `type` | Integer | - | 来源类型: 1-rtb 2-mmp 3-sdk 4-file |
| `integration_type` | Integer | - | 接入方式: 1-api 2-postback 3-upload |
| `status` | Integer | - | 状态: 0-InActive 1-Active 2-Abnormal |

---

### 7.2 创建数据来源

**接口：** `POST /api/v1/data-sources`

**描述：** 添加新的数据来源配置，名称不能重复

**请求体：**

```json
{
  "name": "string",                 // 必填，名称
  "type": 1,                       // 必填，来源类型: 1-rtb 2-mmp 3-sdk 4-file
  "integration_type": 1,            // 必填，接入方式: 1-api 2-postback 3-upload
  "status": 1,                     // 可选，状态: 0-InActive 1-Active 2-Abnormal
  "auth_token": "string",           // 可选，鉴权Token
  "ip_whitelist": ["1.2.3.4"],  // 可选，IP白名单
  "target_regions": ["US","CN"],   // 可选，覆盖国家/地区
  "integration_config": {},          // 可选，字段映射与接入规则
  "cleaning_rules": {},            // 可选，清洗与ETL规则
  "downstream_policy": {}           // 可选，下游流控策略
}
```

---

### 7.3 数据来源下拉选项

**接口：** `GET /api/v1/data-sources/selector`

**描述：** 返回所有 Active 状态的数据来源，用于前端下拉选择

---

### 7.4 获取数据来源详情

**接口：** `GET /api/v1/data-sources/{id}`

---

### 7.5 更新数据来源

**接口：** `PUT /api/v1/data-sources/{id}`

**描述：** 更新数据来源的名称、类型、接入方式、状态及各项配置

---

### 7.6 删除数据来源

**接口：** `DELETE /api/v1/data-sources/{id}`

**描述：** 物理删除数据来源配置，不可恢复

---

## 测试接口

### 8.1 测试：创建用户

**接口：** `POST /api/v1/test/create-user`

**描述：** [仅测试] 无需 Token，直接创建用户，不校验超级管理员权限

---

## 附录

### 状态码说明

| 状态码 | 说明 |
|--------|------|
| 0 | 成功 |
| 400 | 参数错误 |
| 401 | 未授权 |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

### 枚举值对照表

#### 广告主状态
| 值 | 说明 |
|-----|------|
| 0 | Disabled |
| 1 | Enabled |
| 2 | Suspended |
| 3 | Frozen |

#### 广告系列状态
| 值 | 说明 |
|-----|------|
| 0 | 暂停 |
| 1 | 运行中 |
| 2 | 已结束 |
| 3 | 待审核 |
| 4 | 熔断 |

#### 人群包状态
| 值 | 说明 |
|-----|------|
| 0 | draft |
| 1 | calculating |
| 2 | ready |
| 3 | failed |
| 4 | invalid |

#### 数据来源类型
| 值 | 说明 |
|-----|------|
| 1 | RTB |
| 2 | MMP |
| 3 | SDK |
| 4 | File |

#### 接入方式
| 值 | 说明 |
|-----|------|
| 1 | API |
| 2 | Postback |
| 3 | Upload |

---

**文档版本：** v1.0.0  
**最后更新：** 2026-06-25
