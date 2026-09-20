# Notes: “CMake构建”章节

# 项目观察

- 目标文件 `08-工程与项目/CMake构建.md` 只有 frontmatter、学习目标和空的“要点/示例与实践”小节。
- 相邻的《源文件、编译与链接》已经解释编译器、目标文件和链接器，本章应承接这些概念，重点说明 CMake 如何组织并驱动它们。
- 项目整体面向学习者，章节使用中文解释、短代码块和 Obsidian Wiki 链接；本章保留该风格，但需要提供一个足够完整的多目标示例。

## 内容设计

- 先建立三层关系：编译器负责翻译 C++，底层构建工具负责执行命令，CMake 负责生成和协调构建系统。
- 按 configure → build → test → install 的顺序解释 CMake 工作流，避免把 `cmake -S/-B` 和 `cmake --build` 混为同一步。
- 以 Windows 为主线，分别展示 Visual Studio 多配置生成器与 MinGW Makefiles 单配置生成器；强调 `--config Debug` 只适用于多配置生成器的常见用法。
- 完整示例包含 `greeting` 静态库、`greet` 可执行程序和 `greet_test` 测试目标，展示 target、`target_link_libraries`、`target_include_directories`、编译特性和 CTest。
- 覆盖源目录/构建目录、私有/公开依赖、生成器缓存、配置选项、警告、常见错误和清理重配等初学者高频问题。

## 验证清单

- [ ] 章节保留 frontmatter、学习目标、要点、示例与实践、关联结构。
- [ ] 代码块中的 CMake 语法、PowerShell 命令和 C++ 示例相互一致。
- [ ] 主示例可从全新构建目录执行配置、构建、测试和运行。
- [ ] 对单配置/多配置生成器、`PRIVATE/PUBLIC/INTERFACE` 和 `configure/build/test` 的说明准确。
- [ ] 完成后检查 Markdown 围栏、链接、标题层级和工作区差异。
