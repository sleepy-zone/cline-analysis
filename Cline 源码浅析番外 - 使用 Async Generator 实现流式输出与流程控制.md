# Cline 源码浅析番外 - 使用 Async Generator 实现流式输出与流程控制

随着自动 AI Chat 的兴起，**流式输出**已成为默认的交互方式。尽管如今 AI Agent 逐渐取代传统聊天模式成为主流，但流式输出凭借其“打字机”般的实时反馈效果，依然是提升用户体验的核心环节。

而像 Cline 这类 Code Agent，在实际运行中不仅需要处理流式响应，还频繁面临**工具调用、用户确认、流程暂停、失败重试**等复杂控制场景。如何优雅地实现这些功能？答案正是：Async Generator —— 这或许是 JavaScript 中最被低估的高级特性之一。

![Cline 工具调用用户确认界面](https://raw.githubusercontent.com/sleepy-zone/mp-pic-bed/main/2025/08/25/1756112338745-2d0ba5f0-157a-4027-a232-4650e5c32ca9.png)

## 什么是 Generator

在深入了解 Async Generator 之前，先回顾一下它的“前身”——Generator。

Generator 是一种特殊的函数，可以在执行过程中**暂停和恢复**，通过 yield 关键字逐个返回值。

```js
function* simpleGenerator() {
  yield 1;
  yield 2;
  yield 3;
}
const gen = simpleGenerator();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }
```

Generator 返回一个**迭代器（Iterator）**，每次调用 `.next()` 返回都会返回一个形如 `{ value, done }` 的对象：

* value：当前产出的值。
* done：布尔值，表示迭代是否完成。false 表示还有值可产出，true 表示结束。

由于返回的是迭代器，因此可以直接使用 for...of 循环遍历，无需手动调用 .next()。实际上，for...of 的底层机制就是自动调用对象的 Symbol.iterator 方法获取迭代器，并持续调用 .next()，直到 done: true。

> Symbol.iterator 是一个特殊的 Symbol，JavaScript 引擎用它来识别对象是否“可被迭代”。

例如，数组之所以能被 for...of 遍历，正是因为内部实现了 Symbol.iterator：

```js
const arr = [1, 2, 3];
const iterator = arr[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
```
只要在自定义对象上实现 Symbol.iterator，就能享受 for...of 带来的便利。

## Async Generator：异步世界的流式引擎

Async Generator（异步生成器）是 ES2018 引入的强大特性，它融合了 Generator 的“暂停/恢复”能力与 async/await 的异步处理优势。你可以在一个函数中既 yield 值，又 await 异步操作。

一个简单例子：

```js
async function* asyncGenerator() {
  yield 1;
  yield 2;
  // 可以 await 异步操作
  const data = await fetch('/api/data');
  yield data.json();
}
```

```js
const asyncGen = asyncGenerator();

asyncGen.next().then(console.log); // { value: 1, done: false }
asyncGen.next().then(console.log); // { value: 2, done: false }
asyncGen.next().then(result => console.log(result.value)); // 解析后的 JSON 数据
```

调用该函数后返回一个 **Async Iterator（异步迭代器）**，其 .next() 方法返回的是一个 Promise，解析结果为 { value, done }：

异步迭代器使用 `for await...of` 进行遍历：

```js
async function consumeAsyncGenerator() {
  for await (const value of asyncGenerator()) {
    console.log(value); // 依次输出 1, 2, 解析后的数据
  }
}
```

这种“按需产出 + 异步等待”的机制，使得 Async Generator 成为处理**流式数据和复杂流程控制**的理想选择。接下来，我们结合 AI 场景，实战演示它的强大之处。

## 代码实战

### 实现流式输出

我们可以直接使用 fetch 读取服务端的流式响应：

```ts
export async function* chatWithLLM(prompt: string) {
  const stream = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({
      model: 'qwen3-coder-plus',
      messages: [
        {
          role: 'system',
          content: prompt
        }
      ]
    })
  });

  if (!stream.ok) {
    throw new Error(`HTTP ${stream.status}`);
  }

  const reader = stream.body?.getReader();
  const decoder = new TextDecoder();
  while (true) {
    // @ts-ignore
    const { done, value } = await reader?.read();
    if (done) {
      break;
    }
    const chunk = decoder.decode(value);
    yield { type: 'text', content: chunk };
  } 
}
```

当然，也可以借助 OpenAI SDK 等封装好的工具。以下是 Cline 的真实实现：（src/core/api/providers/openai.ts）：

```ts
import OpenAI, { AzureOpenAI } from "openai"

const stream = await client.chat.completions.create({
  model: modelId,
  messages: openAiMessages,
  temperature,
  max_tokens: maxTokens,
  reasoning_effort: reasoningEffort,
  stream: true,
  stream_options: { include_usage: true },
})
for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta
  if (delta?.content) {
    yield {
      type: "text",
      text: delta.content,
    }
  }

  if (delta && "reasoning_content" in delta && delta.reasoning_content) {
    yield {
      type: "reasoning",
      reasoning: (delta.reasoning_content as string | undefined) || "",
    }
  }

  if (chunk.usage) {
    yield {
      type: "usage",
      inputTokens: chunk.usage.prompt_tokens || 0,
      outputTokens: chunk.usage.completion_tokens || 0,
      // @ts-ignore-next-line
      cacheReadTokens: chunk.usage.prompt_tokens_details?.cached_tokens || 0,
      // @ts-ignore-next-line
      cacheWriteTokens: chunk.usage.prompt_cache_miss_tokens || 0,
    }
  }
}
```

通过 yield 逐步输出不同类型的消息（文本、推理过程、Token 使用量），前端即可实时渲染内容，实现流畅的“打字机”效果。

### 实现暂停和恢复

Async Generator 天然支持“暂停并恢复”的语义。我们只需控制执行流程的开关即可：

```ts
let iterator: AsyncGenerator<any>;
let isRunning = true;

async function start() {
  if (!iterator) iterator = chatWithLLM('使用 js 实现 debounce 函数');

  if (!isRunning) return;

  while (isRunning) {
    const { value, done } = await iterator.next();
    if (done) break;
    console.log(value)
  }
}

function pause() { isRunning = false; }
function resume() { isRunning = true; start(); } // 继续从上次 yield 开始
```

Generator 会自动保存执行上下文，无需手动维护状态。Cline 在处理用户确认时，也采用了类似思路。例如在 `src/core/task/index.ts` 中的 ask 方法：

```ts
await pWaitFor(() => this.taskState.askResponse !== undefined || this.taskState.lastMessageTs !== askTs, {
  interval: 100,
})
```

这段代码会持续轮询，直到用户完成确认或消息时间戳更新，期间整个生成器暂停，等待响应到来后再恢复执行。

(p-wait-for：https://github.com/sindresorhus/p-wait-for)

### 错误重试

网络请求可能失败，我们需要在出错时自动重试。借助 Async Generator，可以非常优雅地实现递归重试逻辑：

```js
async function *attempt () {
  const iterator = chatWithLLM('使用 js 实现 debounce 函数');

  try {
    console.log('attempt')
    const firstChunk = await iterator.next();
    yield firstChunk.value
  } catch(error) {
    console.log('error', error);
    yield* attempt();
    return;
  }

  yield* iterator;
}
```

```js
let isRunning = true;

async function start() {
  const stream = attempt();
  const iterator = stream[Symbol.asyncIterator]();
  while (isRunning) {
    const { value, done } = await iterator.next();
    if (done) break;
    console.log(value)
  }
}

function pause() { isRunning = false; }
function resume() { isRunning = true; start(); } // 继续从上次 yield 开始
```

这里关键使用了` yield*` 语法：

`yield* iterator` 的作用是“**把 iterator 中的所有 value 依次 yield 出去**”，直到 done: true。等价于：

```js
for await (const value of iterator) {
  yield value;
}
```
但 `yield*` 更简洁、性能更好，且能正确处理 return 值和异常传播。

这里我们其实是参考了 Cline  的实现，下面是简化后的代码：

```ts
async *attemptApiRequest(previousApiReqIndex: number): ApiStream {
  const stream = this.api.createMessage(systemPrompt, contextManagementMetadata.truncatedConversationHistory)
  const iterator = stream[Symbol.asyncIterator]();
  
  try {
      // awaiting first chunk to see if it will throw an error
      this.taskState.isWaitingForFirstChunk = true
      const firstChunk = await iterator.next()
      yield firstChunk.value
      this.taskState.isWaitingForFirstChunk = false
	} catch (error) {
      yield* this.attemptApiRequest(previousApiReqIndex)
      return;
    }
    yield* iterator
}
```

通过递归调用 attemptApiRequest 并使用 yield* 代理输出，实现了无感重试，同时保持了流式输出的连续性。

## 总结

Async Generator 是处理异步流式数据与复杂流程控制的利器。它结合了 Generator 的“暂停/恢复”能力和 async/await 的异步表达力，使得我们能够以同步的写法处理异步流，极大提升了代码的可读性与可维护性。

在 Cline 这类 AI Agent 中，Async Generator 被广泛用于：

* **流式输出**：实时返回模型生成的文本片段；
* **流程暂停**：等待用户确认后再继续执行；
* **错误重试**：请求失败时自动重试，不影响整体流程。

通过 yield、for await...of、yield* 等语法，我们能以极简的方式构建出健壮、灵活的异步控制流。可以说，Async Generator 是构建现代 AI 应用底层逻辑的基石之一。

如果你正在开发一个需要处理流式响应或复杂状态流转的系统，不妨试试 Async Generator —— 它可能正是你一直在寻找的那个“优雅解”。




