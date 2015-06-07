---
layout: page
title: RapidJSON 代码剖析（一）：混合任意类型的堆栈
categories:
    - rapidjson
image:
    title: header_rapidjson_stack.jpg
    thumb: header_rapidjson_stack_thumb.jpg
    caption: Image by Gozhanet
    caption_url: https://unsplash.com/gozhanet
---

大家好，这个专栏会分析 RapidJSON中一些有趣的 C++ 代码，希望对读者有所裨益。

## C++ 语法解说

我们先来看一行代码（[document.h][StartArray]）：

~~~cpp
bool StartArray() {
    new (stack_.template Push<ValueType>()) ValueType(kArrayType); // <--
    return true;
}
~~~

或许你会问，这是什么C++语法？

这里其实用了两个可能较少接触的C++特性。第一个是 [placement new][placement new]，第二个是 [template disambiguator][template disambiguator]。

## Placement new

简单来说，placement new 就是不分配内存，由使用者给予内存空间来构建对象。其形式是：

~~~cpp
new (T*) T(...);
~~~

第一个括号中的是给定的指针，它指向足够放下 T 类型的内存空间。而 `T(...)` 则是一个构造函数调用。那么，上面 `StartArary()` 里的代码，分开来写就是：

~~~cpp
bool StartArray() {
    ValueType* v = stack_.template Push<ValueType>(); // (1)
    new (v) ValueType(kArrayType);                    // (2)
    return true;
}
~~~

这么分拆，(2)应该很容易理解吧。那么(1)是什么样的语法？为什么中间会有 `template` 这个关键字？

## template disambiguator

(1)其实只是调用 Stack 类的模板成员函数 `Push()`。如果删去这个 template，代码就显得正常一点：

~~~cpp
    ValueType* v = stack_.Push<ValueType>(); // (1)
~~~

这里 `Push<ValueType>`是一个 dependent name，它依赖于 `ValueType` 的实际类型。这里编译器不能确认 `<` 为小于运算符，还是模板的 `<`。为了避免歧义，需要加入`template` 关键字。这是C++标准的规定，缺少这个 template 关键字 gcc 和 clang 都会报错，而 vc 则会通过（C++标准也容许实现这样的编译器）。和这个语法相近的还有 typename disambiguator。

理解这些语法之后，我们进入核心问题。

## 混合任意类型的堆栈

处理树状的数据结构时，我们经常需要用到堆栈（stack）这种数据结构。C++ 标准库也提供了 [`std::stack`][stdstack] 这个容器。然而，这个模板类容器的实例，只能存放一种类型的对象。在 RapidJSON 的解析过程中，我们希望它能同时存放已解析的 Value 对象，以及 Member 对象（key-value对）。或者我们从另一个角度去想，程序堆栈（program stack）本身就是可储存各种类型数据的堆栈。在 RapidJSON 中的其它地方也有这种需求。

在 [internal/stack.h][stack.h] 中的 Stack 类实现了这个构思，其声明是这样的：

~~~cpp
class Stack {
    Stack(Allocator* allocator, size_t stackCapacity);
    ~Stack();

    void Clear();
    void ShrinkToFit();
    
    template<typename T> T* Push(size_t count = 1);
    template<typename T> T* Pop(size_t count);
    template<typename T> T* Top();
    template<typename T> T* Bottom();

    Allocator& GetAllocator();
    bool Empty() const;
    size_t GetSize();
    size_t GetCapacity();
};
~~~

这个类比较特殊的地方，就是堆栈操作使用模板成员函数，可以压入或弹出不同类型的对象。另外，为了完全防止拷贝构造函数调用的可能性，这些函数都是返回指针。虽然引用也可以，但使用指针在一些应用情况下会更自然。

例如，要压入4个 int，再每次弹出两个：

~~~cpp
Stack s;
*s.Push<int>() = 1;
*s.Push<int>() = 2;
*s.Push<int>() = 3;
*s.Push<int>() = 4;
for (int i = 0; i < 2; i++) {
    int* a = s.Pop<int>(2);
    std::cout << a[0] << " " << a[1] << std::endl;
}
// 输出：
// 3 4
// 1 2
~~~

注意到，Pop() 返回弹出的最底端元素的指针，我们仍然可以通过这指针合法地访问这些弹出的元素。

## 重要事项（坑出没注意）

在 StartArray() 的例子里，我们看到使用 placement new 来构建对象。在普通的情况下，new 和 delete 应该是成双成对的，但使用了 placement new，就通常不能使用 delete，因为 delete 会调用析构函数**并**释放内存。在这个例子里，stack_ 对象提供了内存空间，所以我们只需要调用 ValueType 的析构函数。例如，如果解析在中途终止了，我们要手动弹出已入栈的 ValueType 并调用其析构函数：

~~~cpp
while (!stack_.Empty())
    (stack_.template Pop<ValueType>(1))->~ValueType();
~~~

另一个问题是，如果压入不同的数据类型，可能会有内存对齐问题，例如：

~~~cpp
Stack s;
*s.Push<char>() = 'f';
*s.Push<char>() = 'o';
*s.Push<char>() = 'o';
*s.Push<int >() = 123; // 对齐问题
~~~

123写入的地址不是4的倍数，在一些CPU下可能造成崩溃。如果真的要做紧凑的packing，可以用 std::memcpy：

~~~cpp
int i = 123;
std::memcpy(s.Push<int>(), &i, sizeof(i));

int j;
std::memcpy(&j, s.Pop<int>(1), sizeof(j));
~~~

## 代码复用

由于 RapidJSON 不依赖于 STL，在实现一些功能时缺少一些容器的帮忙。后来想到，一些地方其实可以把 Stack 当作可动态缩放的缓冲区来使用。例如，我们想从DOM生成JSON的字符串，就实现了 [GenericStringBuffer][genericstringbuffer.h]：

~~~cpp
template <typename Encoding, typename Allocator = CrtAllocator>
class GenericStringBuffer {
public:
    typedef typename Encoding::Ch Ch;
    
    // ...    

    void Put(Ch c) { *stack_.template Push<Ch>() = c; }

    const Ch* GetString() const {
        // Push and pop a null terminator. This is safe.
        *stack_.template Push<Ch>() = '\0';
        stack_.template Pop<Ch>(1);

        return stack_.template Bottom<Ch>();
    }

    size_t GetSize() const { return stack_.GetSize(); }

    // ...

    mutable internal::Stack<Allocator> stack_;
};
~~~

想在缓冲器末端加入字符，就使用 Stack::Push<Ch>()，想把整个缓冲取出来，就简单地回传底端的指针。不过这里有个特别的地方，因为需要空字符作结尾，在 GetString() 时，会压入并立即弹出一个空字符。如前所述，弹出后、压入其他东西前，刚弹出的内容仍然是合法的。而由于我们希望GetString() 是 const 函数，所以这里让 stack_ 加上 mutable 修饰词。

## 结语

[RapidJSON][rapidjson] 为了一些内存及性能上的优化，萌生了一个混合任意类型的堆栈类 [rapidjson::internal::Stack][stack.h]。但使用这个类要比 STL 提供的容器危险，必须清楚每个操作的具体情况、内存对齐等问题。而带来的好处是更自由的容器内容类型，可以达到高缓存一致性（用多个 [std::stack][stdstack] 不利此因素），并且避免不必要内存分配、释放、对象拷贝构造等。从另一个角度看，这个类更像一种特殊的内存分配器。

[rapidjson]: https://github.com/miloyip/rapidjson
[userguide-zh]: http://miloyip.github.io/rapidjson/zh-cn/
[StartArray]: https://github.com/miloyip/rapidjson/blob/v1.0.1/include/rapidjson/document.h#L1892
[placement new]: http://en.cppreference.com/w/cpp/language/new
[template disambiguator]: http://en.cppreference.com/w/cpp/language/dependent_name
[stdstack]: http://en.cppreference.com/w/cpp/container/stack
[stack.h]: https://github.com/miloyip/rapidjson/blob/v1.0.1/include/rapidjson/internal/stack.h
[genericstringbuffer.h]: https://github.com/miloyip/rapidjson/blob/v1.0.1/include/rapidjson/stringbuffer.h