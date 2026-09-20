---
title: macOS 定时任务实战：50 天重启策略与内存告警（附一次翻车）
published: 2026-09-18
description: 用 launchd 给 Mac 做长期值守——定期重启、内存监控、告警推送。以及我写错一行解析代码，导致机器被误重启的完整复盘。
tags: [launchd, macOS, 自动化, Shell, 踩坑]
category: 自动化
draft: false
---

## 为什么需要这些

这台 Mac 长期当服务端跑（网关、容器、代理、agent），几个月不关机是常态。

有一次发现问题：**连续运行 130 天**后，系统进入一种微妙的状态——空闲内存只剩 300 多 MB、内存压缩接近 2GB，随之而来的是 gateway 进程被反复冻结，UI 操作卡住。

重启后立刻恢复：空闲内存 3.9GB、压缩归零。

于是决定把两件事自动化：**定期重启** + **内存监控**。

## 一、50 天重启策略

### 设计

- 由 launchd 每天凌晨 3:30 触发一次检查；
- 检查"距上次启动是否 ≥ 50 天"，满足才重启；
- 所有决策写进日志，便于回溯。

### 判断运行时长

```bash
boot_sec=$(sysctl -n kern.boottime | awk '{print $4}' | tr -d ',')
now=$(date +%s)
days=$(( (now - boot_sec) / 86400 ))
```

## ⚠️ 二、翻车现场：我的脚本把用户的机器重启了

上面那行 `awk` 是**修好之后**的版本。第一版我写的是：

```bash
boot_sec=$(sysctl -n kern.boottime | sed -E 's/.*sec = ([0-9]+).*/\1/')
```

看起来没问题，实际输出是什么呢？

```bash
$ sysctl -n kern.boottime
{ sec = 1789887597, usec = 783081 } Sun Sep 20 14:59:57 2026
```

`sed` 的 `.*` 是**贪婪匹配**，它匹配到了 **`usec = ` 里的 `sec = `**，于是提取出来的数字是 `783081` 之后的东西——一个完全错误的值。

结果：

```
已连续运行 20707 天 (≥50 天) → 触发重启
```

**脚本算出来机器已经跑了 56 年，于是毫不犹豫地执行了重启。**

更糟的是第二层错误：我是在用户的生产机上"顺手测试"这个脚本的，**没有加任何 dry-run 保护**。

### 修复：三重防护

```bash
# 1) 可靠解析：用 awk 取字段，不用贪婪正则
boot_sec=$(sysctl -n kern.boottime | awk '{print $4}' | tr -d ',')

# 2) 校验一：必须是纯数字
case "$boot_sec" in
  ''|*[!0-9]*) echo "解析失败，拒绝执行" >> "$LOG"; exit 1 ;;
esac

# 3) 校验二：必须是 9~11 位（约 1973~2286 年）
[ ${#boot_sec} -ge 9 ] && [ ${#boot_sec} -le 11 ] || exit 1

# 4) 校验三：运行天数必须落在 0~10000
[ "$days" -ge 0 ] && [ "$days" -le 10000 ] || exit 1

# 5) dry-run 开关
[ "${REBOOT_POLICY_DRY_RUN:-0}" = "1" ] && { echo "DRY-RUN，跳过"; exit 0; }
```

现在的行为：

```
已运行 0 天 (<50)，跳过
```

### 这一课的总结

1. **解析系统命令输出时，永远别信 `.*` 贪婪正则**——用 `awk '{print $N}'` 取精确字段；
2. **会触发副作用的脚本，必须内置 dry-run**，并且测试时先 dry-run；
3. **"解析失败"必须当作异常退出**，而不是让错误值继续参与运算；
4. 关键操作加**范围校验**（天数不可能超过一万）——这是最后一道闸。

## 三、内存监控

思路很简单：定时采集、阈值判断、超限告警。

### 采集

```bash
page_size=$(vm_stat | awk '/page size/{print $8}')
free_pages=$(vm_stat | awk '/Pages free/{print $3}' | tr -d '.')
comp_pages=$(vm_stat | awk '/occupied by compressor/{print $5}' | tr -d '.')
```

### 阈值

| 指标 | 健康 | 告警线 |
|---|---|---|
| 空闲内存 | > 2GB | **< 400MB** |
| 压缩内存 | < 2GB | **> 4GB** |

> 为什么看这两个？macOS 会主动用内存做缓存，**"空闲内存少"本身不是问题**；但**压缩内存**和 **swap** 一开始增长，说明系统真的在硬撑了。

### 告警要能"送到手上"

本地弹窗（`osascript display notification`）只在你坐在电脑前时有用。真正有用的是**推到手机**。

这里的坑：想推 QQ，但渠道插件的命令行发送有兼容问题：

```
❌ outLog.debug is not a function
```

试了另一条路（agent 投递）也不行：

```
Unable to resolve a session route for channel "qqbot"
```

**最后可行的是"自动化任务 + announce 投递"通道**——由脚本触发一个已配置好投递目标的任务，消息就能正常到达。

顺带一个经验：**脚本的告警内容和日志要分开写**。第一版把任务触发输出追加到了监控日志里，导致告警读取逻辑读到了自己的输出。改成写独立文件后就干净了。

## 四、launchd 配置速查

```xml
<!-- 每天 03:30 跑一次 -->
<key>StartCalendarInterval</key>
<dict>
  <key>Hour</key><integer>3</integer>
  <key>Minute</key><integer>30</integer>
</dict>

<!-- 或者：每 1800 秒跑一次 -->
<key>StartInterval</key><integer>1800</integer>

<key>RunAtLoad</key><true/>
<key>ProcessType</key><string>Background</string>
<key>StandardOutPath</key><string>/tmp/job.out.log</string>
<key>StandardErrorPath</key><string>/tmp/job.err.log</string>
```

加载与查看：

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.local.mem-watch.plist
launchctl list | grep mem-watch
```

**注意**：系统睡眠时定时任务可能延后到唤醒后补跑，别指望精确到分钟。

## 小结

| 经验 | 说明 |
|---|---|
| 解析输出用 awk | 贪婪正则会把相邻字段也吃掉 |
| 加范围校验 | 异常值要拒绝执行，不能"继续算" |
| 副作用脚本先 dry-run | 这是最容易忽略的一条 |
| 监控看压缩/swap | 别被"空闲内存"的数字吓到 |
| 告警要有推送通道 | 本地弹窗只解决一半问题 |
| 日志与告警数据分离 | 避免自读自写 |
