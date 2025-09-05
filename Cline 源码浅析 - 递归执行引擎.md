# Cline 源码浅析 - 递归执行引擎

前面我们聊了 Cline [从输入到输出的流程](https://github.com/sleepy-zone/cline-analysis/blob/main/Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%E4%BB%8E%E8%BE%93%E5%85%A5%E5%88%B0%E8%BE%93%E5%87%BA.md)、[MCP 的调用](https://github.com/sleepy-zone/cline-analysis/blob/main/Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20MCP%20%E8%B0%83%E7%94%A8.md)以及[Prompt 设计](https://github.com/sleepy-zone/cline-analysis/blob/main/Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%20Prompt%20%E8%AE%BE%E8%AE%A1.md)，这篇文章，我们再回过头看一下 Cline 内很重要的一部分 - **递归执行引擎**。

如果你使用过 Cline 这类 AI Agent，你可能会注意到：一个任务往往不会在一次模型交互中完成。聊天界面会多次出现 `API Request...` 的提示，说明系统正在与模型进行多轮对话。每一轮都基于前一次的结果（如用户反馈或工具调用结果）推进，直到模型主动确认任务完成，整个流程才终止。这种“逐步推进、持续反馈”（ReAct）的机制，正是由 **递归执行引擎** 驱动的任务循环。

### Prompt 设计

我们还是先来看下 Cline 的 Prompt 设计，如何保证模型有条不紊的执行的每次循环。

首先 Cline 对模型强调了工具的使用规则：

```
您可以访问一组工具，这些工具会在用户批准后执行。您每次消息中只能使用一个工具，并将在用户的回复中收到该工具使用的执行结果。您会逐步使用工具来完成给定任务，每次工具的使用都会基于前一个工具使用的结果进行决策。
```

> You have access to a set of tools that are executed upon the user's approval. You can use one tool per message, and will receive the result of that tool use in the user's response. You use tools step-by-step to accomplish a given task, with each tool use informed by the result of the previous tool use.

1. 工具需要用户批准进行
2. 每次消息只能使用一个工具
3. 根据用户反馈，逐步使用工具完成任务

对于复杂任务，这就保证了模型不必一次性完成任务，而是可以在多轮会话中使用多种工具共同协作完成任务。

那模型在任务完成后如何告知我们呢？这就是 `attempt_completion` 工具的作用了，Prompt 如下：

```markdown
## 尝试完成  
描述：每次使用工具后，用户会回复该工具的执行结果，即是否成功以及失败的原因。一旦您收到了工具使用的执行结果，并可以确认任务已经完成，请使用此工具向用户展示您的工作成果。您可以选择性地提供一个 CLI 命令来展示您的工作成果。如果用户对结果不满意并给出反馈，您可以根据反馈进行改进并再次尝试。  

重要提示：在确认用户之前的工具使用都已成功之前，**不能**使用此工具。否则会导致代码损坏和系统故障。在使用此工具之前，您必须在 `<thinking></thinking>` 标签中自问：是否已从用户那里确认所有先前的工具使用都是成功的？如果不是，则**不要**使用此工具。  

参数：  
- **result**（必填）：任务的结果。请以最终形式表述这个结果，不需要用户进一步输入。不要以问题或提供进一步帮助的提议结尾。  
- **command**（可选）：一条 CLI 命令，用于向用户实时演示结果。例如，使用 `open index.html` 来显示创建的 HTML 网站，或者使用 `open localhost:3000` 来显示本地运行的开发服务器。但**不要**使用像 `echo` 或 `cat` 这类只打印文本的命令。该命令应适用于当前操作系统，并确保格式正确，不包含任何有害指令。  

Usage:
<attempt_completion>
<result>
Your final result description here
</result>
<command>Command to demonstrate result (optional)</command>
</attempt_completion>
```

模型确认任务完成后，调用 `attempt_completion` 这个工具，根据我们之前聊过的，Cline 会从模型的消息中解析工具的使用，在解析到模型消息中包含 `attempt_completion` 这个工具的使用时，我们可以确认任务完成了。

### Task：递归执行的核心

后续就是 Cline 对模型消息的处理和递归的逻辑了，这部分代码在 `src/core/task/index.ts` 文件下。

`recursivelyMakeClineRequests`，看名字就知道是在做递归调用，主要是做一些 User Prompts 的拼装、聊天消息的组装，处理模型请求的流式消息等，只要模型没明确调用 `attempt_completion` ，就一直会调用自身。

关于 `attempt_completion` 工具的真实调用还是在 `presentAssistantMessage` 方法内，这里就不展开了。

下面是一个输出示例，使用 Cline 将截图转为 React 代码，任务完成后，模型输出如下：
```markdown
<thinking>
Now I'll complete the task by summarizing what I've done and presenting the final result.
</thinking>

<attempt_completion>
<result>
I've successfully converted the provided module screenshot into a React component structure following all the specified requirements. Here's what I've created:

....
....

The implementation closely follows the visual design from the provided screenshot, with proper styling, layout, and component structure.
</result>
</attempt_completion>
```

可以看到，模型在 `<thinking>` 中进行了自我验证，确认任务已完成，随后通过 `attempt_completion` 输出最终结果
### 总结

Cline 的递归执行引擎，本质上是一个**基于 Prompt 引导和工具调用驱动的任务循环系统**。它通过精心设计的 System Prompt，约束模型的行为模式，使其在“使用工具 → 等待反馈 → 判断进展 → 决定下一步”的路径上稳步前行。

> Prompt 是“方向盘”，工具是“油门与刹车”，而递归执行引擎，则是让这辆车持续前行的“发动机”。


