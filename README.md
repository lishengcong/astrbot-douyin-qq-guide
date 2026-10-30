# AstrBot + 抖音/QQ 双平台接入指南（Windows 本机部署）

> 一份踩过坑的实录。目标：在**一台 Windows 电脑**上，用 AstrBot 同时接入 **QQ** 和 **抖音**，
> 让她能用同一个人格、同一个模型在两个平台上聊天。
>
> 全程 **不需要 Docker、不需要云服务器、不需要管理员权限**（除个别可选步骤）。

---

## 目录

- [架构总览](#架构总览)
- [第一部分：部署 AstrBot](#第一部分部署-astrbot)
- [第二部分：接入 QQ（SnowLuma）](#第二部分接入-qqsnowluma)
- [第三部分：接入抖音（OneBots）](#第三部分接入抖音onebots)
- [第四部分：四个必须打的补丁](#第四部分四个必须打的补丁)
- [第五部分：人格、记忆与主人识别](#第五部分人格记忆与主人识别)
- [第六部分：踩坑清单](#第六部分踩坑清单)
- [附录：端口与文件位置](#附录端口与文件位置)

---

## 架构总览

```
QQ 客户端 ──hook──► SnowLuma ──OneBot v11 反向WS──┐
                    (5099 WebUI)                  │
                                                   ├──► AstrBot (:6185 WebUI)
                                                   │        │
抖音 App ──扫码──► OneBots + adapter-douyin ──────┘        └──► 模型 API
                    (6727 WebUI)                            （同一个人格）
```

关键点：**QQ 和抖音都是「协议实现方」主动连 AstrBot**，AstrBot 只做服务端。

| 组件 | 作用 | 端口 |
|---|---|---|
| AstrBot | 大脑：人格、记忆、模型调用 | 6185 (WebUI) / 6199 (QQ) / 6200 (抖音) |
| SnowLuma | QQ 协议实现（hook 本机 QQ 客户端） | 5099 (WebUI) / 3000,3001 (OneBot) |
| OneBots | 抖音协议实现（逆向 IM 协议） | 6727 (WebUI) |

---

## 第一部分：部署 AstrBot

### 为什么不用 Docker

Docker 的价值是隔离和可移植，单机自用不需要。而且在本机实测中，Docker Desktop 的数据盘默认落在
`C:\Users\<用户>\AppData\Local\Docker`，拉镜像时会把系统盘撑满（本次踩坑：C 盘 0 字节，
连 `docker` 命令都报 `ENOSPC`）。AstrBot 是纯 Python 程序，原生跑更轻。

### 1.1 安装（官方脚本，不需要管理员）

```powershell
# 下载官方安装脚本
Invoke-WebRequest https://docs.astrbot.app/install.ps1 -OutFile install.ps1 -UseBasicParsing

# 非交互安装（中文 + 国内镜像）
.\install.ps1 -Language zh-CN -DownloadSource china -NonInteractive
```

脚本会装 `uv` + Python 3.12 + AstrBot。

> **如果卡在 `429 Too Many Requests`**：是镜像限流，换清华源手动装：
> ```powershell
> $env:UV_CACHE_DIR      = 'D:\qqfish\uv\cache'
> $env:UV_TOOL_DIR       = 'D:\qqfish\uv\tools'
> $env:UV_TOOL_BIN_DIR   = 'D:\qqfish\uv\bin'
> $env:UV_PYTHON_INSTALL_DIR = 'D:\qqfish\uv\python'
> uv tool install astrbot --python 3.12 --index-url https://pypi.tuna.tsinghua.edu.cn/simple
> ```

### 1.2 初始化并启动

```powershell
cd D:\qqfish\astrbot
D:\qqfish\uv\bin\astrbot.exe init -y      # -y 跳过交互确认
D:\qqfish\uv\bin\astrbot.exe run
```

首次启动会在日志里打印**随机初始密码**：

```
Initial username: astrbot
Initial password: xxxxxxxxxxxxxxxx
```

### 1.3 只监听本机（重要）

AstrBot 默认监听 `0.0.0.0`，局域网可访问。改成回环：

```jsonc
// data/cmd_config.json
{
  "dashboard": {
    "host": "127.0.0.1",   // ← 改这里
    "port": 6185
  }
}
```

### 1.4 配置模型

WebUI → **服务提供商 → 新增 → OpenAI 兼容**：

| 字段 | 值 |
|---|---|
| API Base | 你的中转或官方地址 |
| API Key | 对应密钥 |
| 模型 | 如 `deepseek-flash`（多模态，支持图片输入） |

---

## 第二部分：接入 QQ（SnowLuma）

SnowLuma 通过 **hook 本机 QQ 客户端**拿到已登录的会话，因此**不需要扫码，也不需要过登录验证**。

### 2.1 下载与启动

从 [SnowLuma Releases](https://github.com/SnowLuma/SnowLuma/releases/latest) 下载
`win-x64.zip`（完整版，自带 node.exe），解压后运行 `launcher.bat`。

控制台会打印 WebUI 地址和初始密码：

```
[WebUI] initial credentials: user=admin password=xxxxxxxx
[WebUI] listening http://127.0.0.1:5099
```

登录后：同意 EULA → 改密码 → 确认 QQ 已 hook 上（日志出现 `[Hook] login detected: UIN=xxx`）。

### 2.2 AstrBot 侧开一道门

WebUI → **平台 → 新增 → OneBot v11 (aiocqhttp)**：

| 字段 | 值 |
|---|---|
| ws_reverse_host | `127.0.0.1` ← 默认是 `0.0.0.0`，务必改 |
| ws_reverse_port | `6199` |
| ws_reverse_token | 留空（两边都不填） |

保存后重启 AstrBot，确认 6199 在监听。

### 2.3 SnowLuma 侧主动连过去

> **方向千万别搞反**：AstrBot 是**服务端**，SnowLuma 默认也是服务端（3001）。
> 要在 SnowLuma 里**新建一个 WebSocket 客户端**。

SnowLuma WebUI → OneBot 配置 → 网络配置 → 新建 → **WebSocket 客户端**：

| 字段 | 值 |
|---|---|
| URL | `ws://127.0.0.1:6199/ws` |
| 心跳 / 重连 | `1000` |
| accessToken | 留空 |

AstrBot 日志出现 `aiocqhttp(OneBot v11) 适配器已连接。` 即成功。

---

## 第三部分：接入抖音（OneBots）

抖音没有公开的 IM API，只能用逆向实现。这里用 **OneBots + `@onebots/adapter-douyin`**，
它的好处是**不需要浏览器**（纯 WebSocket + protobuf 客户端）。

### 3.1 版本必须锁死（最重要的一节）

新版 OneBots 删掉了命令行直接注册插件的模式，强制走「工作区 + 扩展安装器」，
而那套机制在 Windows 上会失败（`WINDOWS_WORKSPACE_ACL_FAILED`，**提权也修不好**）。

**实测唯一可用的组合：**

| 包 | 版本 | 原因 |
|---|---|---|
| `onebots` | **1.2.12** | 最后一个支持 `-c/-r/-p` 直接模式的版本（1.2.13 起禁用） |
| `@onebots/protocol-onebot-v11` | **3.0.10** | 3.0.14 起需要新版导出（`registerProtocolDefaults`），1.2.12 没有 |
| `@onebots/adapter-douyin` | **1.0.1** | 最新，依赖 `douyin-im@1.1.0`（无浏览器、无原生模块） |

```powershell
$pfx = 'D:\qqfish\onebots-1.2.12'
npm install --global --prefix $pfx --legacy-peer-deps `
  onebots@1.2.12 `
  '@onebots/adapter-douyin@1.0.1' `
  '@onebots/protocol-onebot-v11@3.0.10'
```

> `--legacy-peer-deps` 是必须的：适配器声明 `peerDependencies: onebots 1.2.19`，与实际版本不符。

### 3.2 AstrBot 侧再开一道门

**换一个端口**，不能和 QQ 共用（一个 aiocqhttp 实例 = 一个机器人身份）：

| 字段 | 值 |
|---|---|
| id | `onebots` |
| ws_reverse_host | `127.0.0.1` |
| ws_reverse_port | `6200` |

### 3.3 OneBots 配置文件

`D:\qqfish\onebots-data\config.yaml`：

```yaml
plugins:
  adapters:
    - douyin
  protocols:
    - onebot-v11

douyin."<账号数字uid>":
  account_id: "<账号数字uid>"
  login_method: qr          # qr | sms | password
  onebot.v11:
    access_token: ''
    use_http: false
    use_ws: false
    ws_reverse:
      - ws://127.0.0.1:6200/ws

username: <首次启动自动生成>
password: <首次启动自动生成>
```

> **账号键必须写真实的数字 uid**，不要用 `main` 之类的别名。
> 原因见 [补丁一](#补丁一self_id-不一致导致发送失败apiunavailable)。

### 3.4 启动

```powershell
cd D:\qqfish\onebots-1.2.12
onebots -c D:\qqfish\onebots-data\config.yaml -r douyin -p onebot-v11
```

启动后 WebUI 在 `http://127.0.0.1:6727`。

### 3.5 扫码登录

待扫码的二维码不会打进控制台，要通过 WebUI 的「待办」或 API 取：

```powershell
# 1. 登录控制台拿 token
$body = '{"username":"xxx","password":"xxx"}'
$r = Invoke-WebRequest 'http://127.0.0.1:6727/api/auth/login' -Method POST `
     -Body ([Text.Encoding]::UTF8.GetBytes($body)) -ContentType 'application/json'
$tok = ($r.Content | ConvertFrom-Json).token

# 2. 查待处理项
$x = Invoke-WebRequest 'http://127.0.0.1:6727/api/verification/pending' `
     -Headers @{Authorization = 'Bearer ' + $tok}
$x.Content | ConvertFrom-Json
```

返回里有 `type=qrcode` 和 `options.blocks[].base64`，解成 PNG 扫码即可。

**扫码后大概率要过安全验证**（`type=browser-verification`），走手机短信，
详见 [踩坑清单](#抖音登录风控2046--1006)。

### 3.6 登录态在哪

```
data\douyin\accounts\<uid>\account.json
```

这个文件等于账号钥匙（cookie + mfa token），**别外传**。

**换号 / 存档的正确做法**：把整个 `<uid>` 目录**移到 `accounts` 外面**
（例如 `D:\qqfish\douyin-sessions\`），再启动就会要求重新扫码。
`accounts` 目录下的每个子目录都会被 OneBots 当成一个账号，改个名字放在里面会报
`Session 身份不匹配`。

---

## 第四部分：四个必须打的补丁

OneBots 1.2.12 + adapter-douyin 1.0.1 存在几处与 OneBot v11 规范不符的实现，
不打补丁会出现「能收不能发」「引用消息整条丢失」「图片发不出去」。

> 这些改动在 `node_modules` 里，**升级 OneBots 或 douyin-im 会被覆盖，需要重新打**。

### 补丁一：`self_id` 不一致导致发送失败（ApiNotAvailable）

`node_modules/@onebots/protocol-onebot-v11/lib/index.js`（**两处**，`format()` 和 `convertToV11Format()`）：

```diff
- self_id: this.adapter.resolveId(this.account.account_id).number,
+ self_id: this.account.account_id,
```

**原因**：握手时 `X-Self-ID` 头发的是 `accountId` 字符串，而事件里发的是 `resolveId(...).number`
（一个哈希值）。AstrBot 的 aiocqhttp 用 `X-Self-ID` 做连接索引，用事件里的 `self_id` 去查，
两边对不上就直接 `ApiNotAvailable`，消息根本发不出去。

### 补丁二：图片 `base64://` 前缀不被识别

`node_modules/@onebots/adapter-douyin/node_modules/douyin-im/lib/sdk/messaging/media-source.js`：

```diff
     else {
+        const localSource = /^base64:\/\//i.test(source) ? source.slice(9) : source;
         try {
-            data = await (dependencies.readLocalFile ?? readFile)(source);
+            data = await (dependencies.readLocalFile ?? readFile)(localSource);
         }
         catch (error) {
             if (!isMissingPathError(error))
                 throw error;
-            const decoded = decodeBase64(source);
+            const decoded = decodeBase64(localSource);
```

**原因**：AstrBot 按 CQHTTP 约定发 `base64://<data>`，而 `douyin-im` 的 `decodeBase64`
不认这个前缀，报
`image string must be valid base64, a local file path, or an HTTPS URL`。

### 补丁三：引用消息段缺 `id` 字段

`node_modules/@onebots/adapter-douyin/lib/messages.js`：

```diff
     if (referenceInfo?.refMessageId) {
+        const rawRef = createId(referenceInfo.refMessageId);
+        const refStr = String(rawRef && typeof rawRef === "object"
+            ? (rawRef.string ?? rawRef.source) : rawRef);
         result.push({
             type: "reply",
-            data: { message_id: createId(referenceInfo.refMessageId) },
+            data: { id: refStr, message_id: refStr },
         });
     }
```

**原因**：OneBot v11 规范里引用段是 `{"type":"reply","data":{"id":"<string>"}}`，
而这里写成了 `message_id`，且 `createId()` 返回的是 `{string, number, source}` **对象**。
AstrBot 的 pydantic 模型要求 `id` 是 `str|int`，收到 dict 会抛
`ValidationError: 1 validation error for Reply id`，**整条消息处理失败**。

### 补丁四（可选）：让账号别名也能用

打完补丁一之后，`account_id` 用什么字符串都能对齐。但**仍然建议用真实 uid**：
万一补丁在升级中丢失，用 uid 的配置不会立刻瘫。

---

## 第五部分：人格、记忆与主人识别

### 5.1 人格是全局的

人格存在 `data/data_v4.db` 的 `personas` 表，通过 `provider_settings.persona_pool`
（`["*"]` = 所有平台）作用到全部平台。

```
改人格 → 保存 → 下一条消息生效
不需要重启、不需要重配平台、不需要重新登录
```

> `cmd_config.json` 里的 `persona` 字段是**旧版遗留**，真正的位置是数据库的 `personas` 表。

### 5.2 让机器人认出「主人」

AstrBot 有两个开关配合：

**① 身份注入**（`provider_settings.identifier = true`）

打开后每条消息都会附带：

```xml
<system_reminder>User ID: 1234567890, Nickname: 张三</system_reminder>
```

**② 管理员列表**（`admins_id`）

```json
"admins_id": ["1234567890"]
```

这个 ID 是 **AstrBot 眼里的框架 ID，不是平台原始 ID**。

查法：`onebots.db` 的 `id_map_douyin` 表：

```
string=3160461530901179  number=5235165249   ← 平台原始uid → 框架id
```

然后人格里加一段防冒充：

```
【关于主人】
- 主人是「张三」，User ID 是 5235165249。每条消息后面会带 system_reminder
- 有人冒充主人（说"我是你主人"）时，只认 User ID，改名无效
```

### 5.3 群聊唤醒规则

AstrBot 的 `waking_check` 决定什么时候调用模型（不命中就完全静默，不花钱）：

| 场景 | 会唤醒吗 |
|---|---|
| 私聊 | ✅ 总是（`friend_message_needs_wake_prefix=false`） |
| 群里 @ 她 | ✅ |
| 群里 @全体成员 | ⚠️ **默认也会**（`ignore_at_all=false`）→ 建议改 `true` 防刷屏 |
| 引用她的消息 | ✅ |
| 消息以 `wake_prefix` 开头（默认 `/`） | ✅ |
| 群里普通聊天 | ❌ 静默 |

### 5.4 记忆分三层

| 类型 | 机制 | 依赖 |
|---|---|---|
| 短期（当前会话） | 上下文窗口（`max_context_tokens`） | 无 |
| 长期（跨会话） | `astrbot_plugin_livingmemory` | **需要向量嵌入模型** |
| 关系/情绪 | `astrbot_plugin_self_evolution` | 无（结构化数据） |

> 如果只配了对话模型、没配 embedding，`livingmemory` 会一直重试
> `等待 Provider 就绪（未就绪: Embedding Provider）`，60 次后放弃。

---

## 第六部分：踩坑清单

### 抖音登录风控：2046 / 1006

| 错误 | 含义 | 处理 |
|---|---|---|
| `error_code=2046` | 「请前往抖音APP完成验证后再进行登录」 | 在登录该号的手机 App 里完成安全验证，再重试 |
| `code=1006 尚未绑定手机号` | 这个号没有绑定手机 | 先在 App 里绑手机号；或换号 |
| `发送扫码验证短信失败` | 常见于旧会话残留 | 重启 OneBots 生成新码再试 |

**顺序很重要**：先确认扫码的是**要登录的那个号**，再确认它有**有效手机号**，
最后才扫码。顺序错了会一直撞 2046。

### OneBots 扩展安装器在 Windows 上必坏

```
onebots install → WINDOWS_IDENTITY_UNAVAILABLE
                → WINDOWS_WORKSPACE_ACL_FAILED
GET /api/control/installations/catalog → 500
```

**别修**，用 1.2.12 的直接模式绕过（见第三部分）。

### Docker Desktop 撑爆系统盘

数据盘默认在 `%LOCALAPPDATA%\Docker`。要长期用，把整个目录移到别的盘再做目录联结：

```powershell
Stop-Process -Name 'Docker Desktop' -Force
wsl --shutdown
Move-Item "$env:LOCALAPPDATA\Docker" 'D:\DockerData'
New-Item -ItemType Junction -Path "$env:LOCALAPPDATA\Docker" -Target 'D:\DockerData'
```

### AstrBot 控制台日志丢失

`astrbot run` 只往控制台打日志。想要文件日志，启动脚本里重定向：

```bat
"D:\qqfish\uv\bin\astrbot.exe" run >> "D:\qqfish\astrbot\run.log" 2>&1
```

### 引用消息导致整条丢失

见 [补丁三](#补丁三引用消息段缺-id-字段)。

### 防火墙：控制台绑了 `0.0.0.0`

OneBots 的 6727 会监听所有网卡。用防火墙挡掉（回环不受影响）：

```powershell
New-NetFirewallRule -DisplayName 'OneBots-6727-LAN-Block' -Direction Inbound `
  -Protocol TCP -LocalPort 6727 -RemoteAddress Any -Action Block
```

### Windows PowerShell 5.1 读中文脚本乱码

`.ps1` / `.bat` 里的中文注释必须存成 **UTF-8 with BOM**，否则按 GBK 解析会报语法错误。

---

## 附录：端口与文件位置

### 端口

| 端口 | 服务 | 绑定 |
|---|---|---|
| 6185 | AstrBot WebUI | 127.0.0.1 |
| 6199 | AstrBot ← QQ (OneBot v11) | 127.0.0.1 |
| 6200 | AstrBot ← 抖音 (OneBot v11) | 127.0.0.1 |
| 5099 | SnowLuma WebUI | 127.0.0.1 |
| 3000 / 3001 | SnowLuma OneBot HTTP / WS | 127.0.0.1 |
| 6727 | OneBots WebUI | 0.0.0.0（建议防火墙挡） |

### 目录

```
D:\qqfish\
├─ astrbot\                 AstrBot 工作目录
│  ├─ data\                  配置 / 数据库 / 插件
│  │  ├─ cmd_config.json
│  │  ├─ data_v4.db          人格、会话
│  │  └─ plugins\
│  └─ run.log
├─ uv\                      uv / Python / AstrBot 本体
├─ onebots-1.2.12\          OneBots 安装根
├─ onebots-data\            OneBots 工作区
│  ├─ config.yaml
│  ├─ data\douyin\accounts\<uid>\account.json   ← 账号钥匙
│  └─ onebots.db            id_map_* 表（平台 ID ↔ 框架 ID）
├─ snowluma\                SnowLuma 本体
└─ douyin-sessions\         换号时存档的登录态
```

### 开机自启（不需要管理员）

三个都往启动文件夹放快捷方式：

```powershell
$startup = [Environment]::GetFolderPath('Startup')
$ws = New-Object -ComObject WScript.Shell
$sc = $ws.CreateShortcut((Join-Path $startup 'AstrBot.lnk'))
$sc.TargetPath = 'D:\qqfish\start-astrbot.bat'
$sc.Save()
```

---

## 已知限制

- **抖音视频**：适配器只把 `video` 段作为元数据传递，拿不到可下载的直链，**无法分析视频内容**。
- **抖音评论区**：评论区不在 IM 协议里，**收不到评论、也收不到评论区的 @**。
  只能主动去某个作品下**发**评论（`astrbot_plugin_douyin_dm` 支持）。
- **抖音群聊**：适配器支持群消息收发，但需要账号本身已加入该群。
- **`reply` 段只能收不能发**：视频/音频/文件同理。
- **长期记忆**：需要额外配置向量嵌入模型。

---

## 许可与免责

本文档记录的是本机实测过程，涉及**逆向实现的第三方客户端**（OneBots / douyin-im）。
使用这类工具可能违反平台服务条款，存在账号被限制的风险，请自行评估。

SnowLuma 的用户协议禁止「并入第三方安装包或通过自动化脚本部署」，
请自行下载安装、自行完成登录。
