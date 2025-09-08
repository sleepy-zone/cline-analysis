# Cline 架构解析

之前的源码分析多聚焦于单点模块，本文从 Cline 的架构出发，理清系统的各个部分是怎么连接和协作的，对 Cline 有一个整体的认识。

![](https://raw.githubusercontent.com/sleepy-zone/mp-pic-bed/main/2025/09/04/1756966035854-a10012bc-3fd7-41b8-8d66-0c6b761582a6.png)

### 底层：统一的模型接入层

最底层是各类大模型服务：OpenAI、Anthropic、Qwen 等官方模型厂商、OpenRouter 这类第三方中转服务商或者是本地部署的 LLM 模型（如 Ollama），他们提供大模型的 API。

为了统一调用模型，Cline 在 `src/core/api` 目录下为每一家模型服务商封装了独立的 Provider。这些 Provider 负责请求格式转换、流式响应解析、错误重试等适配逻辑。

所有 Provider 最终通过一个核心函数统一暴露能力：

```ts
export function buildApiHandler(
  configuration: ApiConfiguration, mode: Mode
): ApiHandler {}
```

### 交互层：基于 VSCode Webview 的 UI 构建

用户通过 VSCode 扩展与 Cline 交互。所有界面——包括任务输入、设置面板、历史记录、MCP 市场等功能——都运行在一个由 VSCode 提供的 Webview 环境中。

本质上，这是一个嵌入在编辑器中的 React Web 应用。Webview 和 VSCode 通过 **RPC 消息**进行通信（如果你了解 Electron，那对这部分并不会陌生）：

```ts
window.addEventListener("message", handleResponse)

vscode.postMessage({
  type: "grpc_request",
  grpc_request: {
    service: service.fullName,
    method: method.name,  // method 为 newTask
    message: encodedRequest, // message 为输入的文本、图片、文件消息等
    request_id: requestId,
    is_streaming: false,
  },
})
```

（未来视情况，应该会推出一篇VSCode Webview 扩展开发教程）

### 核心引擎：Agent 任务循环

在 Cline 博客[什么是 Coding Agent?](https://mp.weixin.qq.com/s/1fP5IeGHnyzttD5oUJkvuw)一文中指出，Code Agent 包括三大核心（模型 + 指令 + 工具）以及一个递归处理引擎。

用户输入的消息，进入 Agent 的任务循环处理引擎内。

Cline 提供了 [**Plan + Act** 的任务执行范式](https://mp.weixin.qq.com/s/ZKwiCE_wb-TqIvhJ4K-CiQ)，这也是当前被广泛认可的 Agent 最佳实践。它不会盲目执行，而是先规划路径，再分步行动，确保任务推进的合理性与可控性。

整个任务循环包含以下几个关键环节：

* **Prompt 模块化**：结合 System Prompt（角色设定、规则约束）和 User Prompt（用户输入），构建完整的上下文。[Cline 源码浅析 - Prompt 设计](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%20Prompt%20%E8%AE%BE%E8%AE%A1.md)
* **工具调用**：灵活调度内置工具（如代码解释器、文件读写）或通过 MCP（Model Calling Protocol）集成的外部能力。[Cline 源码浅析 - MCP 调用](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20MCP%20%E8%B0%83%E7%94%A8.md)
* **上下文管理**：实时监控 token 使用情况，防止超出模型上下文窗口，必要时进行摘要或压缩。[Cline 源码浅析 - 上下文窗口管理](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%E4%B8%8A%E4%B8%8B%E6%96%87%E7%AA%97%E5%8F%A3%E7%AE%A1%E7%90%86.md)
* **功能增强**：支持任务中断、继续、修改等操作、关键操作需用户确认，保证用户对流程的掌控权。[Cline 源码浅析番外 - 使用 Async Generator 实现流式输出与流程控制](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%E7%95%AA%E5%A4%96%20-%20%E4%BD%BF%E7%94%A8%20Async%20Generator%20%E5%AE%9E%E7%8E%B0%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA%E4%B8%8E%E6%B5%81%E7%A8%8B%E6%8E%A7%E5%88%B6.md)
* **检查点机制**：每一步操作都设有检查点，便于回溯、调试和异常恢复。[Cline 源码浅析 — 检查点：用影子 Git 实现安全可靠的代码变更追踪](Cline%20%E6%A3%80%E6%9F%A5%E7%82%B9%EF%BC%9A%E7%94%A8%E5%BD%B1%E5%AD%90%20Git%20%E5%AE%9E%E7%8E%B0%E5%AE%89%E5%85%A8%E5%8F%AF%E9%9D%A0%E7%9A%84%E4%BB%A3%E7%A0%81%E5%8F%98%E6%9B%B4%E8%BF%BD%E8%B8%AA.md)

### 架构定位：专注软件工程的单 Agent 设计

Cline 是一个专为软件工程场景打造的 Agent，采用单 Agent 架构，意味着它不追求复杂的多智能体协作，设计上简单直接。正如他们提到的：**“最有效的AI智能体，往往是设计最简单的那个”**。
