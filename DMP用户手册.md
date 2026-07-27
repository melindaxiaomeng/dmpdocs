# DMP 平台用户手册

> 适用范围：运营、数据分析、投放人员日常使用；技术对接可参考「规则提交结构」小节。

---

## 一、平台概览

### 1.1 登录与导航

打开平台后使用账号登录。登录成功后进入主界面，左侧为导航菜单，按业务分为三组：

| 分组 | 菜单项 | 说明 |
|---|---|---|
| OVERVIEW | 仪表盘 | 平台总览数据 |
| OVERVIEW | 数据源报表 | 各数据源的接入/清洗报表 |
| BUSINESS | 广告活动 | 投放活动管理，可关联人群包 |
| BUSINESS | 广告主管理 | 广告主账户信息 |
| BUSINESS | **人群包管理** | **本手册重点** |
| SYSTEM & DATA | 数据源管理 | 接入数据源与清洗规则配置 |
| SYSTEM & DATA | 标准字段 | 标准字段字典维护 |
| SYSTEM & DATA | 标准事件 | 标准事件字典维护 |
| SYSTEM & DATA | 用户管理 | 后台账号与权限 |

### 1.2 通用交互

- **列表页**：支持按名称/状态等条件筛选，分页栏可调整每页条数。
- **表单**：以右侧**抽屉（Drawer）**形式弹出，点击「取消」或遮罩外区域可关闭；关闭后已填内容会清空。
- **编辑**：列表操作列的「编辑」会先拉取最新数据回填，再允许修改。

---

## 二、人群包管理（核心）

人群包（Audience）是 DMP 的核心对象：它用一套**规则表达式**圈选出满足条件的用户集合，供广告活动定向投放。

### 2.1 列表页说明

列表列含义：

| 列 | 含义 |
|---|---|
| 人群包 ID | 系统生成的唯一标识（如 `aud_xxx`） |
| 名称 | 运营可读的名称 |
| 状态 | 见 2.5 状态流转 |
| 设备数 | 最近一次计算命中并去重后的设备量级 |
| 更新策略 | 静态 / 每小时 / 每天 |
| 创建时间 | 鼠标悬停单元格可查看完整时间 |
| 操作 | 编辑 / 关联（关联广告活动）/ 运行 / 结束 等 |

### 2.2 新建人群包

点击右上角「新建人群包」，弹出右侧抽屉，分两部分填写：**基本信息** 与 **规则表达式**。

#### 基本信息

| 字段 | 必填 | 说明 / 取值 |
|---|---|---|
| 名称 | 是 | 最多 150 字 |
| 描述 | 否 | 最多 500 字，仅备注用途 |
| 状态 | 是（默认草稿） | `0-草稿` / `1-等待计算`，新建时只能选这两个 |
| 更新策略 | 是（默认静态） | `1-静态` / `2-每小时` / `3-每天` |
| 刷新时间 | 仅动态 | **静态策略不填**；每天填 `03:00`，每小时填 `:30` |
| 过期时间(天) | 是（默认 3） | 人群包有效期，到期自动失效 |

> 关于「刷新时间」格式：
> - 更新策略 = **每天** → 填每日计算时刻，如 `03:00`（表示每天凌晨 3 点重算）
> - 更新策略 = **每小时** → 填分钟偏移，如 `:30`（表示每小时第 30 分钟重算）
> - 更新策略 = **静态** → 该框置灰，无需填写（一次性计算）

#### 规则表达式（重点）

这是圈选用户的核心。编辑器由「条件组」组成，支持无限嵌套。详见下一节。

---

### 2.3 规则表达式编辑器详解

规则 = 一棵**条件树**，结构为：

```
条件组（逻辑 AND / OR）
 ├─ 条件1（画像 或 行为）
 ├─ 条件2（画像 或 行为）
 └─ 子条件组（可再嵌套 AND/OR）
```

#### 2.3.1 逻辑组合

- 每个条件组有一个**组逻辑**：`AND`（同时满足）或 `OR`（满足其一）。
- 条件组可以**嵌套子条件组**，实现复杂逻辑。例如：`(国家=US AND 系统=iOS) OR (LTV>100)`。

#### 2.3.2 画像条件（用户属性）

维度共 12 个，下表列出每个维度支持的操作符与值类型：

| 维度（字段） | 中文 | 支持操作符 | 值类型 | 多值 |
|---|---|---|---|---|
| `last_country` | 最后活跃国家 | in / not_in / eq | 字符串 | 是 |
| `device_os` | 操作系统 | eq / in | 字符串 | 是 |
| `device_model` | 设备型号 | eq / in / like | 字符串 | 是 |
| `is_high_value` | 高价值用户 | eq | 布尔(true/false) | 否 |
| `total_revenue` | 累计充值 LTV | gt / lt / between | 浮点 | 是(between) |
| `purchase_count` | 累计充值次数 | gt / lt / between | 整数 | 是(between) |
| `tags` | 画像标签 | contains_any / contains_all / not_contains | 字符串 | 是 |
| `source_id` | 数据源 | eq / in / not_in | 整数 | 是 |
| `device_id_type` | 设备 ID 类型 | eq / in | 字符串 | 是 |
| `lang` | 语言 | eq / in / not_in | 字符串 | 是 |
| `is_new_device` | 新设备 | eq | 布尔 | 否 |
| `last_seen_time` | 最后活跃时间 | gt / lt / between | 整数(时间戳) | 是 |

**操作符速查：**
- `eq`：等于（单值）
- `in` / `not_in`：在列表内 / 不在列表内（多值，逗号分隔）
- `gt` / `lt`：大于 / 小于
- `between`：区间（起,止）
- `like`：模糊匹配（设备型号用，如 `%iPhone%`）
- `contains_any` / `contains_all` / `not_contains`：标签包含任一 / 全部 / 不含

#### 2.3.3 行为条件（用户做过的事）

行为条件描述「用户在某个时间窗内，对某类事件做了什么」，结构比画像条件更丰富：

- **事件类型**：从「标准事件」字典动态加载（如 `purchase`、`login`、`install`）。
- **操作符**：`eq`（单一事件）/ `in`（多个事件之一）。
- **时间窗（time_range）**：
  - 相对时间：`最近 N 天` 或 `最近 N 小时`（如近 30 天）。
  - 绝对时间：指定起止日期时间。
- **事件过滤（event_filters）**：对事件附加限制，如「仅限 app_id=123 的 purchase」。
  - 字段：`app_id` / `campaign_id` / `source_id`
  - 操作符：`eq` / `in` / `not_in` / `like`
- **聚合度量（agg_config）**：对事件做量化阈值，如「近 30 天 purchase 次数 ≥ 3」。
  - 聚合方式：`count`（计数）/ `sum`（求和，需指定字段）
  - 阈值操作符：`gt` / `gte` / `lt` / `eq` / `between`

#### 2.3.4 多值填写规范（重要）

当维度支持多值（如国家、标签、数据源）时：

- 使用 `in` 或 `not_in` 操作符；
- 在值输入框中**用逗号分隔多个值**，例如：

  ```
  US, CN, JP
  ```

- 系统会自动按逗号拆分并去掉空格，最终提交为独立数组 `["US","CN","JP"]`。
- ⚠️ **不要用 `eq` 填多个值**——`eq` 会把整串当作单个值（变成 `["US, CN, JP"]`），导致圈选失败。
- 标签（tags）同理：`VIP, 回流用户` 表示标签包含 VIP 或回流用户（配合 `contains_any`）。

#### 2.3.5 数值与时间

- 数值类维度（LTV、次数）填写的数字会**自动转为数字**提交，无需关心引号。
- `between` 区间填写：`100, 500`（起始值, 结束值）。
- 时间类（`last_seen_time`、行为时间窗）使用日期/时间选择器，提交为时间戳。

---

### 2.4 状态流转

人群包有严格的状态机，编辑时只能流转到允许的目标状态。完整状态枚举：

| 状态码 | 含义 | 可流转到 |
|---|---|---|
| 0 | 草稿 | 1（等待计算） |
| 1 | 等待计算 | 2（计算中）、8（已禁用）、10（取消计算） |
| 2 | 计算中 | 3（就绪）、6（计算失败） |
| 3 | 就绪 | 4（等待更新）、8（已禁用）、9（已过期） |
| 4 | 等待更新 | 5（更新中）、3、8、9 |
| 5 | 更新中 | 3、7（更新失败）、9 |
| 6 | 计算失败 | 1、0、8 |
| 7 | 更新失败 | 4、6、8、9 |
| 8 | 已禁用 | 1、0、9 |
| 9 | 已过期 | （终态，不可再流转） |
| 10 | 取消计算 | 1、0 |

> 编辑抽屉的「状态」下拉会**只列出当前状态允许流转到的选项**，并标注「(当前)」。
> 运营常用的动作：草稿 → 等待计算（提交计算）；就绪 → 关联广告活动投放；需要暂停时 → 已禁用。

---

### 2.5 关联广告活动

在人群包列表操作列点击「关联」，弹出该人群包与广告活动的关联抽屉：

- 查看已关联的活动列表；
- 新增关联（选择广告活动）；
- 解绑用「停用」（status=2），并非物理删除。

关联后，广告活动即可针对该人群包定向投放。

---

### 2.6 实战示例库（从简到繁）

下面每个示例都给出：**目标 → 界面操作 → 提交的规则结构**。规则结构供技术排查或高级用户参考。

#### 示例 1：定向多个国家

> 目标：圈选最后活跃国家为美国、中国、日本的用户的合集。

- 维度：`last_country`
- 操作符：`in`
- 值：`US, CN, JP`

规则结构（节选）：
```json
{
  "type": "profile",
  "field": "last_country",
  "operator": "in",
  "value": ["US", "CN", "JP"]
}
```

#### 示例 2：高价值 iOS 用户

> 目标：操作系统是 iOS，且被标记为高价值用户。

- 条件组逻辑：`AND`
- 条件1：维度 `device_os`，操作符 `eq`，值 `iOS`
- 条件2：维度 `is_high_value`，操作符 `eq`，值 `true`

```json
{
  "logic": "AND",
  "conditions": [
    { "type": "profile", "field": "device_os", "operator": "eq", "value": ["iOS"] },
    { "type": "profile", "field": "is_high_value", "operator": "eq", "value": [true] }
  ],
  "children": []
}
```

#### 示例 3：近 30 天充值次数 ≥ 3 次

> 目标：对 `purchase` 事件，在近 30 天内发生次数不少于 3 次的用户。这是一个**行为条件 + 聚合度量**。

- 条件类型：行为
- 事件类型：`purchase`
- 操作符：`in`
- 时间窗：相对，最近 `30` 天
- 聚合度量：count，`gte`，值 `3`

```json
{
  "type": "behavior",
  "field": "event_type",
  "operator": "in",
  "value": ["purchase"],
  "time_range": { "type": "relative", "unit": "day", "start_offset": 30, "end_offset": 0 },
  "agg_config": { "agg_type": "count", "operator": "gte", "value": [3] }
}
```

#### 示例 4：排除新设备

> 目标：圈选非新设备的用户（排除刚激活的设备）。

- 维度：`is_new_device`
- 操作符：`eq`
- 值：`false`

```json
{ "type": "profile", "field": "is_new_device", "operator": "eq", "value": [false] }
```

#### 示例 5：带有标签的用户

> 目标：标签中同时包含「VIP」和「回流用户」。

- 维度：`tags`
- 操作符：`contains_all`
- 值：`VIP, 回流用户`

```json
{ "type": "profile", "field": "tags", "operator": "contains_all", "value": ["VIP", "回流用户"] }
```

> `contains_any` 表示「包含任一即可」；`not_contains` 表示「不含这些标签」。

#### 示例 6：LTV 区间 + 操作系统组合

> 目标：累计充值 LTV 在 100~500 之间，且操作系统为 Android 或 iOS。

- 条件组逻辑：`AND`
- 条件1：`total_revenue` `between` `100, 500`
- 条件2：`device_os` `in` `Android, iOS`

```json
{
  "logic": "AND",
  "conditions": [
    { "type": "profile", "field": "total_revenue", "operator": "between", "value": [100, 500] },
    { "type": "profile", "field": "device_os", "operator": "in", "value": ["Android", "iOS"] }
  ],
  "children": []
}
```

#### 示例 7：带事件过滤的行为条件

> 目标：近 7 天内，在 app_id=123 下发生过 `login` 的用户。

- 行为条件：事件 `login`，时间窗相对 7 天
- 事件过滤：`app_id` `eq` `123`

```json
{
  "type": "behavior",
  "field": "event_type",
  "operator": "in",
  "value": ["login"],
  "time_range": { "type": "relative", "unit": "day", "start_offset": 7, "end_offset": 0 },
  "event_filters": [{ "field": "app_id", "operator": "eq", "value": ["123"] }]
}
```

#### 示例 8：复杂嵌套（AND / OR 组合）

> 目标：`(国家=US 且 系统=iOS) 或 (LTV>100)`。

- 根条件组逻辑：`OR`
  - 子条件组 A（逻辑 AND）：
    - `last_country` `eq` `US`
    - `device_os` `eq` `iOS`
  - 子条件组 B（逻辑，这里单个条件）：
    - `total_revenue` `gt` `100`

```json
{
  "logic": "OR",
  "conditions": [],
  "children": [
    {
      "logic": "AND",
      "conditions": [
        { "type": "profile", "field": "last_country", "operator": "eq", "value": ["US"] },
        { "type": "profile", "field": "device_os", "operator": "eq", "value": ["iOS"] }
      ],
      "children": []
    },
    {
      "logic": "AND",
      "conditions": [
        { "type": "profile", "field": "total_revenue", "operator": "gt", "value": [100] }
      ],
      "children": []
    }
  ]
}
```

#### 示例 9：设备型号模糊匹配

> 目标：设备型号包含 iPhone 的所有机型。

- 维度：`device_model`
- 操作符：`like`
- 值：`%iPhone%`（系统自动补 `%`，填 `iPhone` 即可）

```json
{ "type": "profile", "field": "device_model", "operator": "like", "value": ["%iPhone%"] }
```

---

### 2.7 AI 智能圈人（Text-to-Rule）

为了让运营无需钻研规则语法，平台在「新建/编辑人群包」抽屉的**规则表达式顶部**内置了 **AI 智能圈人** 模块：用一句话描述你想要的人群，AI 自动把它转换成可编辑的规则卡片，回填到下方规则表达式编辑器。

> 入口位置：**人群包管理 → 新建人群包 / 编辑 → 规则表达式** 区域最上方（紫色边框卡片「AI 智能圈人」）。
> ⚠️ AI 生成是**全量覆盖**当前规则表达式：生成前请确认下方没有需要保留的手动规则，生成后请人工核对再保存。

#### 2.7.1 怎么用（三步）

1. **写描述**：在输入框里用自然语言描述人群，例如「近 30 天充值大于 500 元的 VIP 用户」。
2. **（可选）用模板**：点击输入框下方的紫色标签（推荐句式）可一键填入，再按需修改。
3. **魔法生成**：点击「魔法生成」按钮。也可以按 `Ctrl/⌘ + Enter` 快捷键直接生成。
   - 生成期间按钮显示「正在为你生成规则卡片...」，请稍候。
   - 生成成功会提示「规则已生成，请检查确认」，下方规则树已刷新为 AI 给出的条件卡片。
   - 未命中任何属性/行为、或AI服务异常时，会提示「未找到相关属性/行为，请尝试重新描述。」且**不改动**你原有的规则。

#### 2.7.2 AI 能识别什么（字段与操作符）

AI 的「词汇表」与规则表达式编辑器当前支持的维度**完全一致**，由后端 `dictionary.py` 与前端 `profileConfig` 对齐维护。包含：

**A. 画像属性（18 个）**

| 字段 key | 中文 | 支持操作符 | 值类型 | 多值 |
|---|---|---|---|---|
| `last_country` | 最后活跃国家 | in / not_in / eq | string | 是 |
| `device_os` | 操作系统 | eq / in | string | 是 |
| `device_model` | 设备型号 | eq / in / like | string | 是 |
| `is_high_value` | 高价值用户 | eq | bool | 否 |
| `total_revenue` | 累计充值 LTV | gt / lt / between | float | 是 |
| `purchase_count` | 累计充值次数 | gt / lt / between | int | 是 |
| `tags` | 画像标签 | contains_any / contains_all / not_contains | string | 是 |
| `source_id` | 数据源 | eq / in / not_in | int | 是 |
| `device_id_type` | 设备 ID 类型 | eq / in | string | 是 |
| `lang` | 语言 | eq / in / not_in | string | 是 |
| `is_new_device` | 新设备 | eq | bool | 否 |
| `last_seen_time` | 最后活跃时间 | gt / lt / between | int(时间戳) | 是 |
| `vip_level` | VIP 等级 | gt / lt / eq / between | int | 是 |
| `gender` | 性别 | eq / in / not_in | string | 是 |
| `city` | 城市 | in / not_in / eq | string | 是 |
| `province` | 省份 | in / not_in / eq | string | 是 |
| `age` | 年龄 | gt / lt / between / eq | int | 是 |
| `register_channel` | 注册渠道 | eq / in / not_in | string | 是 |

**B. 动态行为（标准事件）**
事件类型（如 `purchase`、`login`、`recharge`、`register`、`order_pay` 等）从「标准事件」字典**动态加载**，与你在前端「标准事件」模块看到的下拉列表保持一致；后端会优先从主后端 `/api/v1/standard-events/selector` 实时同步。

**C. 操作符规范（AI 只输出下列 token，不输出 `>`/`<` 等符号）**
- 数值/等级比较：`gt`(大于) / `lt`(小于) / `gte`(大于等于) / `lte`(小于等于) / `eq`(等于) / `between`(区间，值取 `[min,max]`)
- 集合：`in`(包含任一) / `not_in`(排除) / `contains_any` / `contains_all` / `not_contains`
- 模糊：`like`（仅 `device_model`，如 `iPhone%`）
- 事件未发生：`not_executed`（value 为 `null`，表示「从未发生过该事件」，如「未登录过」）

**D. 时间窗规范（仅行为条件，相对时间）**
统一格式：`past_N_days` / `past_N_hours` / `past_N_months`（N 为正整数，如 `past_30_days`）。AI 会自动把「近 30 天」「过去 7 天」「最近 24 小时」等换算成该格式；**不支持绝对日期**。

#### 2.7.3 转换规则（理解 AI 的“脑回路”）

- 多条独立条件默认用 **AND** 组合；用户明确说「或 / 任意满足其一」时才用 **OR**。
- 多值（如「多个国家 / 多个渠道」）自动拆成数组 + `in` / `not_in`。
- 「没 / 未 / 没有 + 事件」（如「未登录过」）→ `operator=not_executed`，`value=null`，并给一个合理时间窗。
- 提到的属性若不在字典里，AI 会**跳过该条件**（不会编造字段）。
- value 类型自动与字段 `value_kind` 对齐：`int/float` 给数字，`bool` 给 `true/false`，`string` 给字符串。

#### 2.7.4 实战示例（自然语言 → 规则卡片）

**示例 A：近 30 天充值大于 500 元的 VIP 用户**（前端默认模板）
- 输入：`近30天充值大于500元的VIP用户`
- 生成（渲染为条件组 AND）：
  - 画像 `vip_level` `gt` `0`（VIP 用户）
  - 行为 `recharge`，时间窗 `past_30_days`，数值 `gt` `500`
- 说明：充值类金额比较会在行为条件上挂「聚合度量」（count/sum + 阈值）渲染，运营可在卡片上继续微调。

**示例 B：过去 7 天浏览商品但未下单的用户**（前端默认模板）
- 输入：`过去7天浏览商品但未下单的用户`
- 生成（条件组 AND）：
  - 行为 `page_view`，时间窗 `past_7_days`（浏览商品）
  - 行为 `order_pay`，时间窗 `past_7_days`，`not_executed`（从未下单）

**示例 C：位于中国且系统为 iOS 的高价值用户**（前端默认模板）
- 输入：`位于中国且系统为iOS的高价值用户`
- 生成（条件组 AND）：
  - 画像 `last_country` `in` `["CN"]`
  - 画像 `device_os` `eq` `["iOS"]`
  - 画像 `is_high_value` `eq` `true`

**示例 D：未登录过的用户**
- 输入：`最近30天从未登录过的用户`
- 生成（条件组 AND）：
  - 行为 `login`，时间窗 `past_30_days`，`not_executed`，`value=null`

**示例 E：多个国家 + 高 LTV 的组合**
- 输入：`美国和日本的高价值用户，且累计充值LTV大于1000`
- 生成（条件组 AND）：
  - 画像 `last_country` `in` `["US","JP"]`
  - 画像 `is_high_value` `eq` `true`
  - 画像 `total_revenue` `gt` `1000`

#### 2.7.5 技术参考（部署 / 联调，给技术）

- **独立服务**：AI 圈人由独立的 AI 微服务（`ai-service/`）实现，**与主后端（:8080）不同服务器、不同代理**。服务监听 `:9000`，路由 `POST /api/v1/ai/text-to-rule`，健康检查 `GET /health`。
- **模型**：默认调用 **DeepSeek-V3**（`deepseek-chat`，OpenAI 兼容协议）。通过 `.env` 配置 `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL` / `DEEPSEEK_MODEL`；设 `AI_MOCK=1` 或无 key 时走关键词 Mock，便于跑通链路。
- **零依赖**：`ai-service` 用 Python 标准库（http.server + urllib）实现，无需 `pip install`，直接 `python3 app/main.py` 即可启动。
- **代理分流**：开发态前端 dev server 已配置两条代理——`/api/v1/ai/text-to-rule` → `:9000`(AI 服务)，`/api` → 主后端 `:8080`；生产由 Nginx 按相同路径分流到两个 upstream，前端无需改动。
- **字典对齐（关键）**：`ai-service/app/dictionary.py` 的 `PROFILE_FIELDS`（18 个画像字段）与 `EVENTS`（兜底事件，运行时动态同步主后端标准事件）**必须与前端 `RuleExpressionEditor.profileConfig` 及「标准字段 / 标准事件」模块保持口径一致**，否则 AI 产出的 key/operator 在前端无法正确渲染。
- **服务端校验闸**：LLM 原始产出在返回前端前会经 `validate.py` 做归一化+丢弃非法项（未知字段 / 不支持的操作符 / 类型无法转换 / between 缺两值），保证前端 100% 可渲染；全部丢弃时返回空 rules（前端提示「未找到相关属性/行为，请重新描述」）。

---

## 三、数据源管理（概览）

数据源定义「用户行为数据从哪里来、怎么解析、怎么清洗」。新建为 4 步向导：

1. **基本配置**：名称、来源类型（RTB / MMP / SDK / File）、接入方式（API / Postback / Upload）、状态（禁用 / 启用 / 异常）、鉴权 Token、IP 白名单。
2. **字段映射规则**：数据根节点（如 `payload`）、需 JSON 解析的字段、`字段映射`（标准字段 → 源字段路径）。
   - ⚠️ 标准字段下拉显示「名称 (code)」，但提交时使用的是**标准字段的名称（name）**，如 `ip`，不是 `FIELD_IP`。
3. **清洗规则**：数据模式（字段类型/必填/Trim/默认值）、标准化器（device_identity / network_ip / event_time / geo_country）、事件映射、丢弃过滤。
4. **下游策略**：去重策略（滑动窗口 / 自然周期）、聚合风控（监控窗口 + 阈值 + 动作 + 通知 Webhook）。

---

## 四、广告活动（概览）

广告活动是投放单元，可关联人群包进行定向。

- **状态**：暂停(0) / 运行中(1) / 已结束(2) / 待审核(3) / 熔断(4)。
- **必填**：包名（package_name）；**计费**：payout_amount 单位为 USD。
- **关联人群包**：在广告活动列表操作列点「关联」，弹出关联抽屉，可增删关联、停用（解绑）关联。
- **延迟策略（delay_strategy）**：已做成结构化表单，运营无需手写 JSON，可选：
  - `immediate` 立即
  - `random_range` 随机区间
  - `adaptive` 自适应
  - `adaptive_heatmap` 自适应热力图

---

## 五、广告主管理（概览）

管理广告主账户信息。列表的「创建时间」列内容不换行，超长时省略号截断，**鼠标悬停该单元格可查看完整时间**。

---

## 六、标准字段 / 标准事件 / 用户管理（简述）

- **标准字段**：维护标准字段字典（name / code / 类型等），供数据源字段映射与人群包画像维度引用。
- **标准事件**：维护标准事件字典（event_value / description）。人群包「行为条件」的事件类型即从此处动态加载。
- **用户管理**：后台账号与权限维护。

---

## 七、常见问题（FAQ）

**Q1：多国家怎么填？**
用 `in` 操作符，值框填 `US, CN, JP`（逗号分隔），不要用 `eq`。

**Q2：填了多个值但圈选不到人？**
检查是否误用了 `eq` 操作符——`eq` 会把整串当单值。改 `in` 即可。

**Q3：行为条件的「聚合度量」是干嘛的？**
用于把「事件」转成可比较的量化指标，例如「近 30 天 purchase 次数 ≥ 3」就靠 `count + gte 3` 实现。

**Q4：为什么编辑时某些状态选不了？**
状态机限制了合法流转，下拉只显示允许的目标状态。

**Q5：人群包关联活动后如何解绑？**
在关联抽屉里对关联记录做「停用」（status=2），不是物理删除。

**Q6：静态策略和动态策略的刷新时间怎么填？**
静态不填；每天填 `03:00`；每小时填 `:30`（见 2.2 节）。

**Q7：AI 圈人和手工写规则有什么区别？**
没有本质区别——AI 只是把自然语言「翻译」成与手工完全同构的规则卡片，生成后你仍然可以像手工规则一样任意增删、嵌套、微调，最后一起提交。AI 生成会**整体覆盖**当前规则表达式，生成前请确认下方没有要保留的手动规则。

**Q8：AI 生成后提示「未找到相关属性/行为」？**
通常是描述里涉及到的属性/事件不在 AI 的已知字典中（如自定义字段尚未录入标准字段，或事件未登记到「标准事件」）。换更通用的说法、或先去「标准字段 / 标准事件」补充字典，再试一次。

---

> 文档依据前端 `dmp-backend-frontend` 实际实现整理，若后端字段有调整，以接口文档为准。
