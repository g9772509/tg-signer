# 自动化规则/Handler 扩展设计草案（对话总结）

本文档整理了关于“可扩展 Monitor/Talker、支持定时触发与可插拔 handler、低代码/UI 配置”的需求与方案，作为后续实现的指导文档。

## 1. 背景需求（用户诉求）

### 1.1 签到场景（更细粒度的冷却时间）
- Bot 回复 “签到成功，你需要 X 分钟后才能再次签到”，X 为随机数。
- 需要解析 X，并按 **X + Y** 分钟后再次签到（Y 为用户固定偏移）。
- Bot 的回复是可跳转 reply，需要避免误识别他人的签到回复。
- 需要支持“指定回复模式”（引用某条消息进行回复）。

### 1.2 自动回复场景（通用化）
- **自动发言**：每隔一段时间（定时任务）从发言池随机取一句发送。
- **AI 聊天**：每隔一段时间获取群聊最新几条消息，进行 AI 回复。
- 黑名单过滤：如果 AI 回复命中黑名单则不发送。

### 1.3 产品方向
以上需求较具体，倾向于抽象为一个“规则引擎 + 可插拔 handler”的通用框架：
- 支持定时触发与消息触发；
- 每条消息可应用一个或多个 handler；
- handler 可由用户扩展；
- 支持低代码或 UI 方式配置生成。

## 2. 初步方案（抽象架构）

### 2.1 事件模型（Event）
统一事件结构，支持：
- `message`：新消息事件
- `timer`：定时触发事件
- `startup`：启动事件

可包含字段：
- `event.type`
- `event.chat_id`
- `event.message`（仅 message 事件）
- `event.now`

### 2.2 规则模型（Rule）
每条规则描述“触发 + 过滤 + handler 链”：
- `id`
- `enabled`
- `triggers`（cron/interval/消息触发）
- `filters`（chat_id/from_user/regex/是否 reply to me）
- `handlers`（顺序执行）

### 2.3 Handler（低代码）
统一结构：
```
{"handler": "send_text", "params": {...}}
```
返回 `continue/stop/defer`，可读写 `context.vars`。

建议内置 handler：
- `send_text`: 发送文本
- `reply_text`: 回复消息（reply_to_message_id）
- `random_pick`: 从池子选一句
- `extract_regex`: 从消息里提取变量
- `delay`: 延迟
- `schedule_next`: 设置下次触发时间（支持 X+Y）
- `blacklist_filter`: 命中则停止
- `ai_reply`: 大模型回复
- `forward`: 转发
- `store_state` / `load_state`: 轻量状态读写

### 2.4 用户扩展 handler
两种方向：
1. 纯低代码：仅内置 handler
2. 低代码 + Python 插件：用户在 `~/.signer/handlers/` 写函数，通过配置引用

## 3. 典型需求映射到规则

### 3.1 签到冷却解析（X + Y）
可能规则链：
- `extract_regex` 提取 X
- `schedule_next` 计算 X + Y 并设定下次执行时间
- 可选 `reply_text` 进行指定回复

### 3.2 避免误识别他人回复
过滤策略优先级：
1. 只处理“reply_to 我刚发的签到消息”的回复
2. 同时校验来自指定 bot（user id/username）

### 3.3 自动发言/AI 闲聊
定时触发：
- `random_pick` 从发言池选句并发送
- `ai_reply` 根据最近 N 条消息生成回复
- `blacklist_filter` 过滤内容

## 4. UI/低代码配置设想

### 4.1 WebUI 结构
新增“自动化规则”Tab：
- 规则列表
- 规则详情
  - 触发器配置
  - 过滤条件配置
  - handler 列表（可拖拽排序）

### 4.2 JSON/YAML 直接配置
保留文本配置入口供高级用户使用。

## 5. 落地改动方向（代码层）

可能涉及模块：
- `tg_signer/config.py`：新增 `AutomationConfig/RuleConfig` 等模型
- `tg_signer/automation/engine.py`：自动化执行引擎与调度
- `tg_signer/automation/handlers.py`：内置 handler 与插件加载
- `tg_signer/cli/automation.py`：CLI 命令入口

## 6. 已确认决策
1. 用户扩展 handler 的方式：**低代码 + Python 插件**（默认内置 handler + 可插拔）。
2. 配置入口优先级：**JSON/YAML 优先**，WebUI 暂不扩展。
3. Monitor 保留为 legacy，但功能可由 automation 规则组合实现。

## 7. Monitor -> Automation 迁移映射（示例）
Monitor 示例配置：
```
{
  "chat_id": -100123456,
  "rule": "contains",
  "rule_value": "关键词",
  "default_send_text": "自动回复",
  "ai_reply": false,
  "send_text_search_regex": null,
  "external_forwards": [{"type": "http", "url": "http://127.0.0.1:8080"}],
  "push_via_server_chan": true
}
```

对应 automation 规则（核心映射）：
```
{
  "id": "monitor_like",
  "enabled": true,
  "triggers": [{"type": "message", "chat_id": -100123456}],
  "filters": {"text_rule": "contains", "text_value": "关键词"},
  "handlers": [
    {"handler": "send_text", "params": {"text": "自动回复"}},
    {"handler": "external_forward", "params": {"targets": [{"type": "http", "url": "http://127.0.0.1:8080"}]}},
    {"handler": "server_chan", "params": {"title": "Monitor", "body": "{message.text}"}}
  ]
}
```

## 8. 插件示例（最小）
在 `<workdir>/handlers/custom.py`：
```
async def echo(event, ctx, params):
  ctx.log(f"echo: {event.chat_id}")
  await ctx.worker.send_message(event.chat_id, params.get("text", "hello"))
  return "continue"

HANDLERS = {"echo": echo}
```

---
本草案已进入实现阶段，后续只需按模块逐步落地即可。
