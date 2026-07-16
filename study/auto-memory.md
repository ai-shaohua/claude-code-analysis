# Auto Memory 是什么

> 配套阅读：[第四章：Agent Memory 机制](../analysis/04-agent-memory.md)

## 一句话结论

**Auto Memory 是「自动」从会话里提取、并「自动」在合适时机回灌的那一层持久记忆。名字里的 "Auto" 强调的是「不需要用户 / agent 手动记，系统自动帮你沉淀」。**

它不是一个抽象概念，而是对应代码里真实的 `autoMemory` 机制——常量 `isAutoMemoryEnabled()` / `autoMemoryEnabled` / `autoMemoryDirectory` 都来自这里。

- 相关实现：`[src/memdir/paths.ts](../src/memdir/paths.ts)`
- 相关实现：`[src/services/extractMemories/extractMemories.ts](../src/services/extractMemories/extractMemories.ts)`

## "Auto" 体现在两个动作上

### 1. 自动写入（提取）

`[src/services/extractMemories/extractMemories.ts:1](../src/services/extractMemories/extractMemories.ts)` 的头注释说得很清楚：

```text
Extracts durable memories from the current session transcript
and writes them to the auto-memory directory (~/.claude/projects/<path>/memory/).

It runs once at the end of each complete query loop (when the model produces
a final response with no tool calls) via handleStopHooks in stopHooks.ts.

Uses the forked agent pattern (runForkedAgent) — a perfect fork of the main
conversation that shares the parent's prompt cache.
```

拆开看：

- 每次一个完整 query loop 结束时（模型给出最终回复、且没有 tool 调用），通过 `handleStopHooks` **自动触发一次**
- 用 forked agent（复刻主会话、共享 prompt cache）去读当前 transcript
- 把值得长期保留的信息（durable memories）自动落盘到 `~/.claude/projects/<path>/memory/`

也就是说，用户不需要显式说「记住这个」，系统会在回合结束时自己判断要不要沉淀。

当然也保留了手动入口，见 `[src/memdir/paths.ts:40](../src/memdir/paths.ts)`：

```text
extractMemories turn-end fork、autoDream、/remember、/dream、team sync
```

### 2. 自动召回（relevant recall）

就是第四章第 5 节讲的 `findRelevantMemories()`：每一轮**自动**从 memory 目录里挑最多 5 个相关文件回灌当前上下文，而不是把所有历史记忆全塞进去。

## 关键点：Auto Memory 是整个记忆体系的总开关

这一点最容易被忽略，但它直接决定了四层记忆的边界关系。

`isAutoMemoryEnabled()`（`[src/memdir/paths.ts:30](../src/memdir/paths.ts)`）不只是控制 Auto Memory 自己，它是**全局闸门**。

例如 Agent Memory 的注入逻辑（第四章 7.7 节）：

```text
if (isAutoMemoryEnabled() && parsed.memory) {
  return systemPrompt + '\n\n' + loadAgentMemoryPrompt(agentType, parsed.memory)
}
```

**Agent Memory 是否注入，要先过 Auto Memory 这个总开关。** 所以四层记忆不是完全平级的：

- `Auto Memory` = 自动提取 + 自动召回的**通用**持久记忆，同时它的 enable 逻辑是**全局闸门**
- `Agent Memory` / `Session Memory` / `Team Memory` = 各自独立目录和策略，但都受这个闸门约束

## 四层记忆边界对比

| 记忆层 | 谁触发写入 | 作用域 | 是否受 Auto 总开关约束 |
| --- | --- | --- | --- |
| Auto Memory | 系统自动（turn-end fork）+ 手动 `/remember` | 用户 / 项目通用 | 自身即总开关 |
| Session Memory | 系统自动（后台 forked subagent） | 当前会话 | 是 |
| Agent Memory | agent 自己显式读写 markdown | 某个 agent 类型（user / project / local） | 是 |
| Team Memory | 团队同步（pull / push + watcher） | 团队 repo 级共享 | 是 |

## 回到最初的问题

第四章第 12 行提出的问题：

> Agent Memory 和 Auto / Session / Team Memory 的边界是什么

边界的核心区别就是：

**Auto Memory 是「系统自动、面向用户 / 项目通用」的那层，而 Agent Memory 是「绑定到某个 agent 类型、由 agent 自己手动读写」的那层。**

一个是「系统帮你记的通用记忆」，一个是「某个角色自己维护的专属记忆」——加上 Auto 还兼任全局闸门，这就是它们最本质的分工。
