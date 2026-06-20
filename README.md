# TG-FileBed-Bot

一个运行在 Cloudflare Workers 上的 Telegram 机器人，用于将 Telegram 中的文件自动上传到自建图床，并返回永久访问链接。支持图片、视频、音频、文档等 300+ 种格式，以及大文件分片上传。

---

## 功能特性

- **全格式支持**：图片、视频、音频、动画/GIF、文档、压缩包、安装包、光盘镜像、3D文件、源代码等 300+ 种格式
- **WebP 自动转换**：可选接入 imgproxy，将上传图片自动转为 WebP 格式以节省空间
- **分片上传**：支持将超过 20MB 的大文件拆分为多个分片依次发送，自动合并后上传
- **文件备注**：发送文件时附带的文字说明会作为备注保存，支持按备注搜索
- **上传历史**：查询个人上传记录，支持按类型过滤、关键词搜索、分页浏览
- **统计分析**：查看上传数量、存储用量、成功率、日/周/月报告
- **管理员功能**：禁封/解封用户、广播消息、查看全局统计、设置自动清理
- **仅管理员模式**：可限制只有指定用户才能使用机器人

---

## 部署步骤

### 1. 创建 Cloudflare Worker

登录 [Cloudflare Dashboard](https://dash.cloudflare.com)，进入 **Workers & Pages** → **Create**，创建一个新的 Worker，将 `worker.js` 的内容粘贴进去并保存。

### 2. 创建 KV 命名空间

进入 **Workers & Pages** → **KV**，新建一个命名空间（例如命名为 `STATS_STORAGE`）。

回到 Worker 的设置页，在 **Settings → Bindings** 中添加 KV 绑定：

| 变量名 | KV 命名空间 |
|--------|------------|
| `STATS_STORAGE` | 刚创建的命名空间 |

### 3. 配置环境变量

在 Worker 的 **Settings → Variables** 中添加以下环境变量：

#### 必填

| 变量名 | 说明 |
|--------|------|
| `BOT_TOKEN` | Telegram Bot Token，从 [@BotFather](https://t.me/BotFather) 获取 |
| `IMG_BED_URL` | 图床上传接口地址，例如 `https://your-imgbed.com/api/upload` |

#### 选填

| 变量名 | 说明 |
|--------|------|
| `AUTH_CODE` | 图床访问密码，有密码保护时填写，通过 `X-Access-Password` 请求头传递 |
| `ADMIN_USERS` | 管理员用户 ID 列表，多个用逗号分隔，例如 `123456,789012` |
| `ADMIN_ONLY` | 设为 `true` 则只有管理员可以使用机器人 |
| `ADMIN_CHAT_ID` | 内部错误通知发送目标的 Chat ID，不填则发给触发错误的用户 |
| `BOT_USERNAME` | Bot 的用户名（不含 @），用于群组内命令识别，例如 `MyUploadBot` |
| `ENABLE_WEBP_CONVERT` | 设为 `true` 开启 WebP 自动转换（需同时配置 imgproxy） |
| `IMGPROXY_URL` | imgproxy 服务地址，例如 `https://imgproxy.example.com` |
| `IMGPROXY_KEY` | imgproxy 签名 Key（Hex 格式），未配置则使用 insecure 模式 |
| `IMGPROXY_SALT` | imgproxy 签名 Salt（Hex 格式），未配置则使用 insecure 模式 |

### 4. 绑定 Webhook

部署完成后，在浏览器访问以下地址完成 Webhook 注册：

```
https://<your-worker-domain>/setup-webhook
```

成功后会返回 `Webhook设置成功: https://...`，之后 Telegram 消息会自动推送到 Worker。

---

## 用户命令

| 命令 | 说明 |
|------|------|
| `/start` | 启动机器人，注册用户信息 |
| `/help` | 查看使用说明 |
| `/formats` | 查看支持的文件格式列表 |
| `/analytics` | 查看综合上传统计 |
| `/analytics storage` | 查看存储空间使用情况 |
| `/analytics daily` | 查看今日上传报告 |
| `/analytics weekly` | 查看近 7 天报告 |
| `/analytics monthly` | 查看近 30 天报告 |
| `/analytics success` | 查看上传成功率分析 |
| `/history` | 查看上传历史（最近 5 条） |
| `/history page2` | 查看第 2 页历史 |
| `/history image` | 只显示图片记录 |
| `/history search:关键词` | 按文件名或备注搜索 |
| `/history desc:关键词` | 只按备注搜索 |
| `/chunk_upload 3 file.zip` | 启动分片上传，共 3 片，文件名 file.zip |
| `/chunk_cancel` | 取消当前进行中的分片上传 |

---

## 管理员命令

所有管理员命令格式为 `/admin <子命令> [参数]`，需要用户 ID 在 `ADMIN_USERS` 中。

| 命令 | 说明 |
|------|------|
| `/admin` | 显示管理员命令帮助 |
| `/admin ban <用户ID> [原因]` | 禁止指定用户使用机器人 |
| `/admin unban <用户ID>` | 解除对指定用户的封禁 |
| `/admin list` | 查看所有被封禁的用户及原因 |
| `/admin users` | 查看所有使用过机器人的用户列表（支持翻页） |
| `/admin users 2` | 查看用户列表第 2 页 |
| `/admin stats` | 查看全局使用统计 |
| `/admin broadcast <消息>` | 向所有用户发送广播消息 |
| `/admin autoclean <天数>` | 设置自动清理 N 天前的上传历史（0 为关闭） |
| `/admin autoclean status` | 查看当前自动清理设置 |

---

## 分片上传说明

Telegram Bot API 单文件限制 20MB。对于更大的文件，可使用分片上传：

1. 在电脑上将大文件切分为若干小于 20MB 的分片（可使用 `split` 命令或 7-Zip 等工具）
2. 发送命令 `/chunk_upload 分片数量 文件名`，例如：
   ```
   /chunk_upload 5 large_video.mp4
   ```
3. 按顺序逐个发送每个分片文件
4. 所有分片收齐后，机器人自动合并并上传，返回最终链接

---

## 图床接口兼容

图床上传接口需满足以下条件：

- 接受 `multipart/form-data` 格式的 POST 请求，文件字段名为 `file`
- 请求 URL 附带 `?returnFormat=full` 参数
- 如配置了 `AUTH_CODE`，认证信息通过 `X-Access-Password` 请求头传递
- 返回 JSON，包含文件访问 URL（支持 `url`、`src`、`data.url` 等多种响应结构）

兼容 [Lsky Pro](https://github.com/lsky-org/lsky-pro)、[Telegraph](https://telegra.ph) 等主流图床的 API 格式。

---

## 注意事项

- Cloudflare Workers KV 免费套餐每天有 10 万次读写限制，用户量较大时建议升级或减少统计频率
- 分片合并时数据暂存于 KV，单个 Value 上限为 25MB，因此每个分片应控制在 20MB 以内
- 自动清理功能在每次收到请求时触发检查，但两次清理之间至少间隔 6 小时
- Telegram 通过 Bot API 下载文件的链接有效期有限，建议及时处理
