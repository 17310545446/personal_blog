---
title: 国内开发环境生存指南：代理、镜像源与那些诡异的连不通
published: 2026-09-18
description: 两个月里遇到的网络问题合集——谁需要走代理、哪个源该用哪个镜像、DNS 解析陷阱，以及一次被 Chrome 启动参数坑惨的经历。
tags: [网络, 代理, 镜像源, npm, brew, Docker]
category: 网络
draft: false
---

在国内做开发，"能不能连上"是个天天要面对的问题。这篇把两个月踩过的坑整理成一份速查。

## 一、代理：不是所有流量都该走

本地是 **Shadowsocks + SOCKS5（127.0.0.1:1080）** 的方案。关键认知：**分流比全局更快**。

- **git**：只让 `github.com` 走代理

  ```bash
  git config --global http.https://github.com.proxy socks5h://127.0.0.1:1080
  ```

  注意是 `socks5h`——`h` 表示**由代理解析 DNS**。这点很重要，后面会说。

- **SSH**：GitHub 的 22 端口经常被掐，改走 443：

  ```
  Host github.com
    HostName ssh.github.com
    Port 443
    ProxyCommand sh -c "nc -X 5 -x 127.0.0.1:1080 %h %p 2>/dev/null || exec nc %h %p"
  ```

  最后的 `|| exec nc %h %p` 是**回落**：代理没开时自动直连，不至于把自己锁死。

- **国内站点**：一律直连，**别走代理**（会绕远路甚至超时）。

## 二、DNS：比想象中更微妙

### 一个虚惊

查 `gitee.com` 的解析，结果是这样的：

```
gitee.com → gitee.com-xxxx.baiduads.com → 180.76.x.x
```

看到 `baiduads`（像是"百度广告"）时我以为是 **DNS 劫持**。但用多个公共 DNS（阿里、腾讯、百度）查询，**返回完全一致**——说明这是真实配置：Gitee 用百度云 CDN，那个域名只是 CDN 的 CNAME 而已。

**教训**：怀疑劫持前，先用多个可信 DNS 交叉验证。

### `socks5` vs `socks5h`

- `socks5://`：**本地**解析 DNS，再把 IP 交给代理——本地 DNS 被污染时，代理也救不了；
- `socks5h://`：**交给代理**解析——能绕过本地 DNS 污染。

两个月的经验：**能用 `socks5h` 就用它**。

## 三、DNS/协议之外的"连不上"

### Gitee 的 SSH 谜题

现象：`ssh -T git@gitee.com` 被关掉，还收到一段 **HTTP 400**：

```
kex_exchange_identification: banner line 0: HTTP/1.1 400 Bad Request
Server: ADAS/1.0.214
```

看起来像 SSH key 没配好。实际排查下来：

- **22 端口完全正常**（`Hi <user>! You've successfully authenticated`）；
- 问题出在 SSH 配置里被改成了 **443 端口**——而 443 是 HTTPS 服务，SSH 握手打上去，被云厂商的防护层（ADAS）当成异常请求返回 400。

**结论**：GitHub 可以用 443 跑 SSH，**Gitee 不要照抄这个配置**。

### 代理"假活"

现象：`ss-local` 进程在、端口通，但连不上远端节点（HTTP 000）。

这类状态不用纠结原因，重启服务即可：

```bash
brew services restart shadowsocks-libev
```

### 谁直连可用？（实测结论）

| 目标 | 直连 | 备注 |
|---|---|---|
| `api.github.com` | ✅ | 拿数据首选 API |
| `github.com` / raw | ❌ | 走代理 |
| GitHub Release CDN | ✅ | 实测 1.3MB/s，不用镜像 |
| `registry.npmjs.org` | ⚠️ | 极慢，会被 SIGKILL |
| `registry-1.docker.io` | ❌ | `curl → 000`，必须用镜像源 |

## 四、镜像源清单

| 生态 | 源 |
|---|---|
| Homebrew | 中科大（`mirrors.ustc.edu.cn`） |
| npm | `https://registry.npmmirror.com` |
| pip | 清华 TUNA |
| Docker | `docker.m.daocloud.io`、`docker.1ms.run` |

**Homebrew 特别提醒**：非交互 shell 不会加载 `.zshrc` 里的镜像变量，`brew` 会卡在 `formulae.brew.sh`。要么显式带上变量，要么关掉自动更新：

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install <pkg>
```

## 五、两个"怪现象"的根因

### npm 挂起不动

不是网速问题，是 **IPv6 解析卡死**：

```bash
export NODE_OPTIONS=--dns-result-order=ipv4first
npm install --fetch-timeout=30000 --fetch-retries=3
```

这套组合基本解决了所有 npm 挂起。

### 大文件下载"太慢"

别急着找镜像——**先测速**。

- 某次 270MB 的 DMG，直觉觉得"国内肯定慢"，实测直连 **1.3MB/s，3 分钟下完**；
- 另一次 200MB 的二进制，只有 **570KB/s**，那就老老实实多线程下载。

**时段和节点差别很大，测了再决定。**

## 六、最冤的一次：Chrome 打不开国内网站

现象：某个国内控制台打不开，无痕模式也不行；但 `curl` 直连返回 200。

**根因**：Chrome 主进程是带着这个参数启动的：

```
--proxy-server=socks5://127.0.0.1:1080
```

于是**所有窗口（包括无痕）都被强制走代理**，国内站自然超时。

而更隐蔽的是：**Chrome 已经在运行时，重新执行"带参数启动"命令不会改变已有进程的参数**——你以为切回来了，其实没有。

**解法**：

1. `Cmd+Q` **完全退出** Chrome；
2. 不带参数重新打开；
3. 验证：`ps aux | grep proxy-server` 应该为空。

**教训**：给浏览器加代理要用**独立 profile 或扩展**，别用启动参数。

## 小结

- 代理要**分流**：国外走代理、国内直连；
- DNS 一律优先 `socks5h`（远端解析）；
- 怀疑劫持先**多方交叉验证**；
- 镜像源要**定期验证有效性**（会挂）；
- npm 卡住先试 `ipv4first`；
- 下载慢**先测速**再决定要不要折腾镜像；
- 浏览器代理**别用启动参数**。
