---
title: 在 MacBook 上跑幻兽帕鲁服务器：Rosetta 翻车与 Box64 得救
published: 2026-09-19
description: 帕鲁服务端只有 x86_64 版本，Mac 没有原生支持。记录从 Rosetta 模拟失败到 Box64 方案成功的过程，以及存档编辑踩到的 9 小时大坑。
tags: [Docker, OrbStack, Palworld, 游戏服务器, Box64]
category: 折腾
draft: false
---

## 背景

手上这台 MacBook Pro（Apple Silicon / 32GB）想兼职当幻兽帕鲁服务器，给几个朋友联机用。

第一道坎就来了：**帕鲁官方服务端只有 x86_64 的 Linux/Windows 版本，没有 macOS 原生版**。

于是必须在 ARM 上跑 x86_64 容器，模拟层无非两条路：

| 路线 | 原理 | 结果 |
|---|---|---|
| 路线 A | OrbStack + Rosetta（`platform: linux/amd64`） | ❌ 失败 |
| 路线 B | 原生 ARM 镜像 + 内置 Box64 | ✅ 成功 |

## 路线 A：Rosetta 失败在哪

镜像选的是 `jammsen/palworld-dedicated-server`，容器确实能起来，但服务端在初始化存档系统时崩了：

```
Save data is corrupted (PalSaveGameManager.cpp:2053)
```

连 `Pal.log` 都写不出来。排查过程包括删 `GameUserSettings.ini`、删 `SaveGames`、调性能参数——**全部无效**。

结论：UE5 的存档初始化在 Rosetta 下有问题，不是配置能绕过去的。

期间还顺手踩了个坑：镜像里的 steamcmd 默认用 `STEAM_PLATFORM=linux32`，32 位客户端在 Rosetta 下直接 **SIGSEGV**（日志表现是卡在 `Loading Steam API...`）。要在 compose 里显式覆盖：

```yaml
environment:
  - STEAM_PLATFORM=linux64    # 列表语法，不是 map
```

## 路线 B：Box64 成功

换 `thijsvanloef/palworld-server-docker`——这个镜像**原生支持 Apple Silicon，内置 box64 做 x86→ARM 翻译**。

关键点：

- **不要**写 `platform: linux/amd64`（用原生 arm64 镜像）；
- `UPDATE_ON_BOOT=false` 跳过容器内 steamcmd。

起来之后一切正常：UDP 8211 监听中、容器 healthy、存档和自动备份都工作。

### 游戏本体怎么下

容器内 steamcmd 不可用（Rosetta 崩溃 + 网络），所以用**宿主机 macOS 版的 steamcmd** 下载：

- 内容 CDN 直连速度约 **30MB/s**，总计 5.15GB，比容器里快得多。

## Docker 网络：国内拉镜像

`registry-1.docker.io` 和 `auth.docker.io` 直连都是 `curl → 000`。解决办法是配镜像源：

```json
// ~/.orbstack/config/docker.json
{"registry-mirrors": ["https://docker.m.daocloud.io", "https://docker.1ms.run"]}
```

**两个教训**：

1. 镜像源要定期验证——配过的 `docker.1panel.live` 后来就挂了；
2. ⚠️ **不要给 OrbStack 设 `network_proxy`**。当时试过 `orb config set network_proxy socks5://...`，结果容器的国际出口**全挂**（连 steamcdn 都超时），改回 `auto` 并完整重启才恢复。

## 运维踩坑

### 改 ini 无效！

第一次想调参，直接改容器里的 `PalWorldSettings.ini`——重启后**全被覆盖**。

原因：镜像每次启动都用 **模板 + 环境变量重建 ini**。

所以：

- 改配置 → 改 `docker-compose.yml` 的 env；
- 改完必须 `docker compose up -d`（**recreate**），`restart` 不会应用新 env。

### 在线人数怎么判断

一开始看日志判断玩家在线，结果踩了大坑：**异常断线时日志没有 leave 记录**——有个值守任务因此空等了 7 小时。

正确姿势是查 **REST API**：

```bash
curl -u admin:<密码> http://<容器IP>:8212/v1/api/players
```

（RCON 端口通但协议层无响应，box64 模拟下不可用。）

## 存档编辑：9 小时的血泪

因为想给朋友们做几个"满强化"的帕鲁，深入折腾了 `Level.sav` 的二进制结构。

最大的坑是 **GUID → 存档槽位** 的映射：

```python
# ✅ 正确：每 4 字节一组做字节级反转
bytes.fromhex(guid)[i:i+4][::-1]

# ❌ 错误：整体字符串反转
```

用错方法的后果极其隐蔽：**回读验证全部通过，但服务器重启后帕鲁消失**——因为被当成孤儿数据静默清理了。

排查了整整 9 个小时才定位到这一行。

另外两个发现：

- 改档工具别放 `/tmp`——macOS 会清理，工具链丢过一次（含编译好的二进制），后来迁移到 `~/develop-tool/pal-tools/`；
- GUID 与存档槽是**一一对应**的，多人共享存档时改档要小心冲突。

## 小结

| 事项 | 结论 |
|---|---|
| ARM 上跑 x86 服务 | 优先找**内置 box64** 的原生 ARM 镜像，别硬上 Rosetta |
| Docker 镜像 | 国内必须配镜像源，且要定期验证 |
| OrbStack | 别动 `network_proxy` |
| 容器配置 | 改 env + recreate，不是改 ini + restart |
| 状态判断 | 用 REST API，别信日志 |
| 二进制结构 | 验证要"重启后仍然有效"，不能只看回读 |
