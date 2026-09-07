# Notes: C++11 到 C++23 章节补充

## Repository Findings
- 目标文件 `04-现代C++/C++11到C++23.md` 只有元数据和空的四个章节，需要完整补充。
- 相邻现代 C++ 文档使用“学习目标 / 要点 / 示例与实践 / 关联”结构，链接采用 Obsidian 双链。
- `01-基础入门/C++概览.md` 已介绍标准、编译器和 C++11/14/17/20/23 的总体关系，本章应避免重复基础定义，重点放在按版本学习和代码迁移。

## Teaching Outline
- 总览：标准版本不是编译器名称；新项目优先以项目工具链支持的 C++17 或 C++20 为基线。
- C++11：现代 C++ 起点，覆盖 `auto`、范围 `for`、`nullptr`、`enum class`、Lambda、智能指针、移动语义、`constexpr`。
- C++14：泛型 Lambda、返回类型推导、`make_unique`、更宽松的 `constexpr`。
- C++17：结构化绑定、`if constexpr`、折叠表达式、`optional`/`variant`/`any`、`string_view`、`filesystem`、并行算法。
- C++20：概念、范围、协程、模块、三路比较、`consteval`、`jthread`/`stop_token`、指定初始化；强调工具链差异。
- C++23：`expected`、`print`/`println`、范围转容器和 `zip` 等库增强、显式对象参数、`if consteval`、`mdspan`；标明支持度需实测。
- 困难部分重点：移动语义不是“强制搬内存”，`string_view`/范围是非拥有视图，协程是可暂停的状态机，概念是编译期接口约束。

## Verification Notes
- 示例应尽量是单文件、标准库代码，并在代码注释中解释新手容易误解的地方。
- 文档校对重点：标题层级、反引号、双链目标、代码块语言标记、C++23 特性不误写为 C++20。
