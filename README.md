# 森空岛自动签到 · Cloudflare Workers

[English](./README_EN.md) | 简体中文

将森空岛自动签到运行在 Cloudflare Workers 上，无需自建服务器。Worker 通过 Cron Trigger 每天自动执行签到，也提供带鉴权的 HTTP 接口用于手动触发。

> [!IMPORTANT]
> 本项目是非官方工具，仅供学习和个人使用，与鹰角网络、森空岛及 Cloudflare 无关。账号 Token 等同于登录凭证，请自行评估使用风险，切勿将 Token 提交到 Git、粘贴到 Issue，或分享在日志和截图中。

## 功能

- 支持《明日方舟》和《明日方舟：终末地》森空岛签到
- 支持多个森空岛账号
- 使用 Cloudflare Cron Trigger 定时执行
- 支持 Server酱³、Qmsg 和 PushPlus 结果通知
- 支持通过带 Bearer 鉴权的 HTTP `POST` 请求手动触发
- 支持在手动请求的 JSON 请求体中临时传入森空岛 Token
- 使用 Cloudflare Secrets 保存 Token，不在源码和 Wrangler 配置中保存敏感信息

## 准备工作

- [Node.js](https://nodejs.org/) 当前 LTS 版本（建议 Node.js 20 或更高版本）
- npm
- 一个有效的森空岛 Token

只有部署线上 Worker 时才需要 [Cloudflare](https://dash.cloudflare.com/) 账号。本地运行不需要 Cloudflare 账号，也不需要执行 `wrangler login`。

## 获取森空岛 Token

1. 使用浏览器登录[森空岛官网](https://www.skland.com/)。

   ![森空岛官网](./assets/img.png)

2. 保持登录状态，访问 <https://web-api.skland.com/account/info/hg>。
3. 页面会返回 JSON。找到 `data.content` 字段，只复制它的字符串值，不要复制字段名、引号或整段 JSON。

返回内容的结构大致如下，其中 `这里才是需要复制的Token` 是示例占位符：

```json
{
  "code": 0,
  "data": {
    "content": "这里才是需要复制的Token"
  },
  "msg": "..."
}
```

> [!IMPORTANT]
> 请只在本地 `.dev.vars` 或 Cloudflare Secrets 中保存该值。不要将真实 Token 写入 `.dev.vars.example`、`wrangler.toml`、源代码或文档。

## 部署

### 1. 获取代码并安装依赖

```bash
git clone https://github.com/xiaoyulejia/skyland-auto-sign-CFworker.git
cd skyland-auto-sign-CFworker
npm ci
npm run check
```

### 2. 创建本地 Secret 文件

Windows PowerShell：

```powershell
Copy-Item .dev.vars.example .dev.vars
```

macOS / Linux：

```bash
cp .dev.vars.example .dev.vars
```

编辑 `.dev.vars`：

```dotenv
TOKEN="你的森空岛Token"
WORKER_AUTH="一段足够长且随机的字符串"
```

默认的 Cron 定时签到需要同时配置 `TOKEN` 和 `WORKER_AUTH`：

- `TOKEN`：Cron 定时签到使用的森空岛登录凭证。如果只使用 HTTP 请求体临时传入 Token，可以不配置该 Secret。
- `WORKER_AUTH`：保护 HTTP 手动触发接口的独立密码，请勿与森空岛 Token 相同；使用 HTTP 接口时必须配置。

多账号可用英文逗号分隔：

```dotenv
TOKEN="第一个Token,第二个Token,第三个Token"
```

如需结果推送，可在 `.dev.vars` 中配置对应项目：

```dotenv
SC3_SENDKEY="Server酱³ SendKey"
SC3_UID="Server酱³ UID，可留空"
QMSG_KEY="Qmsg Key"
PUSHPLUS_KEY="PushPlus Token"
```

`.dev.vars` 已被 `.gitignore` 排除。提交前仍建议运行 `git status --short`，确认它没有出现在待提交文件中。

### 3. 本地测试

```bash
npm run dev
```

Wrangler 默认监听 `http://localhost:8787`。在另一个终端中手动触发：

```bash
curl -X POST "http://localhost:8787/" \
  -H "Authorization: Bearer 你的WORKER_AUTH"
```

测试定时任务：

```bash
curl "http://localhost:8787/cdn-cgi/handler/scheduled?format=json"
```

以上两种测试都会访问森空岛并可能执行真实签到。

### 4. 发布到 Cloudflare

登录 Cloudflare：

```bash
npx wrangler login
```

先做一次仅构建检查：

```bash
npx wrangler deploy --dry-run
```

首次发布时，将 `.dev.vars` 中的 Secret 与 Worker 一起安全上传：

```bash
npx wrangler deploy --secrets-file .dev.vars
```

部署完成后，Wrangler 会显示 Worker 的 `workers.dev` 地址。Secret 的值不会写入代码仓库或 `wrangler.toml`。

以后只更新某个 Secret 时，可使用交互式命令并按提示粘贴值：

```bash
npx wrangler secret put TOKEN
npx wrangler secret put WORKER_AUTH
```

可选推送服务同理，只设置实际使用的项目：

```bash
npx wrangler secret put SC3_SENDKEY
npx wrangler secret put SC3_UID
npx wrangler secret put QMSG_KEY
npx wrangler secret put PUSHPLUS_KEY
```

修改代码后重新发布：

```bash
npm run check
npm run deploy
```

## 验证线上部署

使用部署输出的地址发送带鉴权的 `POST` 请求：

```bash
curl -X POST "https://skyland-auto-sign.<你的子域>.workers.dev/" \
  -H "Authorization: Bearer 你的WORKER_AUTH"
```

查看实时日志：

```bash
npx wrangler tail
```

接口只接受 `POST`。缺少或错误的 `Authorization` 会返回 `401`；未配置 `WORKER_AUTH` 会返回 `503`。

### 使用 curl 临时传入 Token

也可以在 JSON 请求体中提供 `token`。该 Token 仅用于当前请求，不会被 Worker 保存或写入日志；请求体未提供 `token` 时，Worker 会继续使用 Cloudflare 中的 `TOKEN` Secret。

不建议把真实 Token 和 `WORKER_AUTH` 直接写进命令，因为它们可能保留在 Shell 历史中。先设置当前终端的环境变量：

Windows PowerShell：

```powershell
$env:SKLAND_TOKEN = "你的森空岛Token"
$env:WORKER_AUTH = "你的WORKER_AUTH"

$body = @{ token = $env:SKLAND_TOKEN } | ConvertTo-Json -Compress
curl.exe -X POST "https://skyland-auto-sign.<你的子域>.workers.dev/" `
  -H "Authorization: Bearer $env:WORKER_AUTH" `
  -H "Content-Type: application/json" `
  --data-binary $body
```

macOS / Linux（需要 `jq`）：

```bash
read -rsp "Skland Token: " SKLAND_TOKEN && echo
read -rsp "WORKER_AUTH: " WORKER_AUTH && echo
export SKLAND_TOKEN WORKER_AUTH

printf '%s' "$SKLAND_TOKEN" \
  | jq -Rs '{token: .}' \
  | curl -X POST "https://skyland-auto-sign.<你的子域>.workers.dev/" \
      -H "Authorization: Bearer ${WORKER_AUTH}" \
      -H "Content-Type: application/json" \
      --data-binary @-

unset SKLAND_TOKEN WORKER_AUTH
```

多个账号仍可在 `token` 字符串中用英文逗号分隔。线上必须配置 `WORKER_AUTH`；如果只使用这种手动调用方式，可以不配置 `TOKEN` Secret，但 Cron 定时签到仍然需要 `TOKEN` Secret。

## 定时任务

[wrangler.toml](./wrangler.toml) 默认配置：

```toml
[triggers]
crons = ["0 23 * * *"]
```

Cloudflare Cron 使用 UTC，因此该配置表示每天 UTC 23:00，即北京时间次日 07:00。修改 Cron 后重新运行 `npm run deploy`；配置传播可能需要几分钟。

## MAA 开始任务前自动签到

MAA 的“开始前脚本”可以在每次任务开始前调用已经部署好的 Worker。仓库提供：

- Windows：[scripts/maa-sign.bat](./scripts/maa-sign.bat)
- macOS / Linux：[scripts/maa-sign.sh](./scripts/maa-sign.sh)
- 本地配置模板：[scripts/maa-curl.env.example](./scripts/maa-curl.env.example)

### 1. 创建 MAA 本地配置

复制模板，但不要修改或填写 `.example` 文件本身。

Windows PowerShell：

```powershell
Copy-Item scripts\maa-curl.env.example scripts\maa-curl.env
```

macOS / Linux：

```bash
cp scripts/maa-curl.env.example scripts/maa-curl.env
```

编辑 `scripts/maa-curl.env`：

```dotenv
WORKER_URL=https://skyland-auto-sign.<你的子域>.workers.dev/
WORKER_AUTH=你的WORKER_AUTH
SKLAND_TOKEN=从data.content复制的森空岛Token
```

### 2. 先手动测试脚本

Windows：

```powershell
scripts\maa-sign.bat
```

macOS / Linux：

```bash
chmod +x scripts/maa-sign.sh
./scripts/maa-sign.sh
```

### 3. 添加到 MAA

进入 MAA 的“设置 → 连接设置”，找到“开始前脚本”，选择 `maa-sign.bat` 的绝对路径。macOS / Linux 版本填写 `maa-sign.sh` 的绝对路径。不同 MAA 版本的选项名称可能略有差异。

![MAA 开始前脚本位置](./assets/img_3.png)

配置完成后，MAA 每次开始任务都会先运行签到脚本。日志中显示“完成任务：开始前脚本”代表脚本流程已经结束；签到是否成功仍应以脚本输出的 Worker JSON 为准。

![MAA 开始前脚本完成](./assets/img_5.png)

> [!IMPORTANT]
> MAA 自动调用需要一个可访问的线上 `WORKER_URL`。可以使用你自己部署的 Worker，或使用你信任的服务提供者明确提供的地址和 `WORKER_AUTH`。不要把 Token 发送给来源不明的公共 Worker，因为 Worker 的运营者在请求处理期间能够接触该 Token。

## 无需部署：直接在本地运行

这种方式适合只想立即执行一次签到的用户。程序运行在自己的电脑上，Token 不需要交给任何第三方，也不需要 Cloudflare 账号。关闭本地 Wrangler 后服务就会停止，因此这种方式不会自动执行每天的 Cron 签到。

### 1. 下载并安装

```bash
git clone https://github.com/xiaoyulejia/skyland-auto-sign-CFworker.git
cd skyland-auto-sign-CFworker
npm ci
```

### 2. 配置本地调用密码

Windows PowerShell：

```powershell
Copy-Item .dev.vars.example .dev.vars
```

macOS / Linux：

```bash
cp .dev.vars.example .dev.vars
```

编辑 `.dev.vars`，将 `WORKER_AUTH` 改为一个仅供本机使用的密码。使用下面的 curl 临时传入 Token 时，`TOKEN` 可以留空：

```dotenv
TOKEN=""
WORKER_AUTH="choose-a-local-password"
```

### 3. 启动本地 Worker

```bash
npm run dev
```

保持该终端运行。Wrangler 默认会在 `http://localhost:8787` 启动本地服务。

### 4. 使用 curl 签到

在另一个 PowerShell 窗口执行：

```powershell
$token = Read-Host "请输入森空岛 Token"
$body = @{ token = $token } | ConvertTo-Json -Compress

curl.exe -X POST "http://localhost:8787/" `
  -H "Authorization: Bearer choose-a-local-password" `
  -H "Content-Type: application/json" `
  --data-binary $body
```

macOS / Linux（需要 `jq`）：

```bash
read -rsp "Skland Token: " SKLAND_TOKEN && echo

printf '%s' "$SKLAND_TOKEN" \
  | jq -Rs '{token: .}' \
  | curl -X POST "http://localhost:8787/" \
      -H "Authorization: Bearer choose-a-local-password" \
      -H "Content-Type: application/json" \
      --data-binary @-

unset SKLAND_TOKEN
```

将命令里的 `choose-a-local-password` 替换为 `.dev.vars` 中完全相同的 `WORKER_AUTH`。多个账号可在 Token 中使用英文逗号分隔。执行结束后，在运行 Wrangler 的窗口按 `Ctrl+C` 停止服务。

> [!NOTE]
> “无需部署”指在用户自己的电脑上运行。项目不会公开你的线上 Worker 地址或共享 `WORKER_AUTH`；共享调用密码会使任何知道密码的人都能消耗你的 Worker 资源。
