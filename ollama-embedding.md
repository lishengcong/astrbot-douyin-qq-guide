# 用 Ollama 给 AstrBot 补上本地向量嵌入（长期记忆）

> 场景：AstrBot 装了长期记忆插件（如 `astrbot_plugin_livingmemory`），但它一直报
> `初始化失败: Embedding Provider`。本文记录如何用**本地 Ollama** 补上这一环，
> 全程装到 D 盘、不依赖任何付费 API。

---

## 目录

- [问题是什么](#问题是什么)
- [为什么云 API 解决不了](#为什么云-api-解决不了)
- [第一步：装 Ollama 到 D 盘](#第一步装-ollama-到-d-盘)
- [第二步：拉模型（这里有个坑）](#第二步拉模型这里有个坑)
- [第三步：手动导入 GGUF 绕过限制](#第三步手动导入-gguf-绕过限制)
- [第四步：接进 AstrBot](#第四步接进-astrbot)
- [第五步：验证](#第五步验证)
- [运维与调优](#运维与调优)

---

## 问题是什么

长期记忆插件的工作流程：

```
聊天内容 ──► embedding 模型 ──► 1024 维向量 ──► 存进向量库
                                                   │
新消息   ──► embedding 模型 ──► 查询向量 ──► 相似度检索 ──► 召回旧记忆
```

**对话模型负责「说话」，embedding 模型负责「记住」。** 两者不能混用。

只配对话模型时，插件会一直重试直到放弃：

```
等待 Provider 就绪（未就绪: Embedding Provider）...（已尝试 60/60 次）
以下 Provider 在 60 次尝试后仍未就绪，初始化失败: Embedding Provider
```

## 为什么云 API 解决不了

实测几个常见来源都**没有 embedding 接口**：

| 来源 | 情况 |
|---|---|
| DeepSeek 官方 | `/v1/models` 只有 `deepseek-flash`、`deepseek-v4-pro`，**无 embedding** |
| 常见中转站 | 同上，通常只代理对话模型 |
| OpenAI / 智谱 / 阿里百炼 | 有，但要另外注册、要 key、按量计费 |

所以最省事的是**本地跑一个**。AstrBot 支持五种 embedding 来源：

```
OpenAI Embedding / Gemini Embedding / NVIDIA Embedding
Ollama Embedding / DashScope Embedding
```

---

## 第一步：装 Ollama 到 D 盘

### 为什么不走 winget

`winget install Ollama.Ollama` 默认装到 C 盘（程序 + 模型都占 C）。
系统盘紧张时，用**官方 zip 版**直接解压到目标盘，完全不碰 C。

### 下载

```powershell
$dir = 'D:\Ollama'
New-Item -ItemType Directory -Path $dir -Force
curl.exe -L -o "$dir\ollama.zip" `
  'https://github.com/ollama/ollama/releases/latest/download/ollama-windows-amd64.zip'
```

约 1.4~1.5 GB。**建议后台下载**，网速慢时容易超时中断；`curl -C -` 可以续传。

### 解压

```powershell
tar.exe -xf "$dir\ollama.zip" -C $dir
```

> 用 `tar` 而不是 `Expand-Archive`：1.5 GB 的包前者几秒，后者可能要几分钟。

### 让模型也落在 D 盘

**这步必须在启动前做**，否则默认会写进 `%USERPROFILE%\.ollama\models`（C 盘）。

```powershell
[Environment]::SetEnvironmentVariable('OLLAMA_MODELS', 'D:\Ollama\models', 'User')
```

### 启动脚本 + 开机自启

`D:\Ollama\start-ollama.bat`：

```bat
@echo off
chcp 65001 >nul
title Ollama
set "OLLAMA_MODELS=D:\Ollama\models"
set "OLLAMA_HOST=127.0.0.1:11434"
cd /d D:\Ollama
ollama.exe serve
```

开机自启（不需要管理员）：

```powershell
$startup = [Environment]::GetFolderPath('Startup')
$ws = New-Object -ComObject WScript.Shell
$sc = $ws.CreateShortcut((Join-Path $startup 'Ollama.lnk'))
$sc.TargetPath = 'D:\Ollama\start-ollama.bat'
$sc.WorkingDirectory = 'D:\Ollama'
$sc.WindowStyle = 7      # 最小化
$sc.Save()
```

启动后 `127.0.0.1:11434` 应处于监听状态。

---

## 第二步：拉模型（这里有个坑）

按常规做法应该是：

```powershell
ollama pull bge-m3
```

**但在使用代理（Clash / sing-box 的 TUN 或 fake-IP 模式）的网络里，这一定会失败：**

```
Error: max retries exceeded: Get "https://dd20bb8919...r2.cloudflarestorage.com/...":
redirect target not allowed: ...r2.cloudflarestorage.com resolves to non-public 198.18.0.41
```

### 为什么会这样

Ollama 在下载模型时会跟随一个重定向到 Cloudflare R2，并且**自带 SSRF 防护**：

> 如果目标域名解析出来是内网/保留地址，就拒绝访问。

而 fake-IP 模式的代理会把所有外网域名解析到 `198.18.0.0/15` 这一段（保留给基准测试的地址），
在 Ollama 看来这就是「非公网地址」，于是直接拒绝。

**给 Ollama 设 `HTTPS_PROXY` 也没用** —— 它的检查在本地解析域名时完成，
请求还没发出去就被拦了。

> 同类问题也会出现在其他带 SSRF 防护的工具上：只要你的网络把域名解析成 `198.18.x.x`
> 或 `10.x` / `127.x`，这些工具就会判定为内网并拒绝。

---

## 第三步：手动导入 GGUF 绕过限制

思路：**用不检查 DNS 的工具下载模型文件，再让 Ollama 从本地文件建模型。**

### 3.1 下载 GGUF

`bge-m3` 在 HuggingFace 上有现成的 GGUF（量化版，体积小、质量够）：

```powershell
$dir = 'D:\Ollama\gguf'
New-Item -ItemType Directory -Path $dir -Force

curl.exe -L --retry 3 -o "$dir\bge-m3-Q8_0.gguf" `
  'https://huggingface.co/gpustack/bge-m3-GGUF/resolve/main/bge-m3-Q8_0.gguf'
```

约 **605 MB**。`curl` 没有 SSRF 检查，能正常下。

> 国内访问 HuggingFace 慢的话，把域名换成 `hf-mirror.com`，路径不变。

### 3.2 用 Modelfile 导入

在 `D:\Ollama\gguf` 下建 `Modelfile`：

```
FROM ./bge-m3-Q8_0.gguf
```

然后在同目录执行：

```powershell
$env:OLLAMA_HOST = '127.0.0.1:11434'
cd D:\Ollama\gguf
D:\Ollama\ollama.exe create bge-m3 -f Modelfile
```

输出 `success` 即完成：

```
copying file sha256:950f4a8e... 100%
parsing GGUF
writing manifest
success
```

确认：

```powershell
ollama list
# NAME             ID              SIZE      MODIFIED
# bge-m3:latest    96d5d72df5aa    634 MB    ...
```

### 3.3 顺手验证接口

```powershell
$body = '{"model":"bge-m3","input":"今天天气不错"}'
Invoke-WebRequest 'http://127.0.0.1:11434/api/embed' -Method POST `
  -Body ([Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType 'application/json'
```

返回里 `embeddings[0]` 的长度就是维度 —— **bge-m3 是 1024**。

> 新接口是 `/api/embed`（参数 `input`），老接口 `/api/embeddings`（参数 `prompt`）仍可用。

---

## 第四步：接进 AstrBot

改两处配置文件，改完**重启 AstrBot**（配置文件是启动时读的）。

### 4.1 AstrBot 主配置 `data/cmd_config.json`

**① 加一个 provider source**（`provider_sources` 数组里）：

```jsonc
{
  "id": "ollama_embedding",
  "type": "ollama_embedding",
  "provider": "ollama",
  "provider_type": "embedding",
  "enable": true,
  "embedding_api_base": "http://127.0.0.1:11434",
  "embedding_model": "bge-m3",
  "embedding_dimensions": 1024,
  "timeout": 60,
  "proxy": ""
}
```

**② 再加一个 provider 条目**（`provider` 数组里），把它和上面的源绑定：

```jsonc
{
  "id": "bge-m3",
  "provider_source_id": "ollama_embedding",
  "enable": true
}
```

> **两个都要加。** `provider_sources` 只描述「怎么连」，实例是从 `provider` 数组创建的；
> 只加 source 不会产生可用的 provider 实例。

### 4.2 插件配置

`data/config/astrbot_plugin_livingmemory_config.json`：

```jsonc
{
  "provider_settings": {
    "embedding_provider_id": "bge-m3",                    // ← 上面 provider 的 id
    "llm_provider_id": "deepseek/deepseek-flash",         // ← 对话模型的 id
    "llm_stream_mode": "auto"
  }
}
```

**`llm_provider_id` 也要填**，否则会同时报 `LLM Provider` 未就绪。

`provider` 的 id 怎么查：在 AstrBot 后台 → 服务提供商 里能看到，形如 `deepseek/deepseek-flash`。

### 4.3 重启

```powershell
# 按你平时的启动方式重启 AstrBot
```

> ⚠️ 如果还接着 OneBots / SnowLuma 这类反向 WS 客户端，**AstrBot 重启会把它们带掉**，
> 需要一并重启（它们不会自动重连）。

---

## 第五步：验证

日志里应该出现：

```
[astrbot_plugin_livingmemory] 配置验证成功
[astrbot_plugin_livingmemory] LivingMemory 插件开始后台初始化...
[astrbot_plugin_livingmemory] LivingMemory 插件初始化成功。
[Core] plugin(...) added LLM tool: recall_long_term_memory
```

三条关键点：

| 标志 | 含义 |
|---|---|
| `LivingMemory 插件初始化成功` | 记忆系统跑起来了 |
| `added LLM tool: recall_long_term_memory` | 模型多了一个「回忆」工具 |
| 没有 `等待 Provider 就绪` 的重复日志 | embedding 已经被认到 |

**注意**：索引是空的，要**积累一两天**才有效果。不是一开就立刻「记得住」。

---

## 运维与调优

### 模型怎么选

| 模型 | 体积 | 维度 | 适合 |
|---|---|---|---|
| **bge-m3** | 634 MB (Q8_0) | 1024 | **中文最佳**，多语言，推荐 |
| nomic-embed-text | 274 MB | 768 | 轻量，英文为主 |

换成别的模型时，四样东西要一起改：

```
1. Modelfile 的 FROM 指向新 gguf
2. ollama create 的名字
3. AstrBot 的 embedding_model
4. AstrBot 的 embedding_dimensions   ← 维度错会直接报错
```

**换模型后旧索引作废**，需要在插件里重建索引。

### 内存占用

`bge-m3` 加载后常驻约 **600~800 MB**。16 GB 内存的机器没什么压力，
但如果同时跑很多程序，可以只在需要时启动 Ollama。

### 端口与安全

Ollama 默认只监听 `127.0.0.1:11434`，**局域网访问不到**，这是安全的默认值。
千万不要改成 `0.0.0.0` —— Ollama 没有鉴权，暴露到局域网等于谁都能调用。

### 不用了怎么办

```powershell
# 停服务
Get-Process -Name ollama | Stop-Process -Force

# 删掉开机自启
Remove-Item (Join-Path ([Environment]::GetFolderPath('Startup')) 'Ollama.lnk')

# 彻底清理（程序 + 模型）
Remove-Item D:\Ollama -Recurse -Force
[Environment]::SetEnvironmentVariable('OLLAMA_MODELS', $null, 'User')
```

然后在 AstrBot 里把 `embedding_provider_id` 清空、停用记忆插件，否则它会继续重试。

---

## 常见错误对照表

| 现象 | 原因 | 处理 |
|---|---|---|
| `redirect target not allowed: ... resolves to non-public 198.18.x.x` | 代理 fake-IP + Ollama 的 SSRF 防护 | 用本文的手动 GGUF 方案 |
| `waiting for server to become available` | Ollama 服务没起 | 运行 `ollama serve` |
| `model requires more system memory` | 模型太大 | 换更小的量化版（Q4_K_M） |
| 插件一直 `等待 Provider 就绪` | id 填错 / 只加了 source 没加 provider | 核对 4.1 的两处配置 |
| `Provider xxx 不是 EmbeddingProvider 类型` | `embedding_provider_id` 指到了对话模型 | 改成 embedding provider 的 id |
| 维度不匹配报错 | `embedding_dimensions` 和模型实际维度不一致 | bge-m3 是 1024，nomic 是 768 |
