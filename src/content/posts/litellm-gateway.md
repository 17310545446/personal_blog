---
title: 本地 LLM 网关：用 LiteLLM 把 Anthropic 协议接到国产模型
published: 2026-09-19
description: Claude Code、worker 这些工具只会说 Anthropic 协议，而国产模型只有 OpenAI 兼容端点。用 LiteLLM 在中间做一次翻译，顺便统一了模型别名和开机自启。
tags: [LiteLLM, 网关, 百炼, DeepSeek, launchd]
category: 服务搭建
draft: false
---

## 起因

手上几个工具都只会说 **Anthropic 协议**：

- Claude Code CLI 默认请求 `/v1/messages`；
- 项目里的 worker 也依赖 `ANTHROPIC_BASE_URL`；
- 但可用的模型是百炼（qwen 系列）和 DeepSeek。

探测了一圈：

- **DeepSeek**：没有原生 `/v1/messages`（真的 key 请求返回 404），只有 OpenAI 兼容的 `/chat/completions` 和 `/responses`；
- **百炼**：同样是 OpenAI 兼容端点。

结论：中间需要一层协议转换。**LiteLLM** 正好干这个——它对外提供 Anthropic 格式端点，对内转发到任意后端。

## 架构

```
Claude Code / worker / dsh
        │  Anthropic 协议（/v1/messages）
        ▼
   LiteLLM  :4000
        │  OpenAI 协议
        ├──► 百炼（dashscope）
        └──► DeepSeek 直连
```

好处是**一处配置、多处复用**：后来装的 dsh、Qwen Code 全都指向 `127.0.0.1:4000`，不用在每个工具里重复填 key。

## 配置要点

模型别名把工具期望的名字映射到真实后端：

```yaml
model_list:
  - model_name: claude-sonnet-4-5          # 工具以为自己在用 Claude
    litellm_params:
      model: dashscope/qwen3.8-max          # 实际打到百炼
  - model_name: deepseek-v4-flash
    litellm_params:
      model: deepseek/deepseek-v4-flash
```

### ⚠️ 最大的坑：前缀决定端点

用 `openai/` 前缀时，LiteLLM 会去调 **`/responses`** 端点——而百炼没有这个接口，直接 404。

**必须用后端原生前缀**：

```yaml
model: dashscope/qwen3.8-max   # ✅
model: openai/qwen3.8-max      # ❌ 走 /responses，404
```

这个坑排查了一阵子，因为报错只显示 404，看不出是端点路由问题。

### 其他配置

- `drop_params: true`：工具会传 Anthropic 独有参数，转发前丢掉不支持的；
- `master_key: null`：本地监听，不做鉴权（但**不要**把端口暴露到公网）；
- key 放在 `.env`（权限 600），启动时加载。

## 开机自启（launchd）

网关是需要长期在线的服务，写个 LaunchAgent：

```xml
<key>Label</key><string>ai.local.litellm-gw</string>
<key>ProgramArguments</key>
<array>
  <string>/bin/zsh</string>
  <string>/Users/me/develop-tool/litellm-gw/start.sh</string>
</array>
<key>RunAtLoad</key><true/>
<key>KeepAlive</key><true/>
```

`start.sh` 负责加载 `.env` 并拉起进程：

```bash
#!/bin/zsh
cd ~/develop-tool/litellm-gw
set -a && source .env && set +a
exec venv/bin/litellm --config litellm-gw-config.yaml --port 4000
```

管理命令：

```bash
launchctl kickstart -k gui/501/ai.local.litellm-gw   # 重启
launchctl bootout gui/501/ai.local.litellm-gw        # 停止
```

**资源占用**：内存约 350MB、空闲 CPU 0%——比跑在容器里轻得多。

## 小插曲

启动时 LiteLLM 会去 GitHub 拉模型价格表，国内会超时。**无害**，它会回落本地缓存，只是日志里多几条警告。

## 小结

- 协议不匹配时，中间加一层转换比自己改客户端划算；
- LiteLLM 的 `model` 前缀直接决定请求端点，**别想当然用 `openai/`**；
- 长期服务交给 launchd，比 `nohup` 可靠得多，重启机器也不用手动拉起来；
- 本地网关天然不鉴权，务必只监听 `127.0.0.1`。
