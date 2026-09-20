---
title: AI 编码 CLI 四件套怎么装：Claude Code / Codex / Qwen Code / Kimi Code
published: 2026-09-18
description: 四个终端编码 agent 在国内网络下的安装与配置实录——官方脚本、npm 遮蔽、协议差异、以及"零密钥接本地网关"的做法。
tags: [CLI, Claude Code, Codex, Qwen Code, Kimi Code, 工具链]
category: 工具链
draft: false
---

终端里的 AI 编码助手越来越多，装法各不相同，国内网络下还各有各的坑。这篇把四个主流的安装配置记录一下。

## 一句话对比

| 工具 | 装法 | 特点 | 最大坑 |
|---|---|---|---|
| Claude Code | npm / 官方脚本 | 原版体验 | 需要 Anthropic key（可用网关中转） |
| Codex | npm | 只支持 `responses` 协议 | 国内模型大多不支持该协议 |
| Qwen Code | 官方脚本（阿里 OSS） | 国内下载快 | 旧版 npm 包会遮蔽新版 |
| Kimi Code | 官方脚本 | 无 Node 依赖 | 二进制约 200MB，下载慢 |

## 一、Claude Code：用网关解决 key 问题

没有真的 Anthropic key，但 Claude Code 支持自定义 base URL——**指着本地 LiteLLM 网关就行**：

```
ANTHROPIC_BASE_URL=http://127.0.0.1:4000
ANTHROPIC_API_KEY=<占位符>
```

网关那边把 `claude-*` 映射到国产模型（见《本地 LLM 网关》那篇）。效果是 Claude Code 的交互体验不变，后端换成国内模型，成本骤降。

## 二、Codex：协议是硬门槛

### 安装

```bash
npm i -g @openai/codex@<版本> --registry=https://registry.npmmirror.com
```

官方 `install.sh` 被墙，`codex.openai.com` 不通——**npm 镜像是最省事的路**。

### ⚠️ 配置格式变了

网上很多教程是旧版写法，直接抄会报错：

```toml
# v0.146 正确写法
approval_policy = "on-request"        # 顶层枚举，不是表
notify = ["exit"]                     # 数组，不是 [notify] 表
model_reasoning_effort = "medium"
```

### ⚠️ 只支持 `responses` 协议

新版**移除了 `chat` wire API**，只认 `wire_api = "responses"`。

我实测了国内几家：

| 服务商 | `/v1/responses` |
|---|---|
| DeepSeek | ✅ 存在（401 = 需要 key） |
| 百炼 / 智谱 | ✅ 存在 |
| Kimi / 硅基流动 | ❌ 404 → 必须走网关转换 |

### 401 排查记录

报错：

```
401 auth header format should be Bearer sk-...
```

第一反应是配置写错了，改来改去更糟（甚至改出一个 `Missing API KEY`）。**真正原因是时序问题**：第一次运行时凭据文件还没生成，codex 手上根本没 key。

**结论**：别急着改配置，先确认凭据文件已经写入。另外 codex 要求在 git 仓库（或受信任目录）里运行，否则得加 `--skip-git-repo-check`。

## 三、Qwen Code：官方脚本最稳

```bash
curl -fsSL https://qwen-code-assets.oss-cn-hangzhou.aliyuncs.com/installation/install-qwen-standalone.sh | bash
```

走的是阿里 OSS，**国内速度很好**。

### ⚠️ 旧版遮蔽

之前用 npm 装过旧版本，二进制落在 `/opt/homebrew/bin/qwen`，而新版在 `~/.local/bin/qwen`——**旧版会把新版遮住**，`qwen --version` 看到的还是老的。

```bash
npm uninstall -g @qwen-code/qwen-code   # 清掉旧的
```

### 零密钥接本地网关

```json
// ~/.qwen/settings.json
{
  "modelProviders": {
    "openai": {
      "baseUrl": "http://127.0.0.1:4000/v1",
      "models": ["qwen3.8-max", "deepseek-v4-pro"]
    }
  }
}
```

**踩坑**：`settings.json` 里的 `env` 字段**不生效**——key 必须通过 `.env` 文件或环境变量提供：

```bash
# ~/.qwen/.env
OPENAI_API_KEY=placeholder    # 本地网关不鉴权，占位即可
```

### 权限模式

| 模式 | 行为 |
|---|---|
| `plan` | 只读，适合"先看看" |
| `default` | 每步确认 |
| `auto-edit` | 自动改文件（推荐日常用） |
| `yolo` | 全自动 |

`--acp` 参数还能把它接进支持 ACP 的宿主。

## 四、Kimi Code：两个安装脚本别搞混

```bash
# 新版（推荐）
curl -fsSL https://code.kimi.com/kimi-code/install.sh | bash

# 旧版（kimi-cli）
curl -fsSL https://code.kimi.com/install.sh | bash
```

路径不同，**装错会得到两套东西**。新版无需 Node，装到 `~/.kimi-code/bin/kimi`。

**注意**：ARM 二进制 200MB+，源站速度一般（~570KB/s），要等几分钟。支持 `kimi web`（浏览器 UI）和 `kimi acp`。

**额度问题**：OAuth 登录后如果提示 `403 monthly usage limit`，说明套餐额度用完了——要么升级，要么改成 API key 计费。

## 五、通用经验

### 一律用镜像

```bash
npm install --registry=https://registry.npmmirror.com
export NODE_OPTIONS=--dns-result-order=ipv4first
```

### 装完验证 PATH

装在 `~/.local/bin` 或 `~/.kimi-code/bin` 的工具，记得确认新终端能找到：

```bash
zsh -ic 'which qwen kimi codex claude'
```

### 优先"零密钥"方案

四个工具里有三个（Claude Code、Qwen Code、dsh）都能指向本地网关——**key 只配一处**，换模型不用动客户端。

## 小结

- **Claude Code**：base URL 指网关，解决 key 问题；
- **Codex**：先确认协议（`responses`）和凭据文件，别乱改配置；
- **Qwen Code**：官方脚本 + 卸载旧 npm 版 + key 放 `.env`；
- **Kimi Code**：认准新脚本路径，注意额度；
- 通用：**镜像源 + IPv4 优先 + 网关统一密钥**。
