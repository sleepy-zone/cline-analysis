# Cline 源码浅析 - 上下文窗口管理

![](https://mmbiz.qpic.cn/sz_mmbiz_png/c7AATQLC3mExrzU9NksicialQoACyt7ia7aPJt4rFJia9jHhy0kYPh8ibia7jCG59N3NWpiaxGeqiaWWO3sqdicGHpMLRnA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1)

## 上下文和上下文窗口

### 什么是上下文（Context）

> 大模型的上下文（Context of a Large Language Model）指的是在与模型交互时，输入给模型的一切内容，以及模型在生成输出时所依赖的相关信息。这些内容和信息构成了模型理解用户意图并生成合理回复的基础。

[Cline 中的上下文包括：](https://cline.bot/blog/clines-context-window-explained-maximize-performance-minimize-cost)

1. 任务指令，即用户输入。
2. 环境详情：关于你的工作区的信息，如可见文件、打开的标签和当前工作目录。
3. 工具定义：Cline 可以使用的工具的描述（如 `read_file` 、 `execute_command` 等）。
4. System Prompt：指导 Cline 的行为。
5. 对话历史：当前任务中和模型的交互。

简单来说，每一次的输入和输出信息构成了 Cline 的上下文。

### 什么是上下文窗口（Context Window）

> 上下文窗口是大语言模型能够处理的输入信息的范围或容量。它定义了模型在一次交互中可以考虑或“记住”的内容的最大长度（通常以**token**为单位），用来理解输入内容并生成输出。

关于 token 可以点击下面的链接体验：https://tiktokenizer.vercel.app/?model=gpt-4o

![gpt-4o 的分词方式](https://fastly.jsdelivr.net/gh/bucketio/img19@main/2025/06/24/1750772722605-2d1610d8-9cff-4413-86d9-57948f429091.png)

也就是说，上下文不能无限的增长，你应该听过使用 Cursor 或者 Cline 的建议：当模型输出结果不稳定时，新开一个会话可能就有效果，很有可能随着任务的处理，上下文暴涨，是的模型并不能很好的从上下文中获取有效信息，此时需要重新构建上下文。Cline 官方建议： **当使用率达到 70-80%时，考虑开始新的会话以保持最佳性能。**。

既然上下文窗口有限制，那么如何更好的管理上下文窗口就显得尤为重要。

## Cline 怎么做的

Cline 很贴心给用户提供了可视化的上下文窗口管理的功能，展示在会话记录的最上方。

![](https://fastly.jsdelivr.net/gh/bucketio/img2@main/2025/06/24/1750772965380-1e159175-7325-43a4-83e7-7aea4e93944d.png)

1. ↑表示输入令牌 input tokens（发送给 LLM 的内容）
2. ↓表示输出令牌 output tokens（LLM 生成的内容）
3. 进度条显示已经使用了的上下文窗口
4. 上下文窗口的最大值（例如，Claude 3.5-Sonnet 的 200k）

接下来我们从源码看一下 Cline 是如何管理上下文窗口的。

### Token 统计

依然是 Cline 任务处理的核心文件： `src/core/task/index.ts`，在 `recursivelyMakeClineRequests` 方法内，调用请求模型的方法：

```ts
const stream = this.attemptApiRequest(previousApiReqIndex)
```
模型返回的 type 为 `usage` 的 chunk 会带有请求的消耗的 token：

```ts
switch (chunk.type) {
  case "usage":
    didReceiveUsageChunk = true
    inputTokens += chunk.inputTokens
    outputTokens += chunk.outputTokens
    cacheWriteTokens += chunk.cacheWriteTokens ?? 0
    cacheReadTokens += chunk.cacheReadTokens ?? 0
    totalCost = chunk.totalCost
    break
```

流式请求处理完成后，将模型回复的消息和 token 信息塞进 `this.clineMessages` 消息内。

```ts
const updateApiReqMsg = (cancelReason?: ClineApiReqCancelReason, streamingFailedMessage?: string) => {
  const currentApiReqInfo: ClineApiReqInfo = JSON.parse(this.clineMessages[lastApiReqIndex].text || "{}")
  delete currentApiReqInfo.retryStatus // Clear retry status when request is finalized

  this.clineMessages[lastApiReqIndex].text = JSON.stringify({
    ...currentApiReqInfo, // Spread the modified info (with retryStatus removed)
    tokensIn: inputTokens,
    tokensOut: outputTokens,
    cacheWrites: cacheWriteTokens,
    cacheReads: cacheReadTokens,
    cost:
    totalCost ??
    calculateApiCostAnthropic(
      this.api.getModel().info,
      inputTokens,
      outputTokens,
      cacheWriteTokens,
      cacheReadTokens,
    ),
    cancelReason,
    streamingFailedMessage,
  } satisfies ClineApiReqInfo)
}
```

`this.clineMessages` 也会用到上下文管理的界面中。前端代码位于 `webview-ui/src/components/chat/ChatView.tsx` 和 `webview-ui/src/components/chat/task-header/TaskHeader.tsx`，这里不多展开了。

### 上下文窗口逻辑概况

上下文窗口管理的逻辑位于 `src/core/context/context-management/ContextManager.ts`。Task 类任务处理的过程中会调用上下窗口管理的逻辑。

上面提到，上下文窗口是有限制的，上下文不能无限制扩大，所以 Cline 每次向模型发起请求时，都会检查已有上下文信息 Token 数量是否已经超过了“最大值”，如果超过了，则需要对当前的上下文信息进行裁剪，以保证留足空间应对接下来的模型请求。

在 `attemptApiRequest` 方法内，每次要发起模型求前都会调用 `this.contextManager.getNewContextMessagesAndMetadata`，上下文窗口管理的逻辑都在这个方法里。

```ts
const contextManagementMetadata = await this.contextManager.getNewContextMessagesAndMetadata(
  this.apiConversationHistory,
  this.clineMessages,
  this.api,
  this.conversationHistoryDeletedRange,
  previousApiReqIndex,
  await ensureTaskDirectoryExists(this.getContext(), this.taskId),
)
```

### 智能缓冲管理

`getNewContextMessagesAndMetadata` 第一件事就是判断此时是否上下文窗口已经逼近极限，也就是我们上面说的最大值，Cline 称之为 Smart Buffer Management（智能缓冲管理）。

```ts
const timestamp = previousRequest.ts
const { tokensIn, tokensOut, cacheWrites, cacheReads }: ClineApiReqInfo = JSON.parse(previousRequest.text)
const totalTokens = (tokensIn || 0) + (tokensOut || 0) + (cacheWrites || 0) + (cacheReads || 0)
const { maxAllowedSize } = getContextWindowInfo(api)

// This is the most reliable way to know when we're close to hitting the context window.
if (totalTokens >= maxAllowedSize) {
  // 如果超过最大值，则执行后面的逻辑
}
```

计算缓冲的方法位于 `src/core/context/context-management/context-window-utils.ts`：

```ts
switch (contextWindow) {
  case 64_000: // deepseek models
    maxAllowedSize = contextWindow - 27_000
    break
  case 128_000: // most models
    maxAllowedSize = contextWindow - 30_000
    break
  case 200_000: // claude models
    maxAllowedSize = contextWindow - 40_000
    break
  default:
    maxAllowedSize = Math.max(contextWindow - 40_000, contextWindow * 0.8) // for deepseek, 80% of 64k meant only ~10k buffer which was too small and resulted in users getting context window errors.
}
```

contextWindow 是你在 Cline 配置的模型上下文窗口大小。对于不同模型给出不同的方法，总之既能确保新交互的空间，同时最大化了可用的上下文。

### 智能截断

如果此时逼近上下文窗口极限，Cline 设计了一套算法来处理已有的上下文。

首先，根据下面的逻辑确定需要裁剪多少信息，裁剪一半或者3/4（如果上下文远大于窗口值）。

```ts
const keep = totalTokens / 2 > maxAllowedSize ? "quarter" : "half"
```

然后进入裁剪逻辑，Cline 为上下文裁剪花了不好功夫。

1、重复文件去重

试想一下，如果多次读取一个文件作为上下文，那么只保留一个即可，这样就可以一些空间。入口方法为 `applyContextOptimizations`

```ts
let [anyContextUpdates, uniqueFileReadIndices] = this.applyContextOptimizations(
  apiConversationHistory,
  conversationHistoryDeletedRange ? conversationHistoryDeletedRange[1] + 1 : 2,
  timestamp,
)
```
`apiConversationHistory` 为用户和模型的交互记录，可以看到基本为 role user 和 assistant 交替出现。

![](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/06/25/1750860927736-f92bd074-5802-409b-a0f7-d5b0ac447c3d.png)

role 为 user 的消息内除了 user prompt 还有工具的执行记录。

![](https://fastly.jsdelivr.net/gh/bucketio/img7@main/2025/06/25/1750861045532-d0492506-bdd3-4b0e-accb-2ec451fc3843.png)

`applyContextOptimizations` 方法内调用了 `findAndPotentiallySaveFileReadContextHistoryUpdates`，内部分别为 `getPossibleDuplicateFileReads` 和 `applyFileReadContextHistoryUpdates`。

这两个方法的内容很长，我找 GPT 帮我读了下，作用是根据历史记录，从 role user 的历史消息中，找出可能重复读取过的文件（查找依据就是调用了 read_file, write_to_file, replace_in_file 这些工具或者使用 @ file），如果有重复的文件的读取，则记录他们的消息序号到 `uniqueFileReadIndices`，同时记录详细的更新信息至 `this.contextHistoryUpdates`（作用见下文）。

有了这些消息序号，进入 `calculateContextOptimizationMetrics` 方法内。

这里先计算前两条消息和后面所有消息的字符总数，同时结合 `uniqueFileReadIndices`，可以计算出可以节省的字数总数。如果这样就可以节省 30%，那我们就不用再做其他裁剪动作了，直接生成最终消息了（见下文）。

2、暴力裁剪，保留最近的上下文信息。

否则继续接下来的粗暴裁剪，根据需要裁剪的比例，确定要删除的内容，比如下面是删除一半的消息，这样主要保证能够成对的删除 user assistant 的消息。

```ts
messagesToRemove = Math.floor((apiMessages.length - startOfRest) / 4) * 2
```

最终获取到需要删除的消息序号为`[2, messagesToRemove]`，保留前两条消息，原始消息可能和项目结构或者任务相关。

接下来进入 `getAndAlterTruncatedMessages` 执行裁剪动作，

```ts
const firstChunk = messages.slice(0, 2) // get first user-assistant pair
const secondChunk = messages.slice(startFromIndex) // get remaining messages within context
const messagesToUpdate = [...firstChunk, ...secondChunk]
```

另外，我们之前记录了 `this.contextHistoryUpdates`，其内部已经记录了哪些消息可以精简的文件内容，基于它移除重复文件内容进一步精简 messagesToUpdate，比如之前的消息内记录了完整的文件内容：

```
'The user has provided feedback on the results. Consider their input to continue the task, and then attempt completion again.\n<feedback>\n\'src/index.tsx\' (see below for file content)  \n</feedback>\n\n<file_content path="src/index.tsx">\nimport \'./font.css\';\nimport { FC } from \'react\';\nimport styles from \'./index.module.css\';\nimport CoinCouponCard from \'./components/CoinCouponCard\';\nimport Appear from \'@ali/appear\';\n\ninterface CoinCouponModuleProps {}\n\nconst CoinCouponMo…猫币`);\n  };\n\n  return (\n    <Appear onAppear={() => { console.log(\'模块出现\') }}>\n      <div className={styles.container}>\n        <CoinCouponCard\n          userCoinAmount={userCoinAmount}\n          validPeriod={validPeriod}\n          coupons={coupons}\n          onUserCoinsClick={handleUserCoinsClick}\n          onQuestionClick={handleQuestionClick}\n          onExchangeCoupon={handleExchangeCoupon}\n        />\n      </div>\n    </Appear>\n  );\n};\n\nexport default CoinCouponModule;\n\n</file_content>'
```

精简后，file content 替换为了一句被替换的提醒。

```
The user has provided feedback on the results. Consider their input to continue the task, and then attempt completion again.\n<feedback>\n'src/index.tsx' (see below for file content)  \n</feedback>\n\n<file_content path="src/index.tsx">[[NOTE] This file read has been removed to save space in the context window. Refer to the latest file read for the most up to date version of this file.]</file_content>`
```

裁剪完成后，将新的对话记录作为上下文信息，就继续进入下一轮对话了。

## 总结

可以看出 Cline 对于上下文窗口的管理还是花了很多心思，从重复文件到消息裁剪做了很多事情。

不过正因为有了 Cline 的上下文窗口管理，一方面我们可以无需关心上下文窗口溢出；另外，我们也可以通过界面及时了解上下文窗口现状，以便在后续的工作中即使调整对策，比如重新开始或者拆分更小的任务等等。推荐查看[如何更好的使用 Cline - 正确的管理 Cline 的上下文](https://mp.weixin.qq.com/s/8iq2ekW_P6WDojML2KV1NA)了解 Cline 上下文的更多相关内容。
