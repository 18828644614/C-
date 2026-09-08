---
type: topic
status: draft
created: 2026-09-01
updated: 2026-09-08
tags: [cpp, stl, lambda, 函数对象, 泛型编程]
---

# 函数对象与 Lambda

## 学习目标

- [ ] 理解“可调用对象”的共同概念，知道普通函数、函数指针、函数对象和 Lambda 的关系。
- [ ] 能读懂 Lambda 的捕获列表、参数列表、返回类型和 `mutable`。
- [ ] 能用 Lambda 作为谓词、比较器和转换操作，配合标准算法处理容器。
- [ ] 能判断按值捕获和按引用捕获的差异，避免悬空引用、意外修改和数据竞争。
- [ ] 知道什么时候使用 Lambda，什么时候定义一个可复用的函数对象，什么时候才需要 `std::function`。

## 要点

### 1. 先理解“可调用对象”

在 C++ 中，**可调用对象（callable）** 是指可以使用类似 `object(arguments)` 形式调用的东西。常见种类包括：

| 类型 | 例子 | 特点 |
| --- | --- | --- |
| 普通函数 | `int add(int, int)` | 有名字，适合在多个地方复用 |
| 函数指针 | `int (*operation)(int, int)` | 保存函数地址，可以作为参数传递 |
| 函数对象 | 重载了 `operator()` 的类对象 | 可以保存状态，类型明确 |
| Lambda | `[](int x) { return x * 2; }` | 就地创建一个匿名函数对象 |
| `std::function` | `std::function<int(int)>` | 统一保存多种可调用对象，但有类型擦除成本 |

它们可以被算法以相似的方式使用。算法通常只关心一件事：**“给你一个值，你能不能按照约定被调用？”**

```cpp
#include <functional>
#include <iostream>

int add(int left, int right) {
    return left + right;
}

struct Multiplier {
    int factor;

    int operator()(int value) const {
        // 调用对象时，实际上是调用 object.operator()(value)。
        return value * factor;
    }
};

int main() {
    auto function_pointer = &add;
    Multiplier triple{3};
    auto lambda = [](int value) { return value * 2; };

    std::cout << function_pointer(2, 3) << '\n'; // 普通函数指针：5
    std::cout << triple(4) << '\n';              // 函数对象：12
    std::cout << lambda(5) << '\n';              // Lambda：10

    // std::function 可以保存不同类型的可调用对象，只要调用签名相同。
    std::function<int(int)> operation = lambda;
    std::cout << operation(6) << '\n';            // 12
}
```

这里的 `Multiplier` 是一个最小的函数对象：对象内部保存 `factor`，调用时把输入乘上这个状态。Lambda 其实也会生成一个编译器管理的匿名类对象，所以可以把 Lambda 理解成“更方便地书写函数对象”。

### 2. 函数对象：把行为和状态放在一起

函数对象的关键是重载函数调用运算符 `operator()`。与普通函数相比，它可以在对象中保存状态；与全局变量相比，状态又被封装在对象内部，多个对象之间互不干扰。

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

class GreaterThan {
public:
    explicit GreaterThan(int limit) : limit_(limit) {}

    bool operator()(int value) const {
        return value > limit_;
    }

private:
    int limit_;
};

int main() {
    const std::vector<int> numbers{1, 5, 8, 2, 10};

    // count_if 会把每个元素传给 GreaterThan 对象。
    const auto count = std::count_if(numbers.begin(), numbers.end(),
                                     GreaterThan{5});
    std::cout << count << '\n'; // 2：8 和 10 大于 5
}
```

`operator()` 通常应写成 `const`，表示调用函数对象不会改变它的配置状态。这样一个 `const` 函数对象也可以交给只读算法使用。如果确实需要在调用中累计状态，可以去掉 `const`，但要明确算法是否会复制函数对象，以及状态最终保存在哪个对象中。

例如，下面的计数器能统计调用次数，但不要把它当作算法返回结果的唯一来源：某些算法可能复制谓词对象，复制后每个副本都有自己的计数。

```cpp
struct CallCounter {
    int calls = 0;

    bool operator()(int value) {
        ++calls;
        return value % 2 == 0;
    }
};
```

#### 比较器必须形成“严格弱序”

把函数对象或 Lambda 传给 `std::sort` 时，返回值不是“两个元素是否相等”，而是“左边元素是否应该排在右边元素之前”。例如升序比较器应写成 `left < right`，而不是 `left <= right`。

```cpp
std::sort(numbers.begin(), numbers.end(),
          [](int left, int right) {
              return left < right; // 正确：相等时返回 false
          });
```

比较器需要满足一种稳定的一致关系：如果 `a` 排在 `b` 前面，`b` 就不应又排在 `a` 前面；相等元素之间可以互不优先。新手最容易犯的错误是使用 `<=`，这可能破坏算法的前提并导致未定义行为或不可预测结果。

### 3. Lambda 的基本写法

Lambda 的一般形式是：

```cpp
[捕获列表](参数列表) -> 返回类型 {
    // 函数体
};
```

各部分的作用如下：

- `[]`：捕获列表，说明 Lambda 如何访问外部局部变量；没有外部变量时可以为空。
- `()`：参数列表，和普通函数参数类似；没有参数时可以省略括号。
- `-> 返回类型`：尾置返回类型，通常可以省略，让编译器推导。
- `{}`：函数体；最后的分号属于 Lambda 表达式所在的语句。

```cpp
auto is_even = [](int value) {
    return value % 2 == 0; // 返回 bool，返回类型可以自动推导
};

const bool result = is_even(6); // true
```

Lambda 没有一个可直接书写的类型名，因此初学时通常用 `auto` 保存它：

```cpp
auto twice = [](int value) { return value * 2; };
```

注意：即使两个 Lambda 的代码看起来一模一样，它们的类型也不同，不能直接互相赋值。若确实需要把它们放进同一个变量或容器，可以使用相同类型的函数对象，或者使用 `std::function`（见第 8 节）。

### 4. 捕获列表：Lambda 如何看到外部变量

Lambda 体内可以直接使用参数、局部变量和成员变量。参数由调用者传入；外部局部变量则必须通过捕获列表明确说明：

| 写法 | 含义 | 新手提醒 |
| --- | --- | --- |
| `[]` | 不捕获任何外部局部变量 | 最安全、依赖最少 |
| `[value]` | 按值捕获 `value` 的一份副本 | 之后外部变量改变，不影响这份副本 |
| `[&value]` | 按引用捕获 `value` | Lambda 使用期间，原变量必须仍然存在 |
| `[=]` | 默认按值捕获需要的变量 | 不要无意中捕获过多状态 |
| `[&]` | 默认按引用捕获需要的变量 | 生命周期和并发风险更高 |
| `[&, limit]` | 默认引用捕获，`limit` 例外按值 | 可混合指定 |
| `[=, &total]` | 默认按值捕获，`total` 例外按引用 | 可混合指定 |

```cpp
#include <iostream>

int main() {
    int limit = 10;
    int total = 0;

    auto by_value = [limit]() {
        // 捕获的是 limit 的副本。
        std::cout << limit << '\n';
    };

    auto by_reference = [&total](int value) {
        // 直接修改外部的 total。
        total += value;
    };

    limit = 20;
    by_value();        // 仍输出 10，不受外部 limit 后续赋值影响
    by_reference(3);
    std::cout << total << '\n'; // 3
}
```

#### 按值捕获不等于只读外部变量

按值捕获的意思是“把值复制到闭包对象中”，并不表示原变量是 `const`。Lambda 的调用运算符默认是 `const`，所以**默认不能修改这份内部副本**。如果确实要修改副本，需要加 `mutable`：

```cpp
#include <iostream>

int main() {
    int start = 10;

    auto next_value = [start]() mutable {
        // mutable 允许修改 Lambda 自己保存的 start 副本。
        // 不会修改外部的 start。
        return ++start;
    };

    std::cout << next_value() << '\n'; // 11
    std::cout << next_value() << '\n'; // 12：副本的状态被保留
    std::cout << start << '\n';        // 10：外部变量没有改变
}
```

`mutable` 改变的是闭包对象内部副本的可修改性，不会让按值捕获变成按引用捕获。

#### 初始化捕获：在 Lambda 内创建一个新成员

C++14 的初始化捕获可以在捕获列表中写初始化表达式。它特别适合移动不可复制对象，或给捕获的值改一个更清楚的名字：

```cpp
#include <memory>

int main() {
    auto resource = std::make_unique<int>(42);

    // std::move(resource) 把 unique_ptr 的所有权移动进 Lambda。
    auto use_later = [owned = std::move(resource)] {
        return *owned;
    };

    // resource 现在为空；资源由 use_later 闭包对象拥有。
    return use_later() == 42 ? 0 : 1;
}
```

这是“谁拥有对象”的问题，而不仅是语法问题。按值捕获普通整数只复制一个数；按值捕获 `std::unique_ptr` 需要移动所有权；按引用捕获则不会转移所有权，仍必须保证被引用对象活得足够久。

### 5. 捕获的生命周期：最容易出错的部分

按引用捕获要求原变量在 Lambda **每一次调用时**都还活着。Lambda 被立即调用时通常没有问题；如果 Lambda 要保存起来、返回出去、交给线程或异步任务执行，就必须重新检查生命周期。

```cpp
#include <functional>

std::function<int()> make_bad_callback() {
    int value = 42;

    // 错误：函数返回后 value 已经销毁，返回的 Lambda 保存了悬空引用。
    // return [&value] { return value; };

    // 正确：按值捕获，让闭包对象拥有自己的副本。
    return [value] { return value; };
}
```

下面几条规则可以帮助排查捕获问题：

1. Lambda 只在当前表达式中立即使用时，按引用捕获通常容易控制。
2. Lambda 会跨函数返回、保存到成员、交给线程或异步任务时，优先考虑按值捕获或显式共享所有权。
3. 按值捕获只能解决“变量本身”的生命周期，不会自动延长变量内部指针所指向对象的生命周期。
4. 多线程中按引用捕获还可能造成数据竞争；“变量还活着”不代表“多个线程可以同时安全访问”。

成员函数中使用 `[this]` 时，捕获的是 `this` 指针的副本，不是整个对象的独立副本。若 Lambda 可能在对象销毁后才执行，`[this]` 仍会悬空。C++17 的 `[*this]` 才是复制当前对象，但复制对象是否符合业务语义仍要认真判断。

```cpp
class Printer {
public:
    auto callback() const {
        // 调用 callback() 返回后，Printer 对象必须继续存在，才能安全调用该 Lambda。
        return [this] { return number_; };
    }

private:
    int number_ = 7;
};
```

### 6. 谓词、比较器和算法：Lambda 最常见的用法

**谓词（predicate）** 是返回布尔值、用于判断条件的可调用对象。例如“是否为偶数”“是否超过阈值”“两个元素谁应该排在前面”。标准算法不会猜测你的业务规则，而是把元素交给谓词判断。

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> numbers{1, 2, 3, 4, 5, 6};

    const auto even_count = std::count_if(
        numbers.begin(), numbers.end(),
        [](int value) {
            return value % 2 == 0; // 谓词：偶数返回 true
        });

    const auto first_large = std::find_if(
        numbers.begin(), numbers.end(),
        [](int value) {
            return value > 4;
        });

    std::vector<int> squares;
    squares.reserve(numbers.size());
    std::transform(numbers.begin(), numbers.end(),
                   std::back_inserter(squares),
                   [](int value) {
                       return value * value; // 转换操作：输入一个值，产生一个新值
                   });

    std::cout << even_count << '\n'; // 3
    std::cout << *first_large << '\n'; // 5；这里已知查找成功
    for (int value : squares) {
        std::cout << value << ' '; // 1 4 9 16 25 36
    }
}
```

使用 `find_if` 时必须先判断返回值是否等于 `end()`，不能无条件解引用；上面的固定示例知道一定能找到 5，实际处理外部数据时应写成：

```cpp
if (first_large != numbers.end()) {
    std::cout << *first_large << '\n';
}
```

Lambda 也常用于排序和删除：

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

struct Student {
    std::string name;
    int score;
};

int main() {
    std::vector<Student> students{
        {"Alice", 88}, {"Bob", 95}, {"Cindy", 88}, {"David", 59}
    };

    std::sort(students.begin(), students.end(),
              [](const Student& left, const Student& right) {
                  if (left.score != right.score) {
                      return left.score > right.score; // 分数高的在前
                  }
                  return left.name < right.name;       // 分数相同时按名字升序
              });

    // remove_if 只重新排列元素并返回新的逻辑末尾，erase 才真正缩短 vector。
    const auto new_end = std::remove_if(
        students.begin(), students.end(),
        [](const Student& student) {
            return student.score < 60; // 删除不及格学生
        });
    students.erase(new_end, students.end());

    for (const auto& student : students) {
        std::cout << student.name << ": " << student.score << '\n';
    }
}
```

### 7. 泛型 Lambda：让同一个小函数接受多种类型

C++14 起，Lambda 参数可以写成 `auto`。这会生成一个模板化的调用运算符，使同一个 Lambda 能接受不同类型：

```cpp
#include <iostream>
#include <string>

int main() {
    auto print_twice = [](const auto& value) {
        std::cout << value << ' ' << value << '\n';
    };

    print_twice(3);                    // int
    print_twice(1.5);                  // double
    print_twice(std::string{"hi"});   // std::string
}
```

泛型 Lambda 并不是“运行时什么类型都接受”。每次调用仍会在编译期生成一个具体版本，因此传入的类型必须支持函数体中的操作。例如上例要求 `value` 能被 `std::cout` 输出。

如果只接受一种类型，显式写参数类型通常更容易读；如果确实希望泛化，再使用 `auto`，并确保函数体对所有预期类型都成立。

### 8. `std::function`：便利的统一容器，不是默认选择

`std::function<返回类型(参数类型...)>` 可以保存普通函数、函数指针、函数对象和 Lambda，只要它们的调用签名兼容：

```cpp
#include <functional>
#include <iostream>

int apply(int value, const std::function<int(int)>& operation) {
    return operation(value);
}

int main() {
    auto add_one = [](int value) { return value + 1; };
    std::cout << apply(4, add_one) << '\n'; // 5
}
```

但 `std::function` 需要做**类型擦除**：它把具体可调用对象隐藏在统一接口后面，可能有额外的间接调用和存储开销。选择时可以按下面的顺序思考：

- 算法参数：优先直接使用模板参数，让编译器保留具体类型，通常写成 `template <typename Predicate>`。
- 只在当前位置使用一次：直接写 Lambda。
- 需要复用、保存状态或多个重载：定义命名函数对象。
- 需要把不同类型的回调存入同一个成员、容器或接口：再考虑 `std::function`。

标准算法本身通常使用模板，而不是 `std::function`，原因就是保留了可调用对象的具体类型，避免不必要的类型擦除。

### 9. Lambda、函数对象和普通函数如何选择

| 需求 | 建议 |
| --- | --- |
| 一次性的短规则，紧挨算法使用 | Lambda |
| 规则较长，需要名字和单元测试 | 普通函数或命名函数对象 |
| 规则需要保存阈值、计数器等状态 | 函数对象或带捕获的 Lambda |
| 多个地方要使用同一规则，且类型不重要 | Lambda 工厂或 `std::function` |
| 需要高性能泛型调用，类型在编译期已知 | 模板参数接收可调用对象 |
| 需要运行时替换不同回调 | `std::function` 或函数指针 |

“更现代”不等于“到处写 Lambda”。如果 Lambda 已经长到需要滚动屏幕，通常应提取成有名字的函数对象或普通函数，让调用位置只表达意图。

## 常见错误

1. **把 `[&]` 当成默认安全写法**：短小的立即调用尚可；保存、返回、异步执行时应优先检查生命周期。
2. **误以为按值捕获会同步外部变量**：按值捕获的是副本，外部变量后续变化不会自动传入。
3. **忘记 `mutable` 的含义**：它只允许修改 Lambda 内部的按值捕获副本，不会修改外部变量。
4. **用 `<=` 写排序比较器**：比较器应描述“严格在前”，相等时通常返回 `false`。
5. **无条件解引用 `find_if` 的结果**：没有找到时返回 `end()`，解引用属于未定义行为。
6. **以为 `remove_if` 会缩短容器**：它只移动元素，`vector` 仍需调用 `erase`。
7. **在回调里捕获 `this` 却没确认对象寿命**：对象销毁后调用回调会访问悬空指针。
8. **为了“统一类型”过早使用 `std::function`**：它方便，但可能隐藏类型和增加调用成本。

## 示例与实践

### 综合示例：筛选、排序和统计订单

下面的程序把一个 Lambda 同时用于筛选和排序，并演示按值捕获配置、按引用累计结果。阅读时可以先看每个 Lambda 的输入和返回值，再看它被哪个算法调用。

```cpp
#include <algorithm>
#include <iomanip>
#include <iostream>
#include <iterator>
#include <string>
#include <vector>

struct Order {
    std::string customer;
    double amount;
    bool paid;
};

int main() {
    const std::vector<Order> orders{
        {"Alice", 120.0, true},
        {"Bob", 80.0, false},
        {"Cindy", 240.0, true},
        {"David", 150.0, true},
    };

    const double minimum_amount = 100.0;
    std::vector<Order> eligible;

    std::copy_if(orders.begin(), orders.end(), std::back_inserter(eligible),
                 [minimum_amount](const Order& order) {
                     // 按值捕获配置：筛选过程中只读取阈值。
                     return order.paid && order.amount >= minimum_amount;
                 });

    std::sort(eligible.begin(), eligible.end(),
              [](const Order& left, const Order& right) {
                  return left.amount > right.amount; // 金额从高到低
              });

    double total = 0.0;
    std::for_each(eligible.begin(), eligible.end(),
                  [&total](const Order& order) {
                      // 按引用捕获累计总额；total 必须活到算法调用结束。
                      total += order.amount;
                  });

    std::cout << std::fixed << std::setprecision(2);
    std::cout << "合格订单数: " << eligible.size() << '\n';
    std::cout << "合计金额: " << total << '\n';
    for (const auto& order : eligible) {
        std::cout << order.customer << ": " << order.amount << '\n';
    }
}
```

### 动手练习

1. 给 `std::vector<int>` 写一个 Lambda，统计大于用户输入阈值的元素数量。
2. 使用 `std::transform` 把一组摄氏温度转换为华氏温度，并把结果保存到新 `vector`。
3. 定义一个 `ByLength` 函数对象，按字符串长度排序；长度相同时按字典序排序。
4. 写一个返回回调的函数：先用引用捕获局部变量，解释为什么危险，再改成按值捕获或初始化捕获。
5. 比较直接把 Lambda 作为模板参数传入函数，与把 Lambda 包装成 `std::function` 后传入的接口差异。

## 关联

- [[算法]]
- [[迭代器]]
- [[顺序容器]]
- [[关联容器]]
- [[左值、右值与移动语义]]
- [[线程与任务]]
- [[C++11到C++23]]
