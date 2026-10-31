# 从 AstrBot 迁移到 DSH：可行性与分期计划

> **状态：暂定，未实施。** 本文只是把调研结论和实施方案存档，等有需要时再执行。
>
> 当前生产环境仍是 AstrBot + OneBots，见 [README.md](README.md)。

---

## 目录

- [为什么考虑迁移](#为什么考虑迁移)
- [一个容易被忽略的差别：prompt 负担](#一个容易被忽略的差别prompt-负担)
- [DSH 市场调研](#dsh-市场调研)
- [缺口：抖音私信](#缺口抖音私信)
- [目标架构](#目标架构)
- [三条设计红线](#三条设计红线)
- [实施阶段](#实施阶段)
- [工作量与风险](#工作量与风险)
- [必须先验证的三个技术点](#必须先验证的三个技术点)
- [数据迁移清单](#数据迁移清单)
- [回滚策略](#回滚策略)

---

## 为什么考虑迁移

当前架构（AstrBot）在「抖音私信」这条线上已经跑通并稳定。考虑迁移的理由是**生态**：

| | AstrBot | DSH |
|---|---|---|
| 插件市场 | 1835（Python） | **7723**（TypeScript/Cordis） |
| 抖音相关插件 | 全是「私信平台适配器」 | **22 个**（浏览 / 评论 / 发布 / 解析） |
| 人格插件 | 有，偏简单 | `dsh-persona-shell`、`dsh-soul`、`dsh-xingye` |
| 记忆插件 | `livingmemory` | `dsh-engram`、`dsh-dream`、`dsh-linghun` |
| 自我进化 | `self_evolution` | `dsh-fish-reflection`、`dsh-host-rsi` |

**注意：两套插件不通用。** AstrBot 插件是 Python + Star/filter API；DSH 插件是
TypeScript + Cordis service。跨框架直接搬不存在，能力要重新拼。

---

## 一个容易被忽略的差别：prompt 负担

评估架构时容易只算「网络跳数」，但真正影响首 token 延迟的是**每条消息要背多少 prompt**。

### 旧架构（外部桥 → DSH 会话）

每条消息都要注入一大段脚手架，实测内容包括：

```
身份说明 + 会话令牌 + 反AI味规则
表情包策略（12 条子规则）
首回复节奏 + wait_for_messages 用法
沉睡前强制等待（一段闸门逻辑）
时间感判断（10 分钟 / 120 分钟分档）
回复检查流程
本次消息快照（JSON）
+ 十几个 qq_* 工具的定义
```

粗估 **1500~2000 汉字 + 十几个工具 schema**，每条消息重来一遍。

### AstrBot（现在）

```
人格文本（约 2500 字）
+ <system_reminder>User ID / Nickname / Current datetime</system_reminder>
```

### 本质差别：「要不要回」由谁判断

| | 旧架构 | AstrBot |
|---|---|---|
| 是否该回复 | **模型读规则自己判断** | **代码判断**（`waking_check`） |
| 何时沉睡 | 模型调工具 | 框架自动 |
| 时间感 | 模型分档推理 | 代码拼一行 |

旧架构里「要不要说话」本身要花掉一轮推理；AstrBot 里是代码判断
（命中 @ 才唤醒，否则完全不调用模型，连费用都不产生）。

> **迁移时必须守住这个优势**，否则会退回「背着两千字脚手架聊天」的状态。

---

## DSH 市场调研

市场目录：`https://awesome-dsh-plugin.com/plugins.json`（约 9.7 MB，7723 条）

### 抖音相关（22 个，按用途分类）

**刷视频**

| 插件 | 能力 |
|---|---|
| `dsh-douyin-agent` | 专用浏览器刷抖音，每日观看限额，共享本地人格与兴趣 |
| `dsh-douyin-subagent` | 闲时刷视频、读评论、回报，带 `dy_*` 工具 |
| `dsh-moyu` | 鼠标悬停即刷的摸鱼版 |
| `dsh-douyin` | 侧边栏播放器，系列导航 + 历史回放 |

**互动与发布**

| 插件 | 能力 |
|---|---|
| `dsh_douyin_tools` | 每日挑一轮视频、读评论、写日报、点赞和评论 |
| `dsh-douyin-publish` | 本地视频 → 创作者上传页，走完整发布流程 |
| `dsh-douyin-webview` | 侧边栏内嵌真实抖音窗口（Electron webview guest，绕开 X-Frame-Options） |

**内容解析**

| 插件 | 能力 |
|---|---|
| `linkdigest-mcp` | 小红书/抖音/TikTok/YouTube/X 链接 → 逐句文本 + 时间码 + 屏幕文字 |
| `dsh-whale-dive` | 从会话里读 B 站与抖音（WBI 签名 + 限速） |
| `dsh-wenan` | 提取短视频口播稿与文案 |
| `dsh-video-kit` | 本地视频 → ffmpeg + ASR 时间轴转录 |

**记忆 / 人格 / 进化**

| 插件 | 能力 |
|---|---|
| `dsh-engram` | 跨会话长期记忆：本地 SQLite + FTS5 + 本地向量混合检索 |
| `dsh-dream` | 经验记忆与反思 |
| `dsh-linghun` | 认知循环与判断核心 |
| `dsh-persona-shell` | 用指定人格重述每条回复，不影响工作提示词 |
| `dsh-soul` | 个人化面板（关于你 / 语气 / 风格 / 自定义指令） |
| `dsh-xingye` | 建角色、写人设、多存档分支 |
| `dsh-fish-reflection` | 自我重写 3~7 条世界观 / 价值观 / 偏好 |
| `dsh-host-rsi` | 反思式自我进化 |

### IM 层

`dsh-im` 存在，`dsh-im-companion` 的描述里提到「按 Agent 分组飞书 / 微信 / QQ 机器人」——
**支持飞书、微信、QQ，不含抖音**。

---

## 缺口：抖音私信

上面 22 个抖音插件**全是浏览器自动化**（浏览、读评论、发评论、发视频、解析内容），
**没有一个能「作为账号收发私信」**。

而 `dsh-im` 覆盖飞书 / 微信 / QQ，不覆盖抖音。

**结论：迁移后「私信收发」这一环必须自己补。** 而它恰好是整个项目最难的部分
（抖音风控 2046、OneBot v11 的 `self_id` 路由、`base64://` 图片、`reply` 段字段），
目前由 OneBots 解决且已验证。

---

## 目标架构

**核心思路：OneBots 不动，只把「接协议并生成回复」的那一端从 AstrBot 换成 DSH 插件。**

```
抖音
  ↓ 扫码 / 风控（OneBots 负责，已验证）
OneBots 1.2.12（原生，6727）
  ↓ OneBot v11（反向 WebSocket，独立端口）
自研薄插件 dsh-douyin-im
  ↓ ctx.sessions.prompt(...)
她的 DSH 会话
  ↓
市场插件：dsh-engram（记忆）+ dsh-persona-shell（人格）
```

关键点：

- OneBots 的登录态**原样复用**（`data/douyin/accounts/<uid>/account.json`）
- 不用重新扫码、不用重走风控
- 插件只承担「协议 ↔ 会话」的翻译，不承担人格、记忆、决策

---

## 三条设计红线

写代码前先定死，否则会退回旧架构的臃肿状态。

| 红线 | 具体要求 |
|---|---|
| **① 唤醒判断在代码里** | 私聊必回；群聊只在「@她 / 引用她 / 带 `/` 前缀」时回。**不让模型读规则决定** |
| **② 系统提示只放人格** | 不塞会话令牌、不塞 JSON 快照、不塞工具用法、不塞沉睡流程 |
| **③ 常驻工具 ≤ 3 个** | 收发消息 + 记忆召回。工具 schema 本身也是 prompt 负担 |

---

## 实施阶段

### 阶段 0：立红线（10 分钟）

无代码，只是把上面三条写进设计文档，后面每次改动对照检查。

### 阶段 1：给 OneBots 加第二个协议出口（约半小时）

OneBots 支持多个出口并存。给插件单独开 **6300**，AstrBot 的 6200 暂时保留：

```yaml
douyin."336910714929712":
  account_id: "336910714929712"
  login_method: qr
  onebot.v11:
    use_http: false
    use_ws: false
    ws_reverse:
      - ws://127.0.0.1:6200/ws   # 现有：AstrBot
      - ws://127.0.0.1:6300/ws   # 新增：DSH 插件
```

> ⚠️ **必须分端口。** 一个账号挂多个客户端没问题，但两个客户端挤同一端口会触发
> `self_id` 路由歧义（本项目实际踩过：`ApiNotAvailable`）。

登录态完全不动。

### 阶段 2：写薄插件（1~2 天，唯一开发量）

**先做 spike（2 小时）**：不写插件，先用脚本验证「把消息 prompt 给会话」这条路。

```js
// 验证用伪代码
const r = await acp.request('session/prompt', {
  sessionId: 'session-xxxx',
  prompt: '[抖音私信] 思忠: 在不'
})
```

通过再写插件，避免白写。

**插件结构**

```
dsh-douyin-im/
  index.js        插件入口（Cordis）
  ws-server.js    监听 6300，收 OneBot v11 事件
  decide.js       代码判断要不要回（纯函数，好测）
  tools.js        注册 douyin_reply
```

**核心逻辑（目标 < 100 行）**

```js
onEvent(ev) {
  if (ev.post_type !== 'message') return
  if (!shouldReply(ev)) return
  const kind = ev.message_type === 'group' ? '群' : '私信'
  const text = `[抖音${kind}] ${ev.sender.nickname}: ${plain(ev)}`
  ctx.sessions.prompt(sessionId, text)   // 一行，没有脚手架
}

function shouldReply(ev) {
  if (ev.message_type === 'private') return true
  const at = ev.message.find(s => s.type === 'at' && String(s.data.qq) === selfId)
  const quote = ev.message.find(s => s.type === 'reply')
  return Boolean(at || quote) || plain(ev).startsWith('/')
}
```

**她说话的通道**

```js
ctx.tools.register('douyin_reply', {
  description: '回复当前抖音会话',
  schema: { text: 'string' },
  handler: ({ text }) => ws.send({ action: 'send_msg', params: { ... } })
})
```

> 发送时仍需带正确的 `self_id`（与握手头 `X-Self-ID` 一致），否则 OneBots 侧路由失败。

### 阶段 3：装市场插件补能力（半天）

```text
dsh-engram          长期记忆（SQLite + 本地向量，可复用已有 Ollama bge-m3）
dsh-persona-shell   人格表达
dsh-douyin-agent    刷抖音（专用浏览器 + 每日限额 + 共享兴趣）
dsh_douyin_tools    每日视频 / 读评论 / 日报 / 点赞评论
dsh-douyin-publish  发视频
dsh-fish-reflection 自我进化
```

### 阶段 4：搬数据（半天，需写导出脚本）

见 [数据迁移清单](#数据迁移清单)。

### 阶段 5：并行跑一周再切（不做一次性切换）

```text
第 1 周   6200 → AstrBot（保底）   +   6300 → DSH（试验）
          同一条私信两边都会回，正好对比人格与速度差异

第 2 周   确认 DSH 侧满意 → 去掉 6200 → 停用 AstrBot
```

**不要一次性切。** 本项目经验：看起来配好了，实际常撞风控或字段不兼容。

---

## 工作量与风险

| 阶段 | 动手时间 | 风险 | 说明 |
|---|---|---|---|
| 0 立红线 | 10 分钟 | — | 纯设计 |
| 1 加协议出口 | 半小时 | 低 | 配置层面，登录态不动 |
| 2 写插件（含 spike） | 1~2 天 | 中 | 协议细节已知，但会话 prompt 形态待验证 |
| 3 装市场插件 | 半天 | 低 | 现成能力 |
| 4 搬数据 | 半天 | 低 | 需写导出脚本 |
| 5 并行观察 | 1 周（被动） | 中 | 期间两边都会回复 |
| **合计动手** | **约 3 天** | | 观察期不计 |

---

## 必须先验证的三个技术点

写代码前先确认，否则会返工：

1. **`ctx.sessions.prompt` 的参数形态**
   是纯文本还是结构化？会不会自动跑工具调用循环？
   已确认 DSH 内部存在 `session/prompt`（ACP 风格 RPC，`agent.prompt(ctx.params, ctx.client)`），
   但插件侧的调用方式需要查文档或试跑。

2. **怎么拿到她的回复文本**
   prompt 的返回值里有最终文本？还是必须订阅事件流？
   **这一点最关键** —— 拿不到回复就无法发回抖音。

3. **群聊 `at` 段的解析**
   OneBot v11 的 `at` 段 `qq` 字段，与 OneBots 实际发出的值是否一致
   （本项目在 `self_id` 上就踩过「握手头与事件值不一致」的坑）。

---

## 数据迁移清单

| 数据 | 现位置 | 迁移方式 |
|---|---|---|
| 人格「ds小肥鱼」（约 2500 字） | AstrBot `data/data_v4.db` → `personas` 表 | 导出为 Markdown → 写入 DSH 人格配置 |
| 长期记忆 | `data/plugin_data/astrbot_plugin_livingmemory/*.db` | 导出为 Markdown → 喂 `dsh-engram` 重建索引 |
| 好友关系（好感度等） | `plugin_data/astrbot_plugin_self_evolution/self_evolution.db` → `user_relationships` | 摘要成几段文字，作为记忆种子 |
| 情绪状态 | 同上 → `persona_state` | 可丢弃（属于瞬时状态） |
| 抖音登录态 | `onebots-data/data/douyin/accounts/<uid>/` | **不迁移**，OneBots 继续使用 |
| QQ 登录态 | SnowLuma 配置 | 不迁移（本机已关闭 QQ 平台） |

**不做自动迁移**，写一次性导出脚本即可，量很小。

---

## 回滚策略

整个方案是**并行**而非替换，回滚成本几乎为零：

```text
阶段 1 之后   随时删掉 6300 那段配置 → 回到现状
阶段 2 之后   插件目录删掉即可，不影响 AstrBot
阶段 5 之后   恢复 6200 配置 + 启动 AstrBot → 立刻回到现状
```

前提：**OneBots 始终不动**，它是唯一不可替代的资产（登录态 + 风控解法）。

---

## 结论

**方案可行，且 DSH 的插件生态比 AstrBot 更适合「养鱼」这个方向**
（刷视频、读评论、发布、自我反思都有现成插件）。

**但迁移的核心不是「换框架」，而是「保持轻」。** 只要守住三条红线，
新架构可以和现在一样快；一旦把旧桥那种脚手架搬回去，就会立刻变慢。

**唯一不可替代的是 OneBots。** 抖音登录与风控是最难的一段，已经跑通，任何方案都不该动它。
