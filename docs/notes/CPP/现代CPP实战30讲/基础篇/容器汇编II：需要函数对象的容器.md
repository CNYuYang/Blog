# 容器汇编 II：需要函数对象的容器

你好，我是吴咏炜。

上一讲我们学习了 C++ 的序列容器和两个容器适配器，今天我们继续讲完剩下的标准容器（**[1]**）。

## 函数对象及其特化 

在讲容器之前，我们需要首先来讨论一下两个重要的函数对象，**less** 和 **hash**。

我们先看一下 **less**，小于关系。在标准库里，通用的 **less** 大致是这样定义的：

```cpp
template <class T>
struct less
  : binary_function<T, T, bool> {
  bool operator()(const T& x,
                  const T& y) const
  {
    return x < y;
  }
};
```

也就是说，**less** 是一个函数对象，并且是个二元函数，执行对任意类型的值的比较，返回布尔类型。作为函数对象，它定义了函数调用运算符（**operator()**），并且缺省行为是对指定类型的对象进行 **&lt;** 的比较操作。

有点平淡无奇，是吧？原因是因为这个缺省实现在大部分情况下已经够用，我们不太需要去碰它。在需要大小比较的场合，C++ 通常默认会使用 **less**，包括我们今天会讲到的若干容器和排序算法 **sort**。如果我们需要产生相反的顺序的话，则可以使用 **greater**，大于关系。

计算哈希值的函数对象 **hash** 就不一样了。它的目的是把一个某种类型的值转换成一个无符号整数哈希值，类型为 **size_t**。它没有一个可用的默认实现。对于常用的类型，系统提供了需要的特化 **[2]**，类似于：

```cpp
template <class T> struct hash;

template <>
struct hash<int>
  : public unary_function<int, size_t> {
  size_t operator()(int v) const
    noexcept
  {
    return static_cast<size_t>(v);
  }
};
```

这当然是一个极其简单的例子。更复杂的类型，如指针或者 **string** 的特化，都会更复杂。要点是，对于每个类，类的作者都可以提供 **hash** 的特化，使得对于不同的对象值，函数调用运算符都能得到尽可能均匀分布的不同数值。

我们用下面这个例子来加深一下理解：

```cpp
#include <algorithm>   // std::sort
#include <functional>  // std::less/greater/hash
#include <iostream>    // std::cout/endl
#include <string>      // std::string
#include <vector>      // std::vector
#include "output_container.h"

using namespace std;

int main()
{
  //  初始数组
  vector<int> v{13, 6, 4, 11, 29};
  cout << v << endl;

  //  从小到大排序
  sort(v.begin(), v.end());
  cout << v << endl;

  //  从大到小排序
  sort(v.begin(), v.end(),
       greater<int>());
  cout << v << endl;

  cout << hex;

  auto hp = hash<int*>();
  cout << "hash(nullptr)  = "
       << hp(nullptr) << endl;
  cout << "hash(v.data()) = "
       << hp(v.data()) << endl;
  cout << "v.data()       = "
       << static_cast<void*>(v.data())
       << endl;

  auto hs = hash<string>();
  cout << "hash(\"hello\")  = "
       << hs(string("hello")) << endl;
  cout << "hash(\"hellp\")  = "
       << hs(string("hellp")) << endl;
}
```

在 MSVC 下的某次运行结果如下所示：

> { 13, 6, 4, 11, 29 }
> 
> { 4, 6, 11, 13, 29 }
> 
> { 29, 13, 11, 6, 4 }
> 
> hash(nullptr) = a8c7f832281a39c5
> 
> hash(v.data()) = 7a0bdfd7df0923d2
> 
> v.data() = 000001EFFB10EAE0
> 
> hash("hello") = a430d84680aabd0b
> 
> hash("hellp") = a430e54680aad322
> 


可以看到，在这个实现里，空指针的哈希值是一个非零的数值，指针的哈希值也和指针的数值不一样。要注意不同的实现处理的方式会不一样。事实上，我的测试结果是 GCC、Clang 和 MSVC 对常见类型的哈希方式都各有不同。

在上面的例子里，我们同时可以看到，这两个函数对象的值不重要。我们甚至可以认为，每个 **less**（或 **greater** 或 **hash**）对象都是等价的。关键在于其类型。以 **sort** 为例，第三个参数的类型确定了其排序行为。

对于容器也是如此，函数对象的类型确定了容器的行为。

## priority_queue 

**priority_queue** 也是一个容器适配器。上一讲没有和其他容器适配器一起讲的原因就在于它用到了比较函数对象（默认是 **less**）。它和 **stack** 相似，支持 **push**、**pop**、**top** 等有限的操作，但容器内的顺序既不是后进先出，也不是先进先出，而是（部分）排序的结果。在使用缺省的 **less** 作为其 **Compare** 模板参数时，最大的数值会出现在容器的“顶部”。如果需要最小的数值出现在容器顶部，则可以传递 **greater** 作为其 **Compare** 模板参数。

下面的代码可以演示其功能：

```cpp
#include <functional>  // std::greater
#include <iostream>    // std::cout/endl
#include <memory>      // std::pair
#include <queue>       // std::priority_queue
#include <vector>      // std::vector
#include "output_container.h"

using namespace std;

int main()
{
  priority_queue<
    pair<int, int>,
    vector<pair<int, int>>,
    greater<pair<int, int>>>
    q;
  q.push({1, 1});
  q.push({2, 2});
  q.push({0, 3});
  q.push({9, 4});
  while (!q.empty()) {
    cout << q.top() << endl;
    q.pop();
  }
}
```

输出为：

> (0, 3)
> 
> (1, 1)
> 
> (2, 2)
> 
> (9, 4)
> 


## 关联容器 

关联容器有 **set**（集合）、**map**（映射）、**multiset**（多重集）和 **multimap**（多重映射）。跳出 C++ 的语境，**map**（映射）的更常见的名字是关联数组和字典 **[3]**，而在 JSON 里直接被称为对象（object）。在 C++ 外这些容器常常是无序的；在 C++ 里关联容器则被认为是有序的。

我们可以通过以下的 xeus-cling 交互来体会一下。

```cpp
#include <functional>
#include <map>
#include <set>
#include <string>
using namespace std;
```

```cpp
set<int> s{1, 1, 1, 2, 3, 4};
```

```null
s
```

> { 1, 2, 3, 4 }
> 


```cpp
multiset<int, greater<int>> ms{1, 1, 1, 2, 3, 4};
```

```null
ms
```

> { 4, 3, 2, 1, 1, 1 }
> 


```cpp
map<string, int> mp{
  {"one", 1},
  {"two", 2},
  {"three", 3},
  {"four", 4}
};
```

```null
mp
```

> { "four" => 4, "one" => 1, "three" => 3, "two" => 2 }
> 


```javascript
mp.insert({"four", 4});
```

```null
mp
```

> { "four" => 4, "one" => 1, "three" => 3, "two" => 2 }
> 


```javascript
mp.find("four") == mp.end()
```

> false
> 


```javascript
mp.find("five") == mp.end()
```

> (bool) true
> 


```javascript
mp["five"] = 5;
```

```null
mp
```

> { "five" => 5, "four" => 4, "one" => 1, "three" => 3, "two" => 2 }
> 


```cpp
multimap<string, int> mmp{
  {"one", 1},
  {"two", 2},
  {"three", 3},
  {"four", 4}
};
```

```null
mmp
```

> { "four" => 4, "one" => 1, "three" => 3, "two" => 2 }
> 


```javascript
mmp.insert({"four", -4});
```

```null
mmp
```

> { "four" => 4, "four" => -4, "one" => 1, "three" => 3, "two" => 2 }
> 


可以看到，关联容器是一种有序的容器。名字带“multi”的允许键重复，不带的不允许键重复。**set** 和 **multiset** 只能用来存放键，而 **map** 和 **multimap** 则存放一个个键值对。

与序列容器相比，关联容器没有前、后的概念及相关的成员函数，但同样提供 **insert**、**emplace** 等成员函数。此外，关联容器都有 **find**、**lower_bound**、**upper_bound** 等查找函数，结果是一个迭代器：

- **find(k)** 可以找到任何一个等价于查找键 k 的元素（**!(x < k || k < x)**）

- **lower_bound(k)** 找到第一个不小于查找键 k 的元素（**!(x < k)**）

- **upper_bound(k)** 找到第一个大于查找键 k 的元素（**k < x**）

```javascript
mp.find("four")->second
```

> 4
> 


```javascript
mp.lower_bound("four")->second
```

> 4
> 


```javascript
(--mp.upper_bound("four"))->second
```

> 4
> 


```javascript
mmp.lower_bound("four")->second
```

> 4
> 


```javascript
(--mmp.upper_bound("four"))->second
```

> -4
> 


如果你需要在 **multimap** 里精确查找满足某个键的区间的话，建议使用 **equal_range**，可以一次性取得上下界（半开半闭）。如下所示：

```cpp
#include <tuple>
multimap<string, int>::iterator
  lower, upper;
std::tie(lower, upper) =
  mmp.equal_range("four");
```

```javascript
(lower != upper)  //  检测区间非空
```

> true
> 


```shell
lower->second
```

> 4
> 


```sql
(--upper)->second
```

> -4
> 


如果在声明关联容器时没有提供比较类型的参数，缺省使用 **less** 来进行排序。如果键的类型提供了比较算符 **&lt;** 的重载，我们不需要做任何额外的工作。否则，我们就需要对键类型进行 **less** 的特化，或者提供一个其他的函数对象类型。

对于自定义类型，我推荐尽量使用标准的 **less** 实现，通过重载 **&lt;**（及其他标准比较运算符）对该类型的对象进行排序。存储在关联容器中的键一般应满足严格弱序关系（strict weak ordering；**[4]**），即：

- 对于任何该类型的对象 x：**!(x < x)**（非自反）

- 对于任何该类型的对象 x 和 y：如果 **x < y**，则 **!(y < x)**（非对称）

- 对于任何该类型的对象 x、y 和 z：如果 **x < y** 并且 **y < z**，则 **x < z**（传递性）

- 对于任何该类型的对象 x、y 和 z：如果 x 和 y 不可比（**!(x < y)** 并且 **!(y < x)**）并且 y 和 z 不可比，则 x 和 z 不可比（不可比的传递性）

大部分情况下，类型是可以满足这些条件的，不过：

- 如果类型没有一般意义上的大小关系（如复数），我们一定要别扭地定义一个大小关系吗？

- 通过比较来进行查找、插入和删除，复杂度为对数 O(log(n))，有没有达到更好的性能的方法？

## 无序关联容器 

从 C++11 开始，每一个关联容器都有一个对应的无序关联容器，它们是：

- unordered_set

- unordered_map

- unordered_multiset

- unordered_multimap

这些容器和关联容器非常相似，主要的区别就在于它们是“无序”的。这些容器不要求提供一个排序的函数对象，而要求一个可以计算哈希值的函数对象。你当然可以在声明容器对象时手动提供这样一个函数对象类型，但更常见的情况是，我们使用标准的 **hash** 函数对象及其特化。

下面是一个示例（这次我们暂不使用 xeus-cling，因为它在输出复数时有限制，不能显示其数值）：

```cpp
#include <complex>        // std::complex
#include <iostream>       // std::cout/endl
#include <unordered_map>  // std::unordered_map
#include <unordered_set>  // std::unordered_set
#include "output_container.h"

using namespace std;

namespace std {

template <typename T>
struct hash<complex<T>> {
  size_t
  operator()(const complex<T>& v) const
    noexcept
  {
    hash<T> h;
    return h(v.real()) + h(v.imag());
  }
};

}  // namespace std

int main()
{
  unordered_set<int> s{
    1, 1, 2, 3, 5, 8, 13, 21
  };
  cout << s << endl;

  unordered_map<complex<double>,
                double>
    umc{{{1.0, 1.0}, 1.4142},
        {{3.0, 4.0}, 5.0}};
  cout << umc << endl;
}
```

输出可能是（顺序不能保证）：

> { 21, 5, 8, 3, 13, 2, 1 }
> 
> { (3,4) => 5, (1,1) => 1.4142 }
> 


请注意我们在 **std** 名空间中添加了特化，这是少数用户可以向 **std** 名空间添加内容的情况之一。正常情况下，向 **std** 名空间添加声明或定义是禁止的，属于未定义行为。

从实际的工程角度，无序关联容器的主要优点在于其性能。关联容器和 **priority_queue** 的插入和删除操作，以及关联容器的查找操作，其复杂度都是 O(log(n))，而无序关联容器的实现使用哈希表 **[5]**，可以达到平均 O(1)！但这取决于我们是否使用了一个好的哈希函数：在哈希函数选择不当的情况下，无序关联容器的插入、删除、查找性能可能成为最差情况的 O(n)，那就比关联容器糟糕得多了。

## array 

我们讲的最后一个容器是 C 数组的替代品。C 数组在 C++ 里继续存在，主要是为了保留和 C 的向后兼容性。C 数组本身和 C++ 的容器相差是非常大的：

- C 数组没有 **begin** 和 **end** 成员函数（虽然可以使用全局的 **begin** 和 **end** 函数）

- C 数组没有 **size** 成员函数（得用一些模板技巧来获取其长度）

- C 数组作为参数有退化行为，传递给另外一个函数后那个函数不再能获得 C 数组的长度和结束位置

在 C 的年代，大家有时候会定义这样一个宏来获得数组的长度：

```objectivec
#define ARRAY_LEN(a) \
  (sizeof(a) / sizeof((a)[0]))
```

如果在一个函数内部对数组参数使用这个宏，结果肯定是错的。现在 GCC 会友好地发出警告：

```cpp
void test(int a[8])
{
  cout << ARRAY_LEN(a) << endl;
}
```

> warning: sizeof on array function parameter will return size of ‘int *’ instead of ‘int [8]’ [-Wsizeof-array-argument]
> 
>     cout << ARRAY_LEN(a) << endl;
> 


C++17 直接提供了一个 **size** 方法，可以用于提供数组长度，并且在数组退化成指针的情况下会直接失败：

```cpp
#include <iostream>  // std::cout/endl
#include <iterator>  // std::size

void test(int arr[])
{
  //  不能编译
  // std::cout << std::size(arr)
  //           << std::endl;
}

int main()
{
  int arr[] = {1, 2, 3, 4, 5};
  std::cout << "The array length is "
            << std::size(arr)
            << std::endl;
  test(arr);
}
```

此外，C 数组也没有良好的复制行为。你无法用 C 数组作为 **map** 或 **unordered_map** 的键类型。下面的代码演示了失败行为：

```cpp
#include <map>  // std::map

typedef char mykey_t[8];

int main()
{
  std::map<mykey_t, int> mp;
  mykey_t mykey{"hello"};
  mp[mykey] = 5;
  //  轰，大段的编译错误
}
```

如果不用 C 数组的话，我们该用什么来替代呢？

我们有三个可以考虑的选项：

- 如果数组较大的话，应该考虑 **vector**。**vector** 有最大的灵活性和不错的性能。

- 对于字符串数组，当然应该考虑 **string**。

- 如果数组大小固定（C 的数组在 C++ 里本来就是大小固定的）并且较小的话，应该考虑 **array**。**array** 保留了 C 数组在栈上分配的特点，同时，提供了 **begin**、**end**、**size** 等通用成员函数。

**array** 可以避免 C 数组的种种怪异行径。上面的失败代码，如果使用 **array** 的话，稍作改动就可以通过编译：

```cpp
#include <array>     // std::array
#include <iostream>  // std::cout/endl
#include <map>       // std::map
#include "output_container.h"

typedef std::array<char, 8> mykey_t;

int main()
{
  std::map<mykey_t, int> mp;
  mykey_t mykey{"hello"};
  mp[mykey] = 5;  // OK
  std::cout << mp << std::endl;
}
```

输出则是意料之中的：

> { hello => 5 }
> 


## 内容小结 

本讲介绍了 C++ 的两个常用的函数对象，**less** 和 **hash**；然后介绍了用到这两个函数对象的容器适配器、关联容器和无序关联容器；最后，通过例子展示了为什么我们应当避免 C 数组而考虑使用 **array**。通过这两讲，我们已经完整地了解了 C++ 提供的标准容器。

## 课后思考 

请思考一下：

1. 为什么大部分容器都提供了 **begin**、**end** 等方法？

2. 为什么容器没有继承一个公用的基类？

欢迎留言和我交流你的看法。

## <span data-slate-string="true">参考资料</span> 

**[1] cppreference.com, “Containers library”.**[https://en.cppreference.com/w/cpp/container](https://en.cppreference.com/w/cpp/container)

**[1a] cppreference.com, “容器库”.**[https://zh.cppreference.com/w/cpp/container](https://zh.cppreference.com/w/cpp/container)

**[2] cppreference.com, “Explicit (full) template specialization”.**[https://en.cppreference.com/w/cpp/language/template_specialization](https://en.cppreference.com/w/cpp/language/template_specialization)

**[2a] cppreference.com, “显式（全）模板特化”.**[https://zh.cppreference.com/w/cpp/language/template_specialization](https://zh.cppreference.com/w/cpp/language/template_specialization)

**[3] Wikipedia, “Associative array”.**[https://en.wikipedia.org/wiki/Associative_array](https://en.wikipedia.org/wiki/Associative_array)

**[3a] 维基百科, “关联数组”.**[https://zh.wikipedia.org/zh-cn/ 关联数组](https://zh.wikipedia.org/zh-cn/ 关联数组)

**[4] Wikipedia, “Weak ordering”.**[https://en.wikipedia.org/wiki/Weak_ordering](https://en.wikipedia.org/wiki/Weak_ordering)

**[5] Wikipedia, “Hash table”.**[https://en.wikipedia.org/wiki/Hash_table](https://en.wikipedia.org/wiki/Hash_table)

**[5a] 维基百科, “哈希表”.**[https://zh.wikipedia.org/zh-cn/ 哈希表](https://zh.wikipedia.org/zh-cn/ 哈希表)

