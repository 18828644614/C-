---
type: topic
status: draft
created: 2026-09-01
updated: 2026-09-09
tags: [cpp, 类型推导, auto, decltype]
---

# auto、decltype与类型推导

## 学习目标

- [ ] 理解 `auto` 如何根据初始化表达式推导类型。
- [ ] 区分值、引用、顶层 `const` 和底层 `const` 在推导中的差异。
- [ ] 掌握 `decltype` 的三条核心规则，尤其是 `x` 与 `(x)` 的区别。
- [ ] 理解 `decltype(auto)`、函数模板推导和转发引用的基本用法。
- [ ] 能根据对象生命周期和是否需要修改原对象，选择值、引用或常量引用。

## 1. 为什么需要类型推导

C++ 是静态类型语言：编译器在编译时必须知道每个表达式和变量的类型。传统写法需要把类型完整写出来：

~~~cpp
std::vector<std::string>::const_iterator it = names.cbegin();
~~~

这段代码没有错，但类型名很长，而且左边和右边重复表达了同一件事。C++11 引入 `auto` 后，可以让编译器根据右侧初始值推导变量类型：

~~~cpp
auto it = names.cbegin(); // it 的类型由 cbegin() 的返回值决定
~~~

`auto` 不是“动态类型”。变量一旦完成推导，类型就固定了；它也不会让 C++ 变成运行时随意改变类型的语言：

~~~cpp
auto count{10};
// count = "ten"; // 错误：count 已经是 int，不能改成字符串
~~~

可以把 `auto` 理解成“请编译器替我填写这里的类型”，而不是“忽略类型”。

## 2. `auto`：从初始化表达式推导类型

### 2.1 基本规则：必须有初始化表达式

声明 `auto` 变量时必须同时提供初始值，因为没有右侧表达式就没有推导依据：

~~~cpp
auto number{42};       // 推导为 int
auto price{19.9};      // 推导为 double
auto name{"Alice"};    // 推导为 const char*（字符串字面量会退化为指针）

// auto value;          // 错误：没有初始化表达式，无法推导类型
~~~

推导只发生一次。后面给变量赋值时，右侧值必须能转换成已经推导出的类型。

### 2.2 复制式 `auto` 会忽略顶层引用和顶层 `const`

最常见的写法 `auto x = expression`，可以先按“像把表达式的值复制一份”来理解。它会忽略表达式本身的引用属性，以及对象类型最外层的 `const`（称为**顶层 `const`**）：

~~~cpp
int value{10};
const int limit{100};
int& alias{value};
const int& read_only{limit};

auto a = value;      // int：复制 value
auto b = limit;      // int：顶层 const 被忽略，b 可以修改
auto c = alias;      // int：引用被“取出”，c 是独立副本
auto d = read_only;  // int：同样是独立副本

b = 200;             // 合法，不会修改 limit
c = 20;              // 合法，不会修改 value
~~~

如果 `const` 出现在指针指向的对象上，它属于**底层 `const`**，不会被随意丢掉：

~~~cpp
int number{10};
const int* pointer{&number}; // 指针指向 const int

auto copy = pointer;         // const int*：指向对象的 const 被保留
// *copy = 20;               // 错误：不能通过 copy 修改 number
copy = nullptr;              // 合法：指针本身不是 const
~~~

可用 `static_assert` 在编译期验证推导结果：

~~~cpp
#include <type_traits>

int value{0};
const int limit{0};
int& alias{value};

static_assert(std::is_same_v<decltype(value), int>);
static_assert(std::is_same_v<decltype(limit), const int>);

auto a = value;
auto b = limit;
auto c = alias;
static_assert(std::is_same_v<decltype(a), int>);
static_assert(std::is_same_v<decltype(b), int>);
static_assert(std::is_same_v<decltype(c), int>);
~~~

### 2.3 想保留引用或 `const`，要明确写出来

| 写法 | 推导后的大致含义 | 常见用途 |
| --- | --- | --- |
| `auto x = expr` | 得到一个值（通常是副本） | 独立保存结果、避免依赖原对象 |
| `auto& x = expr` | 左值引用，保留被引用对象的 `const` | 修改原对象，或避免复制 |
| `const auto& x = expr` | 只读引用，也可绑定临时对象 | 只读访问大对象、函数参数 |
| `auto&& x = expr` | 根据值类别推导的转发引用 | 泛型代码；初学时谨慎使用 |

~~~cpp
std::string text{"hello"};
const std::string title{"C++"};

auto copy = text;                    // std::string，修改 copy 不影响 text
auto& writable = text;               // std::string&，修改 writable 就是修改 text
const auto& view = text;             // const std::string&，只能读取 text
const auto& temporary = std::string{"temporary"};
// temporary 的生命周期会延长到当前作用域结束

// auto& bad = std::string{"temporary"};
// 错误：普通左值引用不能绑定到临时对象
~~~

`auto&` 会保留 `const`，不能借此绕过只读限制：

~~~cpp
const int limit{100};
auto& same_limit = limit; // 推导为 const int&，而不是 int&
// same_limit = 200;       // 错误
~~~

### 2.4 `auto&&` 与引用折叠

当 `auto&&` 用变量初始化时，它是一个**转发引用**（也叫万能引用）：

- 右侧是左值时，`auto` 推导为 `T&`，最终得到 `T&`；
- 右侧是右值时，`auto` 推导为 `T`，最终得到 `T&&`。

~~~cpp
int value{10};

auto&& from_lvalue = value;        // auto 推导为 int&，结果是 int&
auto&& from_rvalue = 42;            // auto 推导为 int，结果是 int&&
const int limit{100};
auto&& from_const = limit;          // auto 推导为 const int&，结果是 const int&

from_lvalue = 20;                   // 修改 value
// from_const = 200;                // 错误：保留了 const
~~~

背后的规则叫**引用折叠**：

| 组合 | 折叠结果 |
| --- | --- |
| `T&` + `&` | `T&` |
| `T&` + `&&` | `T&` |
| `T&&` + `&` | `T&` |
| `T&&` + `&&` | `T&&` |

实际编写普通业务代码时，优先使用意图更清楚的 `auto`、`auto&` 或 `const auto&`；`auto&&` 主要在泛型代码中出现。

### 2.5 数组、函数和花括号的特殊情况

复制式 `auto` 遵循许多“按值传参”的规则：数组和函数会退化为指针。使用引用才能保留真实类型：

~~~cpp
int numbers[3]{1, 2, 3};

auto p = numbers;                  // int*：数组退化为指向首元素的指针
auto& array_ref = numbers;         // int (&)[3]：保留数组类型和长度

void print();
auto function_ptr = print;         // void (*)()：函数指针
auto& function_ref = print;        // void (&)()：函数引用
~~~

花括号初始化也有容易混淆的规则：

~~~cpp
auto a{42};       // C++11 起推导为 int
auto b = {1, 2};  // 推导为 std::initializer_list<int>
// auto c{1, 2};  // 错误：直接列表初始化要求只能有一个元素
~~~

多个值时要意识到 `auto x = { ... }` 得到的是初始化列表，而不是数组或 `std::vector`。

### 2.6 `auto` 在实际代码中的常见位置

~~~cpp
#include <map>
#include <string>
#include <vector>

std::vector<std::string> names{"小王", "小李"};

for (const auto& name : names) {
    // 只读遍历：不复制字符串，也不能修改容器中的元素
}

std::map<std::string, int> scores{{"数学", 90}, {"英语", 95}};
for (auto& [subject, score] : scores) {
    // auto& 绑定到 map 中的真实元素，可以修改 value
    ++score;
    // subject 是 const key，不能修改
}

auto iterator = names.begin(); // 迭代器类型很长时，auto 更清晰
~~~

不要为了“现代”而到处使用 `auto`。当类型本身能表达重要意图时，显式类型往往更易读，例如 `std::chrono::milliseconds timeout{100};`。

## 3. `decltype`：查询表达式的类型

`decltype(expr)` 不会计算 `expr`，只在编译期询问“这个表达式的类型是什么？”：

~~~cpp
int make_value();
decltype(make_value()) number{}; // number 是 int，但没有调用 make_value()
~~~

### 3.1 先认识值类别：左值、将亡值和纯右值

`decltype` 的规则会用到“左值、将亡值、纯右值”这些术语。初学阶段可以先这样理解：

| 值类别 | 直观含义 | 例子 |
| --- | --- | --- |
| 左值（lvalue） | 有身份、能通过名字或地址找到的对象 | `value`、`array[0]` |
| 将亡值（xvalue） | 仍有身份，但资源可以被复用的对象 | `std::move(value)` |
| 纯右值（prvalue） | 通常是计算结果或临时对象 | `value + 1`、`42`、`std::string{"x"}` |

一个很重要的细节是：**有名字的变量在表达式中通常是左值**，即使它的声明类型是右值引用：

~~~cpp
int&& reference = 42;
// reference 的声明类型是 int&&，但表达式 `reference` 是左值，因为它有名字和稳定身份。
~~~

这正是完美转发需要 `std::forward` 的原因，后面会再次看到。

### 3.2 `decltype` 的三条核心规则

按下面顺序判断：

1. 如果 `expr` 是**不加括号的变量名**、函数名或成员访问，结果是该实体声明时的类型。
2. 否则，如果 `expr` 是左值，结果是 `T&`。
3. 否则，如果 `expr` 是将亡值（xvalue），结果是 `T&&`。
4. 否则，结果是 `T`（通常对应纯右值 prvalue）。

~~~cpp
int value{10};
const int limit{100};

decltype(value) a = value;        // int：变量的声明类型
decltype(limit) b = limit;        // const int：保留声明时的 const
decltype((value)) c = value;      // int&：括号表达式是左值
decltype(value + 1) d = 0;        // int：加法结果是纯右值
// decltype(std::move(value)) e = value; // e 的类型是 int&&
~~~

最容易混淆的是 `decltype(value)` 和 `decltype((value))`：不加括号时是“查询变量的声明类型”；加括号后是普通左值表达式，所以结果是 `int&`。

~~~cpp
int value{10};
decltype(value) first = value;     // int
decltype((value)) second = value;  // int&，second 修改会影响 value
second = 20;
~~~

对于 `object.member`，未加括号时也有特殊规则；加括号后按值类别处理：

~~~cpp
struct Point { int x; };
Point point{1};

decltype(point.x) a = 0;       // int：成员 x 的声明类型
decltype((point.x)) b = point.x; // int&：括号表达式是左值
b = 2;                          // point.x 现在是 2
~~~

### 3.3 `decltype` 与 `auto` 的分工

- `auto`：根据值推导一个新变量通常是什么类型；
- `decltype`：精确查询表达式的类型，包括引用和 `const`；
- `decltype(auto)`：采用 `decltype` 的完整规则进行推导。

## 4. `decltype(auto)`：保留完整类型

~~~cpp
int value{10};

auto copy = (value);             // int：auto 按值推导
decltype(auto) alias = (value);  // int&：decltype((value)) 得到引用

alias = 20;                      // value 也变成 20
~~~

函数返回值是常见用途：

~~~cpp
int& first_element(std::vector<int>& values) {
    return values[0];
}

auto get_first(std::vector<int>& values) {
    return first_element(values); // 返回 int：auto 返回类型会复制值
}

decltype(auto) get_first_ref(std::vector<int>& values) {
    return first_element(values); // 返回 int&：保留引用
}
~~~

但它也更容易制造悬空引用：

~~~cpp
decltype(auto) dangerous() {
    int local{42};
    return (local); // 推导为 int&；函数返回后 local 已销毁，引用悬空
}

auto safe() {
    int local{42};
    return local;   // 推导为 int，返回一个值
}
~~~

如果函数返回局部对象，通常使用普通 `auto` 按值返回；只有确认引用指向的对象会继续存在时，才使用 `decltype(auto)`。

## 5. 函数模板中的类型推导

~~~cpp
template <typename T>
void by_value(T value);       // 按值：复制，顶层 const 和引用通常被忽略

template <typename T>
void by_reference(T& value);  // 左值引用：保留 const，不能接收临时对象

template <typename T>
void by_forward(T&& value);   // 转发引用：既可接收左值，也可接收右值
~~~

~~~cpp
int value{1};
const int limit{2};

by_value(value);       // T 通常是 int
by_value(limit);       // T 通常仍是 int，函数内得到副本
by_reference(value);   // T 是 int，参数类型为 int&
by_reference(limit);   // T 是 const int，参数类型为 const int&
// by_reference(3);    // 错误：临时对象不能绑定到普通 T&

by_forward(value);     // T 推导为 int&，参数折叠为 int&
by_forward(3);         // T 推导为 int，参数类型为 int&&
~~~

### 5.1 完美转发：为什么需要 `std::forward`

函数参数即使声明为 `T&&`，在函数体内只要它有名字，它就是左值表达式。若要把调用者传入时的左值/右值属性继续传给下一个函数，需要使用 `std::forward<T>`：

~~~cpp
#include <iostream>
#include <string>
#include <utility>

void consume(const std::string& text) {
    std::cout << "读取：" << text << '\\n';
}

void consume(std::string&& text) {
    std::cout << "接收临时字符串：" << text << '\\n';
}

template <typename T>
void relay(T&& value) {
    consume(value);                  // value 有名字，是左值，总会调用 const& 重载
    consume(std::forward<T>(value)); // 恢复调用者的值类别
}
~~~

`std::move` 和 `std::forward` 主要是类型转换工具，真正执行移动的是移动构造函数或移动赋值运算符。相关内容见[[左值、右值与移动语义]]。

## 6. 常见误区与排查方法

| 现象 | 原因 | 建议 |
| --- | --- | --- |
| `auto` 变量修改后原对象没变 | `auto x = expr` 通常创建副本 | 需要修改原对象时使用 `auto&` |
| `auto&` 绑定失败 | 普通左值引用不能绑定临时对象 | 保存副本用 `auto`，只读临时对象用 `const auto&` |
| `decltype(x)` 与 `decltype((x))` 不同 | 未加括号的变量名有特殊规则 | 先判断是否是未加括号的实体名 |
| `auto` 推导出 `std::initializer_list` | 使用了 `auto x = {a, b}` | 若想要容器，直接写 `std::vector<T>{a, b}` |
| 返回值从引用变成值 | 普通 `auto` 返回类型会按值返回 | 需要保留引用时考虑 `decltype(auto)`，并检查生命周期 |
| `decltype(auto)` 返回局部引用 | `return (local)` 推导出 `T&` | 局部对象一般按值返回 |
| 模板转发后总调用左值重载 | 有名字的参数在函数体内是左值 | 使用 `std::forward<T>(value)` |

排查类型问题时，依次检查：值类别；声明形式；顶层/底层 `const`；是否需要副本；引用的生命周期。

## 7. 可运行综合示例

下面示例使用 C++17，演示 `auto` 的值/引用差异、`decltype` 的括号规则、`decltype(auto)` 返回引用，以及转发引用。

~~~cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>
#include <vector>

int& first_ref(std::vector<int>& values) { return values[0]; }

auto first_copy(std::vector<int>& values) {
    return first_ref(values); // auto 按值返回 int
}

decltype(auto) first_reference(std::vector<int>& values) {
    return first_ref(values); // decltype(auto) 保留 int&
}

void print_kind(const std::string& text) {
    std::cout << "左值/只读重载：" << text << '\\n';
}

void print_kind(std::string&& text) {
    std::cout << "右值重载：" << text << '\\n';
}

template <typename T>
void forward_demo(T&& text) {
    print_kind(text);                  // 有名字的 text 是左值
    print_kind(std::forward<T>(text)); // 恢复传入时的值类别
}

int main() {
    int value{10};
    const int limit{100};

    auto copy = limit;                 // int，复制式 auto 忽略顶层 const
    auto& reference = value;           // int&，与 value 是同一个对象
    const auto& read_only = limit;     // const int&，只读引用

    static_assert(std::is_same_v<decltype(copy), int>);
    static_assert(std::is_same_v<decltype(reference), int&>);
    static_assert(std::is_same_v<decltype(read_only), const int&>);
    static_assert(std::is_same_v<decltype(value), int>);
    static_assert(std::is_same_v<decltype((value)), int&>);

    copy = 200;                        // 只修改副本
    reference = 20;                    // 修改 value
    std::cout << "value = " << value << ", limit = " << limit << '\\n';

    std::vector<int> numbers{1, 2, 3};
    auto copied_first = first_copy(numbers);
    auto& linked_first = first_reference(numbers);
    copied_first = 99;                 // 不影响 numbers[0]
    linked_first = 88;                 // numbers[0] 变成 88
    std::cout << "copied_first = " << copied_first << '\\n';
    std::cout << "numbers[0] = " << numbers[0] << '\\n';

    std::string name{"Alice"};
    forward_demo(name);                // 左值调用
    forward_demo(std::string{"Bob"}); // 右值调用
}
~~~

编译命令：

~~~powershell
g++ -std=c++17 -Wall -Wextra -pedantic auto_decltype_demo.cpp -o auto_decltype_demo.exe
.\auto_decltype_demo.exe

cl /std:c++17 /W4 /EHsc /utf-8 auto_decltype_demo.cpp /Fe:auto_decltype_demo.exe
.\auto_decltype_demo.exe
~~~

## 8. 练习

1. 声明一个 `const int`、一个 `int&` 和一个数组，分别用 `auto`、`auto&` 推导，使用 `static_assert` 验证类型。
2. 解释下面两行代码的差异，并说明修改 `second` 是否会影响 `value`：

   ~~~cpp
   int value{1};
   auto first = (value);
   decltype(auto) second = (value);
   ~~~

3. 把一个返回 `std::vector<int>` 第一个元素的函数分别写成 `auto` 返回和 `decltype(auto)` 返回，观察调用者能否修改原容器。
4. 编写一个 `relay` 模板，分别去掉 `std::forward` 和保留 `std::forward`，比较字符串左值/右值重载的输出。
5. 找出一处 `auto` 产生不必要拷贝的代码，再找出一处引用可能导致生命周期问题的代码，分别改成更安全的写法。

完成后应能回答：

- `auto` 为什么通常会丢失引用和顶层 `const`？
- `auto&`、`const auto&` 和 `auto&&` 分别适合什么场景？
- 为什么 `decltype(x)` 和 `decltype((x))` 可能得到完全不同的类型？
- `decltype(auto)` 为什么既有用又危险？
- 有名字的 `T&&` 参数为什么在函数体内是左值？`std::forward` 解决了什么问题？

## 关联

- [[类型、变量与常量]]
- [[左值、右值与移动语义]]
- [[结构化绑定与初始化]]
- [[模板基础]]
