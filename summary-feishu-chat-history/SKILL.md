---
name: summary-feishu-chat-history
description: 拉取飞书会话历史消息，输出全量消息表格，并生成可交互统计（如按月按人发言频次）的总结技能。
---

# summary-feishu-chat-history

用于从飞书会话历史消息中生成两类结果：
1) **全量消息表格**（逐条消息）
2) **可交互统计视图**（默认按“月份 × 发言人”统计）

## 触发条件
当用户希望“读取/导出/总结飞书聊天记录”，或希望“看发言统计（按月、按人、按群）”时使用。

## 需要用户提供的参数
必填：
- `tenant_access_token`
- `container_id_type`：`chat` 或 `thread`
- `container_id`

选填：
- `start_time`、`end_time`（秒级；仅 `chat` 支持）
- `sort_type`（默认 `ByCreateTimeAsc`）
- `timezone`（默认 `Asia/Shanghai`）
- `language`（默认中文）

若缺失必填参数，先追问后再执行。

## API 调用规范
- Method: `GET`
- URL: `https://open.feishu.cn/open-apis/im/v1/messages`
- Header: `Authorization: Bearer <tenant_access_token>`
- Query:
  - 必填：`container_id_type`, `container_id`
  - 选填：`start_time`, `end_time`, `sort_type`, `page_size`, `page_token`

执行要求：
1. `page_size=50`。
2. 首次请求不带 `page_token`，循环拉取直到 `has_more=false`。
3. 若 `container_id_type=thread`，忽略时间范围并提示用户该限制。

## 数据清洗与标准化
对每条消息提取并标准化以下字段：
- `message_id`
- `create_time`（原始毫秒时间戳）
- `create_time_local`（按 `timezone` 转换后的可读时间）
- `chat_id`
- `thread_id`
- `root_id`
- `parent_id`
- `sender_id`（`sender.id`）
- `sender_type`
- `msg_type`
- `text`（从 `body.content` 解析；无文本则为空）
- `mentions_count`
- `deleted`
- `updated`

说明：
- `deleted=true` 的消息**不删除**，保留在全量表中，并在统计时默认排除（除非用户要求纳入）。
- `body.content` 是 JSON 字符串，解析失败时保留原串并标记 `parse_error=true`。

## 输出要求（必须同时满足）

### A. 全量历史消息表格（必出）
按消息时间排序，输出 Markdown 表格（长会话可分段输出）。

**表头固定为：**
`message_id | create_time_local | sender_id | sender_type | msg_type | text | mentions_count | deleted | thread_id`

### B. 可交互统计（必出）
至少输出以下两种可交互载体之一（默认两种都给）：
1. **Vega-Lite 配置 JSON**（可直接粘贴到支持 Vega-Lite 的渲染器）
2. **Plotly HTML 片段**（可在浏览器交互悬浮查看）

并同时输出其底层明细表：
- `month`（YYYY-MM）
- `sender_id`
- `message_count`

默认统计口径：
- 仅统计 `deleted=false`
- 按 `create_time_local` 所在月份聚合
- 指标为“发言条数”（message count）

## 推荐统计项
除“每月每人发言频次”外，可按需追加：
- 每日消息量趋势
- 不同 `msg_type` 占比
- 活跃度 TopN 发言人
- @提及次数 TopN（基于 `mentions_count`）

## 摘要结构
在表格和统计之后，再给简要总结：
1. TL;DR（3 条）
2. 关键决策
3. 关键进展
4. 风险/阻塞
5. 待办（负责人/截止时间/状态）
6. 未决问题

## 输出模板（精简）
```markdown
# 飞书会话结果
- 容器：{{container_id_type}} / {{container_id}}
- 时间范围：{{time_range}}
- 总消息数：{{total_count}}
- 删除消息数：{{deleted_count}}

## 1) 全量历史消息表格
| message_id | create_time_local | sender_id | sender_type | msg_type | text | mentions_count | deleted | thread_id |
|---|---|---|---|---|---|---:|---|---|
| ... |

## 2) 统计明细（每月每人发言频次）
| month | sender_id | message_count |
|---|---|---:|
| 2026-01 | ou_xxx | 128 |

## 3) 可交互统计
### Vega-Lite JSON
```json
{{vega_lite_spec_json}}
```

### Plotly HTML
```html
{{plotly_html_snippet}}
```

## 4) 摘要
### TL;DR
1. ...
2. ...
3. ...
```

## 常见错误处理
- `230001` 参数错误：回显关键参数并建议修正
- `230002` 机器人不在群：提示先拉机器人入群
- `230006` 未启用机器人能力：提示开启机器人能力
- `230027` 权限不足：检查 `im:message*` 及群消息权限

## 安全与隐私
- 不输出 access token。
- 如用户未要求实名，默认仅展示 `sender_id`。
- 敏感文本先脱敏后输出（如密钥、手机号、身份证号）。
