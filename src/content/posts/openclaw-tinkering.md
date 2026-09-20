---
title: OpenClaw 折腾记：更新失败、插件装不上、渠道接不通
published: 2026-09-18
description: 一次版本更新踩的坑、插件系统的实际加载规则、以及"渠道配好了但消息发不出去"的排查思路。
tags: [OpenClaw, 插件, 渠道, 自动化]
category: AI Agent
draft: false
---

跑了一段时间的 OpenClaw（一个多渠道 AI 助理网关），记录三类典型问题。

## 一、更新：为什么它拒绝升级自己

现象：在 Web 控制台点"更新"，日志显示

```
update.run ... status=skipped
```

手动执行也一样：

```
openclaw update detected it is running inside the gateway process tree.
Gateway PID ... is an ancestor of this process
```

**根因**：更新过程需要停机、替换包、再重启——所以更新器**拒绝在它自己所属的 gateway 进程树里运行**（否则等于让人一边跑一边给自己换腿）。

**解法**：把它从 gateway 树里"摘"出来，交给系统的 launchd 启动：

```bash
launchctl submit -l com.me.openclaw-update \
  -o /tmp/oc-update-out.log -e /tmp/oc-update-err.log \
  -- /bin/zsh -lc 'openclaw update --yes --json'

# 完成后清理
launchctl remove com.me.openclaw-update
```

这样父进程是 launchd，不在 gateway 树内，更新器就肯干活了。实测升级成功，**停机时间为 0**（更新器自己刷新了服务定义）。

**另一个选择**：直接在终端里跑（只要不是从 gateway 子进程起）也一样。

## 二、插件：文档说的目录已经不扫描了

某个插件（rtk）的 README 说：

> 把文件放进 `~/.openclaw/extensions/`，重启 gateway 即可。

**这么做完全没反应**。实际规则（当前版本）：

1. **扫描的不是 `extensions/` 目录**；
2. 插件目录需要两个清单文件：
   - `openclaw.plugin.json`
   - `package.json`（含 `openclaw.extensions` 字段）；
3. **必须是编译后的 JS**（`.cjs` / `.mjs` / `.js`）——TS 源文件只在开发路径下被支持；
4. 安装命令：

```bash
openclaw plugins install <目录> --force --accept-capabilities
openclaw gateway restart
```

5. 排查用 `openclaw plugins list`，看 **Source** 列确认实际加载路径。

另一个坑：某些 CLI 的"检查"逻辑还在看旧目录，会误报"未检测到插件"——**以加载日志为准**。

## 三、渠道：配好了 ≠ 能用

接消息渠道时最容易混淆的是"配置成功"和"真的连通"。

### QQ 机器人

- 配置文件里 `enabled: true` ≠ 已连接：**凭据文件不存在时，通道从未真正建立**（日志里一片空白）；
- 用扫码登录后，凭据写入配置，日志才出现：

```
Access token obtained → WebSocket 连接 → gateway READY
```

- 装好之后建定时任务，又踩两个坑：
  1. 独立会话 + 播报投递 → 报 `Channel is required (no configured channels detected)`——**必须等渠道真的连上**才行；
  2. 往主会话注入事件 → 只有注入、**没有投递**（`deliveryStatus: not-requested`），人收不到。

**验证方式**：建完任务别等，直接**手动跑一次**，看返回里 `delivered: true` 才算通。

### 飞书

走扫码接入，记得把私聊策略设成白名单（`dmPolicy=allowlist`）并锁自己的 ID，避免陌生人触发。接入后"手机遥控本机 Mac"的场景就通了。

## 四、定时任务的两个概念

| 类型 | 适用 | 注意 |
|---|---|---|
| 主会话注入 | 让助理"自己想起来"做事 | 不会主动发消息给你 |
| 独立会话 + 播报 | 结果要送到聊天渠道 | 必须指定渠道与目标 |

频率、时区、一次性/周期性都能配；一次性任务可以设"跑完自删"。

## 小结

| 问题 | 结论 |
|---|---|
| 更新被拒 | 从 gateway 进程树外启动（launchd 或独立终端） |
| 插件装不上 | 看加载日志，不看 README；需要编译后 JS + 清单文件 |
| 渠道不通 | 配置 ≠ 连接，去日志里找 `READY` |
| 消息没收到 | 手动跑一次，看 `delivered` |
| 定时任务 | 分清"注入"和"播报" |

**最大的体会**：这类"胶水系统"的问题，九成能在**日志**里找到答案——配置文件看起来永远是对的。
