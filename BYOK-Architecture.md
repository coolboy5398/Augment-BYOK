# Augment BYOK 架构文档

## 概述

Augment BYOK (Bring Your Own Key) 是一个本地代理系统，允许用户使用自己的 API Key 连接到任意 OpenAI 兼容的后端服务，而不是必须使用 Augment 官方服务。

## 核心组件

### 1. byok-bootstrap.js - 引导模块

**职责**：
- 猴子补丁 `ConfigListener._getRawSettings()` 注入配置覆盖
- 启动本地代理服务器
- 注册 BYOK 配置面板命令
- 管理 API Key 的安全存储

**关键全局变量**：
```javascript
globalThis.__AUGMENT_BYOK_OVERLAY    // 配置覆盖对象
globalThis.__AUGMENT_BYOK_UPSTREAM   // 上游配置
globalThis.__AUGMENT_BYOK_SERVER     // 服务器实例
```

### 2. byok-server.js - 代理服务器

**职责**：
- 启动 HTTP 服务器 (127.0.0.1:随机端口)
- 拦截 Augment 扩展的所有 API 请求
- 将 Augment 私有协议转换为 OpenAI 兼容格式
- 转发请求到用户配置的后端

## 请求流程

```
Augment 扩展
    ↓
本地代理 (127.0.0.1:port)
    ↓
协议转换 (Augment → OpenAI)
    ↓
用户配置的 API 端点
```

## 端点映射

| Augment 端点                 | 处理方式  | 说明                  |
| ---------------------------- | --------- | --------------------- |
| `/chat-stream`               | 转换+转发 | 流式聊天，核心功能    |
| `/chat`                      | 转换+转发 | 非流式聊天            |
| `/completion`                | 转换+转发 | 代码补全              |
| `/edit`                      | 转换+转发 | 代码编辑              |
| `/prompt-enhancer`           | 转换+转发 | 提示词优化            |
| `/get-models`                | 混合处理  | 合并本地+官方模型列表 |
| `/agents/codebase-retrieval` | 本地处理  | 使用 ripgrep 本地搜索 |
| `/checkpoint-blobs`          | 透传/Mock | 依赖官方 token        |
| `/batch-upload`              | 透传/Mock | 依赖官方 token        |
| `/find-missing`              | 透传/Mock | 依赖官方 token        |

## 协议转换

### Augment → OpenAI Messages

```javascript
// Augment 格式
{
  message: "用户消息",
  chat_history: [...],
  nodes: [...],           // 包含工具调用、图片等
  tool_definitions: [...]
}

// 转换为 OpenAI 格式
{
  messages: [
    { role: "system", content: "..." },
    { role: "user", content: "..." },
    { role: "assistant", content: "...", tool_calls: [...] },
    { role: "tool", tool_call_id: "...", content: "..." }
  ],
  tools: [...]
}
```

### 工具调用修复

`repairToolCallArguments()` 函数自动修复 LLM 返回的不完整工具参数：
- 填充缺失的必需字段
- 特殊处理记忆相关工具 (memory/remember)

## Context Engine 处理

### 纯 BYOK 模式
- `/agents/codebase-retrieval` 使用本地 ripgrep 搜索
- 其他上下文端点返回空响应 (功能受限)

### 混合模式 (配置了 augmentToken)
- 上下文相关请求透传到 Augment 官方服务
- 保留完整的云端索引功能

## Feature Flags 注入

`handleGetModels()` 强制注入以下功能标志：
- `enable_agent_mode: true`
- `enable_chat_with_tools: true`
- `enable_memory_retrieval: true`
- `enable_chat_multimodal: true`
- 等等...

确保即使不连接官方服务，所有高级功能也能正常使用。

## 配置项

| 配置项           | 说明                 | 存储位置       |
| ---------------- | -------------------- | -------------- |
| baseUrl          | OpenAI 兼容 API 地址 | globalState    |
| model            | 模型名称             | globalState    |
| temperature      | 生成温度             | globalState    |
| maxTokens        | 最大 token 数        | globalState    |
| systemPromptBase | 基础系统提示词       | globalState    |
| apiKey           | API 密钥             | VSCode Secrets |
| augmentBaseUrl   | Augment 官方地址     | globalState    |
| augmentToken     | Augment 官方 token   | VSCode Secrets |

## 使用方法

1. 安装扩展
2. `Ctrl+Shift+P` → `Augment: BYOK 设置...`
3. 配置 API 地址和密钥
4. 正常使用 Augment 功能

## 支持的后端

任何 OpenAI 兼容的 API 端点：
- OpenAI API
- Claude API (通过兼容层)
- Azure OpenAI
- 本地 Ollama
- vLLM / text-generation-webui
- 其他兼容服务


## 工具调用修复机制详解

### 问题背景

LLM 在生成工具调用时，经常会出现以下问题：
1. 缺少必需参数
2. 参数类型不正确
3. 记忆相关工具的内容字段为空或过短

### 修复函数：`repairToolCallArguments()`

```javascript
function repairToolCallArguments({ toolName, rawArguments, toolInfo, contextText }) {
  // 1. 解析原始参数
  const argsObj = safeJsonParse(rawArguments, {});
  
  // 2. 获取工具 schema
  const schema = toolInfo?.schema;
  const properties = schema?.properties || {};
  const required = schema?.required || [];
  
  // 3. 填充缺失的必需字段
  for (const key of required) {
    if (argsObj[key] === undefined) {
      argsObj[key] = defaultValueForSchema(properties[key]);
    }
  }
  
  // 4. 特殊处理记忆相关工具
  if (isMemoryRelatedTool(toolName, toolInfo)) {
    // 自动填充 title/name/summary 字段
    // 自动填充 memory/content/text 字段
    // 自动填充 context/scenario 字段
  }
  
  return JSON.stringify(argsObj);
}
```

### 默认值生成规则

```javascript
function defaultValueForSchema(schema) {
  if (schema.default !== undefined) return schema.default;
  
  switch (schema.type) {
    case "string":  return "";
    case "number":  return 0;
    case "integer": return 0;
    case "boolean": return false;
    case "array":   return [];
    case "object":  return {};
    default:        return null;
  }
}
```

### 记忆工具识别

```javascript
function isMemoryRelatedTool(toolName, toolInfo) {
  const name = toolName.toLowerCase();
  const desc = toolInfo?.description?.toLowerCase() || "";
  
  return name.includes("memory") || 
         name.includes("remember") || 
         desc.includes("memory") || 
         desc.includes("remember");
}
```

### 记忆工具特殊处理

当检测到记忆相关工具时，会进行以下增强：

1. **Title 字段**：从上下文中提取前 80 个字符作为标题
2. **Content 字段**：自动生成包含上下文的完整记忆内容
3. **Context 字段**：填充使用场景说明

```javascript
// 示例：自动生成的记忆内容
{
  title: "用户偏好设置讨论",
  content: "要记住的内容：用户喜欢使用 TypeScript...\n\n使用场景：当后续对话再次涉及该信息时，用于保持一致性与减少重复确认。",
  context: "用户偏好设置讨论"
}
```

### 工具定义索引

为了快速查找工具的 schema，使用 Map 进行索引：

```javascript
function indexToolDefinitions(toolDefinitions) {
  const map = new Map();
  for (const t of toolDefinitions) {
    const name = extractToolName(t);
    if (!name) continue;
    map.set(name, {
      schema: extractInputSchemaJson(t),
      description: extractToolDescription(t)
    });
  }
  return map;
}
```

### 调用时机

工具调用修复在以下位置执行：

1. **流式聊天** (`handleChatStream`)：在收集完所有 tool_calls delta 后
2. **非流式聊天** (`handleChat`)：在解析响应后立即执行

```javascript
// 流式聊天中的调用
const toolIndex = indexToolDefinitions(payload?.tool_definitions);
const ctxText = payload?.message || getTextFromNodes(payload?.nodes);
const repairedToolCalls = toolCalls.map((c) => ({
  ...c,
  arguments: repairToolCallArguments({
    toolName: c.name,
    rawArguments: c.arguments,
    toolInfo: toolIndex.get(c.name),
    contextText: ctxText
  })
}));
```

### 效果

修复机制确保：
- 工具调用始终包含所有必需参数
- 记忆工具能够正确保存有意义的内容
- 减少因参数缺失导致的工具执行失败


## SSE 流式处理实现详解

### 概述

BYOK 代理需要处理两种流式协议的转换：
- **上游**：OpenAI SSE (Server-Sent Events) 格式
- **下游**：Augment NDJSON (Newline Delimited JSON) 格式

### NDJSON 响应初始化

```javascript
function startNdjson(res, statusCode = 200) {
  res.writeHead(statusCode, {
    "content-type": "application/x-ndjson; charset=utf-8",
    "cache-control": "no-cache",
    connection: "keep-alive",
    "x-content-type-options": "nosniff",
  });
  if (typeof res.flushHeaders === "function") res.flushHeaders();
}

function writeNdjson(res, item) {
  res.write(`${JSON.stringify(item)}\n`);
}
```

### SSE 事件解析器

```javascript
async function* iterateSseEvents(reader) {
  const decoder = new TextDecoder();
  let buffer = "";
  
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    
    // 累积数据到缓冲区
    buffer += decoder.decode(value, { stream: true });
    buffer = buffer.replace(/\r/g, "");  // 规范化换行符
    
    // 按双换行符分割事件
    while (buffer.includes("\n\n")) {
      const idx = buffer.indexOf("\n\n");
      const rawEvent = buffer.slice(0, idx);
      buffer = buffer.slice(idx + 2);
      
      // 解析事件数据
      const lines = rawEvent.split("\n");
      const dataLines = [];
      for (const line of lines) {
        if (!line || line.startsWith(":")) continue;  // 跳过注释
        if (line.startsWith("data:")) {
          dataLines.push(line.slice("data:".length).trimStart());
        }
      }
      
      const data = dataLines.join("\n");
      yield { data };
    }
  }
}
```

### 流式聊天处理流程

```javascript
async function handleChatStream(req, res, payload, upstream, logger) {
  // 1. 初始化 NDJSON 响应
  startNdjson(res, 200);
  
  // 2. 构建 OpenAI 请求
  const messages = toOpenAIMessagesFromAugment({ systemPrompt, payload });
  const tools = toOpenAITools(payload?.tool_definitions);
  
  // 3. 设置中断控制器
  const abortController = new AbortController();
  req.on("close", () => abortController.abort());
  req.on("aborted", () => abortController.abort());
  
  // 4. 发起上游请求
  const upstreamResp = await openAiChatCompletions({
    baseUrl: upstream.baseUrl,
    apiKey: upstream.apiKey,
    body: { model, temperature, stream: true, messages, tools },
    signal: abortController.signal,
  });
  
  // 5. 流式处理响应
  const reader = upstreamResp.body.getReader();
  let stopReason = 1;
  const toolCallsByIndex = new Map();
  
  for await (const evt of iterateSseEvents(reader)) {
    const data = evt.data;
    if (!data) continue;
    if (data === "[DONE]") break;
    
    const parsed = safeJsonParse(data, undefined);
    const choice = parsed?.choices?.[0];
    
    // 5a. 处理文本增量
    const deltaText = choice?.delta?.content;
    if (deltaText) writeNdjson(res, { text: deltaText });
    
    // 5b. 累积工具调用增量
    const deltaToolCalls = choice?.delta?.tool_calls || [];
    for (const tc of deltaToolCalls) {
      const idx = tc?.index ?? 0;
      const cur = toolCallsByIndex.get(idx) || { id: "", name: "", arguments: "" };
      if (tc?.id) cur.id = tc.id;
      if (tc?.function?.name) cur.name = tc.function.name;
      if (tc?.function?.arguments) cur.arguments += tc.function.arguments;
      toolCallsByIndex.set(idx, cur);
    }
    
    // 5c. 记录停止原因
    if (choice?.finish_reason) {
      stopReason = toChatStopReason(choice.finish_reason);
    }
  }
  
  // 6. 处理工具调用
  const toolCalls = Array.from(toolCallsByIndex.entries())
    .sort((a, b) => a[0] - b[0])
    .map(([, v]) => v)
    .filter((v) => v.name);
  
  // 7. 修复工具参数
  const repairedToolCalls = toolCalls.map((c) => ({
    ...c,
    arguments: repairToolCallArguments({ ... })
  }));
  
  // 8. 转换为 Augment 格式并发送
  const nodes = toAugmentToolUseNodes(repairedToolCalls);
  writeNdjson(res, nodes.length > 0 
    ? { text: "", nodes, stop_reason: 3 }  // 工具调用
    : { text: "", stop_reason: stopReason }
  );
  
  res.end();
}
```

### 停止原因映射

```javascript
function toChatStopReason(value) {
  switch (value) {
    case "stop":
    case "end_turn":
    case "stop_sequence":
      return 1;  // 正常结束
    case "length":
    case "max_tokens":
      return 2;  // 长度限制
    case "tool_calls":
    case "tool_use":
      return 3;  // 工具调用
    case "content_filter":
    case "safety":
      return 4;  // 内容过滤
    default:
      return 0;  // 未知
  }
}
```

### 工具调用增量累积

OpenAI 的流式工具调用是分块返回的：

```javascript
// 第一个 chunk
{ index: 0, id: "call_abc", function: { name: "read_file" } }

// 后续 chunks
{ index: 0, function: { arguments: '{"path":' } }
{ index: 0, function: { arguments: ' "src/main.ts"}' } }
```

代理需要按 index 累积这些增量：

```javascript
const toolCallsByIndex = new Map();

for (const tc of deltaToolCalls) {
  const idx = tc?.index ?? 0;
  const cur = toolCallsByIndex.get(idx) || { id: "", name: "", arguments: "" };
  
  // 累积各字段
  if (tc?.id) cur.id = tc.id;
  if (tc?.function?.name) cur.name = tc.function.name;
  if (tc?.function?.arguments) cur.arguments += tc.function.arguments;
  
  toolCallsByIndex.set(idx, cur);
}
```

### 转换为 Augment 工具调用节点

```javascript
function toAugmentToolUseNodes(toolCalls) {
  return toolCalls.map((c, idx) => {
    const tool_use_id = c.id || `tooluse_${Date.now()}_${idx}`;
    return {
      id: idx + 1,
      type: 5,  // 工具调用类型
      tool_use: {
        tool_use_id,
        tool_name: c.name,
        input_json: c.arguments || "{}"
      }
    };
  }).filter(Boolean);
}
```

### 错误处理

```javascript
function streamError(res, message) {
  writeNdjson(res, { text: `[augment-byok] ${message}\n` });
  writeNdjson(res, { text: "", stop_reason: 1 });
  res.end();
}
```

### 连接中断处理

```javascript
const abortController = new AbortController();
const onAbort = () => abortController.abort();

req.on("close", onAbort);
req.on("aborted", onAbort);

try {
  // ... 流式处理
} finally {
  req.off("close", onAbort);
  req.off("aborted", onAbort);
}
```

当客户端断开连接时，`AbortController` 会取消上游请求，避免资源泄漏。


## 本地 Ripgrep 代码检索实现

### 概述

当 Augment Agent 需要检索代码库时，会调用 `/agents/codebase-retrieval` 端点。在纯 BYOK 模式下，这个端点使用本地的 ripgrep (rg) 工具进行代码搜索，而不是依赖 Augment 的云端索引。

### 端点处理函数

```javascript
async function handleAgentsCodebaseRetrieval(res, payload, workspaceRoot, logger) {
  // 1. 提取查询文本
  const queryRaw = trimOrEmpty(payload?.information_request) || trimOrEmpty(payload?.query);
  if (!queryRaw) {
    return jsonOk(res, { 
      formatted_retrieval: "codebase-retrieval: missing information_request/query" 
    });
  }

  // 2. 从查询中提取搜索 token
  const tokens = Array.from(
    new Set(
      (queryRaw.match(/[A-Za-z_][A-Za-z0-9_]{2,}/g) || []).slice(0, 8)
    )
  );
  if (tokens.length === 0) {
    tokens.push(queryRaw.slice(0, 64));
  }

  // 3. 对每个 token 执行 ripgrep 搜索
  const perTokenMax = 40;
  const blocks = [];
  
  for (const token of tokens.slice(0, 5)) {
    const resp = await spawnCollect("rg", [
      "-n",                    // 显示行号
      "--no-heading",          // 不显示文件名标题
      "--color=never",         // 禁用颜色
      "-F",                    // 固定字符串匹配（非正则）
      "--max-count", String(perTokenMax),  // 每个文件最多匹配数
      "--",
      token,
      "."
    ], {
      cwd: workspaceRoot,
      timeoutMs: 120000        // 2分钟超时
    });

    if (!resp.ok) {
      if (resp.error?.code === "ENOENT") {
        blocks.push(`# ${token}\nrg 不可用：请安装 ripgrep（rg）`);
      } else {
        blocks.push(`# ${token}\n(rg failed)`);
      }
      continue;
    }

    const out = trimOrEmpty(resp.stdout);
    blocks.push(out ? `# ${token}\n${out}` : `# ${token}\n(no matches)`);
  }

  // 4. 组装结果
  const header = `codebase-retrieval (local rg)\nquery: ${queryRaw}\nworkspaceRoot: ${workspaceRoot}\n`;
  const result = `${header}\n${blocks.join("\n\n")}\n`;
  
  logger.debug("codebase-retrieval ok");
  return jsonOk(res, { formatted_retrieval: result });
}
```

### 子进程执行器

```javascript
async function spawnCollect(command, args, { cwd, timeoutMs }) {
  return await new Promise((resolve) => {
    const child = spawn(command, args, { 
      cwd, 
      stdio: ["ignore", "pipe", "pipe"]  // stdin 忽略，stdout/stderr 捕获
    });
    
    let stdout = "";
    let stderr = "";
    
    // 设置超时
    const timer = timeoutMs 
      ? setTimeout(() => child.kill("SIGKILL"), timeoutMs) 
      : null;
    
    // 收集输出
    child.stdout.on("data", (d) => (stdout += d.toString("utf8")));
    child.stderr.on("data", (d) => (stderr += d.toString("utf8")));
    
    // 错误处理
    child.on("error", (err) => {
      if (timer) clearTimeout(timer);
      resolve({ ok: false, error: err, stdout, stderr });
    });
    
    // 完成处理
    child.on("close", (code) => {
      if (timer) clearTimeout(timer);
      resolve({ ok: code === 0, code, stdout, stderr });
    });
  });
}
```

### Token 提取策略

从用户查询中提取搜索关键词：

```javascript
// 正则匹配：以字母或下划线开头，后跟至少2个字母/数字/下划线
const tokens = queryRaw.match(/[A-Za-z_][A-Za-z0-9_]{2,}/g) || [];

// 去重并限制数量
const uniqueTokens = Array.from(new Set(tokens)).slice(0, 8);

// 如果没有提取到 token，使用查询的前64个字符
if (uniqueTokens.length === 0) {
  uniqueTokens.push(queryRaw.slice(0, 64));
}
```

### Ripgrep 参数说明

| 参数            | 说明                               |
| --------------- | ---------------------------------- |
| `-n`            | 显示行号                           |
| `--no-heading`  | 不在每个文件前显示文件名标题       |
| `--color=never` | 禁用 ANSI 颜色代码                 |
| `-F`            | 固定字符串匹配，不解释为正则表达式 |
| `--max-count N` | 每个文件最多返回 N 个匹配          |
| `--`            | 参数结束标记，后面是搜索模式       |
| `.`             | 在当前目录递归搜索                 |

### 输出格式

```
codebase-retrieval (local rg)
query: 用户的原始查询
workspaceRoot: /path/to/workspace

# token1
src/main.ts:10:  const token1 = "value";
src/utils.ts:25:  function token1() {

# token2
src/config.ts:5:  token2: true,

# token3
(no matches)
```

### 限制与注意事项

1. **依赖 ripgrep**：需要系统安装 `rg` 命令
2. **无语义理解**：只是简单的文本匹配，不理解代码语义
3. **结果数量限制**：每个 token 最多 40 个匹配，最多 5 个 token
4. **超时保护**：2 分钟超时，防止大型代码库搜索卡住
5. **无索引**：每次搜索都是全量扫描，大型项目可能较慢

### 与云端 Context Engine 的对比

| 特性     | 本地 Ripgrep | Augment 云端 |
| -------- | ------------ | ------------ |
| 语义理解 | ❌ 纯文本匹配 | ✅ 语义搜索   |
| 索引     | ❌ 无索引     | ✅ 预建索引   |
| 速度     | 中等         | 快           |
| 离线可用 | ✅            | ❌            |
| 依赖     | ripgrep      | Augment 账号 |
| 代码理解 | ❌            | ✅ AST 分析   |

### 安装 Ripgrep

如果系统没有安装 ripgrep，会返回错误提示。安装方法：

```bash
# macOS
brew install ripgrep

# Ubuntu/Debian
sudo apt install ripgrep

# Windows (Chocolatey)
choco install ripgrep

# Windows (Scoop)
scoop install ripgrep

# Cargo
cargo install ripgrep
```
