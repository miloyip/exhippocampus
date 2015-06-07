---
layout: page
title: RapidJSON 代码剖析（二）：使用 SSE4.2 优化字符串扫描
categories:
    - rapidjson
image:
    title: header_rapidjson_sse42.jpg
    caption: 2008 年发售的 Intel Core i7 芯片，它采用的 Nehalem 是第一个支持 SSE4.2 的微架构
    caption_url: http://www.intel.com/pressroom/archive/releases/2008/20081117comp_sm.htm
---

现在的 CPU 都提供了单指令流多数据流（single instruction multiple data, SIMD）指令集。最常见的是用于大量的浮点数计算，但其实也可以用在文字处理方面。其中，SSE4.2 包含了一些专为字符串而设的指令。我们通过使用这些指令，可以大幅提升某些 JSON 解析的性能。

## 跳过空白字符

我们知道，有一些 JSON 含有缩进（indentation），这些 JSON 有大量的空白字符（whitespace）。在解析 JSON 的时候，需要跳过这些空白字符。这个操作在 RapidJSON 下是这样的（[reader.h][SkipWhitespace]，为配合版面稍改排版）：

~~~cpp
template<typename InputStream>
void SkipWhitespace(InputStream& is) {
    internal::StreamLocalCopy<InputStream> copy(is);
    InputStream& s(copy.s);

    while (s.Peek() == ' '  ||
           s.Peek() == '\n' ||
           s.Peek() == '\r' ||
           s.Peek() == '\t')
    {
        s.Take();
    }
}
~~~

我们先不关注 `StreamLocalCopy` 等东西。这段代码很简单，就是凡在输入流中遇到4种空白字符，都提取出来跳过，直至流里的字符为非空白字符。

但这种代码会带来很多分支（branching），而且我们每次只能处理一个字符。

## SSE4.2

在 Intel 的 SSE4.2 指令集中，有一个 `pcmpistrm` 指令，它可以一次对一组16个字符与另一组字符作比较，也就是说一个指令可以作最多16×16=256次比较。

对于上面跳过空白字符的需求，我们只需要对16个输入流里的字符与4个空白字符比较，即16×4=64次比较。虽然这样未用尽所有计算能力，但一个指令能代替64个比较以及「或」运算，还是很划算的。

我们可以使用 VC/gcc/clang 都支持的 instrinsic 函数去使用这个指令。这个指令的函数命名为 `_mm_cmpistrm()`，在`nmmintrin.h`中定义。

`SkipWhitespace` 的 SSE4.2 版本只能跳过字符串的输入流，其部分代码如下：

~~~cpp
inline const char *SkipWhitespace_SIMD(const char* p) {
    // ... 非对齐处理

    static const char whitespace[16] = " \n\r\t";
    const __m128i w = _mm_load_si128((const __m128i *)&whitespace[0]);

    for (;; p += 16) {
        const __m128i s = _mm_load_si128((const __m128i *)p);
        const unsigned r = _mm_cvtsi128_si32(_mm_cmpistrm(w, s, 
            _SIDD_UBYTE_OPS | _SIDD_CMP_EQUAL_ANY |
            _SIDD_BIT_MASK | _SIDD_NEGATIVE_POLARITY));

        if (r != 0) {   // some of characters is non-whitespace
#ifdef _MSC_VER         // Find the index of first non-whitespace
            unsigned long offset;
            _BitScanForward(&offset, r);
            return p + offset;
#else
            return p + __builtin_ffs(r) - 1;
#endif
}
~~~

解析一下这里 `_mm_cmpistrm()` 用上了的选项：

* `_SIDD_UBYTE_OPS`: 操作单位是无号字节，即16个 `unsigned char`。
* `_SIDD_CMP_EQUAL_ANY`: 每次比较 `s` 里的字符，是否和 `w` 中的任意字符相等。
* `_SIDD_BIT_MASK`: 以比特方式返回结果。
* `_SIDD_NEGATIVE_POLARITY`: 把结果反转。这里指返回值的1代表非空白字符。

然后，我们用`_mm_cvtsi128_si32()`指令，把返回的最低位32字节储存成普通的32位整数。如果含有非空白字符，就使用`_BitScanForward()`或`__builtin_ffs()`计算出最早出现的非空白字符，并把指针跳到那里返回。

## 对齐问题

通过 SSE 读写内存，每次可以读写128位（16字节）数据。理想地是使用 128位对齐的地址来读写，这样会最大化读写速度。

最初我使用了 `_mm_loadu_si128()` 从非对齐的来源字符串读取16个字符。当时我觉得最多就是损失一些时间吧，问题似乎不大。但实际上还是出现了[问题][oldissue104]：

>If rapidjson::SkipWhitespace_SIMD(char const*) is called at close to the end of string buffer which has less than 16 bytes of allocated space, the function will read beyond the memory it owns.

> In our use case, we parse around 50 million JSON files/buffers per day and
we got hit by the bug around 100 times per day on average before the
workaround.

后来，我估计是因为用非对齐读取，有可能在边界会读到未分配的内存分页，做成很低机率的崩溃。因此，修正方法是先用普通代码处理未对齐的地址，然后才使用 SIMD 进行读取。

~~~cpp
inline const char *SkipWhitespace_SIMD(const char* p) {
    // ...

    // 16-byte align to the next boundary
    const char* nextAligned = reinterpret_cast<const char*>(
        (reinterpret_cast<size_t>(p) + 15) & ~15);

    while (p != nextAligned)
        if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
            ++p;
        else
            return p;

    // The rest of string using SIMD
    // ...
}
~~~

## 快速返回

优化其实还要看实际情况。我们发现，有比较多的情况是，第一个字符已是非空白字符。尤其是已去除空白字符的JSON，上面代码的初始时间还是比较大。因此，我们把第一个字符的检测独立出来。

~~~cpp
inline const char *SkipWhitespace_SIMD(const char* p) {
    // Fast return for single non-whitespace
    if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
        ++p;
    else
        return p;

    // ...
}
~~~

## 性能测试

### 测试环境

* iMac 2.7 GHz Intel Core i5
* Apple LLVM version 6.1.0 (clang-602.0.49) (based on LLVM 3.6.0svn)

### 测试用例 1

跳过1M个空白字符1000次。

* 基本实现: 675 ms
* SSE4.2: 86 ms
* [strspn][strspn]: 897 ms

### 测试用例 2

使用 SAX API 去原位解析（in situ parse）一个含缩进的 671KB [sample.json][sample.json]，不处理事件（null handler）。

* 基本实现: 934 ms
* SSE4.2: 650 ms

## 结语

RapidJSON 中使用 SSE4.2 指令集跳过空白字符，可以在一个迭代中进行 64 次字符比较，而且每次读取 128 位数据应该对内存频宽友好。为了兼容更旧的 x86 系 CPU，RapidJSON 也提供了一个 SSE2 的版本，但每个迭代需要执行更多指令，读取可参考[源代码][SkipWhitespaceSSE2]。

此优化只对含缩进的 JSON 有利，但我们通过「快速返回」使非缩进 JSON 也不会减慢，算是一种权衡之策。在后续的 v1.1 版本中，我希望尝试利用 SIMD 指令去快速扫瞄需处理转义（escaping）的字符，不需转义的部分能使用到 128 位复制至目标缓冲。由于转义符在 JSON 的出现率较低，此举应该能进一步提升整体性能。

最后，关于 x86/x64 系的 SIMD 指令，我推荐 [Intel Instrinsic Guide][IntelIntrinsicGuide] 及 Agner Fog 的[5本优化手册][agnerfogmanual]。

这两期都是比较低阶的东西，下期将会谈一些比较高层一点的，敬请关注。

[单指令流多数据流]: http://zh.wikipedia.org/wiki/%E5%8D%95%E6%8C%87%E4%BB%A4%E6%B5%81%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%B5%81
[SkipWhitespace]: https://github.com/miloyip/rapidjson/blob/v1.0.2/include/rapidjson/reader.h#L246
[SkipWhitespaceSSE2]: https://github.com/miloyip/rapidjson/blob/master/include/rapidjson/reader.h#L294
[oldissue104]: https://code.google.com/p/rapidjson/issues/detail?id=104
[sample.json]: https://github.com/miloyip/rapidjson/blob/master/bin/data/sample.json
[strspn]: http://en.cppreference.com/w/cpp/string/byte/strspn
[IntelIntrinsicGuide]: https://software.intel.com/sites/landingpage/IntrinsicsGuide/
[agnerfogmanual]: http://www.agner.org/optimize/#manuals