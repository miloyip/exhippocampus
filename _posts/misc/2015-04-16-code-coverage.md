---
layout: page
title: 实际软件工程中是否真的需要100%代码覆盖率（code coverage）？
categories:
    - misc
---

以下是我的个人经历，可以作为一个案例参考。最后尝试总结。

我近年展开新软件项目时，都尽量以测试驱动的形式开发，常常有不少的单元测试。然而，之前尝试用 gcov/lcov 的结果有点问题，也没有加入连续整合（Continous Integration, CI）中，并不太关注覆盖率。或者更坦白地说，写程序二十多年来，也没怎么做覆盖率的分析。

我问这个问题之时，正在为 RapidJSON 的正式版做准备。刚刚上周末看到别的GitHub项目 [nlohmann/json](https://github.com/nlohmann/json)，它用上了一个叫 [Coveralls](http://coveralls.io/) 的免费服务，并声称该项目是 100% coverage。

![coveralls](/images/code_coverage01.jpg)

好奇心驱使下，我也把这服务整合至RapidJSON中，经调整一些设定后，[miloyip/rapidjson](https://coveralls.io/r/miloyip/rapidjson) 跑出91% coverage。

![RapidJSON coverage](/images/code_coverage02.jpg)

「这也算是不错了吧？」我最初是这么想的。RapidJSON 本身有 5400 多行代码（LOC），而单元测试也另有 3500 多行。「这个测试代码量的比例已经不错了吧？」

然而，当我仔细分析没被覆盖的代码，并尝试把它们都覆盖，就发现一些情况：

1. 找到了1个bug（Fixed a bug in trimming long number sequence），这是最严重的问题。原因是开发期间改变了一些参数，但忘了在一个条件下做出相应的改动。

2. 找到了1个不应存在的公开API（Remove an invalid Document::ParseInsitu() API），这是在重构的时候加上的，当时忘了 insitu parsing 只允许来源编码和目的编码必须一致。

3. 找到了一些未被执行、可以消去的代码（拟似死代码），主要出现于与浮点数转换相关的复杂算法里。这是由于那些代码都是参考别人的实现，逐渐变成现在的模样。例如原来`strtod()`的实现里，只有 FastPath 和 BigNumber 算法，后来在两者中间加入了 DiyFP 的算法，导致 BigNumber 中的一些分支条件不会再被触发。删去这些冗馀的代码可以优化性能，并减少代码量。但这些改动会存在风险，因为不知道是刚好单元测试没有覆盖到，还是真的冗馀。为了增强信心，唯有再审视代码的逻辑，并增加更多的测试数据及代码。

4. 一些应该测试的分支，例如中途返回错误的地方。
一些锁碎的编译器相关问题。例如 gcc 开了`-Wswitch-default`，但在`default`里做`assert(false)`这种没有覆盖的情况。

经过几天努力，60多个 commit、新增1000多行（不少新文件的license header⋯⋯）、删减200多行，终于完成了 [100% line of code coverage](https://github.com/miloyip/rapidjson/pull/304) 的 PR。

![RapidJSON repo stat](/images/code_coverage03.jpg)

如其他答案所谈及，代码覆盖并不等于测试质量、代码质量。除非使用 [Formal method](http://en.wikipedia.org/wiki/Formal_methods)，一般来说测试是无法证明代码的正确性的。但即使代码覆盖（甚至只是最简单的 line of code coverage）有许多潜在问题，适当地使用它作为一种手段／工具，也是有相当有帮助的，以下列出一些原因：

1. 我是粗心大意、善忘的。在这个案例中，发现有许多问题是因为改动代码后，忘记了增加、修改相关测试。当项目开发持续很长时间，也会忘记许多约定、细节，修改代码时只求通过测试，而没考虑到修改会否令其他代码变成死代码。

2. 我是懒惰的（较堂皇的说法是时间所限）。虽然理想地应该为每个函数去做单元测试，但有时候只测试了上一层的代码。并没有深究每个函数中是否完全覆盖。尤其通常只测试正常路径，而容易忽略一些异常路径。如果参考了别人的算法实现，更容易因为一知半解而忽略各种情况。

总括而言，我现在认为，最好是在 CI 中加入代码覆盖分析。如果能在开始开发时，以代码覆盖作为工具粗略分析测试是否充分，可以避免以上的一些问题。并且通过 CI，我们能追踪每个 commit 是否有潜在问题。尤其是分析开源项目中社区的PR时，能客观地看到是否缺乏最低限度的测试。

如果要简单地回答自己设下的问题，就是尽量达到100%的覆盖率。许多程序因为各种原因难以覆盖，或是所费精力太大，不必强求达至100%，但需要清楚了解个中原因。然后，除了关注覆盖率的绝对值，也要关注它的变化，求进步。

本文原发表于[知乎](http://www.zhihu.com/question/29528349/answer/44914382)，经过修改。