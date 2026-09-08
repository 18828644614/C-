# Task Plan: 补充“字符串、视图与范围”章节

## Goal
从 C++ 新手角度完善“字符串、视图与范围”章节，补充难点讲解、可运行示例和代码注释，并保持项目现有 Markdown 风格。

## Phases
- [x] Phase 1: 了解项目结构、章节位置和写作风格
- [x] Phase 2: 梳理章节知识点与示例设计
- [x] Phase 3: 编写并补充章节内容
- [x] Phase 4: 校对 Markdown、代码和概念准确性

## Key Questions
1. 目标章节文件在哪里，前后章节采用什么结构？
2. 新手最容易混淆的 string、string_view、范围和迭代器问题有哪些？
3. 示例是否覆盖生命周期、边界、算法和 C++20 ranges 的常见坑？

## Decisions Made
- 以“先理解对象，再理解视图，最后组合范围算法”的渐进顺序组织内容。
- 示例优先使用标准库和短小可复制代码，并在困难处添加行内注释。

## Errors Encountered
- 初次读取技能文件时使用了错误的根目录，已改用 `C:/Users/Administrator/.agents/skills`。
- 一次 `apply_patch` 同时删除并新增同一文件导致校验失败，随后拆成两个补丁完成。
- 一次 `rg` 命令的正则引号未闭合，改用简单的逐项检查。

## Status
**Completed** - 章节已补写，并完成 Markdown 结构检查与 C++20 示例语法校验。
