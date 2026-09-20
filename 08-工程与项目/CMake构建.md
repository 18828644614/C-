---
type: topic
status: draft
created: 2026-09-01
updated: 2026-09-20
tags: [cpp, cmake, 构建]
---

# CMake构建

## 学习目标
- [ ] 说清 CMake、编译器、链接器和底层构建工具各自负责什么。
- [ ] 完成 CMake 的配置、构建、测试和安装流程。
- [ ] 用目标导向方式组织库、可执行程序、编译选项和测试。
- [ ] 在 Windows 下分别使用 Visual Studio 和 MinGW 完成一个小项目。
- [ ] 能根据错误信息判断是配置、编译、链接还是运行阶段出了问题。

## 要点

### 1. CMake 解决了什么问题

先回顾一个最小的 C++ 构建过程。假设项目中有 `main.cpp` 和 `math.cpp` 两个源文件，编译器通常先分别把它们翻译成目标文件，再由链接器把目标文件和库合并成可执行程序：

```text
main.cpp  --编译-->  main.obj
math.cpp  --编译-->  math.obj
main.obj + math.obj  --链接-->  demo.exe
```

这个过程在源文件很少时可以手动完成，但项目一大就会出现很多重复工作：

- 不同操作系统的编译命令不同，Windows 可能使用 MSVC，Linux 可能使用 GCC 或 Clang。
- Debug 和 Release 的选项不同，32 位和 64 位的选项也可能不同。
- 库之间有依赖关系，修改一个源文件后只应该重新构建受影响的部分。
- 测试、安装、生成文档和静态检查都需要额外的构建目标。

CMake 是一个**构建系统生成器**。它读取项目中的 `CMakeLists.txt`，根据当前平台和生成器（generator）生成适合本机的构建文件，然后由这些构建文件调用真正的编译器和链接器。

可以把几类工具分成四层：

| 工具 | 主要职责 |
| --- | --- |
| CMake | 描述项目结构、依赖和构建规则；生成构建系统 |
| 生成器 | 把 CMake 的规则生成 Visual Studio 工程、Ninja 文件或 Makefile |
| 底层构建工具 | 根据生成的规则执行编译、链接和增量构建 |
| 编译器和链接器 | 编译 `.cpp`，并把目标文件和库链接成程序 |

因此，CMake 不是编译器，也不会替代 MSVC、GCC 或 Clang。`CMakeLists.txt` 是构建规则的源文件，`build` 目录中的工程文件、缓存和中间产物都是生成结果。

### 2. Windows 下需要哪些工具

推荐先安装以下工具：

1. CMake。安装时可以选择把 CMake 加入当前用户的 `PATH`。
2. 一套 C++ 工具链。初学者可以安装 Visual Studio 或 Visual Studio Build Tools，并勾选“使用 C++ 的桌面开发”；也可以安装 MinGW-w64。
3. 可选的 Ninja。它是速度较快的底层构建工具，但本章的完整示例只依赖 Visual Studio 或 MinGW。

安装后在 PowerShell 中检查：

```powershell
cmake --version
g++ --version
```

如果使用 MSVC，`cl.exe` 通常只在“Developer PowerShell for VS 2022”或已经加载 Visual Studio 环境变量的终端中可用：

```powershell
cl
```

看到“无法将 `cmake` 识别为 cmdlet”时，通常是 CMake 未安装，或者安装目录没有加入 `PATH`。修改 `PATH` 后需要重新打开 PowerShell。看到“找不到 `CMAKE_CXX_COMPILER`”时，则通常是 C++ 编译器未安装、终端环境未加载，或者选择的生成器与已安装的工具链不匹配。

### 3. 第一个 `CMakeLists.txt`

先从单个可执行程序开始。目录结构如下：

```text
hello-cmake/
├─ CMakeLists.txt
└─ main.cpp
```

`main.cpp`：

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, CMake!\n";
    return 0;
}
```

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.21)

# project 会声明项目名称，并启用 C++ 语言。
project(hello_cmake LANGUAGES CXX)

# add_executable 创建一个名为 hello 的可执行目标。
add_executable(hello main.cpp)

# 只要求 hello 目标使用 C++20，不污染其他目标。
target_compile_features(hello PRIVATE cxx_std_20)
```

逐行理解：

- `cmake_minimum_required` 声明项目需要的最低 CMake 版本，并让 CMake 使用相应的策略行为。本章使用 3.21，是因为 Windows 示例使用了 Visual Studio 17 2022 生成器。
- `project` 设置项目名称，并告诉 CMake 这个项目需要 C++ 编译器。`LANGUAGES CXX` 可以避免 CMake 额外探测 C 编译器。
- `add_executable` 创建一个目标。目标可以是可执行程序、静态库、动态库，也可以是只提供头文件的接口库。
- `target_compile_features` 只给指定目标设置语言能力。相比把编译选项写成全局变量，这种写法更容易维护，也不容易影响第三方库。

这里的关键思想是：**CMake 主要围绕 target（目标）组织项目，而不是围绕一串全局编译命令组织项目。** 后面的库、程序和测试都应该先创建目标，再把源文件、头文件、依赖和选项附加到目标上。

### 4. 源目录、构建目录和四个阶段

建议把源代码和构建产物分开：

```text
hello-cmake/
├─ CMakeLists.txt       # 源目录中的构建描述
├─ main.cpp
└─ build/               # CMake 生成，通常不提交到 Git
```

从项目根目录执行下面的命令：

```powershell
# 1. 配置（configure）：读取 CMakeLists.txt，检测编译器并生成构建文件。
cmake -S . -B build -G "Visual Studio 17 2022" -A x64

# 2. 构建（build）：让生成器真正编译和链接。
cmake --build build --config Debug --parallel

# 3. 运行程序。Visual Studio 是多配置生成器，程序通常在 build\Debug 下。
.\build\Debug\hello.exe
```

这里 `-S` 是 source directory，`-B` 是 binary/build directory。`-G` 指定生成器，`-A x64` 指定 Visual Studio 的目标架构。

如果使用 MinGW 的 Makefiles 生成器，Debug/Release 要在**配置阶段**通过 `CMAKE_BUILD_TYPE` 选择：

```powershell
# MinGW Makefiles 是单配置生成器，每个构建目录只有一个构建类型。
cmake -S . -B build-mingw -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug
cmake --build build-mingw --parallel
.\build-mingw\hello.exe
```

Visual Studio 这类多配置生成器可以在同一个构建目录中保存 Debug、Release 等配置，所以构建时使用 `--config Debug`；MinGW Makefiles 这类单配置生成器在配置时就确定构建类型，因此通常不需要 `--config`。这是初学者最容易混淆的命令差异之一。

完整的 CMake 工作流可以按下面的顺序理解：

1. **配置**：CMake 检查编译器、平台和依赖，并生成构建系统。配置结果保存在构建目录的 `CMakeCache.txt` 中。
2. **构建**：底层构建工具读取生成的规则，执行编译和链接。没有修改的源文件通常会被复用，这就是增量构建。
3. **测试**：CTest 按 `add_test` 注册的测试命令运行程序，并汇总成功或失败结果。CTest 是测试运行器，不是测试断言库。
4. **安装**：`cmake --install` 按 `install` 规则把程序、库和头文件复制到安装目录。

修改 `CMakeLists.txt` 后，通常再次执行配置命令；修改普通 `.cpp` 后，直接执行构建即可。配置、构建和测试是三个不同的动作，不能把 `cmake -S ... -B ...` 当成“已经编译完成”。

### 5. 常用目标命令

#### 5.1 可执行程序和库

```cmake
add_executable(app app/main.cpp)
add_library(core STATIC src/core.cpp)
add_library(shared_core SHARED src/core.cpp)
```

`STATIC` 生成静态库，Windows 下通常是 `.lib`；`SHARED` 生成动态库，Windows 下通常会同时涉及 `.dll` 和导入库。初学阶段可以先使用静态库，等理解链接过程后再学习动态库的导出和运行时搜索路径。

#### 5.2 头文件目录

假设头文件放在 `include/core/core.h`，库的源文件使用 `#include "core/core.h"`，可以这样设置：

```cmake
target_include_directories(core
    PUBLIC
        ${PROJECT_SOURCE_DIR}/include
)
```

`PUBLIC` 表示两件事：`core` 自己编译时需要这个目录；链接 `core` 的其他目标也会继承这个目录。这样应用程序不需要再次手写同一条 `-I` 或 `/I` 参数。

#### 5.3 链接目标

```cmake
target_link_libraries(app PRIVATE core)
```

这表示 `app` 依赖 `core`，构建 `app` 前要先构建 `core`，链接时也要把 `core` 加入链接命令。传入的是 CMake 目标名，不是手写的 `core.lib` 或 `libcore.a` 文件名，CMake 会根据平台和配置选择正确的文件。

#### 5.4 `PRIVATE`、`PUBLIC` 和 `INTERFACE`

这三个关键字描述的是“依赖是否继续传递”：

| 关键字 | 当前目标使用 | 依赖当前目标的目标使用 |
| --- | --- | --- |
| `PRIVATE` | 是 | 否 |
| `PUBLIC` | 是 | 是 |
| `INTERFACE` | 否 | 是 |

例如，库的公开头文件包含了某个头文件，那么这个头文件所在目录通常应该是 `PUBLIC`；只在库的 `.cpp` 中使用的实现细节通常是 `PRIVATE`。`INTERFACE` 常用于只有头文件、没有需要编译的源文件的库：

```cmake
add_library(config INTERFACE)
target_include_directories(config INTERFACE
    ${PROJECT_SOURCE_DIR}/include
)
target_compile_features(config INTERFACE cxx_std_20)
```

不要把 `PUBLIC` 理解为“公开给用户访问的函数”，也不要把它和 C++ 的 `public:` 访问控制混淆。这里讨论的是构建属性是否沿依赖关系传播。

#### 5.5 编译特性、宏和警告

优先使用目标级命令：

```cmake
target_compile_features(app PRIVATE cxx_std_20)
target_compile_definitions(app PRIVATE APP_VERSION=1)

if(MSVC)
    target_compile_options(app PRIVATE /W4 /permissive-)
else()
    target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)
endif()
```

`/W4` 是 MSVC 的较高警告级别，`-Wall -Wextra -Wpedantic` 是 GCC/Clang 常用的警告选项。不同编译器的选项不完全相同，所以需要用 `if(MSVC)` 等条件区分。不要直接把某个编译器的参数无条件传给所有平台。

### 6. 选项和测试开关

CMake 可以把项目行为做成配置选项：

```cmake
option(ENABLE_WARNINGS "启用项目警告" ON)

if(ENABLE_WARNINGS)
    # 根据编译器给目标附加不同的警告选项。
endif()
```

配置时可以覆盖默认值：

```powershell
cmake -S . -B build -DENABLE_WARNINGS=OFF
```

测试通常使用 CTest 提供的 `BUILD_TESTING` 选项：

```cmake
include(CTest)

if(BUILD_TESTING)
    add_subdirectory(tests)
endif()
```

关闭测试时：

```powershell
cmake -S . -B build -DBUILD_TESTING=OFF
```

注意，`BUILD_TESTING=OFF` 只表示不把测试目标加入当前构建，不能替代对生产代码的编译检查；它也不会自动删除旧的测试可执行文件。构建目录中残留的文件来自旧配置时，应使用一个新的构建目录，或者确认目录确实是生成目录后再清理。

### 7. 目录和文件组织建议

一个小型项目可以从下面的结构开始：

```text
greeting-demo/
├─ CMakeLists.txt
├─ include/
│  └─ greeting/
│     └─ greeting.h
├─ src/
│  └─ greeting.cpp
├─ app/
│  └─ main.cpp
└─ tests/
   └─ greeting_test.cpp
```

这种布局的意图是把公开头文件放到 `include`，库的实现放到 `src`，应用程序入口和测试程序分别放在自己的目录。目录不是 CMake 的硬性要求，但清晰的布局能让依赖边界更容易被看见。

## 示例与实践

### 1. 完整示例：静态库、应用程序和 CTest

下面的示例不依赖 GoogleTest 等第三方库，适合先理解 CMake 的基本机制。它包含三个目标：

- `greeting`：一个静态库，提供生成问候语的函数。
- `greet`：链接 `greeting` 的应用程序。
- `greet_test`：链接同一个库的测试程序，由 CTest 调用。

#### 1.1 顶层 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.21)

project(greeting_demo
    VERSION 1.0
    LANGUAGES CXX
)

# 提供 BUILD_TESTING 选项，并在开启时启用 CTest。
include(CTest)

add_library(greeting STATIC
    src/greeting.cpp
)

# 公开头文件目录：greeting 自己和所有链接 greeting 的目标都需要它。
target_include_directories(greeting
    PUBLIC
        ${PROJECT_SOURCE_DIR}/include
)

# 库的接口要求 C++20，链接它的目标也会继承这个要求。
target_compile_features(greeting PUBLIC cxx_std_20)

if(MSVC)
    target_compile_options(greeting PRIVATE /W4 /permissive-)
else()
    target_compile_options(greeting PRIVATE -Wall -Wextra -Wpedantic)
endif()

add_executable(greet
    app/main.cpp
)

# CMake 会先构建 greeting，再把正确的库文件链接到 greet。
target_link_libraries(greet PRIVATE greeting)

if(BUILD_TESTING)
    add_executable(greet_test
        tests/greeting_test.cpp
    )

    target_link_libraries(greet_test PRIVATE greeting)

    # COMMAND 后面写目标名时，CTest 会使用构建系统生成的正确路径。
    add_test(NAME greet_test COMMAND greet_test)
endif()

# 下面是可选的安装规则，用于练习 cmake --install。
include(GNUInstallDirs)

install(TARGETS greeting greet
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

#### 1.2 公开头文件 `include/greeting/greeting.h`

```cpp
#pragma once

#include <string>

namespace greeting {

// 根据名字生成一条问候语。
std::string make_message(const std::string& name);

}  // namespace greeting
```

#### 1.3 库实现 `src/greeting.cpp`

```cpp
#include "greeting/greeting.h"

namespace greeting {

std::string make_message(const std::string& name) {
    if (name.empty()) {
        return "Hello, C++!";
    }

    return "Hello, " + name + "!";
}

}  // namespace greeting
```

#### 1.4 应用程序 `app/main.cpp`

```cpp
#include "greeting/greeting.h"

#include <iostream>

int main() {
    std::cout << greeting::make_message("CMake") << '\n';
    return 0;
}
```

#### 1.5 测试程序 `tests/greeting_test.cpp`

```cpp
#include "greeting/greeting.h"

#include <iostream>
#include <string>

namespace {

bool expect_equal(const std::string& actual, const std::string& expected) {
    if (actual == expected) {
        return true;
    }

    std::cerr << "实际值: " << actual << "\n"
              << "期望值: " << expected << "\n";
    return false;
}

}  // namespace

int main() {
    // 返回 0 表示测试程序成功，非 0 表示失败。
    if (!expect_equal(greeting::make_message("CMake"), "Hello, CMake!")) {
        return 1;
    }

    if (!expect_equal(greeting::make_message(""), "Hello, C++!")) {
        return 1;
    }

    return 0;
}
```

这个测试程序故意保持简单：CTest 只负责启动它并读取退出码，真正的断言逻辑由测试程序自己完成。实际项目可以把测试框架接入 CMake，但“创建测试目标、链接被测库、用 `add_test` 注册”这三个步骤不会改变。

### 2. 在 Windows 上配置、构建和测试

把上面的文件按目录结构保存后，在 `greeting-demo` 根目录打开 PowerShell。

#### 2.1 Visual Studio 2022

建议使用“Developer PowerShell for VS 2022”，这样 CMake 更容易找到 MSVC：

```powershell
# 配置：使用 Visual Studio 2022 的 64 位生成器。
cmake -S . -B build-msvc -G "Visual Studio 17 2022" -A x64

# 构建 Debug 配置。
cmake --build build-msvc --config Debug --parallel

# 运行 CTest；-C 必须与构建时的配置一致。
ctest --test-dir build-msvc -C Debug --output-on-failure

# 直接运行应用程序。
.\build-msvc\Debug\greet.exe

# 可选：把 Release 版本安装到当前项目下的 install 目录。
cmake --build build-msvc --config Release --parallel
cmake --install build-msvc --config Release --prefix "$((Get-Location).Path)\install"
```

预期的应用程序输出为：

```text
Hello, CMake!
```

#### 2.2 MinGW

如果 `g++ --version` 可以运行，可以使用 MinGW Makefiles：

```powershell
# 单配置生成器在配置时选择 Debug。
cmake -S . -B build-mingw -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug
cmake --build build-mingw --parallel
ctest --test-dir build-mingw --output-on-failure
.\build-mingw\greet.exe

# 如果想构建 Release，使用另一个构建目录，避免混淆缓存。
cmake -S . -B build-mingw-release -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build build-mingw-release --parallel
```

同一个构建目录不要先用 Visual Studio 生成器，再改用 MinGW 生成器。CMake 会把生成器和编译器写入缓存；需要切换工具链时，应使用新的目录，例如 `build-msvc` 和 `build-mingw`。

### 3. 如何阅读这个示例的依赖关系

可以把示例的构建关系画成：

```text
include/greeting/greeting.h
                ↑
src/greeting.cpp ──> greeting（静态库） <── app/main.cpp ──> greet
                                      ↑
                              tests/greeting_test.cpp ──> greet_test
```

更准确地说，`greet` 和 `greet_test` 都通过 `target_link_libraries(... PRIVATE greeting)` 依赖 `greeting`。因为 `greeting` 的头文件目录是 `PUBLIC`，两个消费者都能找到 `greeting/greeting.h`；如果把它误写成 `PRIVATE`，库自身能编译，但消费者可能出现“找不到头文件”的错误。

修改 `src/greeting.cpp` 时，通常只需要重新编译库，然后重新链接 `greet` 和 `greet_test`。修改 `greet` 的 `main.cpp` 时，不需要重新编译库。这就是按目标组织项目比手写一条巨大编译命令更有价值的地方。

### 4. 常见错误和排查顺序

遇到错误时，先判断它发生在哪个阶段，不要看到红色输出就直接删除所有文件。

#### 4.1 配置阶段失败

**`cmake` 找不到**：确认已安装 CMake，并重新打开终端；也可以检查 `PATH` 是否包含 CMake 的 `bin` 目录。

**找不到 C++ 编译器**：Visual Studio 用户应使用 Developer PowerShell 或安装“使用 C++ 的桌面开发”；MinGW 用户应确认 `g++.exe` 所在目录已经加入 `PATH`，并让生成器与工具链匹配。

**生成器不匹配或出现缓存错误**：不要在同一个构建目录切换 Visual Studio 和 MinGW。优先新建构建目录；如果确认 `build` 只包含可重新生成的文件，也可以在 PowerShell 中清理它：

```powershell
Remove-Item -Recurse -Force .\build
```

这条命令具有破坏性，只应该对确认是 CMake 生成目录的路径使用，源代码不要放在其中。

#### 4.2 编译阶段失败

**找不到头文件**：检查 `target_include_directories` 是否设置在正确的目标上，路径是否指向 `include` 而不是 `include/greeting`；同时检查源代码中的 `#include "greeting/greeting.h"` 是否与目录结构一致。

**C++ 标准相关错误**：不要只在 IDE 的某个配置中设置 C++ 标准。使用 `target_compile_features(target PRIVATE cxx_std_20)`，让 Debug、Release 和不同生成器都得到一致的要求。

**把 MSVC 参数传给 GCC，或反过来**：使用 `if(MSVC)`、`if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")` 等条件区分编译器，避免无条件添加 `/W4` 或 `-Wall`。

#### 4.3 链接阶段失败

**`unresolved external symbol` 或“未定义引用”**：通常表示声明存在但实现没有被链接。检查实现源文件是否列在 `add_library` 或 `add_executable` 中，以及应用程序是否调用了 `target_link_libraries`。

**库目标名称和文件名混淆**：在 CMake 中写 `target_link_libraries(greet PRIVATE greeting)`，不要手写 `greeting.lib`。目标名让 CMake 处理 Debug 后缀、平台差异和库搜索路径。

#### 4.4 测试阶段没有测试

如果 CTest 报告没有发现测试，逐项检查：

1. 是否执行了 `include(CTest)` 或 `enable_testing()`。
2. 是否执行了 `add_test(NAME ... COMMAND ...)`。
3. 测试目标是否受 `if(BUILD_TESTING)` 影响，并且配置时没有使用 `-DBUILD_TESTING=OFF`。
4. 多配置生成器运行 CTest 时是否使用了正确的 `-C Debug` 或 `-C Release`。

### 5. 版本控制和日常习惯

通常提交以下内容：

- `CMakeLists.txt`、源文件、头文件和必要的 CMake 模块。
- 明确记录的最低 CMake 版本、编译器版本和依赖版本。

通常不提交以下内容：

- `build/`、`build-msvc/`、`build-mingw/` 等构建目录。
- `CMakeCache.txt`、`CMakeFiles/`、`.sln`、`.vcxproj` 和编译产生的 `.obj`、`.exe`、`.lib` 文件。

可以在 `.gitignore` 中加入：

```gitignore
build*/
CMakeUserPresets.json
```

`CMakeUserPresets.json` 通常保存个人机器上的路径和偏好，不适合提交；团队共享的配置可以使用 `CMakePresets.json`，但应在项目约定中明确它支持的 CMake 版本和工具链。

### 6. 实践任务

完成完整示例后，按顺序尝试下面的练习：

1. 给 `make_message` 增加一个名字为空白时的行为，并为这个行为补充测试。
2. 为 `greet` 增加一个命令行参数，使用 `greeting::make_message` 输出用户输入的名字。
3. 配置 `-DBUILD_TESTING=OFF`，观察 `greet_test` 不再被构建；然后重新配置并打开它。
4. 分别生成 Debug 和 Release，观察它们可以共存于 Visual Studio 的同一个构建目录中；再用 MinGW 验证单配置生成器需要不同构建目录或重新配置。
5. 故意注释掉 `target_link_libraries(greet PRIVATE greeting)`，记录链接错误；恢复后再故意移除 `target_include_directories`，比较两类错误信息的差异。

每次练习都建议从“修改源文件或 CMakeLists.txt → 重新配置（必要时）→ 构建 → 测试 → 运行”的顺序执行，并把遇到的错误归类为配置、编译、链接或运行期问题。

## 关联
- [[源文件、编译与链接]]
- [[单元测试]]
- [[依赖管理与包管理]]
- [[调试与Sanitizer]]
- [[代码规范与静态分析]]
