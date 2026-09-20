# Task Plan: 完善“CMake构建”章节

## Goal
面向 C++ 初学者系统补充“CMake构建”章节，保留现有 Markdown 章节结构，提供 Windows 优先、可运行且概念准确的完整示例。

## Phases
- [x] Phase 1: 检查项目结构、目标文件和相邻章节风格
- [x] Phase 2: 梳理 CMake 基础概念、Windows 工作流和示例设计
- [ ] Phase 3: 编写章节正文与完整项目示例
- [ ] Phase 4: 校对 Markdown、命令、CMake 语义和章节连贯性

## Key Questions
1. 如何向初学者解释“编译器”和 CMake 的分工，而不把 CMake 误讲成编译器？
2. Windows 下 Visual Studio 多配置生成器与 MinGW 单配置生成器的命令差异是什么？
3. 示例是否覆盖源目录/构建目录、目标、依赖、头文件、测试、Debug/Release 和常见错误？

## Decisions Made
- 保留现有 frontmatter、学习目标、要点、示例与实践、关联五部分结构，在空小节下增加循序渐进的子章节。
- 以 CMake 3.20+、C++20 和 Windows PowerShell 为示例基线，同时解释 MSVC 与 MinGW 的差异。
- 使用一个包含静态库、可执行程序和测试目标的小项目贯穿主要命令；测试使用 CTest，不引入额外第三方框架。
- 将用户最容易混淆的 configure、build、test、install 阶段和单配置/多配置生成器单独说明。

## Errors Encountered
- 初次读取技能文件时误用了 `.codex\skills` 路径；已改用清单映射的 `C:\Users\Administrator\.agents\skills`。
- 更新历史计划时补丁上下文与文件实际措辞不一致，已重新读取精确内容并拆分补丁。

## Status
**Currently in Phase 3** - 计划与研究笔记已更新，正在编写 CMake 章节。
