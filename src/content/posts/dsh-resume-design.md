---
title: dsh 的 resume 是怎么实现的：追加式日志与崩溃恢复
published: 2026-09-22
description: 拆解 DeepSeek Harness 的 resume 实现——日志即真相、尾部修复靠追加、未发布会话的独占所有权，以及它如何把崩溃恢复写成模型能理解的上文。
tags: [dsh, Agent, 源码阅读, 崩溃恢复, 设计]
category: AI Agent
draft: false
---

> 承接上一篇《DeepSeek Harness 上手记》，这篇深入它最有意思的部分：**resume**。
> 代码基于 `deepseek-harness` 仓库某次提交，注意它是 developer preview，接口会变。

## 0. 一句话结论

resume **不是"恢复内存快照"**，而是：

```
以持久事件日志为唯一事实源 → 冷读 → 修复崩溃尾部 → 独占预留 → 播种重建 → 续写
```

核心取向：**修复 = 追加事件，绝不改写已有历史**。

---

## 1. 前提：日志即真相

dsh 的会话就是一个 **append-only 事件日志**，内存态只是它的投影。

| 机制 | 代码事实 |
|---|---|
| 事件不可改写 | 事件在被接受时**深冻结**（deep-frozen），快照在下一次 append 前复用 |
| header 不进日志 | 注释原话：header 是"存储关注点，不是可重放会话状态" |
| 构造即播种 | "用已有事件日志播种 = replay/fork 一个会话"——resume 也走同一入口 |
| 双边界 | `firstLiveSeq`（本进程第一个 seq，= 种子长度）vs `header.seedLength`（**持久的 fork 血缘边界**）。resume 时种子是**整份历史**，但 header 保留原始 fork 值 |

---

## 2. 全链路（六个阶段）

### ① 身份决议（agent-loop）

```ts
// sessionId 与 resumeSessionId 互斥
throw new Error(`agent "${id}": sessionId and resumeSessionId are mutually exclusive`)
const exactIdentity = hasResumeId ? resumeSessionId : sessionId
```

### ② 冷读与独占预留（persistence coordinator）

```ts
async prepare(id: SessionId, signal?: AbortSignal): Promise<SessionPreparation> {
  for (;;) {
    await this.waitForRetirement(id, signal)
    if (this.ctx.sessions.get(id) !== undefined)
      throw new Error(`cannot prepare session "${id}" while it is live`)
    const reservation = await this.preparations.reserve(
      id,
      () => this.serialize(id, () => this.prepareCore(id)),          // 冷加载
      source => this.serialize(id, () => this.commitPrepared(source), signal), // 落盘修复
      signal,
    )
    if (reservation === undefined) continue   // 被 invalidate → 重试
    ...
  }
}
```

三个要点：

- **`waitForRetirement`**：等上一个持有者退场；
- **live 检查**：已挂载的会话不能 prepare —— 从根上杜绝"两个实例复活同一会话"；
- **收敛循环**：`reserve` 返回 `undefined` 说明条目被并发写入作废，重试即可。

### ③ 加载与校验

```ts
const stored = await this.backend.loadStored(id)
if (stored === undefined) throw new Error(`session "${id}" not found`)

this.assertStoredId(id, meta)
this.assertVersion(meta)
const storedEvents = adoptStoredEvents(events, id)
this.assertEventsSupported(meta, storedEvents)

// ★ 保留完整的"被中断事件"，只补缺失的收尾
const closers = interruptedTurnClosers(storedEvents).map(adoptSessionEvent)
const balanced = [...storedEvents, ...closers]

const session = this.ctx.sessions.prepare(id, { seed: balanced, meta, seedSource: 'persistence' })
```

注意 `balanced` 这个词：**加载后的日志必须"平衡"**（turn/step 成对闭合），才能喂给模型。

### ④ 崩溃尾部修复（最有技术含量的一段）

**扫描规则**（一次线性遍历）：

| 事件 | 动作 |
|---|---|
| `turn/start` | 更新开放 turn，**清空 pending（防止上一轮的调用泄漏进来）** |
| `step/start` | 记录开放 step |
| `step/end` | 清空 pending |
| `assistant/message` | 把其中的 `tool-call` 块登记为 pending |
| `tool/call` | 给 pending 补上 `callSeq`（说明"确实开始执行了"） |
| `tool/result` | 从 pending 移除 |

**只在日志结尾有未闭合 turn 时才动手**，且合成顺序被严格固定：

```
先补 tool/result（每个未完成的调用）
  → 再补 step/end
    → 最后补 turn/end { reason: { kind: 'interrupted' } }
```

为什么是这个顺序？注释写得很清楚：**provider 不接受悬空的 assistant 调用**，而"step 未关时 turn/end 属于不变式违反"。

**两种错误码区分严重程度**：

| 情况 | 错误码 | 给模型的说明 |
|---|---|---|
| 有 assistant 的 tool-call，**没有** `tool/call` | `TOOL_NOT_STARTED` | "记录它开始之前就中断了，需要就重试" |
| 有 `tool/call` 但**没有**结果 | `TOOL_OUTCOME_UNKNOWN` | "调用已记录但结果未持久化，**结果未知**。只读或幂等才重试；可能有副作用就先验证外部状态或问用户。**别盲目重试**" |

最后一句其实是**把崩溃恢复写成了 agent 能理解的上文**，而不是无声地塞个空结果。

**确定性保证**（可复现、可测试）：

```ts
let seq = last.seq + 1      // seq 从最后事件续号
const time = last.time      // 时间戳复用最后事件,绝不发明"未来时间"
```

### ⑤ 播种 + 所有权

```ts
export class SessionPreparation implements Disposable {
  readonly session: Session                        // 唯一一份"未发布的会话"
  [Symbol.dispose](): void {                       // 同步、幂等
    if (this.released) return
    this.released = true
    this.options.release?.()
  }
}
```

一个 **RAII 式的所有权凭据**：持有"尚未发布"的会话，发布成功就消耗掉，失败就归还或丢弃。

### ⑥ 续跑锚点

```ts
const baseline = this.session.requestHeader()
if (!this.requestHeaderLogged) {
  this.session.append('request/header', {
    header,
    reason: baseline === undefined ? 'initial' : 'resume',
  })
} else if (baseline === undefined || !headerEquals(baseline, header)) {
  this.session.append('request/header', { header, reason: 'change' })
}
```

**resume 的判定标准就是：种子里是否已有 request header。** 于是三种语义被清晰切开：

- `initial` —— 全新会话
- `resume` —— 从历史日志续跑（配置未变）
- `change` —— 续跑但配置变了（换模型/换工具集）

---

## 3. 并发与所有权模型

模块注释一句话概括：**"未发布会话的有界共享与独占预留"**。

| 操作 | 语义 |
|---|---|
| `observe` / `peek` | 冷读**共享**：同一 id 的并发读合流成一次 |
| `reserve` | **独占**预留；先提交挂起的持久化修复，再交付副本 |
| `claim` | 取出预留用于**发布**（拒绝别名） |
| `consume` | 会话已挂载后消费掉预留 |
| `discard` | 调用方只要 inspection，不要会话 |
| `release` | 归还到 ready **LRU** |
| `invalidate` | **持久化写入提交后**作废条目 |

这解决了"同一 session id 被并发打开"的经典问题：**要么共享只读视图，要么独占写入权，不存在中间态**。

---

## 4. 失败语义

```ts
} catch (error) {
  // 格式不支持 → 这是对"完好日志"的拒绝,不是损坏
  if (error instanceof SessionFormatUnsupportedError) throw error
  // 其余 → 判定为损坏,带上 cause
  throw new SessionPersistenceCorruptionError(
    `stored session "${id}" failed validation: ${String(error)}`, { cause: error })
}
```

调用方因此能区分两种完全不同的处置：**"这个客户端版本读不了"** vs **"文件真坏了"**。

---

## 5. 三个入口的差别

| 入口 | 做了什么 | 与 resume 的差异 |
|---|---|---|
| **resume** | 用**完整存储日志**作种子，保留原 header（含 `seedLength` 血缘） | 继续**同一个 session id** |
| **fork** | 用日志前缀作种子，新 id，记 `parentSession` / `seedLength` | 血缘边界是"fork 点" |
| **replay** | 用相同事件重建状态 | 只读或再执行，不续写 |

三者共用同一条构造路径——**一套机制覆盖三种需求**。

还有个细节很讲究：`session/end-seed` 标记被写在**构造期**（后端捕获 creation seed 时它已在日志里，因此**无需加载期写入**），而且**幂等**——注释说得很直白："冷会话在首次触碰时就被 resume，所以反复打开不能让它每次增长"。

---

## 6. 设计取舍总结

| 决策 | 收益 |
|---|---|
| **修复靠追加，不改写** | 可审计、可复现、崩溃不丢数据 |
| **日志是唯一事实源** | resume/fork/replay 复用一条路径 |
| **持久化血缘字段**（`seedLength` / `parentSession` / `delegationDepth` / 能力组合） | resume 后语义不漂移：子代理深度不会重置成顶层、工具门控不会变 |
| **`SessionPreparation` + Disposable** | 未发布会话有确定的所有权与释放时机 |
| **显式错误分类** | 调用方能区分"不支持"与"损坏" |
| **修复文本面向模型** | 把崩溃恢复变成可推理的上文，而非静默补空 |

**注释里明确承认的边界情况**：如果有持续的外部写入者，`prepare` 的收敛循环会因为日志一直"不稳定"而延后完成。

---

## 7. 一图流

```
resume(sessionId)
   │
   ├─ agent-loop: resumeSessionId 决议 ──► identity.resume
   │
   ├─ coordinator.prepare(id)
   │     ├─ waitForRetirement        (等上一个持有者)
   │     ├─ live? → 抛错             (禁止复活已挂载会话)
   │     └─ preparations.reserve
   │           └─ prepareCore(id)
   │                 ├─ backend.loadStored        取日志
   │                 ├─ 校验 id / 版本 / 格式
   │                 ├─ interruptedTurnClosers    修复尾部(只追加)
   │                 └─ sessions.prepare({ seed: 完整日志 + closers })
   │
   ├─ commitPrepared                 把修复落盘
   │
   └─ SessionPreparation(Disposable) ──► 发布 / 归还
                                        │
                                        └─ agent-loop 追加
                                           request/header{ reason: 'resume' }
```

---

## 一句话带走

如果只能记一条：**dsh 把"崩溃恢复"当成了"往日志尾巴上补几笔、并告诉模型刚才发生了什么"**——而不是试图把内存状态复原回去。

这个思路对任何"长跑型 agent"都值得借鉴：**不可变历史 + 追加式修复 + 显式锚点**。
