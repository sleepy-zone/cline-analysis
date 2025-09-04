# Cline 源码浅析 - 从输入到输出

在了解 Cline 的架构后，本文将深入源码，从用户输入指令开始，一步步追踪 Cline 是如何处理请求、构建上下文、调用模型并最终输出结果的。
## 启动 Cline：调试环境准备

首先我们需要本地运行并调试 Cline。根据其 GitHub 仓库 [CONTRIBUTING -  Development Setup](https://github.com/cline/cline/blob/main/CONTRIBUTING.md#development-setup)的指引，我们可以快速搭建开发环境：

```bash
git clone https://github.com/cline/cline.git
code cline
npm run install:all
```

在 VSCode 里 `F5` 或者 `Run->Start Debugging` 即可启动调试会话。此时会打开一个全新的 VSCode 窗口（Extension Host），我们可以在其中测试和调试 Cline 插件。

> ⚠️ **常见问题提示**：
>  - 若构建失败，建议安装 [esbuild Problem Matchers](https://marketplace.visualstudio.com/items?itemName=connor4312.esbuild-problem-matchers) 插件。
>  - Webview 中依赖的 `electron` 包体积较大，下载可能较慢。可配置国内镜像源加速：

```yaml
electron_mirror=https://npmmirror.com/mirrors/electron/
electron_builder_binaries_mirror=https://npmmirror.com/mirrors/electron-builder-binaries/
shamefully-hoist=true
```
## Step by Step：从输入到输出的完整链路

VSCode 对插件调试支持非常完善，主进程可以断点调试，Webview 可以使用 Chrome DevTool。下面我们以一个典型任务为例，逐步解析整个执行流程。

### 前端输入

我们输入一条简单的指令：实现快速排序算法，并插入到 utils 文件内。

```
/newtask 实现快速排序算法，写入 @/src/utils/index.ts 文件
```

这条命令包含了 Cline 的两个关键特性：
- **内置命令（Slash Command） `/newtask`**：触发“新建任务”流程，背后对应一组预设 prompt。
- **Mention 语法 `@/path/to/file`**：引用指定文件内容作为上下文，增强模型理解能力。

> 这里使用 /newtask 和 @ 语法，仅作为演示用。 /newtask 命令一般用作打包当前工作进度重新开启新任务时使用，类似工作交接。

输入后点击发送，前端组件位于 `webview-ui/src/components/chat/ChatView.tsx` ，发送消息的逻辑抽象为 useMessageHandlers hooks：`webview-ui/src/components/chat/chat-view/hooks/useMessageHandlers.ts`。

Webview 负责收集用户输入（文本、图片、文件等），但不进行实际处理，而是将请求通过 `postMessage` 发送给 VSCode 主进程。

```typescript
// 文件位置 webview-ui/src/services/grpc-client-base.ts
window.addEventListener("message", handleResponse)
// Send the request
vscode.postMessage({
	type: "grpc_request",
	grpc_request: {
		service: this.serviceName,
		method: methodName,
		message: this.encode(request, encodeRequest),
		request_id: requestId,
		is_streaming: false,
	},
})
```

message 的格式：

```
files =(0) []
images =(0) []
text ='/newtask 创建一个快速排序算法，写入 @/src/utils/index.ts 文件'
```

VSCode 主进程接收 `grpc_request` 消息进行处理。

### VSCode 主进程接收消息

主进程处理的内容位于：`src/core/controller/grpc-handler.ts`，接收并分发请求：

```typescript
/**
 * Handles a gRPC request from the webview.
 */

export async function handleGrpcRequest(
  controller: Controller,
  postMessageToWebview: PostMessageToWebview,
  request: GrpcRequest,
): Promise<void> {
  if (request.is_streaming) {
	await handleStreamingRequest(controller, postMessageToWebview, request)    }   else {
	await handleUnaryRequest(controller, postMessageToWebview, request)
  }
}
```

可以看到，参数和 Webview 传过来的一致。

### 任务初始化：构建 Task 实例

经过一系列封装，请求最终进入 `src/core/controller/index.ts` 的 `initTask` 方法，初始化一个庞大的 `Task` 类实例：

```typescript
this.task = new Task(
  this.context,
  this.mcpHub,
  this.workspaceTracker,
  (historyItem) => this.updateTaskHistory(historyItem),
  () => this.postStateToWebview(),
  (message) => this.postMessageToWebview(message),
  (taskId) => this.reinitExistingTaskFromId(taskId),
  () => this.cancelTask(),
  apiConfiguration,
  autoApprovalSettings,
  browserSettings,
  chatSettings,
  shellIntegrationTimeout,
  terminalReuseEnabled ?? true,
  enableCheckpointsSetting ?? true,
  customInstructions,
  task,
  images,
  files,
  historyItem,
)
```

这个 `Task` 类（位于 `src/core/task/index.ts`）是整个任务执行的核心容器。我们在后面会经常跟这个文件打交道。

初始化完成后，进入真正的任务执行阶段。

首先，根据用户配置的模型提供商信息创建 AI 的 api handler：

```typescript
// Now that taskId is initialized, we can build the API handler
this.api = buildApiHandler(effectiveApiConfiguration)
```

然后，进入到 `startTask` 方法：

首先调用 `say` 方法，将用户的原始输入展示在聊天面板中：

![](https://fastly.jsdelivr.net/gh/bucketio/img4@main/2025/06/06/1749196272451-b424c65b-cb65-411f-9cbb-92322d586b92.png)

```typescript
await this.say("text", task, images, files)
```

### 构建 Prompt：组装上下文

接下来，构建 userContent：

```typescript
// 为用户输入的消息添加一个 task 标签。
const userContent: UserContent = [
	{
		type: "text",
		text: `<task>\n${task}\n</task>`,
	},
	...imageBlocks,
]

/**
输出：
text =
'<task>\n/newtask 创建一个快速排序算法，写入 @/src/utils/index.ts 文件\n</task>'
type =
'text'
*/
```

如果用户上传了文件，Cline 会读取内容并追加到上下文中：

```typescript
if (files && files.length > 0) {
  const fileContentString = await processFilesIntoText(files)
  if (fileContentString) {
    userContent.push({
      type: "text",
      text: fileContentString,
    })
  }
}
```

### 继续构造 Prompts

用过 Cline 的话，我们会知道 Cline 会循环处理任务，所以 User Prompts 构造完成后，进入到 `initiateTaskLoop` 方法，真正执行任务的方法为 `recursivelyMakeClineRequests`：

进一步处理用户的输入信息：

```typescript
const [parsedUserContent, environmentDetails, clinerulesError] = await this.loadContext(userContent, includeFileDetails)
```

这样处理之后，用户输入重新组装为 parsedUserContent，内容比较长，这里不贴了，大致格式为：

```
<explicit_instructions type="new_task"></explicit_instructions>
<task>用户输入</task>
<fileContent>src/utils/index.ts内的文件</fileContent>
```

再次看一下我们的输入：`/newtask 实现快速排序算法，写入 @/src/utils/index.ts 文件`，上面三段一一对应：<explicit_instructions type="new_task"> 对应 /newtask 命令，这是 Cline 的内置 prompt，@/src/utils/index.ts 添加了该文件内容作为上下文。

同时将 environmentDetails 也添加到 userContent，environmentDetails 有工作目录、当前时间、当前打开的 Tab 等。

```typescript
userContent = parsedUserContent
// add environment details as its own text block, separate from tool results
userContent.push({ type: "text", text: environmentDetails })
```

> 正是这种“尽可能多注入上下文”的设计，让 Cline 能精准理解用户意图，但也因此消耗较多 Token。

### 模型请求

继续进入 `attemptApiRequest` 方法，首先构造 System prompt：

```ts
const isClaude4ModelFamily = await this.isClaude4ModelFamily()
let systemPrompt = await SYSTEM_PROMPT(cwd, supportsBrowserUse, this.mcpHub, this.browserSettings, isClaude4ModelFamily)
```

比较有意思的是，Cline 针对 Claude4 定制了不同的 System Prompt。

> System Prompt 里会告诉模型我们当前已经提供了哪些工具以及这些工具该如何调用（参数和规则），比如本次任务要将模型生成的代码写入已有文件，那么就需要使用到 `replace_in_file` 这个工具，这里贴一下 System Prompt 里关于  `replace_in_file` 的内容。

```markdown
## replace_in_file
Description: Request to replace sections of content in an existing file using SEARCH/REPLACE blocks that define exact changes to specific parts of the file. This tool should be used when you need to make targeted changes to specific parts of a file.
Parameters:
- path: (required) The path of the file to modify (relative to the current working directory ${cwd.toPosix()})
- diff: (required) One or more SEARCH/REPLACE blocks following this exact format:
  \`\`\`
  ------- SEARCH
  [exact content to find]
  =======
  [new content to replace with]
  +++++++ REPLACE
  \`\`\`
  Critical rules:
  1. SEARCH content must match the associated file section to find EXACTLY:
     * Match character-for-character including whitespace, indentation, line endings
     * Include all comments, docstrings, etc.
  2. SEARCH/REPLACE blocks will ONLY replace the first match occurrence.
     * Including multiple unique SEARCH/REPLACE blocks if you need to make multiple changes.
     * Include *just* enough lines in each SEARCH section to uniquely match each set of lines that need to change.
     * When using multiple SEARCH/REPLACE blocks, list them in the order they appear in the file.
  3. Keep SEARCH/REPLACE blocks concise:
     * Break large SEARCH/REPLACE blocks into a series of smaller blocks that each change a small portion of the file.
     * Include just the changing lines, and a few surrounding lines if needed for uniqueness.
     * Do not include long runs of unchanging lines in SEARCH/REPLACE blocks.
     * Each line must be complete. Never truncate lines mid-way through as this can cause matching failures.
  4. Special operations:
     * To move code: Use two SEARCH/REPLACE blocks (one to delete from original + one to insert at new location)
     * To delete code: Use empty REPLACE section
Usage:
<replace_in_file>
<path>File path here</path>
<diff>
Search and replace blocks here
</diff> 
</replace_in_file>
```

同时，System Prompt 也继续追加了用户的偏好语言，方便按照用户偏好的语言进行输出。

接下来发送请求：

```ts
let stream = this.api.createMessage(systemPrompt, contextManagementMetadata.truncatedConversationHistory)
```

### 模型输出解析

针对模型的输出，需要进一步处理，是直接展示在聊天面板，还是需要调用工具处理，这部分逻辑在 `src/core/assistant-message/parse-assistant-message.ts` 的 `parseAssistantMessageV2` 方法内。

针对我们本次创建排序算法的任务，下面是一个模型输出样例：

```markdown
<thinking>\n我需要在`src/utils/index.ts`文件中添加一个快速排序算法。首先，我已经看到了文件的内容，这是一个工具函数集合文件，包含了各种实用函数，如rpx转换、vw计算、类型检查、日志记录等。\n\n我需要在这个文件中添加一个快速排序算法实现。快速排序是一种高效的排序算法，基于分治法的思想。我将实现一个通用的快速排序函数，可以对数组进行排序。\n\n我将使用`replace_in_file`工具来修改文件，在文件末尾添加快速排序算法的实现。我需要确保添加的代码与现有代码风格保持一致，并提供适当的类型注解和注释。\n\n我将在文件末尾添加快速排序算法的实现，包括：\n1. 一个主快速排序函数，接受一个数组和可选的比较函数\n2. 一个辅助函数用于分区操作\n\n让我使用`replace_in_file`工具来修改文件。\n</thinking>\n\n<replace_in_file>\n<path>src/utils/index.ts</path>\n<diff>\n------- SEARCH\nexport const jump2UpdateApp = () => {\n  if (is… (a, b) => {\n  if (a < b) return -1;\n  if (a > b) return 1;\n  return 0;\n}): T[] => {\n  // 创建数组副本，不修改原数组\n  const result = [...arr];\n  \n  // 内部快速排序实现\n  const _quickSort = (arr: T[], left: number, right: number) => {\n    if (left < right) {\n      const pivotIndex = partition(arr, left, right);\n      _quickSort(arr, left, pivotIndex - 1);\n      _quickSort(arr, pivotIndex + 1, right);\n    }\n  };\n  \n  // 分区操作\n  const partition = (arr: T[], left: number, right: number): number => {\n    // 选择最右边的元素作为基
```

`parseAssistantMessageV2` 方法会解析上述输出内容，主要是从模型输出中解析出使用哪些工具以及工具的参数，比如这里我们发现模型输出中有 <replace_in_file>，那就说明这次需要使用 replace_in_file 工具。

接下来调用 `presentAssistantMessage` 方法真正将模型消息展示给用户，一方面展示文本消息，类似下图：

![](https://fastly.jsdelivr.net/gh/bucketio/img9@main/2025/06/06/1749217048970-dba8c0b7-c03c-4767-a014-845c105ab0d9.png)

一方面执行上一步解析出的 tool，将模型生成代码以 diff 的方式写入文件，这里借助 VSCode 的能力创建 diffView。自此这个任务算是完成了。

## 总结

剥离 UI 与工程细节，Cline 的本质可归结为两大核心机制：

1. 上下文驱动的 Prompt 组装：Cline 的强大源于其**极致的上下文注入能力**。它将以下信息融合为一个完整的 prompt：

- 用户输入（文本、图片、文件）
- 内置命令（如 `/newtask`）
- Mention 引用（文件、目录内容）
- 环境信息（时间、路径、打开文件）
- 用户偏好（语言、风格）
- Rules 与 Workflow（自定义逻辑）
- System Prompt（工具定义、行为规范）

这种“上下文即能力”的设计，显著提升了模型输出的准确性与实用性。

2. 工具增强的 Agent 能力：Cline 并非简单的聊天机器人，而是一个具备行动能力的 **Agent**。其关键在于：

- 在 System Prompt 中声明可用工具（如 `replace_in_file`, `run_command`, `browse_url`）
- 模型输出中嵌入工具调用指令
- 主进程解析并执行工具，实现文件修改、命令执行等操作

两者配合大模型共同合作完成用户的编码的任务。

---

本文基于边调试边写作完成，另外 Cline 更新频率也很高，若有疏漏或理解偏差，欢迎交流指正。




