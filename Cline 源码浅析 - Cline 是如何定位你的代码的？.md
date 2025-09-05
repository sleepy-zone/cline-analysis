# Cline 源码浅析 - Cline 是如何定位你的代码的？

我之前翻译过 [为什么 Cline 不会索引你的代码库](https://mp.weixin.qq.com/s/LU_5EPLPgTQNB3cyh2l1iw)，文中提到代码不适合使用 RAG，Cline 会像开发者一样**高效的**逐个文件阅读代码，最终精确定位到目标代码。本文就来讨论下 Cline 是如何做到的。

## Tree Sitter

没错，是 **Tree Sitter**，官网地址：[https://tree-sitter.github.io/tree-sitter/](https://tree-sitter.github.io/tree-sitter/)。

> Tree-sitter 是一个解析器生成工具和增量解析库。它可以为源文件构建具体的语法树，并在源文件编辑时高效地更新语法树。

当我们人类开发者接手一个新项目时，我们不会一头扎进去，逐行阅读每一行代码。我们通常会先浏览一下目录结构，看看有哪些关键的类、函数和模块，在脑子里构建一个项目的宏观蓝图。

Tree Sitter 的优势是足够快，性能足够好，甚至官方称可以一边编辑代码一边生成语法树。

- 足够通用，可以解析任何编程语言
- 足够快速，可以在文本编辑器中每次按键时进行解析
- 足够稳健，即使在存在语法错误的情况下也能提供有用的结果
- 无依赖性，因此运行时库（用纯 C11 编写）可以嵌入到任何应用程序中

（我并没有对 Tree Sitter 进行详细的了解，具体可以参考官方文档，或者 https://zhuanlan.zhihu.com/p/716273346 这篇文章。）

Cline 内相关代码位于：`src/services/tree-sitter` 目录下。

## Cline 的做法

总的来说，Cline 依赖**高性能的**TreeSitter 对文件目录进行初步解析，获取到代码内的类、方法等基础信息，模型可以获取到与本次需求相关联的文件，继续阅读文件内容，给出建议或者修改。下面我们看代码。

### System Prompt

Cline 使用 Tree Sitter 主要对代码进行初步进行分析，这里涉及到的工具为 `list_code_definition_names`。

工具定义如下：
```md
## 列出代码定义名称

描述: 请求列出指定目录中源代码文件顶层使用的定义名称（类、函数、方法等）。这个工具提供了对代码库结构和重要构造的洞察，涵盖了理解整体架构所需的高级概念和关系。

参数:
- 路径: (必需) 要列出顶层源代码定义的目录路径（相对于当前工作目录 ${cwd.toPosix()}）。

使用方法:
<list_code_definition_names>
<path>在此输入目录路径</path>
</list_code_definition_names>
```

何时使用该工具：
```md
你可以使用 `list_code_definition_names` 工具来获取指定目录中所有文件顶层的源代码定义概况。这在需要理解代码某些部分之间的广泛背景和关系时特别有用。为了理解与任务相关的代码库的不同部分，你可能需要多次调用这个工具。

例如，当被要求进行编辑或改进时，你可以先分析初始 `environment_details` 中的文件结构，以获得项目概况，然后使用 `list_code_definition_names` 获取相关目录中文件的源代码定义进一步洞察。接着，使用 `read_file` 检查相关文件的内容，分析代码并提出改进建议或进行必要的编辑，然后使用 `replace_in_file` 工具实施更改。如果你重构的代码可能影响代码库的其他部分，可以使用 `search_files` 确保更新其他文件。
```

### Environment Details

prompt 反复提及 `environment_details`，获取环境信息的代码位于 `src/core/task/index.ts` 的 `getEnvironmentDetails` 方法内。

主要涉及：
1. VSCode 可见文件：当前在 VSCode 编辑器中可见的所有文件路径
2. VSCode 打开的标签页：当前在 VSCode 中打开的所有标签页文件路径
3. 活动终端信息：
	a. 正在运行的终端：显示正在执行命令的终端及其输出
    b. 非活动终端：显示已完成命令的终端及其新输出
4. 最近修改的文件
5. 当前时间信息/文件系统/上下文窗口使用情况/当前模式(PLAN or ACT)信息

模型可以从 VSCode 可见文件 和 VSCode 打开的标签页开始，使用  `list_code_definition_names` 分析源代码结构。

### 工具调用

代码位于 `src/core/task/ToolExecutor.ts` 的 `executeTool` 方法下。

当模型要求调用 `list_code_definition_names` 工具时，就调用 `src/services/tree-sitter` 的 `parseSourceCodeForDefinitionsTopLevel` 方法，本质上是遍历目录，调用 `parseFile` 方法，获取每个方法的基本定义。

```ts
for (const filePath of allowedFilesToParse) {
  const definitions = await parseFile(filePath, languageParsers, clineIgnoreController)
  if (definitions) {
    result += `${path.relative(dirPath, filePath).toPosix()}\n${definitions}\n`
  }
}
```

 `parseFile` 调用 Tree Sitter 的 parse，将解析的结果进行格式化输出。最后填充到上下文中去。

解析结果如下图，可以看到包含了文件名以及文件内的类的定义，方法的定义等信息：
![](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/08/03/1754231294107-a4528cf7-838d-473e-84f6-7bb57dd1a55a.png)

## 总结

理论上，这套工作流非常完美。但在实际测试中，我发现了一个有趣的现象。

无论是用 qwen-coder-plus 还是 Claude 3.7 Sonnet，大模型似乎有点“懒”，它们并不总是优先选择 `list_code_definition_names` 这个最高效的工具。 有时候，它们还是会倾向于用更“笨”的办法——直接读取目录下的文件进行分析。

这可能是因为直接读取对模型来说更简单直接，调用特定工具需要更复杂的思考链。

但这并不影响这个思路给我们的启发：

**在处理复杂问题时，结构化的、由宏观到微观的分析方法，远比无头苍蝇式的蛮干要高效得多。** 这不仅是AI编程的趋势，对我们人类开发者来说，同样是一种值得借鉴的工作智慧。
