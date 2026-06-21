# Telegram 图床机器人 Worker

基于 Cloudflare Workers 的 Telegram Bot，接收用户发送的文件，自动上传到图床，并返回访问链接。支持图片、视频、音频、文档等 400+ 种格式，可选将图片转换为 WebP 后再上传。

---

## 环境变量

### 必填

| 变量 | 说明 |
|------|------|
| `BOT_TOKEN` | Telegram Bot Token，从 [@BotFather](https://t.me/BotFather) 获取 |
| `IMG_BED_URL` | 图床上传接口地址，例如 `https://your-imgbed.com/api/upload` |

### 选填 — 访问控制

| 变量 | 说明 |
|------|------|
| `AUTH_CODE` | 图床认证密码，有则自动附带到请求头 `X-Access-Password` |
| `ADMIN_USERS` | 管理员用户 ID，多个用英文逗号分隔，例如 `123456,789012` |
| `ADMIN_ONLY` | 设为 `true` 则只有管理员可以使用机器人 |

### 选填 — 存储（用于统计和分片上传）

| 变量 | 说明 |
|------|------|
| `STATS_STORAGE` | 绑定的 KV 命名空间，用于统计数据、历史记录、分片上传临时数据 |

### 选填 — WebP 转换

| 变量 | 说明 |
|------|------|
| `WEBP_CONVERT` | 填 `imgproxy` 或 `cloudinary`，不填则不转换，直接上传原图 |
| `WEBP_QUALITY` | WebP 质量，范围 1–100，默认 `85`。使用 imgproxy 时填 `101` 时可开启无损模式 |

**使用 imgproxy 时额外需要：**

| 变量 | 说明 |
|------|------|
| `IMGPROXY_URL` | imgproxy 服务地址，例如 `https://imgproxy.example.com` |
| `IMGPROXY_KEY` | imgproxy 签名 Key（Hex 格式），不填则降级为 insecure 模式 |
| `IMGPROXY_SALT` | imgproxy 签名 Salt（Hex 格式），不填则降级为 insecure 模式 |

**使用 Cloudinary 时额外需要：**

| 变量 | 说明 |
|------|------|
| `CLOUDINARY_CLOUD_NAME` | Cloudinary 控制台的 Cloud Name |
| `CLOUDINARY_API_KEY` | Cloudinary API Key |
| `CLOUDINARY_API_SECRET` | Cloudinary API Secret |
| `CLOUDINARY_DELETE_AFTER_CONVERT` | 设为 `false` 可保留 Cloudinary 上的临时图片，默认转换完成后自动删除 |

---

## 部署步骤

### 1. 创建 Worker

在 [Cloudflare Dashboard](https://dash.cloudflare.com/) 中新建一个 Worker，将 `worker.js` 的内容粘贴进去并保存。

### 2. 配置环境变量

在 Worker 的 **Settings → Variables** 中添加上方必填变量，以及需要的可选变量。

`CLOUDINARY_API_SECRET` 等敏感值建议使用 **Encrypt** 加密存储。

### 3. 绑定 KV（可选）

如需统计功能或分片上传，在 **Settings → Variables → KV Namespace Bindings** 中新建一个 KV 命名空间，变量名填 `STATS_STORAGE`。

### 4. 注册 Webhook

Worker 部署后，访问以下地址完成 Webhook 注册（只需执行一次）：

```
https://<your-worker>.workers.dev/setup-webhook
```

返回 `Webhook设置成功` 即代表注册完成，此后 Telegram 消息会自动推送到 Worker。

---

## 用户命令

| 命令 | 说明 |
|------|------|
| `/start` | 启动机器人 |
| `/help` | 查看使用说明 |
| `/formats` | 查看支持的文件格式 |
| `/analytics` | 查看个人上传统计 |
| `/analytics storage` | 查看存储占用 |
| `/analytics daily/weekly/monthly` | 查看各周期使用报告 |
| `/history` | 查看上传历史（支持翻页、类型筛选、关键词搜索） |
| `/history p2 image` | 第 2 页、仅显示图片 |
| `/history search:关键词` | 按文件名搜索 |
| `/history desc:关键词` | 按备注搜索 |
| `/chunk_upload` | 启动分片上传模式，突破 20 MB 限制 |
| `/chunk_cancel` | 取消当前分片上传 |

---

## 管理员命令

以下命令仅 `ADMIN_USERS` 中的用户可用：

| 命令 | 说明 |
|------|------|
| `/admin` | 查看管理员命令列表 |
| `/admin ban <用户ID> [原因]` | 限制指定用户 |
| `/admin unban <用户ID>` | 解除限制 |
| `/admin list` | 查看所有被限制的用户 |
| `/admin users [页码]` | 查看所有使用过机器人的用户 |
| `/admin stats` | 查看全局统计数据 |
| `/admin broadcast <消息>` | 向所有用户广播消息 |
| `/admin autoclean <天数>` | 设置自动删除 N 天前的历史记录（填 `0` 禁用） |
| `/admin autoclean status` | 查看当前自动清理设置 |

---

## WebP 转换说明

发送图片时，如果配置了 `WEBP_CONVERT`，机器人会在上传前将图片转为 WebP 格式，通常可减小 30%–60% 的文件体积。转换仅作用于静态图片（jpg/jpeg/png/bmp/tiff/heic/heif），GIF、视频、文档等不受影响。

**两个后端的区别：**

- **imgproxy**：需要自行部署服务，转换在自己的服务器上完成，数据不经过第三方。支持无损模式（`WEBP_QUALITY=101`）。
- **Cloudinary**：免费额度内开箱即用，无需额外部署。图片会上传到 Cloudinary 完成转换后被拉取回来，默认转换完毕后自动删除，不占用存储配额。

---

## 注意事项

- Telegram Bot 单次文件上传上限为 **20 MB**，超出限制请使用 `/chunk_upload` 分片上传。
- 发送文件时附带的文字会作为备注保存，可通过 `/history desc:关键词` 搜索。
- 分片上传需要绑定 KV，且每个分片也受 20 MB 限制，请按实际文件大小合理分片。
