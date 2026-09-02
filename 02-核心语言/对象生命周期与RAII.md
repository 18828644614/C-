---
type: topic
status: draft
created: 2026-09-01
updated: 2026-09-02
tags: [cpp, 生命周期, RAII]
---

# 对象生命周期与RAII

## 学习目标
- [ ] 区分作用域、存储期、对象生命周期、资源生命周期和所有权。
- [ ] 判断局部对象、动态对象、静态对象、临时对象和子对象何时构造、何时析构。
- [ ] 根据构造顺序和析构顺序，解释对象离开作用域以及异常发生时的行为。
- [ ] 用 RAII 自动管理文件、互斥锁、动态内存和 Windows `HANDLE` 等资源。
- [ ] 区分拥有资源的对象与只观察资源的裸指针、引用。
- [ ] 选择 `std::unique_ptr`、`std::shared_ptr` 和 `std::weak_ptr`，避免泄漏、重复释放和悬空访问。
- [ ] 理解 Rule of 0、虚析构函数和析构函数不应抛异常等重要规则。
- [ ] 在 Windows 上使用 MSVC 或 MinGW-w64 编译并运行本章示例。

## 要点

### 1. 先建立整体模型：对象、作用域和资源

“生命周期”这个词看起来抽象，其实可以从三个问题开始：

1. 这个对象什么时候出生？
2. 这个对象什么时候离开世界？
3. 对象存在期间，它是否拥有某种必须归还的资源？

#### 1.1 什么是对象生命周期

对象从初始化完成开始，到销毁完成为止，这段时间叫作对象的**生命周期**。生命周期中通常会经历以下阶段：

```text
获得存储空间
    ↓
执行基类和成员的初始化
    ↓
执行构造函数体，对象建立有效状态
    ↓
对象正常使用
    ↓
执行析构函数，释放对象直接管理的资源
    ↓
对象和它占用的存储结束存在
```

初学时容易把下面几个概念混为一谈：

| 概念 | 含义 | 例子 |
| --- | --- | --- |
| 作用域（scope） | 源代码中名字可见的区域 | `if` 代码块、函数体、命名空间 |
| 存储期（storage duration） | 一块存储空间从获得到释放的时间类别 | 自动、动态、静态、线程局部 |
| 对象生命周期 | 某个对象从初始化到销毁的时间 | `std::string name;` 的存在时间 |
| 资源生命周期 | 文件句柄、锁、连接等外部资源从取得到归还的时间 | 文件打开到关闭 |
| 所有权（ownership） | 哪个对象负责在最后使用后释放资源 | `std::unique_ptr` 拥有一个对象 |

作用域和生命周期经常重合，但不是同一个概念。局部变量通常在所在作用域结束时销毁，所以看起来它们是一回事；动态对象却可能没有一个与它对应的名字，必须由 `delete` 或智能指针决定何时销毁。

还要区分“指针变量”和“指针指向的对象”：

~~~cpp
void pointer_demo() {
    int value = 42; // value 是一个局部对象
    int* p = &value; // p 也是局部对象，但它只是保存地址

    // p 和 value 的生命周期都在这个函数结束时结束。
    // p 的销毁不会销毁 value；p 从来没有拥有 value。
}
~~~

如果一个指针指向的对象已经销毁，指针变量本身仍可能存在，但它就成为**悬空指针**。通过悬空指针访问对象属于未定义行为。

#### 1.2 常见存储期和销毁时机

| 对象写法 | 常见存储期 | 生命周期结束时机 |
| --- | --- | --- |
| 函数中的普通局部变量 | 自动存储期 | 离开所在代码块，包括通过异常离开 |
| `new T(...)` 创建的对象 | 动态存储期 | 对应的 `delete` 执行时 |
| 命名空间作用域对象、`static` 局部变量 | 静态存储期 | 程序结束阶段 |
| `thread_local` 对象 | 线程存储期 | 对应线程结束时 |
| 临时对象 | 通常到所在完整表达式结束 | 例如一条语句末尾；某些引用绑定会延长 |
| 类的成员和基类子对象 | 隶属于外层对象 | 外层对象销毁时，按规定顺序销毁 |

例如：

~~~cpp
void lifetime_kinds() {
    int local = 1;              // 自动对象
    int* dynamic = new int(2);  // 动态对象，不会因指针变量离开作用域而自动删除
    delete dynamic;             // 先销毁动态对象，再释放其存储空间

    static int count = 0;       // 静态局部对象：第一次执行到这里时初始化
    ++count;
}
~~~

这段代码中的 `dynamic` 是一个局部指针。函数返回时指针变量会自动销毁，但这不会替我们执行 `delete`；如果漏掉 `delete`，动态对象就泄漏了。因此现代 C++ 通常写成 `std::make_unique<int>(2)`，把动态对象交给 RAII 类型管理。

对 `new` 和 `delete` 可以先记住这条简化规则：`new` 负责取得存储空间并构造对象，`delete` 负责调用析构函数并释放存储空间。数组必须成对使用 `new[]` 和 `delete[]`。实践中则应优先使用 `std::vector`、`std::string` 和智能指针，减少直接写 `new`/`delete` 的机会。

### 2. 对象何时构造、何时析构

#### 2.1 局部对象：离开作用域就析构

代码块使用一对大括号 `{}` 创建作用域。局部对象在声明处构造，在执行流离开作用域时析构；离开方式可以是正常执行到右大括号，也可以是 `return`、`break` 跳出循环，或者异常传播。

~~~cpp
void work() {
    Resource first("first");
    {
        Resource second("second");
        // second 在这里仍然有效
    } // 先析构 second

    // first 仍然有效
} // 再析构 first
~~~

`Resource` 在这里代表任意拥有资源的类型，例如 `std::ofstream` 或 `std::lock_guard`。右大括号不是“建议清理的位置”，而是语言规则明确触发自动对象销毁的位置。

如果函数提前返回：

~~~cpp
int find_value() {
    std::string buffer = "temporary buffer";
    return 0; // 返回前先析构 buffer，然后函数才真正返回
}
~~~

这也是“在函数中创建 RAII 对象，可以保证函数离开时完成清理”的基础。

#### 2.2 动态对象：由 `delete` 决定结束

~~~cpp
void dynamic_object() {
    auto number = std::make_unique<int>(42);
    // number 是局部对象；它拥有的 int 动态对象由 number 管理。
} // 先析构 number，随后 number 自动删除它拥有的 int
~~~

`number` 的生命周期是局部的，但它管理的 `int` 的存储来自动态区。两者结束的动作由 `std::unique_ptr` 的析构函数连接起来，这正是 RAII 的典型应用。

#### 2.3 成员和基类子对象：外层对象负责组织顺序

一个对象内部可能包含基类子对象和成员子对象。它们不是独立的“额外对象”，而是外层对象生命周期的一部分。构造和析构顺序非常固定：

构造时：

1. 虚基类（如果有）；
2. 直接基类，按继承列表顺序；
3. 非静态数据成员，按**类中声明顺序**；
4. 构造函数体。

析构时反过来：

1. 析构函数体先执行；
2. 非静态数据成员按声明顺序的逆序析构；
3. 直接基类按构造顺序的逆序析构；
4. 虚基类按构造顺序的逆序析构。

因此，初始化列表的书写顺序不能改变成员的实际初始化顺序：

~~~cpp
class Example {
public:
    Example()
        : second_(first_), // 看起来先写 second_，实际不会先初始化它
          first_(10) {}

private:
    int first_;
    int second_;
};
~~~

实际顺序仍然是先 `first_`，再 `second_`。应让初始化列表顺序和成员声明顺序一致；这样既清晰，也能避免编译器的初始化顺序警告。需要更完整的构造函数基础时，可参阅[[构造函数与析构函数]]。

同一作用域中，局部对象则按构造顺序的逆序析构：

~~~cpp
Logger logger;       // 先构造
std::ofstream file;   // 后构造
// 离开作用域时先析构 file，再析构 logger
~~~

这个规则很重要：如果 `file` 的清理需要 `logger` 还存在，就必须先声明 `logger`，再声明 `file`。更稳妥的做法是让每个资源包装对象只依赖自己拥有的资源，减少这种顺序依赖。

#### 2.4 异常栈展开：异常也会触发析构

当异常从函数中向外传播时，C++ 会沿着调用栈离开一个个作用域，并析构已经成功构造的局部对象，这个过程叫作**栈展开（stack unwinding）**：

~~~cpp
void parse() {
    std::string input = "data";
    std::vector<int> values{1, 2, 3};

    throw std::runtime_error("parse failed");

    // 抛出异常后不会执行到这里，input 和 values 仍会自动析构。
}
~~~

如果某个类的构造函数抛出异常，说明这个对象没有构造成功，因此不会调用它自身的析构函数；但是已经构造完成的成员和基类子对象会被清理。这就是把资源交给成员对象（如 `std::string`、`std::vector`、`std::unique_ptr`）管理的重要原因。

#### 2.5 临时对象：默认只活到完整表达式末尾

临时对象常由函数返回值或类型转换产生：

~~~cpp
std::string make_name() {
    return "Alice";
} // 返回的字符串可能是临时对象

void use_name() {
    std::cout << make_name() << '\n';
    // 这条完整表达式结束后，临时 string 不再存在。
}
~~~

把临时对象绑定到某些局部引用，可以延长它到引用的生命周期结束：

~~~cpp
const std::string& name = make_name();
// 这里的临时 string 通常会活到 name 所在作用域结束。
~~~

“引用绑定会延长临时对象”有明确的例外，不能把它当成普遍的保活机制。尤其不要从函数返回局部对象或临时对象的引用：

~~~cpp
const std::string& dangerous() {
    return std::string("do not return this reference"); // 返回后引用悬空
}
~~~

另外，`std::string_view` 和普通引用一样不拥有字符串。下面的 `view` 在初始化语句结束后就可能指向已经销毁的临时字符串：

~~~cpp
#include <string>
#include <string_view>

std::string_view view{std::string("temporary")}; // 错误思路：view 不延长临时 string 的生命周期
// 使用 view 可能产生未定义行为
~~~

### 3. RAII：把资源释放绑定到对象析构

RAII 是 **Resource Acquisition Is Initialization** 的缩写，常译为“资源获取即初始化”。它不是一个库函数，而是一种设计思想：

1. 用对象构造表示“取得资源并建立可用状态”；
2. 用析构表示“无论怎样离开作用域，都归还资源”；
3. 让对象的生命周期和它拥有的资源生命周期绑定；
4. 禁止复制或定义移动语义，准确表达资源所有权。

最简单的抽象形式如下：

~~~cpp
class ResourceOwner {
public:
    explicit ResourceOwner(/* 参数 */) {
        // 取得资源；失败时抛出异常，不让对象进入半可用状态
    }

    ~ResourceOwner() noexcept {
        // 释放资源；析构函数通常不应抛异常
    }

    ResourceOwner(const ResourceOwner&) = delete;            // 默认不允许复制所有权
    ResourceOwner& operator=(const ResourceOwner&) = delete;
};
~~~

真实代码中，`ResourceOwner` 往往不直接保存裸资源，而是把资源放进另一个 RAII 成员。例如 `std::ofstream` 管理文件流，`std::vector` 管理动态数组，`std::lock_guard` 管理一次加锁，`std::unique_ptr` 管理一个动态对象。这样可以遵循 **Rule of 0**：类本身不需要手写析构、拷贝和移动操作，让成员类型完成正确清理。

#### 3.1 RAII 为什么能处理异常

手动清理通常要求每个提前退出路径都写一遍：

~~~cpp
// 不推荐：新增一个 return 或 throw 就可能忘记清理
void old_style() {
    Handle handle = acquire_handle();
    if (!handle.is_valid()) {
        return;
    }

    if (something_failed()) {
        release_handle(handle);
        return;
    }

    do_work(handle);
    release_handle(handle);
}
~~~

问题在于 `do_work` 抛异常时，`release_handle` 根本不会执行。RAII 把清理放到对象析构中：

~~~cpp
void modern_style() {
    HandleOwner handle; // 构造时取得资源
    do_work(handle);    // 即使这里抛异常，离开函数前也会析构 handle
}                       // 自动释放资源
~~~

这并不是说异常会“自动清理所有裸资源”。只有已经被 RAII 类型拥有的资源才会这样清理；裸指针、裸 `HANDLE`、手写的整数文件描述符仍然需要明确的拥有者。

#### 3.2 构造失败时如何避免泄漏

构造函数执行到一半抛异常时，自身的析构函数不会运行。因此不要先用裸资源成员保存资源、再期待外层对象析构来清理。更好的结构是让每个资源立即交给一个已经存在的 RAII 成员：

~~~cpp
class Session {
public:
    Session()
        : file_("session.log"), // 如果后面的成员构造失败，file_ 会被析构
          connection_() {}

private:
    FileOwner file_;
    ConnectionOwner connection_;
};
~~~

成员的声明顺序决定清理顺序：如果 `connection_` 依赖 `file_`，声明时应让 `connection_` 在 `file_` 后构造、从而在 `file_` 前析构，或者重新设计依赖关系。资源所有权越清晰，异常路径越容易正确。

### 4. 标准库中的 RAII 工具

C++ 标准库已经把很多常见资源包装好了：

| 类型 | 管理的内容 | 离开作用域时的行为 |
| --- | --- | --- |
| `std::string` | 字符串存储 | 释放内部动态内存 |
| `std::vector<T>` | 连续元素和容量 | 销毁元素并释放存储 |
| `std::ofstream` / `std::ifstream` | 文件流 | 关闭文件流 |
| `std::unique_ptr<T>` | 一个动态对象的唯一所有权 | 调用删除器，默认执行 `delete` |
| `std::shared_ptr<T>` | 动态对象的共享所有权 | 最后一个共享拥有者销毁对象 |
| `std::lock_guard` | 一次互斥锁的加锁状态 | 自动解锁 |
| `std::scoped_lock` | 一个或多个互斥锁 | 自动解锁，适合同时锁多个锁 |

因此，“资源管理”不等于“到处写析构函数”。优先组合这些类型，通常比自己维护计数器、清理标志和异常分支更可靠。

#### 4.1 `std::unique_ptr`：默认优先考虑的动态所有权

`std::unique_ptr<T>` 表示一个动态对象只有一个拥有者。它不能复制，只能移动所有权：

~~~cpp
#include <memory>

void unique_owner() {
    auto first = std::make_unique<int>(42);
    // auto copy = first;              // 错误：unique_ptr 不能复制

    auto second = std::move(first);    // 所有权转移给 second
    // first 仍是一个有效的 unique_ptr，但通常为空，不再拥有 int

    second.reset();                    // 立即删除它拥有的 int
}                                      // 若仍拥有对象，析构时自动删除
~~~

常见操作的含义如下：

- `get()` 只借出地址，不转移所有权；返回的裸指针不能在外部 `delete`。
- `reset()` 释放当前对象，然后可以接管另一个指针。
- `release()` 放弃所有权并返回裸指针；调用者从此必须负责释放它，容易重新引入泄漏，除非确实要把所有权交给某个 C API。
- `std::make_unique<T>(...)` 通常比手写 `std::unique_ptr<T>(new T(...))` 更安全、更简洁。

动态数组通常优先写 `std::vector<T>`；确实需要数组形式时，可以使用 `std::make_unique<T[]>(n)`。

#### 4.2 `shared_ptr` 和 `weak_ptr`：共享不等于随便使用

`std::shared_ptr<T>` 使用引用计数表达“多个对象共同拥有资源”：当最后一个 `shared_ptr` 销毁或重置时，被管理对象才销毁。

~~~cpp
#include <iostream>
#include <memory>

struct Document {
    ~Document() { std::cout << "Document destroyed\n"; }
};

void shared_owner() {
    auto first = std::make_shared<Document>();
    {
        auto second = first; // 引用计数增加
        std::cout << first.use_count() << '\n';
    } // second 销毁，但 Document 仍由 first 拥有
} // first 销毁，引用计数归零，Document 才销毁
~~~

只有在所有权确实是共享的情况下才使用 `shared_ptr`。它会带来控制块和引用计数开销，也不能自动解决被管理对象内部的数据竞争；多个线程同时访问对象本身仍需要同步。

引用计数还可能遇到循环引用：A 拥有 B，B 又用 `shared_ptr` 拥有 A，双方计数都不会归零。表示“返回、父对象、缓存或观察者”等非拥有关系时，应考虑 `std::weak_ptr`：

~~~cpp
#include <memory>

struct Node {
    std::shared_ptr<Node> child; // Node 拥有 child
    std::weak_ptr<Node> parent;   // parent 只是观察者，不增加引用计数
};

void tree_example() {
    auto parent = std::make_shared<Node>();
    auto child = std::make_shared<Node>();
    parent->child = child;
    child->parent = parent;

    if (auto owner = child->parent.lock()) {
        // lock() 成功时，owner 临时取得一份共享所有权
        (void)owner;
    }
}
~~~

本章只需掌握生命周期视角；智能指针的更多选择和接口细节见[[智能指针]]。

#### 4.3 锁也是资源：使用锁守卫

互斥锁的资源不是内存，而是“当前线程拥有进入临界区的许可”。手写 `lock()`/`unlock()` 很容易在 `return` 或异常路径上漏解锁：

~~~cpp
#include <mutex>

std::mutex mutex;
int value = 0;

void add_one() {
    std::lock_guard<std::mutex> lock(mutex); // 构造时加锁
    ++value;
} // lock 析构时自动解锁
~~~

如果临界区中的代码抛出异常，`lock` 依然会析构并解锁。需要同时锁住多个互斥量时，可以使用 `std::scoped_lock lock(mutex_a, mutex_b);`；需要延迟加锁、条件变量或在作用域内转移锁状态时，再考虑 `std::unique_lock`。

### 5. 所有权、观察和生命周期陷阱

在接口设计中，最重要的问题之一是“调用者把资源交给谁”。可以先按下面的约定理解：

| 写法 | 默认含义 | 调用者应该做什么 |
| --- | --- | --- |
| `T&` / `const T&` | 借用并观察一个必须仍然存在的对象 | 不负责释放；保证使用期间对象有效 |
| `T*` | 可选的借用指针，或旧式接口中的输出指针 | 默认不负责释放；必须查文档确认所有权 |
| `std::unique_ptr<T>` | 唯一所有权 | 通过移动转移所有权 |
| `std::shared_ptr<T>` | 共享所有权 | 复制表示增加一个拥有者 |
| `std::weak_ptr<T>` | 对 `shared_ptr` 对象的非拥有观察 | 使用前 `lock()` 检查对象是否仍存在 |

“裸指针一定不拥有资源”不是语言强制规则，而是现代 C++ 的推荐接口约定。老 C API 可能要求调用者 `free` 或 `CloseHandle`，这时要在边界处立刻包装成 RAII 类型，并明确删除器。

常见生命周期错误包括：

#### 5.1 返回局部对象的地址或引用

~~~cpp
const std::string& bad_name() {
    std::string name = "Alice";
    return name; // 错误：函数返回后 name 已析构
}
~~~

应直接按值返回：

~~~cpp
std::string good_name() {
    return "Alice"; // 返回值由调用者接管；通常还有返回值优化
}
~~~

#### 5.2 复制拥有裸指针的类

如果类中有 `T* data_`，编译器默认生成的拷贝构造和拷贝赋值通常只复制地址，而不是复制地址指向的资源。这样两个对象可能在析构时重复释放同一块内存。

优先把成员改成 `std::vector<T>`、`std::string` 或 `std::unique_ptr<T>`，让成员类型定义正确的复制/移动行为。若确实必须直接管理裸资源，就要遵循 Rule of 3/5：

- Rule of 3：析构函数、拷贝构造函数、拷贝赋值运算符需要一起考虑；
- Rule of 5：在此基础上再考虑移动构造函数和移动赋值运算符；
- Rule of 0：使用已有 RAII 成员，因此自己不需要声明这些特殊成员函数。

初学阶段最推荐 Rule of 0；这不是偷懒，而是把复杂且容易出错的资源管理交给经过验证的类型。

#### 5.3 `delete` 配对错误、重复释放和释放后使用

~~~cpp
int* one = new int(1);
delete one;       // 正确配对

int* many = new int[10];
delete[] many;    // 数组必须用 delete[]

// delete[] one;  // 错误：new 和 delete[] 不匹配
// delete one;    // 错误：one 已释放，再次释放
// std::cout << *one; // 错误：释放后使用
~~~

还不能混用不同分配体系：`new` 配 `delete`，`new[]` 配 `delete[]`，`malloc` 配 `free`，Windows `HeapAlloc`/`LocalAlloc` 等 API 则必须使用其规定的释放函数。最好让资源从取得之初就进入正确的 RAII 包装，避免人工记忆配对规则。

#### 5.4 `std::move` 不会自动释放或延长生命周期

`std::move(x)` 只是把 `x` 转换成一个允许移动的表达式，它本身不移动资源，也不延长 `x` 的生命周期。移动由目标类型的移动构造或移动赋值完成；移动后的源对象仍然存在，但通常只保证处于“有效、可析构”的状态。

同样，`get()`、引用和 `string_view` 都只是观察。把它们保存到所有者已经销毁之后再使用，就是生命周期错误。

### 6. 多态对象和虚析构函数

当类要作为多态基类，并且可能通过基类指针销毁派生类对象时，基类析构函数必须是 `virtual`：

~~~cpp
class Base {
public:
    virtual ~Base() = default;
    virtual void run() const = 0;
};

class Derived : public Base {
public:
    void run() const override {}
};

void destroy_polymorphic() {
    std::unique_ptr<Base> object = std::make_unique<Derived>();
    // 离开作用域时先析构 Derived，再析构 Base
}
~~~

如果 `Base::~Base()` 不是虚函数，就通过 `Base*` 执行 `delete` 或让 `std::unique_ptr<Base>` 默认删除一个 `Derived` 对象，行为可能是未定义的。若类完全不用于多态删除，则不必仅为“保险”添加虚析构；可以根据设计选择普通析构函数、`protected` 析构函数，或禁止相应的用法。

### 7. 析构函数、`noexcept` 和异常安全

析构函数应该完成清理而不是报告普通业务错误。析构期间抛异常尤其危险：如果此时已经有另一个异常在传播，两个异常同时存在会导致程序调用 `std::terminate`。因此析构函数通常写成：

~~~cpp
class Connection {
public:
    ~Connection() noexcept {
        // 清理动作应设计为不抛异常
    }
};
~~~

如果关闭文件、回滚事务等操作可能失败，应在显式的 `close()`、`commit()` 或 `flush()` 接口中报告错误；析构函数只做最后的、不能再拖延的清理。不要为了让析构函数“看起来完整”而在其中抛异常。

RAII 通常帮助我们提供三种异常安全保证：

- **无泄漏（基本保证）**：发生异常后资源仍然被释放，对象和程序保持有效状态；
- **强保证**：操作失败时，外部可见状态像什么也没发生过一样；
- **不抛保证**：操作保证不抛异常，例如常见的析构、交换或移动操作。

初学时首先要做到无泄漏，再根据业务需要考虑强保证。把资源放在 RAII 成员中，是实现第一层保证的基础。

### 8. 静态对象与初始化顺序

命名空间作用域的全局对象会在 `main` 前后参与程序初始化和销毁。不同源文件中的全局对象初始化顺序通常不适合拿来建立依赖；如果一个全局对象在初始化时使用另一个源文件中的全局对象，可能遇到**静态初始化顺序问题**。

一种常见的改进方式是使用函数内的局部静态对象：

~~~cpp
Logger& global_logger() {
    static Logger logger; // 第一次调用时初始化，程序结束时自动析构
    return logger;
}
~~~

C++11 起，函数局部静态对象的初始化是线程安全的。它不能解决所有设计问题，但能把初始化时机从“程序启动阶段的隐含顺序”变成“第一次使用时的明确顺序”。更复杂的全局资源仍应通过显式初始化和显式依赖来设计。

### 9. 一套实用判断法

遇到一个需要管理资源的类或函数时，可以依次问：

1. 资源由谁取得？取得失败时对象是否能保持有效状态？
2. 资源由谁拥有？是唯一拥有、共享拥有，还是只观察？
3. 所有退出路径（正常返回、提前 `return`、异常、线程退出）是否都会释放？
4. 复制对象时，资源应该复制、共享，还是禁止复制？
5. 移动对象后，源对象是否仍满足“有效、可析构”？
6. 如果通过基类指针销毁派生对象，基类析构函数是否为虚函数？
7. 清理动作是否可能抛异常？如果可能，是否应改为显式操作报告错误？

如果答案不清楚，先不要把裸资源暴露给调用者，优先寻找标准库容器、智能指针、锁守卫或自定义 RAII 包装器。

## 示例与实践

下面每个示例都是一个独立的完整程序，都包含自己的 `main` 函数。请分别保存为不同的 `.cpp` 文件，不要把它们直接合并编译。

### 1. 完整示例：观察构造、析构、作用域和异常

保存为 `lifetime_demo.cpp`。这个示例使用 ASCII 输出，避免 Windows 终端代码页影响观察结果。

~~~cpp
#include <iostream>
#include <stdexcept>

class Tracker {
public:
    explicit Tracker(const char* name) : name_(name) {
        std::cout << "construct " << name_ << '\n';
    }

    ~Tracker() noexcept {
        std::cout << "destroy  " << name_ << '\n';
    }

private:
    const char* name_;
};

class Box {
public:
    Box() : first_("box.first"), second_("box.second") {
        std::cout << "construct Box body\n";
    }

    ~Box() noexcept {
        // 析构函数体先执行，随后才析构成员 second_ 和 first_
        std::cout << "destroy  Box body\n";
    }

private:
    // 成员的实际构造顺序由声明顺序决定
    Tracker first_;
    Tracker second_;
};

void throw_error() {
    Tracker local("inside throw_error");
    throw std::runtime_error("something failed");
    // 抛出异常后不会执行到这里，但 local 会先析构
}

int main() {
    std::cout << "--- main begins ---\n";
    Tracker first("first local");
    Tracker second("second local");

    {
        Box box;
        std::cout << "--- leave inner scope ---\n";
    } // 先析构 Box，再按逆序析构 second_、first_

    try {
        throw_error();
    } catch (const std::exception& error) {
        std::cout << "caught: " << error.what() << '\n';
    }

    std::cout << "--- main ends ---\n";
} // 先析构 second local，再析构 first local
~~~

在 Visual Studio Developer PowerShell 中：

~~~powershell
cl /std:c++20 /W4 /EHsc /permissive- lifetime_demo.cpp /Fe:lifetime_demo.exe
.\lifetime_demo.exe
~~~

在已配置 MinGW-w64 的 PowerShell 中：

~~~powershell
g++ -std=c++20 -Wall -Wextra -pedantic lifetime_demo.cpp -o lifetime_demo.exe
.\lifetime_demo.exe
~~~

可以重点观察三件事：`Box` 的析构函数体先于成员析构；同一作用域的局部对象按逆序析构；异常离开 `throw_error` 时，`local` 仍然会被析构。

### 2. 完整示例：用 RAII 管理文本文件

保存为 `file_raii_demo.cpp`。程序故意在写文件后抛出异常，用来观察异常路径上文件仍然会被关闭。文件会生成在程序的当前工作目录中。

~~~cpp
#include <fstream>
#include <iostream>
#include <stdexcept>
#include <string>

class TextFile {
public:
    explicit TextFile(const std::string& path) : stream_(path) {
        if (!stream_.is_open()) {
            throw std::runtime_error("cannot open output file");
        }
    }

    // ofstream 本身不可复制，这个包装类也明确禁止复制
    TextFile(const TextFile&) = delete;
    TextFile& operator=(const TextFile&) = delete;

    ~TextFile() noexcept = default; // stream_ 的析构函数会关闭文件

    void write_line(const std::string& text) {
        stream_ << text << '\n';
        if (!stream_) {
            throw std::runtime_error("write failed");
        }
    }

private:
    std::ofstream stream_; // 成员先构造，TextFile 销毁时自动关闭
};

void write_and_fail() {
    TextFile file("raii_text_demo.txt");
    file.write_line("line written before the exception");

    throw std::runtime_error("pretend that later work failed");
    // 离开函数前，file 析构，stream_ 自动关闭
}

int main() {
    try {
        write_and_fail();
    } catch (const std::exception& error) {
        std::cout << "caught: " << error.what() << '\n';
    }

    // 这里重新打开文件，说明上一个 TextFile 已经完成清理
    std::ifstream input("raii_text_demo.txt");
    if (!input) {
        std::cerr << "cannot reopen output file\n";
        return 1;
    }

    std::string line;
    while (std::getline(input, line)) {
        std::cout << "read: " << line << '\n';
    }
}
~~~

编译运行命令：

~~~powershell
# MSVC Developer PowerShell
cl /std:c++20 /W4 /EHsc /permissive- file_raii_demo.cpp /Fe:file_raii_demo.exe
.\file_raii_demo.exe

# MinGW-w64 PowerShell
g++ -std=c++20 -Wall -Wextra -pedantic file_raii_demo.cpp -o file_raii_demo.exe
.\file_raii_demo.exe
~~~

注意：`std::ofstream` 的析构负责关闭文件，但不会把“业务上的写入失败”变成可捕获的异常；需要可靠报告写入错误时，应在写操作后检查流状态，或在设计流对象时显式开启异常。析构函数本身仍应保持不抛异常。

### 3. 完整示例：用 `std::lock_guard` 管理互斥锁

保存为 `lock_raii_demo.cpp`。四个线程共同增加一个计数器。每次进入临界区时构造 `lock_guard`，离开循环体时自动解锁，即使临界区将来加入异常或提前返回，也不会遗留锁。

~~~cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex counter_mutex;
int counter = 0;

void add_many(int times) {
    for (int i = 0; i < times; ++i) {
        std::lock_guard<std::mutex> lock(counter_mutex); // 构造时加锁
        ++counter;
    } // lock 析构时解锁
}

int main() {
    constexpr int thread_count = 4;
    constexpr int increments_per_thread = 10'000;

    std::vector<std::thread> threads;
    threads.reserve(thread_count);

    for (int i = 0; i < thread_count; ++i) {
        threads.emplace_back(add_many, increments_per_thread);
    }

    for (std::thread& thread : threads) {
        thread.join(); // std::thread 不会在析构时自动 join，必须明确处理
    }

    std::cout << "counter = " << counter << '\n';
}
~~~

预期输出为 `counter = 40000`。Windows 下的编译命令：

~~~powershell
# MSVC Developer PowerShell
cl /std:c++20 /W4 /EHsc /permissive- lock_raii_demo.cpp /Fe:lock_raii_demo.exe
.\lock_raii_demo.exe

# MinGW-w64 PowerShell；-pthread 启用线程支持
g++ -std=c++20 -Wall -Wextra -pedantic -pthread lock_raii_demo.cpp -o lock_raii_demo.exe
.\lock_raii_demo.exe
~~~

`std::lock_guard` 不负责创建或销毁 `std::mutex`，它只管理“本次加锁状态”。互斥量本身通常是更外层对象；不要在一个线程仍可能使用互斥量时销毁它。

`std::thread` 也不是一个会自动等待线程结束的 RAII 容器：如果 `std::thread` 对象析构时仍处于 `joinable` 状态，程序会调用 `std::terminate`。C++20 的 `std::jthread` 会在析构时请求停止并等待线程结束，适合需要这种生命周期语义的场景。

### 4. Windows 专属示例：用自定义删除器管理 `HANDLE`

Windows API 的文件句柄不是 C++ 对象，不能用 `delete` 释放，必须调用 `CloseHandle`。可以让 `std::unique_ptr` 使用自定义删除器，把这个平台资源也纳入 RAII。

保存为 `windows_handle_raii.cpp`：

~~~cpp
#define WIN32_LEAN_AND_MEAN
#include <windows.h>

#include <iostream>
#include <memory>

struct HandleCloser {
    // unique_ptr 会使用这个 pointer 类型保存 HANDLE
    using pointer = HANDLE;

    void operator()(HANDLE handle) const noexcept {
        // CreateFile 失败返回 INVALID_HANDLE_VALUE，而不是 nullptr
        if (handle != nullptr && handle != INVALID_HANDLE_VALUE) {
            ::CloseHandle(handle);
        }
    }
};

using unique_handle = std::unique_ptr<void, HandleCloser>;

int main() {
    HANDLE raw_handle = ::CreateFileW(
        L"raii_windows_demo.txt",
        GENERIC_WRITE,
        0,
        nullptr,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        nullptr);

    if (raw_handle == INVALID_HANDLE_VALUE) {
        std::cerr << "CreateFileW failed, error = " << ::GetLastError() << '\n';
        return 1;
    }

    unique_handle file(raw_handle); // 从这里开始，file 拥有 HANDLE

    const char text[] = "written by a Windows HANDLE RAII wrapper\r\n";
    DWORD bytes_written = 0;
    const DWORD bytes_to_write = static_cast<DWORD>(sizeof(text) - 1);

    if (!::WriteFile(
            file.get(),
            text,
            bytes_to_write,
            &bytes_written,
            nullptr) ||
        bytes_written != bytes_to_write) {
        std::cerr << "WriteFile failed, error = " << ::GetLastError() << '\n';
        return 1; // 返回前自动调用删除器，关闭 HANDLE
    }

    std::cout << "file written successfully\n";
} // unique_handle 析构，调用 CloseHandle
~~~

在 Visual Studio Developer PowerShell 中：

~~~powershell
cl /std:c++20 /W4 /EHsc /permissive- windows_handle_raii.cpp /Fe:windows_handle_raii.exe
.\windows_handle_raii.exe
~~~

在 MinGW-w64 的 PowerShell 中：

~~~powershell
g++ -std=c++20 -Wall -Wextra -pedantic windows_handle_raii.cpp -o windows_handle_raii.exe
.\windows_handle_raii.exe
~~~

这个例子有两个值得注意的细节：第一，删除器必须使用 Windows API 规定的 `CloseHandle`，不能写 `delete handle`；第二，`INVALID_HANDLE_VALUE` 是特殊的失败值，不能只用 `nullptr` 判断失败。把裸 `HANDLE` 交给 `unique_handle` 后，后续的 `return`、异常和正常结束都会走同一个关闭路径。

### 5. 练习建议

1. 在 `lifetime_demo.cpp` 中再添加一个嵌套代码块，预测输出后运行验证。
2. 给 `TextFile` 增加 `write_line` 的错误处理，比较“手动 `close()`”和析构自动关闭在异常路径上的差异。
3. 把 `lock_raii_demo.cpp` 中的 `std::lock_guard` 暂时替换成手写 `lock()`/`unlock()`，再在加锁后增加 `throw`，观察为什么不推荐手动管理。
4. 给 Windows HANDLE 包装器增加移动构造和移动赋值，确保移动后源对象不会再次关闭同一个句柄。
5. 写一个返回 `std::vector<int>` 的函数，再和返回局部数组地址的错误写法比较。
6. 创建 `Base` 和 `Derived`，先让 `Base` 的析构函数为 `virtual`，再去掉它并说明为什么通过基类所有权销毁不安全。

完成本章后，应能用自己的话回答：

- 作用域和对象生命周期为什么经常重合，却不是同一个概念？
- 为什么异常发生时，局部 RAII 对象仍能完成清理？
- 为什么成员声明顺序会影响资源释放顺序？
- `unique_ptr`、`shared_ptr`、`weak_ptr` 和裸指针分别表达什么关系？
- Windows `HANDLE` 为什么需要自定义删除器？

## 关联
- [[构造函数与析构函数]]
- [[智能指针]]
- [[异常处理]]
