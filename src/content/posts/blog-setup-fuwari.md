---
title: 从零搭一个博客：为什么最后选了 Fuwari
published: 2026-09-20
description: Hugo、Hexo、Astro 都试了一遍，说说静态博客的选型逻辑、GitHub Pages 部署链路，以及过程中踩的坑。
tags: [博客, Astro, GitHub Pages, 建站]
category: 建站
draft: false
---

折腾了两个月的各种服务之后，觉得该有个地方把这些东西记下来。于是有了这个博客。

这篇文章记录建站过程本身——包括选型怎么定的，以及部署链路的坑。

## 一、静态博客的选型

需求很简单：Markdown 写作、免费托管、访问速度可接受、颜值过得去。

候选方案对比了几轮：

| 方案 | 优势 | 顾虑 |
|---|---|---|
| **GitHub Pages + Jekyll** | 官方原生支持，零配置部署 | 本机 Ruby 版本偏旧，主题生态陈旧 |
| **Hugo + PaperMod** | 构建毫秒级，单二进制，主题多 | 默认设计偏"文档感"，需要自己调 |
| **Hexo** | Node 生态，中文教程多 | 依赖树重，构建慢 |
| **Astro + Fuwari** | 交互最好，View Transitions 丝滑 | 换框架，构建链变复杂 |

最后实际的做法是：**三个主题同时跑起来对比**——Hugo 的 PaperMod（自己加了一版美化 CSS）、Blowfish、以及 Astro 的 Fuwari。

对比下来结论很清晰：

- PaperMod 是最"稳"的选择，零迁移成本，但设计语言偏克制；
- Blowfish 功能最丰富，留在 Hugo 生态里，迁移成本也低；
- **Fuwari 的交互体验是另一个量级**——页面切换有动画、搜索是即时的、整体流畅到不像静态站。

考虑到主要是"记录"而非"功能站"，最后选了 Fuwari。颜值和交互带来的写作积极性，在我这儿权重比较高。

## 二、部署链路

托管用的是 **GitHub Pages**，构建交给 **GitHub Actions**：

```
本地写 Markdown  →  git push  →  Actions 构建 Astro  →  发布到 Pages
```

工作流核心就几步：装 pnpm → 装依赖 → `pnpm build` → 上传产物 → 部署。

`pnpm build` 这里除了 `astro build`，还会跑一次 **Pagefind** 生成搜索索引——这是 Fuwari 搜索功能能工作的原因。

## 三、踩过的坑

### 1. Gitee Pages 走不通

一开始想用 Gitee Pages（国内访问快），结果卡在两个硬性限制上：

- 该账号只能建**私有仓库**（未实名认证）；
- Gitee Pages 免费版要求**公开仓库**。

所以这条路直接放弃，转向 GitHub Pages。

### 2. `git clone` 残留的上游历史

主题是 `git clone` 下来的，目录里天然带着上游仓库的 `.git`。结果：

- `git remote add origin` 报 "remote already exists"；
- `git fetch` 拉回来的是**上游主题仓库**的历史，不是自己的。

正确做法是重建历史：

```bash
rm -rf .git
git init -b main
git add -A && git commit -m "feat: 初始化博客"
git remote add origin git@github.com:<你>/<仓库>.git
git fetch origin
git reset --soft origin/main   # 站在远程历史上
git add -A && git commit -m "feat: 迁移到新主题"
```

这样得到一个**干净、可追溯、不需要强制推送**的迁移提交。

### 3. CI 里的 pnpm 版本冲突

工作流里写了 `version: 9`，而 `package.json` 里有：

```json
"packageManager": "pnpm@9.14.4"
```

两者冲突，CI 直接报错退出：

```
Error: Multiple versions of pnpm specified
```

**修复**：工作流里不要写 pnpm 版本，让它读 `packageManager` 字段就好。

### 4. base 路径

项目部署在 `https://<user>.github.io/<repo>/`，所以 Astro 配置必须写：

```js
site: "https://<user>.github.io",
base: "/<repo>",
```

否则所有静态资源 404。

## 四、日常写作流程

```bash
pnpm new-post 文章标题   # 生成带 frontmatter 的文件
pnpm dev                # 本地预览
git add -A && git commit -m "post: ..." && git push   # 发布
```

文章 frontmatter 长这样：

```yaml
---
title: 标题
published: 2026-09-20
description: 摘要
tags: [标签1, 标签2]
category: 分类
draft: false
---
```

推送后大约 1 分钟，线上就能看到。

## 小结

- 静态博客的关键决策是**主题/框架**，不是托管——托管方案都足够便宜和可靠；
- GitHub Actions 让"写 → 推 → 上线"这条链路完全自动化；
- 国内的取舍：GitHub Pages 稳定但慢，要速度得另找方案；
- 迁移仓库时**先理清 git 历史**，比事后救火省事得多。
