---
title: DeepSeek Harness 上手记：从装不上到一键启停
published: 2026-09-19
description: dsh 是个"万物皆插件"的本地 agent harness。记录安装踩的坑、怎么接进本地网关、以及它会话存储机制的设计细节。
tags: [dsh, DeepSeek, Agent, 源码阅读]
category: AI Agent
draft: false
---

## 这是什么

**DeepSeek Harness（dsh）** 是一个本地运行的 agent harness，核心理念是「万物皆插件」，底层基于 **Cordis** 插件框架。

它能跑成三种形态：

| 形态 | 用法 |
|---|---|
| Web UI | `dsh web`（默认 `127.0.0.1:3080`） |
| headless CLI | `dsh --profile headless "任务"` |
| Python SDK | JSON-RPC over stdio |

配置文件在 `~/.dsh/settings.yaml`，profile 之间通过 `cordis.patch.yml` 做覆盖。

## 一、安装：网络是最大障碍

`npm install` 直接卡死。有效的组合是**三件套**：

```bash
npm install --registry=https://registry.npmmirror.com \
  --fetch-timeout=30000 --fetch-retries=3
```

再加上环境变量：

```bash
export NODE_OPTIONS=--dns-result-order=ipv4first
```

**为什么需要 IPv4 优先**：IPv6 解析会挂起等待超时，`ipv4first` 之后立竿见影。

> 顺带一提：用 pnpm/corepack 装 dsh 会失败——corepack 首次要下载 pnpm 本体，网络不好会直接被 SIGKILL。

## 二、接本地网关

dsh 支持自定义 provider。指向本地 LiteLLM 网关，**完全不需要 API key**：

```yaml
# ~/.dsh/settings.yaml
providers:
  local-gw:
    base_url: http://127.0.0.1:4000
```

headless 模式如果要走官方 API，才需要 `DEEPSEEK_API_KEY`。

## 三、一键启停

Web UI 需要网关 + dsh 两个进程都在，手动拉太麻烦，于是写了两个命令：

```bash
dshup     # 幂等拉起：网关(4000) + dsh web(3080)，并打开浏览器
dshdown   # 停止两者
```

实现里踩到一个 **zsh 特性坑**：

```bash
kill $pids          # ❌ zsh 变量不分词，杀不掉多个 PID
kill ${(f)pids}     # ✅ 或者 xargs
```

## 四、顺手读了一遍它的存储设计

dsh 的会话日志设计挺讲究，值得记一笔（源码在 `packages/session/session-persistence-jsonl/`）：

### 目录布局

```
~/.dsh/sessions/--<cwd 编码>--/<sessionId>/session.jsonl.zstd
```

- cwd 编码做了**注入式转义**（`~XXXX` UTF-16 码元），注释原话是 *no traversal, no collision*——防路径穿越和碰撞；
- 目录权限 `0o700`。

### 写入的原子性

会话文件采用 **temp-write → fsync → link() 发布 → fsync 目录** 的流程。

这样设计的好处：任何时刻崩溃，都不会留下"写了一半可见"的文件。而**崩溃恢复**的做法是：

1. `truncate` 到最后一个完整帧边界；
2. 恢复残帧里的事件；
3. 补齐 closers，再 fsync 两次。

### 压缩

自研的 zstd 拼接帧容器：每帧带 checksum，并且有一种**不解压就能扫帧边界**的能力——这让"从尾部修复"变得便宜。

### 血缘信息

首行 header 里带了 `parentSession` / `origin: 'subagent'` / `delegationDepth`——子 agent 的会话血缘是显式记录的，方便回溯和隔离。

## 五、结论与风险

- 存储层设计扎实：原子发布、崩溃恢复、帧级校验，该有的都有；
- 但要注意它是 **developer preview**：官方明确警告会有破坏性变更、无安全审计，别上生产；
- 接入方式很灵活（本地网关 + 无密钥），适合当"折腾沙盒"。

> 后续想做的：拿它跑一遍完整的任务生命周期，验证 replay/fork/resume 三个能力。
