# pi 内核层知识总纲（pi-agent-core）

> 本文是 pi 内核层（`@earendil-works/pi-agent-core`）全部知识点的总结，配合源码 `packages/agent/src/agent-loop.ts` 和 `agent.ts` 阅读。

---

## 一、Agent 循环（核心中的核心）
[[pi-coding-agent产品层知识总纲]]
**两层循环结构**：

- 外层循环：处理 follow-up（模型干完后的追加消息）
- 内层循环：处理「模型调工具」的反复（一次回复 + 执行工具 + 再问）

**一轮 turn** = 一次模型回复 + 可能的工具执行。

**完整流程**：

```
turn_start
  ↓ ① transformContext（剪裁/注入）
  ↓ ② convertToLlm（翻译/过滤）
  ↓ ③ 拼 llmContext（systemPrompt + messages + tools）
  ↓ ④ 发给模型，流式收回复，emit message_update
  ↓ ⑤ 检查 toolCall
  ↓ ⑥ 有 → 执行工具，结果回填 context.messages
  ↓ ⑦ 还有工具 → 回到①；没有 → 循环结束
turn_end
```

---

## 二、上下文打包（llmContext）

- 全内核唯一「凑齐三件套」的地方
- 三件套：systemPrompt + messages + tools
- 打包前经历：transformContext（改内容）→ convertToLlm（改格式）
- 打包后交给 streamFunction 序列化发给模型

---

## 三、事件流（10 个核心事件）

**生命周期**：`agent_start` / `agent_end`

**轮次**：`turn_start` / `turn_end`

**消息**：`message_start` / `message_update`（流式增量）/ `message_end`

**工具**：`tool_execution_start` / `tool_execution_update` / `tool_execution_end`

**本质**：广播机制，内核靠它解耦——内核只管 emit，订阅者（UI/落盘/扩展）各取所需。

---

## 四、工具执行流水线

```
① 找工具（找不到报错）
② 参数预处理（prepareArguments，可选）
③ 参数校验（validateToolArguments）
④ beforeToolCall（拦截/阻止，可选）
⑤ 真正执行 tool.execute
⑥ afterToolCall（改写结果，可选）
⑦ 结果包装成 toolResult，回填上下文
```

**设计思想**：不信任模型、不信任工具，每步设防御。

**并行 vs 串行**：

- 并行（默认）：无依赖工具同时执行
- 串行：逐个执行；只要批里有一个标了 sequential，整批串行

---

## 五、转换层

- **transformContext**：改内容（剪裁/注入），还在 pi 内部格式
- **convertToLlm**：改格式（AgentMessage → Message），默认是过滤非标准角色

两者都在「发模型前」执行，顺序：先 transformContext，再 convertToLlm。

---

## 六、消息队列

- **steering**（转向）：模型跑着时插话，内层循环处理
- **follow-up**（追加）：模型跑完后补话，外层循环处理
- **pending**：统一工作缓冲区，三个填充点（启动前、每轮后、内层结束后）
- **模式**：one-at-a-time（默认，逐条）/ all（全倒）

---

## 七、AgentState（9 个字段）

```
systemPrompt      身份（开发者可改）
model             当前模型（用户/开发者可改）
thinkingLevel     思考档位（用户/开发者可改）
tools             工具列表（开发者可改）
messages          消息历史（内核自动维护，开发者可重置）
isStreaming       是否流式中（内核维护，只读）
streamingMessage  流式半成品（内核维护，只读）
pendingToolCalls  执行中的工具（内核维护，只读）
errorMessage      最近错误（内核维护，只读）
```

分三类：开发者可改（配置）、内核自动改（运行时）、用户间接影响（发消息/切模型）。

---

## 八、terminate 机制

- 工具结果带 `terminate: true` → 执行完停止循环，不再问模型
- 条件：这批**所有**工具都带 terminate 才生效
- 可选字段，默认不带；大多数工具不该带（需要模型组织人话回复）

---

## 九、stopReason（模型结束原因）

| stopReason | 含义 | 循环行为 |
|---|---|---|
| `stop` | 正常结束 | 结束 |
| `toolUse` | 要调工具 | 执行工具继续 |
| `length` | 输出被截断 | 工具全判失败（防参数不完整） |
| `error` | 出错 | 立即结束 |
| `aborted` | 被取消 | 立即结束 |

---

## 十、三个进阶钩子

| 钩子 | 时机 | 作用 |
|---|---|---|
| `agentLoopContinue` | 出错后手动调 | 从当前状态继续（不重跑，让模型根据现状决定） |
| `shouldStopAfterTurn` | 一轮结束后 | 决定停不停（如压缩前刹车） |
| `prepareNextTurn` | 下一轮开始前 | 调整下一轮配置（压缩/切模型/注入） |

---

## 核心思想回顾

1. **存储和运行分离**：磁盘态（树）和运行态（内存数组）是两个世界，投影是桥，落盘是回流。
2. **内核靠事件解耦**：内核只管广播，订阅者各取所需。
3. **不信任模型**：工具流水线每步设防御（找不到、参数错、拦截、异常）。
4. **每轮全量重发上下文**：因为 LLM 无状态，每次都得重新递身份 + 历史 + 工具。
5. **模型是翻译器**：工具结果是机器格式，通常让模型再加工成人话回复用户。

---

## 学习状态

内核层 10 个主题全部学完，无遗漏：

Agent 循环、上下文打包、事件流、工具流水线、转换层、消息队列、AgentState、terminate、stopReason、进阶钩子。

下一步：产品层（`pi-coding-agent`），从 AgentSession 运行流程开始。

---

# 附录：完整实例——一次带插话、出错、重试的借书过程

## 前置设定

```
系统提示词（systemPrompt）："你是图书管理助手..."
工具（tools）：search_books、search_readers、borrow_book、get_stats、recommend_books 等 8 个
初始消息（context.messages）：[]（全新会话，空）
pending（工作缓冲区）：[]
steering 队列：[]
follow-up 队列：[]
```

场景：用户要借《三体》，中途插话、出错重试、借完又追加推荐，一次完整交互覆盖全部知识点。

消息简写约定（下文每阶段都会完整列出 context.messages）：

- U1 = user:"帮我借《三体》给张三"
- A1 = assistant:"好的，我先查一下这本书"
- T1 = toolResult(search_books):"id=1 《三体》可借 4 本"
- U2 = user:"顺便看看库存"（steering 插话）
- A2 = assistant:"好的，我同时查库存和读者"
- T2 = toolResult(search_readers):"id=1 张三"
- T3 = toolResult(get_stats):"网络错误"（isError:true）
- T4 = toolResult(get_stats):"库存 26 册"（重试成功）
- A3 = assistant:"已确认，要借书吗？"
- T5 = toolResult(borrow_book):"借阅成功，应还日期 2026-10-01"
- A4 = assistant:"借阅成功！张三借出《三体》，应还日期 2026-10-01"
- U3 = user:"再推荐几本科幻书"（follow-up 追加）
- A5 = assistant:"好的，我推荐几本科幻书"
- T6 = toolResult(recommend_books):"《三体》等科幻书"
- A6 = assistant:"推荐《三体》等科幻书"

---

## 阶段 1：启动 + 用户发第一条消息

用户输入 U1，触发 `agent.prompt()`。

**发生了什么**：

```
【知识点：Agent 循环启动】
runLoop 启动

【知识点：消息队列 - 填充点1】
检查 steering 队列 → 空
pending = []

【知识点：事件流】
emit agent_start（run 开始）
emit turn_start（第 1 轮开始）
emit message_start（U1 开始）
emit message_end（U1 结束）
```

**事件广播后订阅者做什么**：

- UI：收到 agent_start、turn_start，可以显示"正在思考中…"
- 落盘系统：收到 message_start/end，把 U1 落盘到 .jsonl

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三"
]
```

---

## 阶段 2：第 1 轮 - 组装上下文问模型

**发生了什么**：

```
【知识点：转换层】
① transformContext：没设，跳过，消息不变
② convertToLlm：过滤，只剩 user/assistant/toolResult，现在只有 U1 一条

【知识点：上下文打包 llmContext】
llmContext = {
  systemPrompt: "你是图书管理助手...",
  messages: [U1],                    ← convertToLlm 后的结果
  tools: [search_books, search_readers, borrow_book, get_stats, recommend_books, ...8个],
}

【知识点：发给模型 + 事件流】
streamFunction 把 llmContext 序列化，发给模型
模型流式吐字："好的，我先查一下这本书"
  每吐一个字 → emit message_update
```

**事件广播后订阅者做什么**：

- UI：收到 message_update，把每个字追加到界面（流式显示"好的，我先查一下这本书"）
- 日志扩展（如果有）：记录"正在流式输出"
- 落盘系统：不响应（message_update 是流式碎片，不是定稿，不落盘）

**模型吐完后**：

```
emit message_end（A1 定稿）
```

**事件广播后订阅者做什么**：

- 落盘系统：收到 message_end，把 A1 落盘到 .jsonl
- UI：标记"这条消息显示完了"

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书"
]
```

---

## 阶段 3：第 1 轮 - 执行 search_books 工具

**发生了什么**：

```
【知识点：检查 stopReason 和 toolCall】
A1 的 stopReason = "toolUse"（模型要调工具）
A1 的内容里有 toolCall：search_books(q="三体")

【知识点：工具执行流水线】
① 找工具：在 tools 里找到 search_books ✅
② 参数预处理：没设 prepareArguments，跳过
③ 参数校验：q="三体" 是合法字符串 ✅
④ beforeToolCall：没设，跳过
⑤ 真正执行：execute → 查 SQLite → 返回 "id=1 《三体》可借 4 本"
⑥ afterToolCall：没设，跳过
⑦ 结果包装成 toolResult 消息（T1）

【知识点：事件流】
emit tool_execution_start（携带 toolName="search_books", args={q:"三体"}）
emit tool_execution_end（携带 result, isError:false）
emit message_start（T1 开始）
emit message_end（T1 结束）
```

**事件广播后订阅者做什么**：

- UI：收到 tool_execution_start，显示工具卡片"🔧 search_books 执行中…"；收到 tool_execution_end，标记"✓ 完成"
- 落盘系统：收到 message_end，把 T1 落盘到 .jsonl

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本"
]
```

**然后检查 terminate**：

```
【知识点：terminate 检查】
T1 结果没带 terminate:true → 不终止，继续循环
```

---

## 阶段 4：第 1 轮结束 + steering 插话

**发生了什么**：

```
【知识点：消息队列 - 填充点2】
emit turn_end（第 1 轮结束）
之后检查 steering 队列

══════════ 关键：用户此时插话 ══════════

用户在模型执行工具的间隙，输入 U2："顺便看看库存"（steering 转向消息）

【知识点：steering 入队】
U2 进入 steering 队列，不立刻处理（模型正忙）

【知识点：填充点2 捞取】
turn_end 后检查 steering 队列 → 发现 U2
→ 把 U2 捞进 pending
```

**此时各缓冲区状态**：

```
pending = [U2:"顺便看看库存"]
steering 队列 = []（已捞空）
follow-up 队列 = []
```

**此时 context.messages 完整内容**（注意：U2 还没进 messages，还在 pending 里排队）：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本"
]
```

---

## 阶段 5：第 2 轮 - 注入 steering，问模型

**发生了什么**：

```
【知识点：内层循环继续】
还有工具调用（上一轮模型要调工具）→ 内层循环继续

【知识点：注入 pending】
emit turn_start（第 2 轮开始）
发现 pending 非空 → 注入：
  context.messages.push(U2)
  pending = []（清空）
  emit message_start(U2) / message_end(U2)
```

**事件广播后订阅者做什么**：

- 落盘系统：收到 message_end，把 U2 落盘到 .jsonl
- UI：显示 U2 这条消息

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存"
]
```

**然后组装上下文问模型**：

```
【知识点：上下文打包】
llmContext = {
  systemPrompt: "你是图书管理助手...",
  messages: [U1, A1, T1, U2],       ← 现在 4 条
  tools: [8个工具],
}

模型看到"顺便看看库存"，回复 A2："好的，我同时查库存和读者"
（要求调 get_stats 和 search_readers 两个工具）

emit message_end（A2 定稿）
```

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者"
]
```

---

## 阶段 6：第 2 轮 - 并行执行两个工具 + 出错

**发生了什么**：

```
【知识点：并行 vs 串行】
A2 要调 2 个工具：get_stats + search_readers
两者无依赖，且都默认并行 → 并行执行

get_stats ────────┐
                  ├─ 同时执行
search_readers ───┘

假设 get_stats 执行时网络出错：

【知识点：工具异常处理】
get_stats 的 execute 抛异常 → 被 try/catch 捕获 → 转成错误结果 T3（isError:true）
search_readers 正常执行 → T2："id=1 张三"

两个工具结果都回填（并行模式：全部完成后一起回填）：
  emit tool_execution_end(get_stats, isError:true)
  emit tool_execution_end(search_readers, isError:false)
  emit message_start/end(T3)
  emit message_start/end(T2)
```

**事件广播后订阅者做什么**：

- UI：显示两个工具卡片，get_stats 标"✗ 失败"、search_readers 标"✓ 完成"
- 落盘系统：把 T3、T2 都落盘

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者",
  T2: toolResult(search_readers):"id=1 张三",
  T3: toolResult(get_stats):"网络错误"（isError:true）
]
```

**此时 AgentState 的 errorMessage**：

```
【知识点：AgentState - errorMessage】
agent.state.errorMessage = "网络错误"（内核自动写入）
```

---

## 阶段 7：出错重试 - agentLoopContinue

**发生了什么**：

```
【知识点：进阶钩子 - agentLoopContinue】
开发者代码检测到 errorMessage 非空（网络错误）
→ 决定重试，调用 agent.continue()（而不是重新 prompt）

【知识点：continue 的约束检查】
检查 context.messages 最后一条：是 T3（toolResult），不是 assistant ✅
→ 满足约束，可以 continue

【知识点：continue 的本质】
不重新执行失败的 get_stats
不重新发用户 prompt
而是直接把当前 context（含"网络错误"这个事实）发给模型
让模型自己看到错误结果，自己决定下一步
```

**continue 后，新的一轮**：

```
emit turn_start
组装 llmContext（messages 是上面 8 条，含 T3 错误）
发给模型

模型看到 T3"网络错误"，决定重试 get_stats
→ 重新调 get_stats → 这次成功 → T4："库存 26 册"
→ 结果回填
```

**事件广播后订阅者做什么**：

- UI：又显示一个 get_stats 工具卡片，这次"✓ 完成"
- 落盘系统：把 T4 落盘

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者",
  T2: toolResult(search_readers):"id=1 张三",
  T3: toolResult(get_stats):"网络错误"（isError:true）,
  T4: toolResult(get_stats):"库存 26 册"（重试成功）
]
```

---

## 阶段 8：第 3 轮 - 借书（beforeToolCall 人机确认）

**发生了什么**：

```
模型现在信息齐了：book_id=1（来自 T1）、reader_id=1（来自 T2）
再问模型 → 模型回复 A3："已确认，要借书吗？"（要调 borrow_book，参数 book_id=1, reader_id=1）

【知识点：工具流水线 - beforeToolCall 钩子】
borrow_book 执行前：
① 找工具 → 找到 borrow_book
② 参数预处理 → 跳过
③ 参数校验 → book_id=1, reader_id=1 都是合法数字 ✅
④ beforeToolCall 钩子触发：
   → 弹确认框（人机确认）"确认借书吗？"
   → 用户点"确认" → 返回不阻止 → 放行
⑤ 真正执行：execute → 借书 → T5："借阅成功，应还日期 2026-10-01"
⑥ afterToolCall → 跳过
⑦ 结果包装成 toolResult（T5）
```

**事件广播后订阅者做什么**：

- UI：显示确认框（beforeToolCall 触发的），然后显示 borrow_book 工具卡片"✓ 完成"
- 落盘系统：把 A3、T5 落盘

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者",
  T2: toolResult(search_readers):"id=1 张三",
  T3: toolResult(get_stats):"网络错误"（isError:true）,
  T4: toolResult(get_stats):"库存 26 册"（重试成功）,
  A3: assistant:"已确认，要借书吗？",
  T5: toolResult(borrow_book):"借阅成功，应还日期 2026-10-01"
]
```

**terminate 检查**：

```
【知识点：terminate 不用】
T5 不带 terminate:true → 不终止，继续循环（需要模型组织人话回复用户）
```

---

## 阶段 9：第 4 轮 - 最终回复，内层循环结束

**发生了什么**：

```
【知识点：再问模型】
第 4 轮，组装 llmContext（messages 现在 11 条）
发给模型
模型回复 A4："借阅成功！张三借出《三体》，应还日期 2026-10-01"

【知识点：检查 stopReason】
A4 的 stopReason = "stop"（正常结束，不再要工具）
A4 内容里没有 toolCall

【知识点：内层循环结束】
hasMoreToolCalls = false
→ 内层循环退出
```

**事件广播后订阅者做什么**：

- UI：流式显示 A4
- 落盘系统：把 A4 落盘

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者",
  T2: toolResult(search_readers):"id=1 张三",
  T3: toolResult(get_stats):"网络错误"（isError:true）,
  T4: toolResult(get_stats):"库存 26 册"（重试成功）,
  A3: assistant:"已确认，要借书吗？",
  T5: toolResult(borrow_book):"借阅成功，应还日期 2026-10-01",
  A4: assistant:"借阅成功！张三借出《三体》，应还日期 2026-10-01"
]
```

---

## 阶段 10：外层循环 - 处理 follow-up 追加

**发生了什么**：

```
【知识点：外层循环 + follow-up】
内层循环退出后，外层循环检查 follow-up 队列

══════════ 关键：用户此时追加消息 ══════════

用户输入 U3："再推荐几本科幻书"（follow-up 追加消息）

【知识点：follow-up 进入 pending】
外层循环发现 follow-up 队列有 U3
→ 把 U3 塞进 pending
→ continue 回外层循环开头
→ 内层循环又启动
```

**此时各缓冲区状态**：

```
pending = [U3:"再推荐几本科幻书"]
follow-up 队列 = []（已捞空）
```

**此时 context.messages 完整内容**（U3 还在 pending，没进 messages）：

```
context.messages = [
  U1 ~ A4（上面的 11 条，不变）
]
```

---

## 阶段 11：处理 follow-up - 最后一轮

**发生了什么**：

```
【知识点：注入 pending 处理 follow-up】
emit turn_start（新一轮）
注入 pending：context.messages.push(U3)，pending 清空
emit message_start/end(U3)

组装 llmContext（messages 现在 12 条）
发给模型
模型回复 A5："好的，我推荐几本科幻书"（要调 recommend_books）
执行 recommend_books → T6："《三体》等科幻书"
结果回填
再问模型 → 模型回复 A6："推荐《三体》等科幻书"（无 toolCall，stopReason="stop"）
```

**事件广播后订阅者做什么**：

- UI：流式显示 A5、A6，显示 recommend_books 工具卡片
- 落盘系统：把 U3、A5、T6、A6 逐条落盘

**此时 context.messages 完整内容**：

```
context.messages = [
  U1: user:"帮我借《三体》给张三",
  A1: assistant:"好的，我先查一下这本书",
  T1: toolResult(search_books):"id=1 《三体》可借 4 本",
  U2: user:"顺便看看库存",
  A2: assistant:"好的，我同时查库存和读者",
  T2: toolResult(search_readers):"id=1 张三",
  T3: toolResult(get_stats):"网络错误"（isError:true）,
  T4: toolResult(get_stats):"库存 26 册"（重试成功）,
  A3: assistant:"已确认，要借书吗？",
  T5: toolResult(borrow_book):"借阅成功，应还日期 2026-10-01",
  A4: assistant:"借阅成功！张三借出《三体》，应还日期 2026-10-01",
  U3: user:"再推荐几本科幻书",
  A5: assistant:"好的，我推荐几本科幻书",
  T6: toolResult(recommend_books):"《三体》等科幻书",
  A6: assistant:"推荐《三体》等科幻书"
]
```

---

## 阶段 12：循环真正结束

**发生了什么**：

```
【知识点：循环真正结束】
再次检查 follow-up 队列 → 空
→ break，整个 run 结束
emit agent_end（携带最终的 messages 列表）
```

**事件广播后订阅者做什么**：

- UI：收到 agent_end，把发送按钮从禁用恢复，允许用户继续输入
- 落盘系统：确认所有消息已落盘

---

## 阶段 13：落盘结果（贯穿全程，最终汇总）

```
【知识点：事件驱动落盘】
整个过程中，产品层订阅 message_start/message_end
每定稿一条消息 → 立刻落盘 .jsonl

最终 .jsonl 文件完整内容：
{"type":"session","id":"...","cwd":"D:/library"}
{"type":"message","id":"m1","parentId":null,"message":{"role":"user","content":"帮我借《三体》给张三"}}
{"type":"message","id":"m2","parentId":"m1","message":{"role":"assistant","content":"好的，我先查一下这本书"}}
{"type":"message","id":"m3","parentId":"m2","message":{"role":"toolResult","content":"id=1 《三体》可借 4 本"}}
{"type":"message","id":"m4","parentId":"m3","message":{"role":"user","content":"顺便看看库存"}}
{"type":"message","id":"m5","parentId":"m4","message":{"role":"assistant","content":"好的，我同时查库存和读者"}}
{"type":"message","id":"m6","parentId":"m5","message":{"role":"toolResult","content":"id=1 张三"}}
{"type":"message","id":"m7","parentId":"m6","message":{"role":"toolResult","content":"网络错误"}}
{"type":"message","id":"m8","parentId":"m7","message":{"role":"toolResult","content":"库存 26 册"}}
{"type":"message","id":"m9","parentId":"m8","message":{"role":"assistant","content":"已确认，要借书吗？"}}
{"type":"message","id":"m10","parentId":"m9","message":{"role":"toolResult","content":"借阅成功，应还日期 2026-10-01"}}
{"type":"message","id":"m11","parentId":"m10","message":{"role":"assistant","content":"借阅成功！张三借出《三体》"}}
{"type":"message","id":"m12","parentId":"m11","message":{"role":"user","content":"再推荐几本科幻书"}}
{"type":"message","id":"m13","parentId":"m12","message":{"role":"assistant","content":"好的，我推荐几本科幻书"}}
{"type":"message","id":"m14","parentId":"m13","message":{"role":"toolResult","content":"《三体》等科幻书"}}
{"type":"message","id":"m15","parentId":"m14","message":{"role":"assistant","content":"推荐《三体》等科幻书"}}
```

注意：磁盘上的消息是**平铺的树（靠 id/parentId 串成链）**，和内存里的 context.messages（扁平数组）是两种形态。整个过程中，内存态每轮追加，磁盘态同步落盘，两者一致。

---

## 知识点全覆盖清单

| 知识点 | 在例子里哪里体现 |
|---|---|
| Agent 循环（两层） | 内层循环跑多轮，外层循环处理 follow-up（阶段10-11） |
| 上下文打包 | 每轮组 llmContext（阶段2/5/7/8/9/11） |
| 事件流 | 全程 emit，每阶段标注广播后订阅者做什么 |
| 工具流水线 7 步 | search_books 完整走 7 步（阶段3） |
| 转换层 | transformContext 跳过 + convertToLlm 过滤（阶段2） |
| 消息队列 steering | 阶段4 用户插话"顺便看库存" |
| 消息队列 follow-up | 阶段10 用户追加"再推荐" |
| 消息队列 pending | 填充点1（阶段1）、填充点2（阶段4）、注入（阶段5/11） |
| AgentState | context.messages 每阶段完整列出；errorMessage（阶段6） |
| terminate | T1/T5 不带，继续循环（阶段3/8） |
| stopReason | toolUse→继续，stop→结束（阶段2/9/11） |
| 三个进阶钩子 | beforeToolCall（阶段8确认）、agentLoopContinue（阶段7重试） |
| 事件广播后做什么 | 每阶段标注 UI/落盘/日志各自响应 |
| 并行 vs 串行 | 阶段6 get_stats + search_readers 并行 |
| 工具异常处理 | 阶段6 get_stats 网络错误被捕获 |
| 落盘机制 | 阶段13 最终 .jsonl 完整内容 |

## 一句话总结

> 这个例子把内核层所有知识点串成一条完整故事：用户借书 → 模型多轮循环查书查读者 → 中途 steering 插话 → 工具并行执行遇网络错误 → agentLoopContinue 重试 → beforeToolCall 人机确认借书 → 模型最终回复 → follow-up 追加推荐 → 全程事件驱动落盘。每个阶段都完整列出了 context.messages 的当前内容，并标注了对应知识点和事件广播后各订阅者的行为。
