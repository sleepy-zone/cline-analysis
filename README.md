<p align="center"><img alt="logo" src="https://cline.bot/assets/images/home/hero/hero-character.webp" width="80" /></p>

<h1 align="center">
  <strong>Cline 源码浅析 </strong>
</h1>

本项目旨在通过分析 [Cline](https://github.com/cline/cline) 的源码实现，深入理解 SWE Agent 的工作原理。从源码入手，不仅能更清晰地掌握其设计思想与核心技术，也有助于我们在实际应用中更有效地使用 Cline 这类智能代理工具。

<p align="center">
	<img src="https://raw.githubusercontent.com/sleepy-zone/mp-pic-bed/main/2025/09/04/1756966035854-a10012bc-3fd7-41b8-8d66-0c6b761582a6.png" />
</p>

✍️ 系列文章首发于我的公众号 **【VibeCode RealLife】**，持续更新中，欢迎关注以获取最新内容。

<p align="center"><img src="https://camo.githubusercontent.com/e29c39af8c4d7763dc27b237bfd067ccba4c27c135ce1abe1cb8180d7897527d/68747470733a2f2f636c6f75642d6d696e6170702d34373830332e636c6f75642e6966616e7275736572636f6e74656e742e636f6d2f317476414d3638437672783362664c522e6a7067" width="240" /></p>

---

**👉 开始阅读：**

* [Cline 源码浅析 - 从输入到输出](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%E4%BB%8E%E8%BE%93%E5%85%A5%E5%88%B0%E8%BE%93%E5%87%BA.md)
* [Cline 源码浅析 - Prompt 设计](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%20Prompt%20%E8%AE%BE%E8%AE%A1.md)
* [Cline 源码浅析 - MCP 调用](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20MCP%20%E8%B0%83%E7%94%A8.md)
* [Cline 源码浅析 - 递归执行引擎](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%E9%80%92%E5%BD%92%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E.md)
* [Cline 源码浅析 - 上下文窗口管理](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20%E4%B8%8A%E4%B8%8B%E6%96%87%E7%AA%97%E5%8F%A3%E7%AE%A1%E7%90%86.md)
* [Cline 源码浅析 - Cline 是如何定位你的代码的？](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%20-%20Cline%20%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9A%E4%BD%8D%E4%BD%A0%E7%9A%84%E4%BB%A3%E7%A0%81%E7%9A%84%EF%BC%9F.md)
* [Cline 源码浅析番外 - 使用 Async Generator 实现流式输出与流程控制](Cline%20%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90%E7%95%AA%E5%A4%96%20-%20%E4%BD%BF%E7%94%A8%20Async%20Generator%20%E5%AE%9E%E7%8E%B0%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA%E4%B8%8E%E6%B5%81%E7%A8%8B%E6%8E%A7%E5%88%B6.md)
* [Cline 源码浅析 — 检查点：用影子 Git 实现安全可靠的代码变更追踪](Cline%20%E6%A3%80%E6%9F%A5%E7%82%B9%EF%BC%9A%E7%94%A8%E5%BD%B1%E5%AD%90%20Git%20%E5%AE%9E%E7%8E%B0%E5%AE%89%E5%85%A8%E5%8F%AF%E9%9D%A0%E7%9A%84%E4%BB%A3%E7%A0%81%E5%8F%98%E6%9B%B4%E8%BF%BD%E8%B8%AA.md)

---

🌟 如果你觉得有收获，欢迎点个 Star ⭐

💬 个人水平有限，欢迎指正，欢迎 issue 和 PR！
