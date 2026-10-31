# 备选方案：用 ADB 真机让 AI「亲自玩抖音」

> **状态：备选思路，未实施。** 当前生产环境仍是 OneBots + AstrBot（见 [README.md](README.md)）。
>
> 本文记录一条与现有方案完全不同的技术路线 —— 不碰抖音协议，而是让 AI
> 通过 ADB 操作一台真实 Android 设备，用"真人方式"使用抖音 App。

---

## 目录

- [为什么会有这条路](#为什么会有这条路)
- [原理](#原理)
- [和现有方案的对比](#和现有方案的对比)
- [硬件要求](#硬件要求)
- [实施步骤（最小验证）](#实施步骤最小验证)
- [常用 ADB 命令速查](#常用-adb-命令速查)
- [成本估算](#成本估算)
- [已知陷阱](#已知陷阱)
- [适用边界](#适用边界)
- [为什么暂不采用](#为什么暂不采用)

---

## 为什么会有这条路

现有方案（OneBots 逆向 IM 协议）只能做**消息收发**，做不到：

- 刷推荐流视频
- 看评论区、发评论
- 发布作品
- 修改资料/简介

这些能力都在抖音 App 里，不在 IM 协议里。想拿到它们只有两条路：

1. 继续逆向 Web/App 接口（`a_bogus` 签名、风控对抗，非常脆）
2. **让 AI 直接操作一台真手机** ← 本文记录的是第二条

社区里已有实践者用这套跑通了：他那条鱼回复私信约 **10~20 秒**，全程在平板上操作，
不经过任何协议桥接。

---

## 原理

```
AI（DSH / 任何 agent）
   │  执行 shell 命令
   ▼
adb（platform-tools）
   │  无线调试（Wi-Fi）
   ▼
真实 Android 设备
   │
   ├─ adb exec-out screencap       截图 → AI 看屏幕
   ├─ adb shell input tap x y      点击
   ├─ adb shell input swipe ...    滑动（刷视频）
   ├─ adb shell input text ...     输入（仅 ASCII）
   ├─ adb shell uiautomator dump   读控件树（可选，比截图省 token）
   └─ ADBKeyboard                  注入中文
   ▼
抖音 App（真实客户端、真实登录、真实网络）
```

**核心：AI 看到的是屏幕，做的是点击。抖音系统里这就是一台普通手机。**

这条路线常被叫做「AI 玩手机」/「手机智能体」，几乎所有实现都是这个结构。

---

## 和现有方案的对比

| 维度 | 现有：OneBots 协议 | 备选：ADB 真机 |
|---|---|---|
| 私信 | ✅ 毫秒级 | ✅ 但 10~20 秒 |
| 刷视频 | ❌ | ✅ |
| 读/发评论 | ❌ | ✅ |
| 点赞 | ❌ | ✅ |
| 发布作品 | ❌ | ✅ |
| 改资料/简介 | ❌ | ✅ |
| 抗抖音改版 | 协议层，影响较小 | UI 层，**改版就坏** |
| 风控 | 协议指纹，有被判定风险 | **最低**（真实设备） |
| 硬件需求 | 无 | **一台常开的 Android 设备** |
| token 成本 | 低（纯文本） | **高**（每步截图） |
| 独占性 | 无 | 设备被占用，不能同时干别的 |

---

## 硬件要求

| 项 | 要求 | 说明 |
|---|---|---|
| 系统 | **Android 11+** 最省事 | 原生「无线调试」，扫码/配对码即可 |
| 旧系统 | Android 7~10 | 先用 USB 连一次，再 `adb tcpip 5555` 转无线 |
| 屏幕 | 5.5 寸以上 | 太小点不准私信界面 |
| 供电 | 长期插电 | 屏幕常亮 + 不锁屏 |
| 网络 | 与电脑同一 Wi-Fi | 无线调试必须同网段 |
| 成本 | 二手安卓平板 500~1000 元 | 旧手机更便宜 |

### 设备侧设置

```text
开发者选项 → 无线调试（开）
开发者选项 → 充电时不锁屏（开）
```

再执行一次，防止息屏：

```bash
adb shell svc power stayon true
```

### ⚠️ iPad 不可行

iOS 没有 adb。iOS 自动化只能走 `XCTest` / `WebDriverAgent`：

- 需要 **macOS + Xcode** 编译签名（Windows 只能靠 Sideloadly 装预编译包）
- 免费开发者证书 **每 7 天失效**，一失效全停
- iPadOS 17+ 还要额外开开发者模式、信任证书

技术上"可能"，实践上维护成本远高于收益 —— **这条路是 Android 专属的。**

---

## 实施步骤（最小验证）

**不要一上来搭全套。** 先花 20 分钟验证「我能不能让 AI 操作这台手机」，
截不到图，后面全是空谈。

### ① 部署 platform-tools

```powershell
# 约 10 MB，解压即用，不需要安装 Android SDK
curl.exe -L -o D:\platform-tools.zip `
  https://dl.google.com/android/repository/platform-tools-latest-windows.zip
tar.exe -xf D:\platform-tools.zip -C D:\
# 得到 D:\platform-tools\adb.exe
```

把它加进 PATH，或直接用绝对路径。

### ② 设备上开始配对

```text
设置 → 开发者选项 → 无线调试 → 使用配对码配对设备
```

会显示一个 **IP:配对端口** 和 **6 位配对码**。

### ③ 配对

```bash
adb pair 192.168.1.x:xxxxx
# 输入 6 位配对码
```

### ④ 连接

配对成功后，回到「无线调试」主界面，那里显示的是**另一个端口**：

```bash
adb connect 192.168.1.x:<连接端口>
adb devices          # 应该能看到设备，状态为 device
```

### ⑤ 验证截图（关键一步）

```bash
adb exec-out screencap -p > D:\screen.png
```

**如果能拿到一张正常的屏幕截图 → 这条路可行，继续往下做。
拿不到（黑屏 / 报错 / 设备 offline）→ 先解决连接问题，别写流程。**

### ⑥ 可选：中文输入

`adb shell input text` **只支持 ASCII**。要打中文得装 `ADBKeyboard`：

```bash
# 1. 安装 ADBKeyboard.apk（GitHub: senzhk/ADBKeyBoard）
adb install ADBKeyboard.apk

# 2. 设为当前输入法
adb shell ime set com.android.adbkeyboard/.AdbIME

# 3. 通过广播注入中文
adb shell am broadcast -a ADB_INPUT_TEXT --es msg '你好呀'
```

---

## 常用 ADB 命令速查

```bash
# 看屏幕
adb exec-out screencap -p > screen.png

# 点击 / 长按 / 滑动
adb shell input tap 540 1200
adb shell input swipe 540 1800 540 400 300     # 上滑（刷下一个视频）

# 按键：返回 / Home / 唤醒
adb shell input keyevent 4
adb shell input keyevent 3
adb shell input keyevent 82

# 读控件树（比截图省 token，但依赖抖音的控件 id）
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml

# 保持常亮
adb shell svc power stayon true

# 当前前台应用
adb shell dumpsys window | findstr mCurrentFocus

# 断开 / 重连
adb disconnect
adb connect 192.168.1.x:5555
```

---

## 成本估算

假设单条私信回复需要 **3~5 张截图**（看消息 → 点开 → 确认输入框 → 打完确认）：

| 项 | 估算 |
|---|---|
| 单张 1080p 截图进上下文 | 1000~1500 token |
| 单条回复 | **4000~7000 token** |
| 对比现有方案 | 约 2500 token，且不含图片 |
| 单条回复耗时 | **10~20 秒** |

**大约是现有路径的 2~3 倍 token、10~20 倍时间。**

优化方向：

- **用 `uiautomator dump` 代替截图** —— 控件树是文本，token 便宜得多（但抖音的控件 id 不一定稳定）
- **裁剪截图** —— 只截需要判断的区域（私信列表 / 输入框），别传全屏
- **只在关键步骤截图** —— 点击和打字不需要视觉确认

---

## 已知陷阱

| 陷阱 | 说明 | 缓解 |
|---|---|---|
| 同账号多端互踢 | 协议登录 + App 登录同时在线，抖音会判定异常 | **用不同账号**，或彻底放弃协议端 |
| 屏幕锁定 | 息屏后 adb 点击全落空 | `svc power stayon true` + 充电不锁屏 |
| 弹窗打断 | 广告、更新提示、青少年模式、验证码 | 流程里加「异常检测 → 截图 → 人工/AI 判断」 |
| UI 改版 | 抖音一改版，写死的坐标/控件全废 | 优先用控件树定位，少用绝对坐标 |
| 无线调试掉线 | 路由器重启、休眠会断 | 定时 `adb devices` 探活，断了自动重连 |
| 设备过热 | 长期亮屏 + 跑视频 | 别用旧旗舰，中端机功耗低 |

---

## 适用边界

```
✅ 适合：每天刷几条、留个言、看看评论区、偶尔发个作品
❌ 不适合：实时聊天（10~20 秒一条，体验会崩）
```

**这也是为什么建议「私信走协议、刷视频走真机」两条线并存** ——
各自用在它擅长的地方。

---

## 为什么暂不采用

1. **没有闲置的 Android 设备** —— 需要额外投入
2. **私信占日常交互的绝大部分** —— 把它从毫秒降到十几秒，体验损失太大
3. **token 成本高** —— 现阶段更值得把钱花在记忆积累上
4. **现有方案已覆盖最常用的能力** —— 私信 + 群聊 + 图片理解 + 长期记忆

**留档原因**：如果哪天「她自己刷抖音」变成核心需求，这条路是唯一可行且风控最低的方案。
届时按上面的「最小验证」五步走，20 分钟就能判断可行性。
