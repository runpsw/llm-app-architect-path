# 今日学习笔记：LLM API 协议的核心心智模型

> 2026-06-11 · 从 SDK 协议到 token 流真相到注入防御

今天真正的收获不是一堆 API 字段，而是五个环环相扣的心智模型。它们按抽象层次从上往下排列，每一层都解释了上一层"为什么这样设计"。

---

## 模型一：API 是无状态的，"对话"是客户端拼出来的幻觉

服务端不记录任何上下文。每次请求都要提交**全量历史**，模型只做一件事：给定这个序列，生成下一条 assistant 消息。

由此推出的结论：
- 对话状态、记忆、上下文管理，全部是**你的**（应用层的）责任；
- 历史可以伪造——assistant 消息是不是模型真说过的，无人验证（few-shot 的原理）；
- 上下文越长越贵，因为每次都是全量从头计算。

## 模型二：role 结构是前端抽象，模型只看见一条 token 流

messages 数组在进模型前被对话模板拍平成线性序列：

```
<|system|>...<|user|>...<|assistant|>...<|user|>...<|assistant|>←从这里续写
```

- **顺序**不需要"被告知"——数组顺序就是物理顺序，messages 是有序列表不是集合；
- **回答哪条**不是挑选——模型永远只是续写序列结尾那个空的 assistant 开口；
- `<|user|>` 这类是词表预留的**特殊 token**，普通文本无法伪造出它们的 token ID——这是 role 隔离在底层的真正保障；
- "向量化"的准确说法：逐 token 查嵌入表，N 个 token 进去是 N 个向量（区别于 RAG 用的 embedding model：整段压一个向量）。

## 模型三：stop_reason 是状态机，agent 的本质是一个 while 循环

`stop_reason`（OpenAI 方言叫 `finish_reason`）是协议中最重要的字段，每个值对应一个必须处理的分支：`end_turn` 正常结束 / `max_tokens` 截断（输出不完整！）/ `tool_use` 要调工具。

Tool use 循环的三条铁律：
1. 第二次请求 = **全量历史** + 原样回传的 assistant 消息（含 tool_use block，一字不改）+ 装着 tool_result 的新 user 消息；
2. 第二次推理是一次与第一次毫无关系的全新推理——模型"在等工具结果"只是应用层叙事；
3. 所有 agent 框架的核心就是 `while stop_reason == "tool_use"`，循环体只做一件事：往 messages 上 append。

## 模型四：业界有两大协议方言，深层同构

| | Anthropic Messages | OpenAI Chat Completions（DeepSeek 原生） |
|---|---|---|
| 鉴权 | `x-api-key` | `Authorization: Bearer`（HTTP 标准方案） |
| system | 顶层字段 | messages 里的一条消息 |
| 回答 | `content[]` block 列表 | `choices[0].message.content` 字符串 |
| 停止原因 | `stop_reason` | `finish_reason` |
| 工具调用 | `tool_use` block，input 是**对象** | `tool_calls[]`，arguments 是 **JSON 字符串**（要二次解析） |
| 工具结果 | user 消息 + tool_result block | `role: "tool"` 消息 |

表层差异不少，深层结构（无状态、全量历史、停止原因状态机、工具闭环）完全同构。多模型网关/路由层做的就是这两种方言间的转换。自测题：口述一个 Anthropic→OpenAI 转换层的字段映射，说出哪里会丢信息。

## 模型五：Prompt 注入没有根治方案，防御重心是限制损害而非阻止上当

- 区分 **Jailbreak**（用户骗模型）与 **Prompt Injection**（第三方把指令藏进模型会读的数据里）——做应用，主要敌人是后者，尤其是**间接注入**（藏在网页/邮件/PDF/RAG 文档里）；
- 根因正是模型二：拍平成 token 流后，"待处理的数据"和"要执行的指令"没有本质区别，token 的来源信任标签在架构上丢失了；
- 因此防御是纵深而非银弹，最有效的在**架构侧**：最小权限（被注入也做不了坏事）、高危操作人在回路、Dual LLM 隔离（碰脏数据的没权限，有权限的不碰脏数据）、工具调用前用确定性代码校验参数；
- 一句话：防的不是"模型被骗"，是"被骗之后能造成的实际损害"。

---

## 工程实操教训

1. **API key 一旦出现在聊天/截图/仓库里即视为泄露，立刻吊销重生成**（今天发生过一次，已处理）。
2. PowerShell ≠ bash：`export` → `$env:VAR = "..."`；PowerShell 的 `curl` 是 Invoke-WebRequest 的别名，要用 `curl.exe` 或干脆换 Git Bash 跑 bash 模板。
3. 模型名有时效：deepseek-chat / deepseek-reasoner 将于 2026-07-24 弃用，用 `deepseek-v4-flash`；网上旧教程的参数（如 temperature、prefill）在新模型上可能已失效，以官方 reference 为准。
4. DeepSeek 提供 Anthropic 兼容端点（`api.deepseek.com/anthropic`），学 Messages 协议不必付 Claude 的价格。

---

## 下一步（按优先级）

1. 把 curl 实验跑完（路线 A 的截断 + tool use 两步是重点）；
2. 跑路线 B，亲手对比两种方言；
3. 回到 Python SDK，对照看 `client.messages.create()` 替你省掉了哪些手工动作——此时 SDK 应该"一眼透明"；
4. 之后的主战场在协议之上：上下文管理、agent 编排、评估、成本与降级——那才是"AI 应用架构"的正题。
