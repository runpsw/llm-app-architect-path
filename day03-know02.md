# LLM 应用开发学习笔记

日期:2026-06-11
场景:调用 DeepSeek 的 Anthropic 兼容端点(`/anthropic/v1/messages`),开启 thinking 模式,实验流式输出。

---

## 1. 流式输出与 SSE 格式

开启流式:请求体加 `"stream": true`(curl 调试时加 `-N` 关闭缓冲,才能看到逐字输出的效果)。

流式响应不是一个完整 JSON,而是 **SSE(Server-Sent Events)事件流**:

```
event: content_block_delta
data: {"type":"content_block_delta","delta":{"text":"我"}}
```

因此**不能把流式响应当完整 JSON 解析**(如直接 `| jq`),会报解析错误。实际遇到的报错:

```
parse error: Invalid numeric literal at line 1, column 6
```

(jq 把第一行 `event: ...` 当 JSON 解析失败。)

正确做法是逐事件处理:取 `data:` 行 → 逐行解析 JSON → 按事件类型分发。命令行调试时的提取管道:

```bash
curl -N -s https://api.deepseek.com/anthropic/v1/messages \
    -H "x-api-key: $DEEPSEEK_API_KEY" \
    -H "content-type: application/json" \
    -d '{
        "model": "<模型名>",
        "max_tokens": 1024,
        "thinking": {"type":"enabled","budget_tokens":16000},
        "stream": true,
        "messages": [{"role":"user","content":"用一句话介绍你自己"}]
    }' \
  | grep '^data:' \
  | sed 's/^data: //' \
  | jq -j 'select(.type=="content_block_delta") | .delta.text // empty'
```

注意:thinking 内容的事件类型/字段名可能与正文不同。如果提取不到,先看原始流确认实际字段,再调整过滤条件。

## 2. thinking 模式与 signature_delta

Anthropic 消息格式中开启思考:`"thinking": {"type":"enabled","budget_tokens":16000}`。

开启后,流中会出现 thinking 块的签名事件:

```
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"signature_delta","signature":"dcdd..."}}
```

- **signature 的作用**:thinking 块的完整性校验/防篡改凭据,证明这段思考由模型生成、未被改动。
- **单轮请求**:可以忽略,不影响回答内容。
- **多轮对话且回传 thinking 历史**:必须把 signature 原样保留并随 thinking 块一起传回,不能丢、不能改,否则可能被服务端拒绝。
- 待确认:这是 Anthropic 格式的设计;DeepSeek 兼容端点的实际校验行为是否完全一致,需查 DeepSeek 官方文档。

## 3. 多服务商差异与架构层沉淀

各家服务商的"兼容 OpenAI/Anthropic 格式"是**有损兼容**:基础对话场景差异不大,但在多轮、工具调用、推理模式(thinking/reasoning)等复杂场景,差异会集中爆发。架构层的职责是用 provider 抽象层(adapter)抹平这些差异——前提是先知道差异在哪。

沉淀知识库的原则:**记结构性、不易变的;易变的交给程序动态获取。**

值得沉淀的四类:

1. **能力差异矩阵**:谁有原生工具调用 / 并行工具调用 / thinking 模式 / JSON mode 或结构化输出 / 视觉、PDF 输入。
2. **格式与协议差异**:SSE 事件类型命名、推理内容字段名(`thinking` vs `reasoning_content` vs `reasoning`)、多轮回传必须原样带回的字段(如 signature)、错误响应结构。
3. **踩坑记录**:文档不写的实际行为,如 `max_tokens` 是否包含思考 token、某参数被某家静默忽略、流式结尾事件缺失等。
4. **抽象层设计依据**:以上差异的最终用途——支撑 adapter 设计,让上层业务不感知具体是哪家。

不值得静态沉淀的:模型名、价格、上下文长度、限流——变化太快,应通过配置中心或各家 models 接口动态查询,而非写死在文档里。
