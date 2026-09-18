---
title: Pi Coding Agent 产品层知识总纲
type: note
topic: 研发/架构与设计
tags: [研发, 架构, pi-agent, 产品层]
created: 2026-09-11
updated: 2026-09-11
status: active
source: team-knowledge 改造前 01-公共区/00-个人区
---

# pi 产品层知识总纲（pi-coding-agent）

> 本文是 pi 产品层（`@earendil-works/pi-coding-agent`）全部知识点的总结，配合源码 `packages/coding-agent/src/core/` 阅读。
> 产品层 = AgentSession（协调者）+ SessionManager（磁盘账本）+ ResourceLoader（资源管家）+ Extension（插件）+ SDK（装配入口）。

---

## 一、三层架构总览

```
pi-coding-agent（产品层）   ← 本文主题
  AgentSession / SessionManager / ResourceLoader / Extension / SDK
        ↓ 调用
pi-agent-core（内核层）     ← 基座
  Agent / runLoop / state.messages / 工具执行 / 事件
        ↓ 调用
pi-ai（模型层）             ← 和模型 API 打交道
```

**核心分工**：

- **内核层管计算**：发模型、跑工具、维护 state.messages（内存账本）。
- **产品层管协调**：持久化、会话、扩展、资源加载、装配。

---

## 二、AgentSession（协调者，核心）

AgentSession 是产品层的总协调者，持有 ~10 个组件（Agent、SessionManager、ResourceLoader、SettingsManager、ModelRuntime、ExtensionRunner、队列、压缩、重试、监听者）。

**主线 = 内核 + 存储 + 资源**。

### 2.1 prompt() 的完整结构（两条线）

```
prompt(text)
  ├─ 线A：输入预处理（8 步骤）—— 处理"文本"
  │     ① 开关判断（expandPromptTemplates，默认 true，一个开关管扩展命令+技能+模板）
  │     ② 扩展命令检查（/xxx → 执行 handler → return，不进模型）
  │     ③ 压缩进行中 → 抛异常
  │     ④ input 拦截（handled 吞掉 / transform 改写 / 放行）
  │     ⑤ 技能+模板展开（/skill:x 展开 md 正文，/文件名 展开模板）
  │     ⑥ 流式检查（必须指定 steer/followUp，否则抛异常）
  │     ⑦ 校验模型+key
  │     ⑧ 压缩检查
  │
  ├─ 组装 user 消息（预处理好的文本 → {role:"user",...}）
  │
  ├─ 线B：发模型前装配
  │     before_agent_start（改系统提示词 + 塞 custom 消息）
  │
  └─ _runAgentPrompt(messages) → 启动内核 runLoop
```

### 2.2 _runAgentPrompt（产品层总调度）

```
_isAgentRunActive = true
  try:
    agent.prompt(messages)               ← 内核跑一轮（runLoop 全程）
    while(_handlePostAgentRun()):        ← 善后循环（runLoop 返回后才跑）
      agent.continue()                    ← 再跑一轮
  finally:
    清临时系统提示词 + flush bash/custom + 发 agent_settled
```

**善后循环三问**（_handlePostAgentRun）：

```
① 出错重试？→ 是 → continue()
② 满了压缩？→ 是 → continue()
③ 有排队？  → 是 → continue()（agent_end 期间扩展又塞了消息）
三问全否 → false → 循环结束
```

### 2.3 执行主线调用链（5 层准备 + 1 层干活）

```
_runAgentPrompt（产品层老板）
  → agent.prompt（检查门锁 activeRun + 归一化输入）
  → runPromptMessages（算历史快照 + 配置两个参数）
  → runWithLifecycle（开门设状态 + 执行 + 关门清状态）
  → runAgentLoop（合并历史+新输入 + 发开场事件）
  → runLoop（真正发模型⇄跑工具的两重循环）
  → finishRun（关门）
  → 回到 _runAgentPrompt 善后循环 → finally
```

### 2.4 _handleAgentEvent（中央接线员，旁路监听者）

**不在调用链上**，是 403 行 `agent.subscribe(_handleAgentEvent)` 登记的事件监听者。每次内核 `emit` → processEvents → 遍历监听者 → 触发它。

五件事：

```
① message_start 且 user → 清理队列显示（发 queue_update）
② await _emitExtensionEvent(event)    ← 转发给扩展（await）
③ _emit(event)                        ← 通知 UI 监听者（同步广播）
④ message_end → 写磁盘 jsonl + 记录 lastAssistantMessage + 重置重试计数
⑤ turn_end → flush 积压的 custom 消息
```

### 2.5 三道闸门（发模型前）

| 闸门 | 位置 | 时机 | 干什么 |
|---|---|---|---|
| before_agent_start | prompt() 里 | 每次输入 1 次 | 改系统提示词 + 塞 custom |
| transformContext | streamAssistantResponse | 每次发模型（可能多次） | 增删改 pi 内部消息（接扩展 context 事件） |
| convertToLlm | streamAssistantResponse | 每次发模型 | 翻译角色（custom→user 等） |

**transformContext = 改内容（AgentMessage→AgentMessage）；convertToLlm = 改语言（AgentMessage→Message）。**

---

## 三、扩展系统（Extension）

### 3.1 扩展能注册的三类东西（三种触发方式）

| 触发方式 | 注册方法 | 触发源 | 触发动作 |
|---|---|---|---|
| 命令 | registerCommand | 人（用户） | 敲 /xxx |
| 事件 | on(...) | 内核（runLoop） | 内核发事件 |
| 工具 | registerTool | 模型（AI） | 模型回复带 toolCall |

### 3.2 扩展生命周期 5 阶段

```
① 发现：discoverExtensionsInDir（项目级 .pi/extensions/ → 全局 ~/.pi/agent/extensions/ → 显式配置）
② 加载：读模块 → createExtension(空壳) → createExtensionAPI(造 pi) → factory(pi)(执行工厂收集注册) → commit/discard
③ 绑定：runner.bindCore（接真实动作：sendMessage/getModel/isIdle...）
④ 运行：事件转发（emit/emitInput/emitContext）、命令执行、工具调用
⑤ 销毁：runner.invalidate()（标记过期）+ clearExtensionCache() + 重新加载
```

### 3.3 input 拦截的 handled / transform

```
on("input", handler) 可返回：
  { action: "handled" }      ← 吞掉输入，直接 return（连 message 都不组装）
  { action: "transform", text }  ← 改写文本，继续走
  不返回                     ← 放行
```

- **handled 多扩展链式**：按加载顺序遍历，handled 一出现就短路（整个 emitInput 函数 return）。
- **transform 多扩展链式**：A 改完，B 在 A 基础上改，接力叠加。

### 3.4 sendMessage / sendCustomMessage

```
sendCustomMessage(message, { triggerTurn })
  ├─ triggerTurn: true  → _runAgentPrompt（塞了再跑）
  ├─ triggerTurn: false → 只塞 state.messages（只塞不跑）
  ├─ 流式 + triggerTurn → 走 steer/followUp 队列
  └─ deliverAs: "nextTurn" → 存 _pendingNextTurnMessages
```

custom 消息在 convertToLlm 里翻译成 user 进模型。

### 3.5 inline 扩展 vs 文件扩展

| | 文件扩展 | inline 扩展 |
|---|---|---|
| 代码位置 | .pi/extensions/*.ts | SDK 代码里 |
| 告诉 pi 的方式 | 放目录自动扫描 | extensionFactories 参数 |
| 路径 | 真实文件路径 | <inline:名字> |

---

## 四、SessionManager（磁盘账本）

### 4.1 JSONL 格式（一行一个 JSON）

```
第一行：header {type:"session", id, cwd, parentSession}
其余行：entry（9 种类型）
```

### 4.2 9 种 Entry 类型

| type | 含义 | 进 LLM 吗 |
|---|---|---|
| message | 普通消息 | ✅ |
| thinking_level_change | 思考级别变更 | ❌ |
| model_change | 模型变更 | ❌ |
| compaction | 压缩摘要 | ✅ |
| branch_summary | 分支摘要 | ✅ |
| custom | 扩展私有数据 | ❌ |
| custom_message | 扩展注入消息 | ✅ |
| label | 书签 | ❌ |
| session_info | 会话名 | ❌ |

每个 entry 有 id / parentId / timestamp，形成树。leafId = 当前叶子指针。

### 4.3 写盘时机（_persist）

```
第一条 assistant 出现之前 → 只攒内存，不写盘
第一条 assistant 出现时   → 一次性写盘（openSync "wx"）
之后                     → 每条 appendFileSync 追加
```

### 4.4 恢复四步（嵌套调用）

```
buildSessionContext（总装）
  ├─ buildSessionPath（找路径：leafId 回溯到根）
  ├─ buildContextEntries（压缩裁剪）
  └─ sessionEntryToContextMessages（投影成消息）
```

### 4.5 分支 + 跨进程恢复

- branch(branchFromId)：移动 leafId 指针到历史 entry，继承之前全部上下文，抛弃之后的旧路径。
- create/open/continueRecent：扫描目录找最新 .jsonl → 读回 entry → 投影成消息 → 塞回内核 state.messages。

---

## 五、ResourceLoader（资源管家）

### 5.1 统一加载 6 类资源

| 资源 | 默认位置 |
|---|---|
| extensions | ~/.pi/agent/extensions/ + .pi/extensions/ |
| skills | ~/.pi/agent/skills/ + .pi/skills/ |
| prompts | ~/.pi/agent/prompts/ + .pi/prompts/ |
| themes | ~/.pi/agent/themes/ + .pi/themes/ |
| agentsFiles | 项目根 + 祖先目录（AGENTS.md） |
| systemPrompt | SYSTEM.md |

### 5.2 reload() 流程（总调度）

```
清缓存 → 处理项目信任 → 重载 settings → packageManager.resolve()（发现路径）
→ 过滤 enabled → 加载扩展 → 加载 skills → 加载 prompts → 加载 themes
→ 加载 context files → 加载 system prompt
```

**ResourceLoader 是调度者，具体读文件委托给各模块的 load 函数（loadSkills/loadExtensions 等）。**

### 5.3 项目级 vs 全局级

每个资源都有"项目级（cwd/.pi/）"和"全局级（~/.pi/agent/）"两个来源，同时加载，同名冲突靠 dedupe 去重（先加载的赢）。

---

## 六、SDK（装配入口）

### 6.1 createAgentSession 8 阶段

```
① 确定 cwd / agentDir
② 创建 ModelRuntime（模型运行时）
③ 创建 SettingsManager + SessionManager
④ 创建 ResourceLoader 并 reload（加载所有资源）
⑤ 恢复/确定 model + thinkingLevel（options → 历史 → 默认）
⑥ 创建内核 Agent（传 convertToLlm、transformContext、streamFn）
⑦ 恢复历史（agent.state.messages = 磁盘恢复的消息）
⑧ 创建 AgentSession（把全部组件塞进去）
```

### 6.2 SDK 和"基座"的关系

```
你的智能体（业务逻辑）      ← 你加书阁工具、UI
  ↑ 复用
SDK（createAgentSession）   ← 帮你装配好，不改源码
  ↑ 底层就是
内核（pi-agent-core）       ← 你的真正基座
```

**开发智能体 = SDK + 业务逻辑**，简单场景不改 SDK，用扩展点定制，只有改底层机制才 fork 源码。

---

## 七、三层账本（内存 vs 磁盘）

| 账本 | 持有者 | 作用 |
|---|---|---|
| state.messages | 内核 Agent | 供 runLoop 发模型 |
| SessionManager.fileEntries | 产品层 | 供恢复、树导航、渲染 |
| .jsonl 文件 | 磁盘 | 持久化，重启后恢复 |

**消息落账双写**：内核 processEvents 写内存 state.messages（无需监听），产品层 _handleAgentEvent 写磁盘 jsonl（靠 subscribe 监听）。

---

## 八、核心思想回顾

1. **三层分工**：内核管计算、产品层管协调、模型层管通信。
2. **存储和运行分离**：磁盘态（树）和运行态（内存数组）是两个世界，投影是桥，落盘是回流。
3. **事件驱动解耦**：内核只管 emit，产品层 subscribe 后各取所需（持久化、UI、扩展）。
4. **可替换插槽**：内核的 convertToLlm/transformContext 是可替换的，产品层 new Agent 时塞入自己的版本。
5. **扩展点是定制入口**：customTools/registerCommand/on/registerTool 等，让人在不改源码的前提下定制行为。
6. **装配式设计**：SDK 的 createAgentSession 把零散组件装配成完整机器。

---

## 学习状态


AgentSession 运行流程、执行主线调用链、三道闸门、扩展系统（三种触发 + 生命周期）、SessionManager（JSONL + 树 + 恢复）、ResourceLoader（资源加载）、SDK（装配入口）。

---

# 附录：完整实例——一次借书的完整执行链

## 前置设定

- 用户敲「帮我借三体」，模型先调查书工具，再回复"已借出"。
- 全程标注每层干了什么、每个事件触发什么。

## 阶段 1：用户输入 → prompt() 预处理

```
【产品层】用户敲"帮我借三体" → session.prompt(text)
8 步骤：无命令/无拦截/无技能模板 → 文本通过
组装 user 消息：messages = [{role:"user", content:"帮我借三体"}]
before_agent_start：无扩展 → 用基础系统提示词
```

## 阶段 2：_runAgentPrompt 启动

```
【产品层】_isAgentRunActive = true
  → agent.prompt(messages)
```

## 阶段 3：内核入口 + runAgentLoop

```
【内核层】agent.prompt → 检查 activeRun → normalizePromptInput（数组原样返回）
  → runPromptMessages → runWithLifecycle（开门：isStreaming=true）
  → runAgentLoop：
      emit agent_start
      emit turn_start（第1个）
      emit message_start(user) / message_end(user)
        → processEvents：state.messages.push(user)      ← 写内存
        → _handleAgentEvent：appendMessage(user)         ← 写磁盘
```

## 阶段 4：runLoop 两重循环

```
【内核层】runLoop：
  第1次 streamAssistantResponse 发模型
    → transformContext（无扩展）→ convertToLlm（user 通过）
    → 模型回复"先查库存" + toolCall(查书)
    → emit message_end(assistant) → 写内存 + 写磁盘

  执行工具"查书"
    → tool_execution_start → 执行 → tool_execution_end
    → emit message_end(toolResult) → 写内存 + 写磁盘
    → 工具结果塞回 currentContext.messages

  emit turn_end（第1个）
  emit turn_start（第2个）

  第2次 streamAssistantResponse 发模型
    → 模型看到 用户输入+工具调用+工具结果
    → 回复"已借出《三体》"
    → emit message_end(assistant) → 写内存 + 写磁盘

  emit turn_end（第2个）
  emit agent_end
```

## 阶段 5：善后循环 + finally

```
【产品层】回到 _runAgentPrompt
  while(_handlePostAgentRun())：三问全否 → false → 循环结束
  finally：
    清临时系统提示词 + flush + emit agent_settled
  prompt() 返回，生命周期结束
```

## 完整事件序列

```
agent_start → turn_start(1) → message_start(user) → message_end(user)
→ [发模型1] message_start(assistant) → message_update×N → message_end(assistant)
→ tool_execution_start → tool_execution_end
→ message_start(toolResult) → message_end(toolResult)
→ turn_end(1) → turn_start(2)
→ [发模型2] message_start(assistant) → message_update×N → message_end(assistant)
→ turn_end(2) → agent_end → agent_settled
```

## state.messages 全程变化

```
初始：          []
用户输入后：    [user]
第1次发模型后： [user, assistant(toolCall)]
工具执行后：    [user, assistant(toolCall), toolResult]
第2次发模型后： [user, assistant(toolCall), toolResult, assistant(最终)]
```

每一步都是 message_end 事件触发：内核写内存 + 产品层写磁盘。

## 一句话总结

> 这个例子把产品层所有知识点串成一条完整链路：用户输入 → prompt 预处理 → 组装消息 → before_agent_start → _runAgentPrompt 启动内核 → runLoop 两重循环（发模型⇄跑工具）→ 每个 message_end 事件触发内核写内存 + 产品层写磁盘 → 善后循环 → finally 发 agent_settled → 生命周期结束。
