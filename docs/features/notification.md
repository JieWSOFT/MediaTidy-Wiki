# 通知服务

MediaTidy 支持多种通知渠道，在关键事件发生时推送消息。每个渠道可独立配置订阅的事件类型。

## 通知渠道

支持四种通知渠道，可同时添加多个：

### Bark

iOS 推送通知服务。

| 配置项 | 说明 |
|--------|------|
| **Server URL** | Bark 服务器地址 |
| **Device Key** | 设备推送 Key |

### Telegram Bot

通过 Telegram Bot 发送通知消息，支持富文本格式和海报图片。

| 配置项 | 说明 |
|--------|------|
| **Bot Token** | 通过 [@BotFather](https://t.me/BotFather) 创建 Bot 获取 |
| **Chat ID** | 接收消息的聊天 ID（个人或群组） |

### 企业微信 (WeCom)

通过企业微信 Webhook 推送消息。

| 配置项 | 说明 |
|--------|------|
| **Webhook URL** | 企业微信群机器人的 Webhook 地址 |

### Webhook

通过 HTTP POST 请求推送到自定义 URL，适合对接第三方系统。

| 配置项 | 说明 |
|--------|------|
| **URL** | 接收通知的 Webhook URL |
| **Headers** | 自定义请求头（如 `Authorization`） |

---

## 渠道配置

每个渠道有以下通用字段：

| 字段 | 说明 |
|------|------|
| `id` | 渠道唯一 ID（自动生成） |
| `type` | 渠道类型：`bark` / `telegram` / `wecom` / `webhook` |
| `name` | 渠道显示名称 |
| `enabled` | 是否启用 |
| `config` | 渠道特有配置（Key-Value） |
| `events` | 订阅的事件类型列表 |

---

## 通知事件

每个渠道可独立选择订阅的事件：

| 事件类型 | 事件 ID | 说明 |
|----------|---------|------|
| **整理任务完成** | `task.done` | 整理任务完成时通知，包含总文件数、成功/失败/跳过/升级统计 |
| **单项整理完成** | `task.item` | 每部电影/剧集转移完成后立即通知，包含海报、评分、质量信息 |
| **STRM 生成完成** | `strm.done` | STRM 全量生成完成时通知 |
| **系统错误** | `system.error` | 严重错误（如风控触发、API 失败）时通知 |
| **Emby 删除** | `emby.delete` | Emby 媒体删除事件（含深度删除） |
| **Emby 入库** | `emby.library` | Emby 新增媒体入库通知 |
| **Emby 播放开始** | `emby.playback.start` | 用户开始播放 |
| **Emby 播放停止** | `emby.playback.stop` | 用户停止播放 |
| **Emby 测试** | `emby.test` | Emby Webhook 连接测试 |

::: tip 自动启用新事件
系统升级后如果新增了事件类型，会自动为所有已有渠道启用新事件，无需手动配置。
:::

---

## 通知模板

每种事件类型支持自定义标题和正文模板，使用 Go text/template 语法。

### task.item 模板变量（单项整理通知）

| 变量 | 说明 | 示例 |
|------|------|------|
| `Title` | 媒体标题 | 流浪地球 |
| `Year` | 年份 | 2019 |
| `MediaType` | 媒体类型 | movie / tv |
| `Episodes` | 集信息（剧集） | S01E01-E03 |
| `PosterURL` | TMDB 海报图片 URL | — |
| `Rating` | TMDB 评分 | 8.5 |
| `Genres` | 类型 | 动作 / 惊悚 |
| `Resolution` | 分辨率 | 4K, 1080p |
| `DynamicRange` | HDR 类型 | Dolby Vision P8 |
| `AudioFormat` | 音频格式 | Dolby TrueHD Atmos 7.1 |
| `Duration` | 处理耗时 | 12s |

### emby.library 模板变量（入库通知）

| 变量 | 说明 |
|------|------|
| `ItemName` | 媒体名称 |
| `ItemType` | 类型（Movie, Series, Episode） |
| `ServerName` | Emby 服务器名 |
| `Rating` | 评分 |
| `Genres` | 类型标签 |
| `Runtime` | 时长（如 128分钟） |
| `ImageURL` | Emby 封面图 URL |

### emby.playback 模板变量（播放通知）

| 变量 | 说明 |
|------|------|
| `UserName` | 播放用户名 |
| `DeviceName` | 设备名 |
| `ClientName` | 客户端名 |
| `ItemName` | 媒体名称 |
| `Progress` | 播放进度（0-100） |

---

## 海报卡片

`task.item` 事件触发时，如果 `PosterURL` 非空，支持的渠道（如 Telegram）会自动发送带海报图片的通知卡片，包含：

- TMDB 海报/背景图
- 标题、年份、评分
- 分辨率、HDR、音频格式
- 类型标签和简介

---

## Telegram Bot 交互

除了接收通知，Telegram Bot 还支持交互式命令：

- 搜索媒体信息
- 查看任务状态
- 触发手动整理
