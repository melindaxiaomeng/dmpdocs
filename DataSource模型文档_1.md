# DataSource 数据来源模型文档

## 1. 概述

`DataSource` 是 DMP 平台最核心的配置模型之一。它定义了上游数据源的全部接入行为规范，包括：

- **身份标识与鉴权**：名称、Access Token、IP 白名单
- **协议与形态**：数据源类型（RTB/MMP/SDK/File）、接入方式（API/Postback/Upload）
- **字段提取与映射**：`integration_config` — 根节点定位、嵌套 JSON 展开、异构字段映射
- **清洗与标准化**：`cleaning_rules` — 数据模式声明、四大核心维度标准化、业务过滤与事件翻译
- **下游流控策略**：`downstream_policy` — 滑动窗口去重、自然周期去重、多维聚合风控

---

## 2. DataSource 主表字段

| 字段 | 类型 | DB列名 | 必填 | 说明 |
|------|------|--------|:---:|------|
| `ID` | `uint` | `id` PK autoIncrement | — | 自增主键 |
| `Name` | `string` | `name` varchar(100) | ✅ | 数据来源名称（唯一） |
| `Type` | `int` | `type` default=1 | ✅ | 来源类型（见 §3） |
| `IntegrationType` | `int` | `integration_type` default=1 | ✅ | 接入方式（见 §4） |
| `Status` | `int` | `status` default=1 | — | 状态（见 §5） |
| `AuthToken` | `*string` | `auth_token` varchar(255) | — | 鉴权 Token（网关接入校验） |
| `IPWhitelist` | `JsonSlice` | `ip_whitelist` JSON | — | IP 白名单，如 `["192.168.1.1","10.0.0.0/24"]` |
| `TargetRegions` | `JsonSlice` | `target_regions` JSON | — | 覆盖国家/地区，如 `["US","JP","CN"]` |
| `IntegrationConfig` | `IntegrationConfig` | `integration_config` JSON | — | 字段提取与映射（§6） |
| `CleaningRules` | `CleaningRules` | `cleaning_rules` JSON | — | 清洗与 ETL 规则（§7） |
| `DownstreamPolicy` | `DownstreamPolicy` | `downstream_policy` JSON | — | 下游流控策略（§8） |
| `CreatedBy` | `uint` | `created_by` | ✅ | 创建人用户 ID |
| `UpdatedBy` | `uint` | `updated_by` | ✅ | 最后修改人用户 ID |
| `CreatedAt` | `time.Time` | `created_at` autoCreateTime:milli | — | 创建时间（毫秒精度） |
| `UpdatedAt` | `time.Time` | `updated_at` autoUpdateTime:milli | — | 更新时间（毫秒精度） |

> **表名**：`data_source`  
> **删除策略**：物理删除。`JsonSlice` 实现 GORM `Scanner`/`Valuer` 接口，自动 JSON 序列化。  
> `IntegrationConfig`、`CleaningRules`、`DownstreamPolicy` 均为结构化类型，通过 GORM 的 JSON 列直接序列化。

---

## 3. 数据来源类型 (`Type`)

| 常量 | 值 | 说明 | 典型场景 |
|------|:---:|------|----------|
| `DataSourceTypeRtb` | 1 | rtb — 流量请求 | 来自 DSP/AdExchange 的竞价请求 |
| `DataSourceTypeMmp` | 2 | mmp — MMP 回调 | AppsFlyer、Adjust 等归因平台 Postback |
| `DataSourceTypeSdk` | 3 | sdk — 自有产品 | 自有 App 内嵌 SDK 埋点 |
| `DataSourceTypeFile` | 4 | file — 文件导入 | 离线 CSV/JSON 文件批量导入 |

可通过 `DataSource.TypeLabel()` 获取文字描述（如 `"mmp"`）。

---

## 4. 接入方式 (`IntegrationType`)

| 常量 | 值 | 说明 |
|------|:---:|------|
| `IntegrationTypeApi` | 1 | API — HTTP 主动推送 |
| `IntegrationTypePostback` | 2 | postback — MMP 回调 URL 接收 |
| `IntegrationTypeUpload` | 3 | upload — 文件上传（COS/SFTP） |

可通过 `DataSource.IntegrationTypeLabel()` 获取文字描述（如 `"postback"`）。

---

## 5. 数据来源状态 (`Status`)

| 常量 | 值 | 说明 |
|------|:---:|------|
| `DataSourceStatusInactive` | 0 | InActive — 禁用（不接收数据） |
| `DataSourceStatusActive` | 1 | Active — 启用（默认值） |
| `DataSourceStatusAbnormal` | 2 | Abnormal — 异常（接入异常但未禁用） |

> Gateway 的 `bizcache` 同步时仅拉取 `status=1` 的记录，禁用/异常的 DataSource 不会参与数据处理。

---

## 6. IntegrationConfig — 字段提取与映射

### 6.1 结构定义

```go
type IntegrationConfig struct {
    Version   string          `json:"version"`    // 配置版本号
    SourceID  string          `json:"-"`          // 运行时注入，不存储
    Extractor ExtractorConfig `json:"extractor"`  // 数据提取器
}
```

`SourceID` 使用 `json:"-"` 标签，**不在数据库中存储**，运行时由 Gateway 动态赋值 `strconv.Itoa(int(ds.ID))`。

### 6.2 ExtractorConfig — 数据提取器

| 字段 | 类型 | 说明 |
|------|------|------|
| `RootNode` | `string` | 真实数据的根节点路径。支持 `.` 分割的层级路径（如 `"data.attributes"`）。留空或 `"root"` 表示顶层即数据 |
| `JSONParseFields` | `[]string` | 需要解析的 JSON 字符串字段路径列表。如 `["payload.inner"]` |
| `FieldMapping` | `map[string]string` | 字段映射字典。Key = 中台标准字段名，Value = 原始数据字段路径 |

### 6.3 数据提取流程 (`ProcessRawData`)

```
原始 JSON (raw)
  ├─ 1. RootNode 定位: 按 . 路径导航到真实数据层
  ├─ 2. JSON 字符串展开: 遍历 json_parse_fields，逐字段解析内嵌 JSON
  └─ 3. 字段映射: 按 field_mapping 提取目标字段到标准输出
```

### 6.4 配置示例

```json
{
  "version": "1.0",
  "extractor": {
    "root_node": "data",
    "json_parse_fields": ["payload.custom_fields"],
    "field_mapping": {
      "device_id": "device.ifa",
      "event_type": "event.name",
      "event_time": "timestamp",
      "revenue": "event.revenue"
    }
  }
}
```

对于如下原始数据：
```json
{
  "data": {
    "device": { "ifa": "A1B2C3D4E5F6" },
    "event": { "name": "purchase", "revenue": 29.99 },
    "timestamp": 1717209600,
    "payload": { "custom_fields": "{\"channel\":\"google\",\"campaign\":\"cmp_618\"}" }
  }
}
```

提取后的输出：
```json
{
  "device_id": "A1B2C3D4E5F6",
  "event_type": "purchase",
  "event_time": 1717209600,
  "revenue": 29.99,
  "custom_fields": { "channel": "google", "campaign": "cmp_618" }
}
```

---

## 7. CleaningRules — 清洗与 ETL 规则

### 7.1 结构定义

```go
type CleaningRules struct {
    Version       string                      `json:"version"`        // 规则版本号
    SourceID      string                      `json:"-"`              // 运行时注入
    Schema        map[string]SchemaField      `json:"schema"`         // 数据模式声明
    Standardizers map[string]StandardizerJob  `json:"standardizers"`  // 四大核心维度标准化
    BusinessLogic BusinessLogicPolicy         `json:"business_logic"` // 业务映射与过滤
}
```

### 7.2 Schema — 数据模式声明

定义数据接收时的"物理形态"约束。Key 为映射后的中台字段名，Value 为约束规则。

```go
type SchemaField struct {
    Type     string      `json:"type"`            // string / int64 / float / bool
    Required bool        `json:"required"`        // 是否必填，必填缺失则丢弃整行
    Trim     bool        `json:"trim"`            // 是否去除首尾空白（string 类型）
    Default  interface{} `json:"default"`         // 默认值
    Min      *float64    `json:"min,omitempty"`   // 数值最小值（int64/float）
    Max      *float64    `json:"max,omitempty"`   // 数值最大值（int64/float）
}
```

**支持的数据类型**：`string`、`int64`、`float`、`bool`

### 7.3 Standardizers — 四大核心维度标准化

系统内置 4 个标准化器，Key 固定不可自定义：

| Key | 名称 | 适用场景 |
|-----|------|----------|
| `device_identity` | 设备身份标准化 | IDFA、GAID、OAID、Android ID、MAC |
| `network_ip` | IP/网络标准化 | 公网 IPv4/IPv6 地址 |
| `event_time` | 事件时间标准化 | 全球各类格式时间戳 |
| `geo_country` | 地域/国家标准化 | 国家编码 |

```go
type StandardizerJob struct {
    SourceFields    []string `json:"source_fields,omitempty"`   // 备选漏斗（多选一，按序取第一个非空）
    SourceField     string   `json:"source_field,omitempty"`    // 单一源字段
    Action          string   `json:"action"`                    // 标准化动作算子
    TargetField     string   `json:"target_field"`              // 标准化后写入的目标字段
    SourceTimezone  string   `json:"source_timezone,omitempty"` // 源时区（仅 event_time 有效）
}
```

#### 7.3.1 `device_identity` — 设备身份标准化

| Action | 说明 |
|--------|------|
| `normalize_raw` | 剔除横杠、下划线、空格，强转大写 |
| `normalize_to_md5` | 先 `normalize_raw`，再计算 MD5 32位（可选大小写） |
| `normalize_to_sha256` | 先 `normalize_raw`，再计算 SHA256 |

#### 7.3.2 `network_ip` — IP/网络标准化

| Action | 说明 |
|--------|------|
| `extract_clean_ip` | 剥离端口号，去前导零，IPv6 标准压缩格式，非法 IP 置空 |
| `mask_ip_gdpr` | `extract_clean_ip` 后 IPv4 末段抹零（GDPR 合规） |

#### 7.3.3 `event_time` — 事件时间标准化

| Action | 说明 |
|--------|------|
| `to_utc_millis` | 自动识别 10 位/13 位整数 → UTC 毫秒时间戳 |
| `string_to_utc_millis` | 字符串格式 → 按 `source_timezone` 解析 → UTC 毫秒时间戳 |

#### 7.3.4 `geo_country` — 国家编码标准化

| Action | 说明 |
|--------|------|
| `to_iso_alpha2` | 任意格式（USA/united states/840）→ 2 位大写 `US` |
| `to_iso_alpha3` | 任意格式 → 3 位大写 `USA` |

### 7.4 BusinessLogic — 业务映射与过滤

```go
type BusinessLogicPolicy struct {
    EventMapping *EventMappingPolicy `json:"event_mapping,omitempty"` // 事件类型翻译
    DropFilters  []DropFilterRule    `json:"drop_filters,omitempty"`  // 强力丢弃规则
}
```

#### 事件映射 (`EventMappingPolicy`)

将上游非标准事件名翻译为中台核心事件：

```json
{
  "source_field": "event_name",
  "map": {
    "install": "activation",
    "purchase": "payment",
    "signup": "register"
  },
  "unmatched": "keep"   // keep / drop / set_default
}
```

| 未匹配策略 | 说明 |
|-----------|------|
| `keep` | 原样保留 |
| `drop` | 丢弃整行数据 |
| `set_default` | 强制修改为 `"unknown"` |

#### 丢弃规则 (`DropFilterRule`)

短路求值，满足任意一条即丢弃整行。

| 操作符 | 说明 | Value 类型 |
|--------|------|-----------|
| `is_empty` | 字段为空 | — |
| `is_not_empty` | 字段非空 | — |
| `in` | 在白名单列表中 | `[]interface{}` |
| `not_in` | 在黑名单列表中 | `[]interface{}` |
| `older_than_days` | 时间戳老于 N 天（拦截历史刷量） | `float64` |
| `in_the_future_hours` | 时间戳超前 N 小时（拦截时钟作弊） | `float64` |

### 7.5 Validate — 全量防卫性校验

```go
func (r *CleaningRules) Validate() error
```

校验链路：
```
Validate()
├── Version 非空校验
├── validateSchema()           // 数据类型/Bool检测/Min>Max校验
├── validateStandardizers()    // 四大Key合法性/Action匹配性/输入字段完整性
└── validateBusinessLogic()    // EventMapping完整性/DropFilter操作符合法性/值类型匹配
```

---

## 8. DownstreamPolicy — 下游流控策略

### 8.1 结构定义

```go
type DownstreamPolicy struct {
    Version             string               `json:"version"`
    SourceID            string               `json:"-"`              // 运行时注入
    DeduplicationPolicy *DeduplicationPolicy `json:"deduplication_policy,omitempty"` // 去重策略
    AggregationPolicy   *AggregationPolicy   `json:"aggregation_policy,omitempty"`   // 聚合风控策略
}
```

### 8.2 DeduplicationPolicy — 去重策略

定义基于有状态分布式缓存（Redis）的数据去重规则，由 ETL 消费者执行。

| 字段 | 类型 | 说明 |
|------|------|------|
| `Enabled` | `bool` | 是否开启去重 |
| `Strategy` | `string` | `sliding_window` / `periodic` |
| `DedupeKeys` | `[]string` | 联合去重主键字段列表，如 `["device_id","event_type"]` |
| `Window` | `*WindowConfig` | 滑动窗口配置（仅 `sliding_window`） |
| `Periodic` | `*PeriodicConfig` | 自然周期配置（仅 `periodic`） |
| `OnDuplicate` | `string` | 命中重复后的行为：`drop_silently` / `route_to_dlq` |

#### WindowConfig — 滑动时间窗口

| 字段 | 说明 | 示例 |
|------|------|------|
| `DurationSeconds` | 窗口跨度（秒） | 600（10 分钟） |
| `RedisTTLSeconds` | 去重键 Redis TTL（秒） | 建议设为 DurationSeconds 的 2 倍 |

#### PeriodicConfig — 自然周期去重

| 字段 | 说明 |
|------|------|
| `Cycle` | 周期类型：`natural_day` / `natural_hour` |
| `Timezone` | 边界计算时区，如 `"Asia/Shanghai"` |

### 8.3 AggregationPolicy — 聚合风控策略

定义多维度的滚动窗口聚合流控与防刷风控规则，由 S2S 变现引擎执行。

```go
type AggregationPolicy struct {
    Enabled bool                `json:"enabled"`
    Windows []AggregationWindow `json:"windows"`  // 多个独立监控窗口
}
```

#### AggregationWindow — 单个监控窗口

| 字段 | 类型 | 说明 |
|------|------|------|
| `Name` | `string` | 监控项标识，用于日志溯源 |
| `AggrKey` | `string` | 聚合维度字段，如 `"client_ip"`、`"channel_id"` |
| `Metric` | `string` | 指标类型：`count` / `sum_revenue` |
| `DurationSeconds` | `int` | 滚动窗口跨度（秒），如 300 代表 5 分钟 |
| `Thresholds` | `[]ThresholdRule` | 阈值规则数组 |

#### ThresholdRule — 阈值规则

| 字段 | 说明 |
|------|------|
| `Operator` | 比较操作符：`">"` / `">="` / `"=="` / `"<"` / `"<="` |
| `Value` | 触发阈值（float64） |
| `Action` | 风控行为：`block_s2s`（阻断 S2S 转发）/ `flag_suspicious`（打标签） |
| `Notify` | 告警通知渠道列表（飞书 Webhook URL 等） |

### 8.4 配置示例

```json
{
  "version": "2.0",
  "deduplication_policy": {
    "enabled": true,
    "strategy": "sliding_window",
    "dedupe_keys": ["device_id", "event_type"],
    "window": {
      "duration_seconds": 3600,
      "redis_ttl_seconds": 7200
    },
    "on_duplicate": "drop_silently"
  },
  "aggregation_policy": {
    "enabled": true,
    "windows": [
      {
        "name": "ip_frequency_guard",
        "aggr_key": "client_ip",
        "metric": "count",
        "duration_seconds": 300,
        "thresholds": [
          { "operator": ">", "value": 100, "action": "block_s2s", "notify": ["https://open.feishu.cn/webhook/xxx"] },
          { "operator": ">", "value": 50,  "action": "flag_suspicious", "notify": [] }
        ]
      }
    ]
  }
}
```

---

## 9. API 接口

### 9.1 端点列表

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/data-sources` | 分页列表（支持 name/type/integration_type/status 筛选） |
| `POST` | `/api/v1/data-sources` | 创建数据来源 |
| `GET` | `/api/v1/data-sources/selector` | 下拉选项（仅返回 Active 状态） |
| `GET` | `/api/v1/data-sources/:id` | 详情 |
| `PUT` | `/api/v1/data-sources/:id` | 更新 |
| `DELETE` | `/api/v1/data-sources/:id` | 物理删除 |

> 所有端点需要 JWT 鉴权。

### 9.2 创建请求示例

```json
{
  "name": "AppsFlyer Postback",
  "type": 2,
  "integration_type": 2,
  "status": 1,
  "auth_token": "sk-af-xxxxxxxxxxxx",
  "ip_whitelist": ["52.1.2.3", "54.4.5.6/32"],
  "target_regions": ["US", "JP", "KR"],
  "integration_config": {
    "version": "1.0",
    "extractor": {
      "root_node": "data",
      "json_parse_fields": ["payload.extra"],
      "field_mapping": {
        "device_id": "device.ifa",
        "event_type": "event.name",
        "event_time": "event.timestamp"
      }
    }
  },
  "cleaning_rules": {
    "version": "1.0",
    "schema": {
      "device_id": { "type": "string", "required": true, "trim": true },
      "revenue": { "type": "float", "min": 0, "max": 99999 }
    },
    "standardizers": {
      "device_identity": {
        "source_field": "device_id",
        "action": "normalize_raw",
        "target_field": "device_id"
      },
      "event_time": {
        "source_field": "event_time",
        "action": "to_utc_millis",
        "target_field": "event_time"
      }
    },
    "business_logic": {
      "event_mapping": {
        "source_field": "event_type",
        "map": { "install": "activation", "purchase": "payment" },
        "unmatched": "keep"
      },
      "drop_filters": [
        { "field": "event_time", "operator": "older_than_days", "value": 7 }
      ]
    }
  },
  "downstream_policy": {
    "version": "2.0",
    "deduplication_policy": {
      "enabled": true,
      "strategy": "sliding_window",
      "dedupe_keys": ["device_id", "event_type"],
      "window": { "duration_seconds": 3600, "redis_ttl_seconds": 7200 },
      "on_duplicate": "drop_silently"
    }
  }
}
```

### 9.3 更新请求说明

更新请求所有字段为可选（指针类型），传入 `null` 的字段不更新。`cleaning_rules` 和 `downstream_policy` 传入时会触发防御性校验（仅当 `Version` 非空时）。

### 9.4 防御性校验

| 错误码 | 说明 |
|:------:|------|
| `14001` | 数据来源不存在 |
| `14002` | 数据来源名称已存在 |
| `14100` | 清洗规则 / 下游策略校验失败（返回具体校验错误详情） |

---

## 10. 运行时使用

### 10.1 SourceID 注入

`IntegrationConfig.SourceID`、`CleaningRules.SourceID`、`DownstreamPolicy.SourceID` 使用 `json:"-"` 标签，**不参与 JSON 序列化，不存储到数据库**。运行时由使用方动态注入：

```go
ds := repo.FindByID(id)
ds.IntegrationConfig.SourceID = strconv.Itoa(int(ds.ID))
ds.CleaningRules.SourceID = strconv.Itoa(int(ds.ID))
ds.DownstreamPolicy.SourceID = strconv.Itoa(int(ds.ID))
```

### 10.2 Gateway 使用流程

```
HTTP Request → 鉴权(Token+IP) → 路由到 DataSource
  → ProcessRawData(raw, ds.IntegrationConfig.Extractor)  // 字段提取
  → CleaningRules 清洗流水线
      ├─ Schema 约束校验
      ├─ Standardizers 四大维度标准化
      └─ BusinessLogic 事件翻译 + 过滤丢弃
  → DownstreamPolicy 下游控制
      ├─ DeduplicationPolicy 去重判断
      └─ AggregationPolicy 风控阈值检查
  → 标准化事件推入 Kafka
```

### 10.3 配置优先级

三个 JSON 配置列均为**可选**。未配置时等同于空结构体，Gateway 按默认行为处理（不执行对应阶段逻辑）。
