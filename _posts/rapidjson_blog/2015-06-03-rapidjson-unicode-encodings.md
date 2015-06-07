---
layout: page
title: RapidJSON 代码剖析（三）：Unicode 的编码与解码
categories:
    - rapidjson
image:
    title: header_rapidjson_unicode.jpg
    caption: 老彼得·布吕赫尔笔下的巴别塔
    caption_url: http://zh.wikipedia.org/zh-cn/%E5%B7%B4%E5%88%A5%E5%A1%94
---

RapidJSON 希望尽量支持各种常用 UTF 编码，用四百多行代码实现了 5 种 Unicode 编码器／解码器，另外加上 ASCII 编码。


根据 [RFC-7159][RFC-7159]：

> 8.1 Character Encoding
> 
> JSON text SHALL be encoded in UTF-8, UTF-16, or UTF-32.  The default encoding is UTF-8, and JSON texts that are encoded in UTF-8 are interoperable in the sense that they will be read successfully by the maximum number of implementations; there are many implementations that cannot successfully read texts in other encodings (such as UTF-16 and UTF-32).

> 翻译：JSON文本应该以UTF-8、UTF-16、UTF-32编码。缺省编码为UTF-8，而且有大量的实现能读取以UTF-8编码的JSON文本，说明UTF-8具互操作性；有许多实现不能读取其他编码（如 UTF-16及UTF-32）

本文会简单介绍它的实现方式。

## 回顾 Unicode、UTF 与 C++

[Unicode][Unicode] 是一个标准，用于处理世界上大部分的文字。在 Unicode 出现之前，每种语言文字会使用不同的编码，例如英文主要用 ASCII、中文主要用 GB 2312 和大五码、日文主要用 JIS 等等。这样会造成很多不便，例如一个文本信息很难混合各种语言的文字。

Unicode 定义了统一字符集（Universal Coded Character Set, UCS），每个字符映射至一个整数码点（code point），码点的范围是 0 至 0x10FFFF。储存这些码点有不同方式，这些方式称为 Unicode 转换格式（Uniform Transformation Format, UTF）。现时流行的 UTF 为 UTF-8、UTF-16 和 UTF-32。每种 UTF 会把一个码点储存为一至多个编码单元（code unit）。例如 UTF-8 的编码单元是 8 位的字节、UTF-16 为 16 位、UTF-32 为 32 位。除 UTF-32 外，UTF-8 和 UTF-16 都是可变长度编码。

UTF-8 成为现时互联网上最流行的格式，有几个原因：

1. 它采用字节为编码单元，不会有字节序（endianness）的问题。
2. 每个 ASCII 字符只需一个字节去储存。
3. 如果程序原来是以字节方式储存字符，理论上不需要特别改动就能处理 UTF-8 的数据。

那么，在处理 JSON 时，若使用 UTF-8，我们为何还需要特别处理？这是因为 JSON 的字符串可以包含 `\uXXXX` 这种转义字符串。例如`["\u20AC"]`这个JSON是一个数组，里面有一个字符串，转义之后是欧元符号`"€"`。在 JSON 中，这个转义符使用 UTF-16 编码。JSON 也支持 UTF-16 代理对（surrogate pair），例如高音谱号(U+1D11E)可写成`"\uD834\uDD1E`"。所以，即使是 UTF-8 的 JSON，我们都需要在解析JSON字符串时做解码／编码工作。

虽然 Unicode 始于上世纪90年代，C++11 才加入较好的支持。RapidJSON 为了支持 C++ 03，需要自行实现一组编码／解码器。

## Encoding 

RapidJSON 的编码（encoding）的概念是这样的（非C++代码）：

~~~cpp
concept Encoding {
    typename Ch;    //! Type of character. A "character" is actually a code unit in unicode's definition.

    enum { supportUnicode = 1 }; // or 0 if not supporting unicode

    //! \brief Encode a Unicode codepoint to an output stream.
    //! \param os Output stream.
    //! \param codepoint An unicode codepoint, ranging from 0x0 to 0x10FFFF inclusively.
    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint);

    //! \brief Decode a Unicode codepoint from an input stream.
    //! \param is Input stream.
    //! \param codepoint Output of the unicode codepoint.
    //! \return true if a valid codepoint can be decoded from the stream.
    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint);

    //! \brief Validate one Unicode codepoint from an encoded stream.
    //! \param is Input stream to obtain codepoint.
    //! \param os Output for copying one codepoint.
    //! \return true if it is valid.
    //! \note This function just validating and copying the codepoint without actually decode it.
    template <typename InputStream, typename OutputStream>
    static bool Validate(InputStream& is, OutputStream& os);

    // The following functions are deal with byte streams.

    //! Take a character from input byte stream, skip BOM if exist.
    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is);

    //! Take a character from input byte stream.
    template <typename InputByteStream>
    static Ch Take(InputByteStream& is);

    //! Put BOM to output byte stream.
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os);

    //! Put a character to output byte stream.
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, Ch c);
};
~~~

由于 C++ 可使用不同类型作为字符类型，如 `char`、`wchar_t`、`char16_t` (C++11)、`char32_t` (C++11)等，实现这个 `Encoding` 概念的类需要设定一个 `Ch` 类型。

这当中最种要的函数是 `Encode()` 和 `Decode()`，它们分别把码点编码至输出流，以及从输入流解码成码点。`Validate()`则是只验证编码是否正确，并复制至目标流，不做解码工作。例如 UTF-16 的编码／解码实现是：

~~~cpp
template<typename CharType = wchar_t>
struct UTF16 {
    typedef CharType Ch;
    RAPIDJSON_STATIC_ASSERT(sizeof(Ch) >= 2);

    enum { supportUnicode = 1 };

    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 2);
        if (codepoint <= 0xFFFF) {
            RAPIDJSON_ASSERT(codepoint < 0xD800 || codepoint > 0xDFFF); // Code point itself cannot be surrogate pair 
            os.Put(static_cast<typename OutputStream::Ch>(codepoint));
        }
        else {
            RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
            unsigned v = codepoint - 0x10000;
            os.Put(static_cast<typename OutputStream::Ch>((v >> 10) | 0xD800));
            os.Put((v & 0x3FF) | 0xDC00);
        }
    }

    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputStream::Ch) >= 2);
        Ch c = is.Take();
        if (c < 0xD800 || c > 0xDFFF) {
            *codepoint = c;
            return true;
        }
        else if (c <= 0xDBFF) {
            *codepoint = (c & 0x3FF) << 10;
            c = is.Take();
            *codepoint |= (c & 0x3FF);
            *codepoint += 0x10000;
            return c >= 0xDC00 && c <= 0xDFFF;
        }
        return false;
    }

    // ...
};
~~~

## 转码

RapidJSON 的解析器可以读入某种编码的JSON，并转码为另一种编码。例如我们可以解析一个 UTF-8 JSON文件至 UTF-16 的 DOM。我们可以实现一个类做这样的转码工作：

~~~cpp
template<typename SourceEncoding, typename TargetEncoding>
struct Transcoder {
    //! Take one Unicode codepoint from source encoding, convert it to target encoding and put it to the output stream.
    template<typename InputStream, typename OutputStream>
    RAPIDJSON_FORCEINLINE static bool Transcode(InputStream& is, OutputStream& os) {
        unsigned codepoint;
        if (!SourceEncoding::Decode(is, &codepoint))
            return false;
        TargetEncoding::Encode(os, codepoint);
        return true;
    }

    // ...
};
~~~

这段代码非常简单，就是从输入流解码出一个码点，解码成功就编码并写入输出流。但如果来源的编码和目标的编码都一样，我们不是做了无用功么？但 C++ 的[模板偏特化（[partial template specialization][partial template specialization]）可以这么做：

~~~cpp
//! Specialization of Transcoder with same source and target encoding.
template<typename Encoding>
struct Transcoder<Encoding, Encoding> {
    template<typename InputStream, typename OutputStream>
    RAPIDJSON_FORCEINLINE static bool Transcode(InputStream& is, OutputStream& os) {
        os.Put(is.Take());  // Just copy one code unit. This semantic is different from primary template class.
        return true;
    }

    // ...
};
~~~

那么，不用转码的时候，就只需复制编码一个单元。零开销！所以，在解析及生成 JSON 时都使用到 `Transcoder` 去做编码转换。

## UTF-8 解码与 DFA

在 UTF-8 中，一个码点可能会编码为1至4个编码单元（字节）。它的解码比较复杂。RapidJSON 参考了 [Hoehrmann][Hoehrmann] 的实现，使用确定有限状态自动机（deterministic finite automation, DFA）的方式去解码。UTF-8的解码过程可以表示为以下的DFA:

![UTF-8 DFA](/images/utf8_dfa.png)

当中，每个转移（transition）代表在输入流中遇到的编码单元（字节）范围。这幅图忽略了不合法的范围，它们都会转移至一个错误的状态。

原来我希望在本文中详细解析 RapidJSON 实现中的「优化」。但几年前在 Windows 上的测试结果和近日在 Mac 上的测试结果大相迳庭。还是等待之后再分析后再讲。

## AutoUTF

有时候，我们不能在编译期决定 JSON 采用了哪种编码。而上述的实现都是在编译期以模板类型做挷定的。所以，后来 RapidJSON 加入了一个运行时做动态挷定的编码类型，称为 `AutoUTF`。它之所以称为自动，是因为它还有检测[字节顺序标记][字节顺序标记]（byte-order mark, BOM）的功能。如果输入流有 BOM，就能自动选择适当的解码器。不过，因为在运行时挷定，就需要多一层间接。RapidJSON采用了函数指针的数组来做这间接层。

## ASCII

有一个用家提出希望写入 JSON 时，能把所有非 ASCII 的字符都写成 `\uXXXX` 转义形式。解决方法就是加入了 `ASCII` 这个模板类：

~~~cpp
template<typename CharType = char>
struct ASCII {
    typedef CharType Ch;

    enum { supportUnicode = 0 };

    // ...

    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
        unsigned char c = static_cast<unsigned char>(is.Take());
        *codepoint = c;
        return c <= 0X7F;
    }

    // ...
};
~~~

通过检测 `supportUnicode`，写入 JSON 时就可以决定是否做转义。另外，`Decode()`时也会检查是否超出 ASCII 范围。

## 总结

RapidJSON 提供内置的 Unicode 支持，包括各种 UTF 格式及转码。这是其他 JSON 库较少做的部分。另外，RapidJSON 是在输入输出流的层面去处理，避免了把整个JSON读入、转码，然后才开始解析。RapidJSON 这么实现节省内存，而且性能应该更优。

最近为了开发 RapidJSON 下一个版本新增的 JSON Schema 功能，实现了一个[正则表达式引擎][regex.h]。该引擎也利用了 `Encoding` 这套框架，轻松地实现了 Unicode 支持，例如可以直接匹配 UTF-8 的输入流。

[RFC-7159]: https://tools.ietf.org/html/rfc7159
[encodings.h]: https://github.com/miloyip/rapidjson/blob/master/include/rapidjson/encodings.h
[Unicode]: http://zh.wikipedia.org/wiki/Unicode
[Hoehrmann]: http://bjoern.hoehrmann.de/utf-8/decoder/dfa/
[字节顺序标记]: http://zh.wikipedia.org/zh-cn/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F
[partial template specialization]: http://en.cppreference.com/w/cpp/language/partial_specialization
[regex.h]: https://github.com/miloyip/rapidjson/blob/regex/include/rapidjson/internal/regex.h