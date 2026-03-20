# claude-qq-channel

QQ Bot channel plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — 将 QQ 私聊消息桥接到 Claude Code 会话，支持双向对话、文件附件和完整的访问控制。

## 工作原理

```
QQ 用户 ──私聊──▶ QQ 开放平台 ──WebSocket──▶ channel server ──stdio──▶ Claude Code
                                                     ◀── reply tool ──
```

- channel server 作为 MCP 子进程由 Claude Code 启动，通过 WebSocket 连接 QQ 开放平台网关
- 收到私聊消息后，经白名单校验，以 `<channel>` 事件推送到 Claude Code 会话
- Claude 通过 `reply` tool 回复，消息经 QQ Bot API 发回给用户
- 仅支持 C2C（单聊），不支持群聊

## 前置条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) v2.1.80 或更高版本，使用 claude.ai 账号登录 (或者参考这里绕过 [linuxdo-哈雷佬-patch](https://linux.do/t/topic/1787422))
- [Bun](https://bun.sh) 运行时 — 安装: `curl -fsSL https://bun.sh/install | bash`
- QQ 开放平台的 Bot AppID 和 AppSecret（见下文获取方式）

## 快速开始

**1. 获取 QQ Bot 凭证**

前往 [QQ 开放平台](https://q.qq.com/) 点击龙虾专用入口，扫码登录。创建一个机器人应用，在应用管理页面获取 **AppID** 和 **AppSecret**。

**2. 安装插件**

以下为 Claude Code 命令 — 先运行 `claude` 启动会话。

```
/plugin install github:hank9999/claude-qq-channel
```

**3. 配置凭证**

```
/qqbot:configure <AppID>:<AppSecret>
```

凭证保存到 `~/.claude/channels/qqbot/.env`。也可以手动写入该文件，或在启动 Claude Code 前设置环境变量 `QQBOT_CREDENTIALS=AppID:AppSecret`。

**4. 启动 channel**

退出当前会话，使用 channel 标志重新启动：

```sh
claude --dangerously-load-development-channels plugin:qqbot
```

**5. 配对**

在上一步启动的 Claude Code 会话中，用 QQ 向你的 Bot 发送任意私聊消息 — Bot 会回复一个配对码。在 Claude Code 中运行：

```
/qqbot:access pair <code>
```

你的下一条 QQ 私聊消息就会到达 Claude。

**6. 锁定访问**

配对完成后，切换到 `allowlist` 模式，阻止陌生人触发配对码：

```
/qqbot:access policy allowlist
```

## 配对与访问控制

channel 采用白名单机制保护：只有经过配对批准的 QQ 用户才能向 Claude 发送消息。

### 访问管理命令

| 命令 | 说明 |
|---|---|
| `/qqbot:access` | 查看当前状态：策略、白名单、待配对列表 |
| `/qqbot:access pair <code>` | 批准配对请求 |
| `/qqbot:access deny <code>` | 拒绝配对请求 |
| `/qqbot:access allow <openid>` | 手动添加用户到白名单 |
| `/qqbot:access remove <openid>` | 从白名单移除用户 |
| `/qqbot:access policy <mode>` | 设置策略：`pairing`、`allowlist`、`disabled` |

### DM 策略说明

| 策略 | 行为 |
|---|---|
| `pairing`（默认） | 未知用户发消息会收到配对码，等待终端批准。适合初始设置阶段 |
| `allowlist` | 仅白名单中的用户可发消息，其他人静默丢弃。**推荐的长期策略** |
| `disabled` | 所有消息静默丢弃 |

## 提供给 Claude 的工具

| 工具 | 用途 |
|---|---|
| `reply` | 回复 QQ 消息。传入 `chat_id` + `text`，可选 `msg_id`（被动回复）和 `files`（绝对路径，附件）。长文本自动分段（单条上限 2000 字符），支持图片和文件附件，单文件上限 50MB。 |
| `download_attachment` | 下载 QQ 消息中的附件到 `~/.claude/channels/qqbot/inbox/`。当 `<channel>` 标签包含 `attachment_count` 时使用。 |

收到消息时自动发送输入状态提示。

## 附件

附件**不会**自动下载。`<channel>` 通知中列出附件名称、类型和大小 — Claude 在需要时调用 `download_attachment` 获取文件，下载到 `~/.claude/channels/qqbot/inbox/`。

## 数据存储

所有状态文件位于 `~/.claude/channels/qqbot/`：

| 文件 | 说明 |
|---|---|
| `.env` | Bot 凭证（`QQBOT_CREDENTIALS=AppID:AppSecret`） |
| `access.json` | 访问控制状态：策略、白名单、待配对列表 |
| `session.json` | WebSocket session 信息（用于断线恢复） |
| `inbox/` | 下载的附件文件 |
| `approved/` | 配对批准标记（channel server 轮询后自动清理） |

## 安全注意事项

- **白名单是必须的**：未配对的用户无法向 Claude 注入任何内容。始终在完成配对后切换到 `allowlist` 策略
- **配对码不可自动批准**：即使只有一个待配对请求，也必须由用户在终端中手动输入配对码确认。这防止通过 channel 消息进行 prompt injection 来自动批准攻击者
- **凭证安全**：`.env` 文件权限为 600，仅当前用户可读。不要将凭证提交到版本控制
- **附件保护**：`reply` tool 禁止发送 channel 状态目录中的文件（`inbox/` 除外），防止泄露 `access.json` 等敏感信息

## 凭证更新后不生效

channel server 在启动时读取 `.env`。更新凭证后需要重启 Claude Code 会话，或运行 `/reload-plugins`。

## License

[MIT](./LICENSE)
