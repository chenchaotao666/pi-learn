# Pi 学习文档

本目录面向希望系统理解 Pi 源码与设计的读者。

1. [Pi 项目详细学习计划](./pi-learning-plan.md)：6 周主线、2 周进阶、每日阅读与实验、验收标准和综合项目。
2. [Pi 技术特点与原理分析](./pi-technical-analysis.md)：架构、Agent 循环、Provider 抽象、Session 树、上下文压缩、扩展、MCP/codemode、接口、TUI、安全与工程化。
3. [Pi 调试指南](./pi-debugging-guide.md)：源码级调试方法（VS Code、Chrome DevTools、测试调试）、source-resolver 原理与常见问题。
4. [Pi 上下文设计与策略](./pi-context-design.md)：会话树、上下文投影、按需加载、压缩预算、摘要保留、溢出恢复与源码阅读路径。

推荐阅读方式：先学习计划，建立整体地图；每周结合技术分析的对应章节阅读源码，并用具体调用链和最小实验验证理解。
