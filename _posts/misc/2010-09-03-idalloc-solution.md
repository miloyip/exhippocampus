---
layout: page
title: 手机分配短讯id的面试题目（下篇：分析解答）
categories:
    - misc
---

看过[《上篇：厘清需求》]({% post_url /misc/2010-08-31-idalloc-clarify %})，读者想到多少个解呢？本篇首先谈及一些基本分析，之后会按两种 API 设计（纯函数 API 和含状态的 API），分别描述多个解。虽然面试时或许不能进行实际测试，但本文还是给出 PC 上的效能测试结果。最后分析比较各解之优劣作为总结。

## 问题分析

原来的问题是要从一个无序`ids`数组里分配一个 id。我们可以用数学方式去更清楚地说明这个问题。

设$m = 256$为所有 id 的个数，集合$U = \left\\{ 0, 1, \ldots, m-1 \right\\}$为所有id的集合。那么，给定一个已分配id的集合$A\subset U$，$A = \left\\{ a_0, a_1, \ldots, a_{n-1} \right\\}$（即参数`ids`），本题目可表示为，求一个$x$（即传回的id），符合条件：

$$
x \in U \setminus A
$$

$\setminus$是补集的意思，即$x$属于$U$但不属于$A$。上回的对答已确定$U \setminus A\ne \oslash$ ，即$x$必然存在。此外，这个条件又可以写成：

$$
(x \in U) \wedge (x \notin A)
$$

以上两种表达式可说明此问题的两种解法，一种编程方向是查找U集里有没有不属于A的id，而另种是计算A的补集再取出其中一个id。

## 纯函数API的解

实现程序之前，如果可以，应先写测试函数。笔者认为，若面试者在情况容许下，也可在解答题目之前，写下测试程序。如果有多个面试者能同样解题，或许同时写下测试程序的面试者能脱颖而出。

### 测试函数

为了简单起见，笔者使用了`assert()`来检测正确性，只于Debug版本有效。而Release版本则用来测试效能。

由于$U$集合的子集合很多，$\left| P(U) \right| = 2^m=2^{256}\approx 10^{76}$ ，不可能穷举所有可能集合。所以，只能够举出随机的集合以作测试。

以下是一些常数（宏）及类型声明，`TEST_COUNT`是测试次数，而`TEST_REPEATCOUNT`是为了测试效能时，重复测试的次数（即Release版本会调用测试函数一百万次）：

~~~cpp
#define M 256 // ID的数目，且所有ID在[0, M)的区间内

#define TEST_COUNT 10000 
#ifdef NDEBUG
#define TEST_REPEATCOUNT 100
#else
#define TEST_REPEATCOUNT 1
#endif

typedef unsigned char byte; 
typedef unsigned long dword; 

typedef byte (*idalloc_func)(byte*, size_t); 
~~~

首先，写一个帮助函数测试某id是否在`ids`集合之内（不熟 C++ 的读者可参考 C 版本）:

~~~cpp
// 检测ids里是否含id (C++ 版本) 
inline bool contain(byte* ids, size_t n, byte id) { 
	assert(ids != NULL); 

	return find(ids, ids + n, id) != ids + n; 
} 
~~~

~~~cpp
// 检测ids里是否含id (C 版本) 
inline bool contain(byte* ids, size_t n, byte id) { 
	assert(ids != NULL); 

	for (size_t i = 0; i < n; i++) 
		if (ids[i] == id) 
			return true; 
	return false; 
}
~~~

笔者首先写了一个测试平均情况的测试平台函数：

~~~cpp
// 测试平均情况 
void test_average(idalloc_func idalloc) { 
	assert(idalloc != NULL); 

	byte ids[M]; 

	for (size_t i = 0 ; i < M; i++) 
		ids[i] = (byte)i; 

	srand(0); // 使每次测试的伪随机数相同 

	size_t n = 0; 
	for (int test = 0; test < TEST_COUNT; test++) { 
		random_shuffle(ids, ids + M); // 把整个数组洗牌 

		for (int repeat = 0; repeat < TEST_REPEATCOUNT; repeat++) { 
			byte id = idalloc(ids, n); 
			(void)id; 
			assert(!contain(ids, n, id)); 

			// 测试是否最小的id 
			for (size_t i = 0; i < id; i++) 
				assert(contain(ids, n, (byte)i)); 
		} 

		n = (n + 1) % M; 
	} 
} 
~~~

简单解释。首先，把ids数组填入所有id值。利用`random_shuffle()`把整个`ids`数组洗牌，而`n`则是在$[0, M)$区间里循环递增。

由于笔者给出的解，都能传回最小的 id，所以也会测试这条件。而最坏情况，就是`ids`含无序的$\left\\{ 0, 1, \ldots M - 2 \right\\}$，分配到的 id 为$M-1$，笔者也为此编了一个最坏情况的效能测试函数。

~~~cpp
// 测试最坏情况(ids为无序的[0, M - 2], 结果必然是id = M - 1) 
void test_worst(idalloc_func idalloc) { 
	assert(idalloc != NULL); 

	const size_t n = M - 1; 
	byte ids[n]; 

	srand(0); // 使每次测试的伪随机数相同 

	for (size_t i = 0 ; i < n; i++) 
		ids[i] = (byte)i; 

	for (int test = 0; test < TEST_COUNT; test++) { 
		random_shuffle(ids, ids + n); 

		for (int repeat = 0; repeat < TEST_REPEATCOUNT; repeat++) { 
			byte id = idalloc(ids, n); 
			(void)id; 
			assert(id == M - 1); 
		} 
	} 
} 
~~~

### 线性查找

最简单的想法，可能是遍历所整个$U$集合（即0至$M-1$），并使用`contain()`函数检测该 id 是否不包含在`ids`数组里。

~~~cpp
// 线性查找 (总是传回最小id) 
// 时间复杂度: O(n^2) 
// 临时内存大小: 0 字节 
// 注: 因为n < M，无论ids内的值为何(甚至有重复元素)，必然可找到一个id，所以id的for不用边界检查。 
byte linear_search(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	// 逐个id检查是否存在于[ids, ids + n) 
	for (byte id = 0; ; id++) 
		if (!contain(ids, n, id)) 
			return id; 
} 
~~~

### 二分查找

网友 Doyle 在 TL 里提出了用二分查找的主意。笔者实现了两种形式，以下这个是不需额外内存。原理是把U集合分割为两个各占一半的区间，分别数算两个区间内的已分配元素数目，若元素数目少于区间大小，即代表该区间内有未分配的 id。再继续分割该区间，直至区间内都是可分配的 id（即找到的元素是零）。

~~~cpp
// 数ids内有多少个id在[min, max)的区间内 
inline size_t count_interval(byte* ids, size_t n, size_t min, size_t max) { 
	size_t count = 0; 

	for (size_t i = 0; i < n; i++) 
		if (ids[i] >= min && ids[i] < max) 
			count++; 

	return count; 
} 

// 二分查找 (总是传回最小id) 
// 时间复杂度: O(n lg n) 
// 临时内存大小: 0 字节 
byte binary_search(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	size_t l = 0, r = M; 

	for(;;) { 
		size_t c = (l + r) / 2; // 把id范围从[l, r)分割为[l, c), [c, r)两个区间 
		size_t count; 

		// 以下的条件测试次序保证了传回最小id 
		if ((count = count_interval(ids, n, l, c)) < c - l) { 
			if (count == 0) 
				return (byte)l; 
			r = c; 
		} 
		else if ((count = count_interval(ids, n, c, r)) < r - c) { 
			if (count == 0) 
				return (byte)c; 
			l = c; 
		} 
		else 
			assert(false); // 因为n < M，不可能找不到任何id 
	} 
}
~~~

这算法在最坏情况比线性查找快，但平均情况下却不一定。

### 排序

以上两个解，都是查找的方式，毋需改动数据。相反，另一类解用的算法需改动`ids`数组内的元素，或是把`ids`复制到另一个临时数组里进行更改型的算法。

最简单的算法，是把无序的`ids`排序。之后就可以从头开始扫描未分配的 id。

~~~cpp
// 排序 (总是传回最小id) 
// 时间复杂度: O(n lg n) 
// 临时内存大小: M 字节(如果可改变ids则是0) 
byte sort_stl(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	byte buffer[M]; 
	memcpy(buffer, ids, n); 

	sort(buffer, buffer + n); // 平均 O(n lg n) 

	for (size_t i = 0; i < n; i++) 
		if (buffer[i] != i) 
			return (byte)i; 

	return (byte)n; 
}
~~~

但读者可能会想到，把整个数组排序可能会做了很多无用工。而且，快速排序（quicksort）的最坏时间复杂度是$\mathrm{O}(n^2)$。因此，就有了下一个解。

### 堆

笔者想到的另一个解是使用堆（heap）数据结构。堆可保证第一个元素是最小的元素（通常是最大的，但这题目里我们希望取得最小的），而每次弹出这个元素，取出第二小的元素只需要$\mathrm{O}(\log n)$的时间。 `sort_stl()`需要完整排序，而使用堆则是逐步进行的，中途找到没用到的 id 就可以停下来，所以平均来说会省下很多时间。

~~~cpp
// 堆 (总是传回最小id) 
// 时间复杂度: O(n lg n) 
// 临时内存大小: M 字节(如果可改变ids则是0) 
byte heap_stl(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	byte buffer[M]; 
	memcpy(buffer, ids, n); 

	byte* end = buffer + n; 
	make_heap(buffer, end, greater()); // O(n) 

	for (byte id = 0; buffer != end; id++, end--) { 
		if (buffer[0] != id) 
			return id; 
		pop_heap(buffer, end, greater()); // O(lg n) 
	} 

	return (byte)n; 
}
~~~

最坏的情况，是要把最小的$M-1$个元素最弹出，才能求得$id = M-1$。这情况其实等价于堆排序（heapsort）。

### 剖分

另一个方法和二分查找相似，就是把数组剖分（partition）为两部分，这应该是 Doyle 提出的原意。原理是，设一个中间`c=M/2`，用它把无序`ids`集合剖分为两个无序集合，前一个集合的元素小于`c`，后一个的元素大于或等于`c`。那么，应该有一个集合的元素数量少于 id 区间的大小，再把该集合继续剖分，直至变成空集。

~~~cpp
// 剖分 (总是传回最小id) 
// 时间复杂度: O(n) 
// 临时内存大小: M 字节(如果可改变ids则是0) 
byte partition_stl(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	byte buffer[M]; 
	memcpy(buffer, ids, n); 

	byte *first = buffer, *last = buffer + n; 
	size_t l = 0, r = M; 

	for (;;) { 
		size_t c = (l + r) / 2; 
		byte* middle = partition(first, last, bind2nd(less(), c)); // O(n) 
		// 后置条件: l <= [first, middle)内元素 < c 及 c <= [middle, last)内元素 < r

		// 以下的条件测试次序保证了传回最小id 
		if (first == middle) 
			return (byte)l; 
		else if ((size_t)distance(first, middle) < c - l) { 
			last = middle; 
			r = c; 
		} 
		else if (middle == last) 
			return (byte)c; 
		else if ((size_t)distance(middle, last) < r - c) { 
			first = middle; 
			l = c; 
		} 
		else 
			assert(false); 
	} 
}
~~~

此算法的妙处在于，时间复杂度仅为$\mathrm{O}(n)$。为什么呢？因为`partition()`的时间复杂度是$\mathrm{O}(n)$，而此算法中每个迭代需处理的元素是$n, n/2, n/4, \ldots$，把这个几何数列求和，得出$2n$，所以此算法为线性时间。

### 布尔集合

也许，最多网友都想到的解，就是把`ids`无序数组变换为另一个集合表示方式，能更快地测试A是否不含某id。这种表达方式是使用一个布尔数组（boolean array），储存某id是否在ids无序数组里。用数学方式，可以称这个数组为一个函数$f:U\rightarrow \{0,1\}$：

$$
f(i)=\left\{
	\begin{matrix}
		1 & \text{if } i \in A\\
		0 & \text{if } i \notin A
	\end{matrix}
\right.
$$

建立这个数组之后，再扫描一次，找出没使用到的 id。

~~~cpp
// 布尔集合 (总是传回最小id) 
// 时间复杂度: O(n) 
// 临时内存大小: M 字节 
byte boolset(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	bool id_used[M] = { false }; 

	// 填充 id_used 
	for (size_t i = 0; i < n; i++) { 
		assert(!id_used[ids[i]]); // 此处断言失败代表ids有重复元素 
		id_used[ids[i]] = true; 
	} 

	// 扫描id_used去找出最小未用id 
	for (size_t i = 0; i < M; i++) 
		if (!id_used[i]) 
			return (byte)i; 

	assert(false); 
	return 0; 
}
~~~

这类解法在纯函数 API 中是最快的，但必须使用额外内存。

### 位集合

上述的解，每个数组元素由于只需储存1个比特，可以把8个布尔值置于字节里，减少额外内存。这种集合称为位集合（bit set）或位图（bitmap）。此外，在32位 CPU 上，可一次检查32位是否全0或全1，这可是一个优化。这次，我们直接储存补集$A$，即是那些分配了的id会把位设为0，那么在扫描时就不需做一个 not 位元运算。

~~~cpp
// 位集合 (总是传回最小id) 
// 时间复杂度: O(n) 
// 临时内存大小: floor((M + 31) / 32) * 4 字节 
byte bitset_standard(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	const size_t dword_count = (M + 31) / 32; 
	dword id_unused_bits[dword_count]; 

	// 开始时设全部id为未用(即设位为1) 
	memset(id_unused_bits, ~0, sizeof(id_unused_bits)); 

	// 填充id_unused_bits (ids内的位清为0) 
	for (size_t i = 0; i < n; i++) { 
		size_t index = ids[i] / 32; 
		dword bitIndex = ids[i] % 32; 
		assert(id_unused_bits[index] & (1 << bitIndex)); 
		id_unused_bits[index] ^= (1 << bitIndex); 
	} 

	// 扫描id_unused_bits，找出最小未用id 
	for (size_t index = 0; index < dword_count; index++) { 
		if (dword bits = id_unused_bits[index]) { 
			for (dword bitIndex = 0; bitIndex < 32; bitIndex++) 
				if (bits & (1 << bitIndex)) { 
					dword id = index * 32 + bitIndex; 
					assert(id < M); 
					return (byte)id; 
				} 
		} 
	} 

	assert(false); 
	return 0; 
}
~~~

在某些 CPU 上，还会支持一个 bsf（bit scan forward）汇编指令，可扫描一个32位值里，第一个为1的位索引（从 LSB 至 MSB）。这正正是我们想要的。以下使用了 Visual C++ 的内部函数（intrinsic function）去使用此指令。

~~~cpp
// 位集合(使用内部函数(intrinsic)) 
byte bitset_intrinsic(byte* ids, size_t n) { 
	assert(ids != NULL); 
	assert(n < M); 

	const size_t dword_count = (M + 31) / 32; 
	dword id_unused_bits[dword_count]; 

	// 开始时设全部id为未用(即设位为1) 
	memset(id_unused_bits, ~0, sizeof(id_unused_bits)); 

	// 填充id_unused_bits (ids内的位清为0) 
	for (size_t i = 0; i < n; i++) { 
		size_t index = ids[i] / 32; 
		dword bitIndex = ids[i] % 32; 
		assert(id_unused_bits[index] & (1 << bitIndex)); 
		id_unused_bits[index] ^= (1 << bitIndex); 
	} 

	// 扫描id_unused_bits，找出最小未用id 
	for (size_t index = 0; index < dword_count; index++) { 
		dword bitIndex; 
		if (_BitScanForward(&bitIndex, id_unused_bits[index])) { 
			dword id = index * 32 + bitIndex; 
			assert(id < M); 
			return (byte)id; 
		} 
	} 

	assert(false); 
	return 0; 
}
~~~

由于建立位集合所需的操作较布尔集合多，扫描的优化未必能弥补，所以位集合的主要好处是减低了临时内存的大小，为布尔集合的八分之一。

## 含状态API的解

笔者对此题目提出另一种 API 的设计，就是保存一些状态:

~~~cpp
struct manager {
    // 这里有一些状态变量(暂未决定)

    byte alloc();
    void dealloc(byte id);
};
~~~

而在工程上，我们都可以估计到，传给纯函数 API 的`ids`数组，其内容实际上是以某方式储存在系统内的。若能改善它们储存的方式，就能加速 id 的分配过程。

### 测试函数

同样，笔者为此 API 设计编写了测试函数。纯函数 API 的测试函数每次都是独立调用，但本测试的对象是有状态的。因此，此函数设计为随机分配为释放 id（各概率约为50%）。

~~~cpp
template <class T>
void test_manager() { 
	T manager; 
	bool id_allocated[M] = { false }; 
	byte allocated_ids[M]; // allocated_ids[0]至allocated_ids[id_used_count - 1]储存无序的已分配id 
	size_t allocated_id_count = 0; 

	srand(0); // 使每次测试的伪随机数相同 

	for (int test = 0; test < TEST_COUNT * TEST_REPEATCOUNT; test++) { 
		// id集为空时必须进行分配，否则若id集未满时，有一半概率进行分配 
		if (allocated_id_count == 0 || (rand() > RAND_MAX / 2 && allocated_id_count < M)) { 
			byte id = manager.alloc(); 
			assert(!id_allocated[id]); 
			id_allocated[id] = true; 
			allocated_ids[allocated_id_count++] = id; 
		} 
		else { 
			// 其他情况，随机抽一个已分配id进行释放 
			assert(allocated_id_count > 0); 
			size_t index = rand() % allocated_id_count; 
			byte id = allocated_ids[index]; 
			assert(id_allocated[id]); 
			manager.dealloc(id); 
			id_allocated[id] = false; 
			allocated_ids[index] = allocated_ids[--allocated_id_count]; // 用列表末的id取代已释放的id 
		} 
	} 
}
~~~

此外，这个测试函数不使用$\mathrm{O}(n)$的`contain()`，所有操作都是$\mathrm{O}(1)$的，测试的开销比较少。

### 布尔集合(含状态)

首先的解是把之前的布尔集合储存为状态，那么就不用每次重新建立该集合。

~~~cpp
// 布尔集合 (总是传回最小id) 
// 分配的时间复杂度: O(n) 
// 释放的时间复杂度: O(1) 
// 状态所需内存: M 字节 
struct boolset_manager { 
	bool id_used[M]; 

	boolset_manager() { 
		for (size_t i = 0; i < M; i++) 
			id_used[i] = false; 
	} 

	byte alloc() { 
		for (size_t i = 0; i < M; i++) { 
			if (!id_used[i]) { 
				id_used[i] = true; 
				return (byte)i; 
			} 
		} 

		assert(0); 
		return false; 
	} 

	void dealloc(byte id) { 
		assert(id_used[id]); 
		id_used[id] = false; 
	} 
};
~~~

当然，亦可以用位集合减少内存。此处就不再详述了。

这个解可以传回最小id，但若是没此需要，则有更快的解。

### 栈

笔者认为，以下这个采用栈（stack）的解可能是本文最简单的一个解，同时，它的分配和释放时间复杂度皆是$\mathrm{O}(1)$，而且系数应为最低，所以是本文最高效的解。

其原理很简单，把整个$U$集合压入栈，分配的时候弹出一个 id，释放的时候压回去。

~~~cpp
// 栈 
// 分配的时间复杂度: O(1) 
// 释放的时间复杂度: O(1) 
// 状态所需内存: M + 4 字节(使用short top会是M + 2 字节) 
struct stack_manager { 
	byte ids[M]; 
	size_t top; 

	stack_manager() : top(M) { 
		for (size_t i = 0; i < M; i++) 
			ids[i] = (byte)i; 
	} 

	byte alloc() { 
		assert(top > 0); 
		return ids[--top]; // 弹出 
	} 

	void dealloc(byte id) { 
		assert(top < M); 
		ids[top++] = id; // 压入 
	} 
};
~~~

### 数组链表

而另一个接近高效的解是 [Qiaojie](http://home.cnblogs.com/Hybird3D/) 提出的，把数组当作链表。这个解的分配和释放时间复杂度亦是$\mathrm{O}(1)$。

~~~cpp
// 数组链表 (来自qiaojie) 
// 分配的时间复杂度: O(1) 
// 释放的时间复杂度: O(1) 
// 状态所需内存: M + 1 字节(若以freelist形式储存，则所需额外内存只是1字节) 
struct arraylinkedlist_manager { 
	byte next[M]; 
	byte head; 

	arraylinkedlist_manager() : head(0) { 
		// 填入完整的环 
		for(int i = 0; i < M; ++i) 
			next[i] = (byte)(i + 1); 
	} 

	byte alloc() { 
		byte id = head; 
		head = next[head]; 

		// next[id]在这里已经不需要，可以用来放短讯或其他数据，这里放置0作为测试。实际上这步是可有可无的。 
		next[id] = 0; 

		return id; 
	} 

	void dealloc(byte id) { 
		next[id] = head; 
		head = id; 
	} 
};
~~~

这个解其实可称为 free list，其优点是，`next`数组的元素若被分配，则本身可以储存其他数据。所以实际上会占用的额外内存只是1个字节！例如，可以把短讯的结构定义为:

~~~cpp
// 用于数组链表的freelist的结构例子 
union sms { 
	byte next; 
	char message[160]; 
};
~~~

此数据结构其实最适合做对象池（object pool）。

## 效能测试

以下是在i7 920、Windows 7、Visual C++ 2008 x86 模式下的结果（单位为秒）：

~~~
  0.068476 test_average(dummy)
  0.545491 test_average(linear_search)
  3.030943 test_average(binary_search)
  4.209131 test_average(sort_stl)
  0.966749 test_average(heap_stl)
  0.424917 test_average(partition_stl)
  0.208690 test_average(boolset)
  0.272523 test_average(bitset_standard)
  0.271665 test_average(bitset_intrinsic)

  0.068385 test_worst(dummy)
 27.025864 test_worst(linear_search)
 11.407150 test_worst(binary_search)
 10.122118 test_worst(sort_stl)
 13.912083 test_worst(heap_stl)
  0.887030 test_worst(partition_stl)
  0.498429 test_worst(boolset)
  0.570213 test_worst(bitset_standard)
  0.458865 test_worst(bitset_intrinsic)

  0.042507 test_manager<dummy_manager>()
  0.073745 test_manager<boolset_manager>()
  0.042462 test_manager<stack_manager>()
  0.042526 test_manager<arraylinkedlist_manager>()
~~~

当中，`dummy`/`dummy_manager`为没有实际计算的测试对象，用以量度测试本身的开销。读者比较时可把测试的时间减去相对的开销。

## 讨论

以下的表简单总括各个解的特性：

|解|传回最小id|平均时间复杂度|额外内存（字节）|
|:-:|:-:|:-:|:-:|
|线性查找|是|$\mathrm{O}(n^2)$|0|
|二分查找|是|$\mathrm{O}(n \log n)$|0|
|排序|是|$\mathrm{O}(n \log n)$（最坏$\mathrm{O}(n^2)$）|$m$或0（可改动`ids`）|
|堆|是|$\mathrm{O}(n \log n)$|$m$ 或 0（可改动`ids`）|
|剖分|是|$\mathrm{O}(n)$|$m$或0（可改动`ids`）|
|布尔集合|是|$\mathrm{O}(n)$|$m$|
|位集合|是|$\mathrm{O}(n)$|$4\lfloor(m+31)/32\rfloor$|
|布尔集合(含状态)|是|$\mathrm{O}(n)$, $\mathrm{O}(1)$|$m$|
|位集合(含状态)|是|$\mathrm{O}(n)$, $\mathrm{O}(1)$ |$4\lfloor(m+31)/32\rfloor$|
|栈|否|$\mathrm{O}(1)$, $\mathrm{O}(1)$ |$m + 4$或$m + 2$|
|数组链表|否|$\mathrm{O}(1)$, $\mathrm{O}(1)$ |$m + 1$或$1$|

原题目中的需求中谈及「……我要求你的程序尽量快，并少用内存。」但时间和空间是两个互相竞争的需求，通常难以同时满足。而在上文中，也把问题的 API 需求细分为:

1.  纯函数API
2.  可改动ids的函数API
3.  含状态API

本文列出的解并没有各方面都完美的解。例如，在无需额外内存的纯函数解里，二分查找在最坏情况下比线性查找的性能好，但平均来说却是相反。

在变动数组（或复制数组）的纯函数解里，剖分在平均和最坏情况下，性能都比排序和堆好。剖分的优点是可以不占内存（当能改动`ids`时），性能又比查找好。

布尔集合和位集合的性能在纯函数解里是最好的，但必须占一些内存（虽然当m=256，位集合只需32字节）。

含状态的解中，若需要传回最小 id，可使用布尔集合和位集合。不然，可采用栈和数组链表。若在数组链表中以 free list 使用，当然是最理想，因为这只占1字节。但栈的性能会好一点点。

## 结语

个人认为，本题是一个不错的面试题目，因为它并没有一个各方面都完美的解。这样，更可以考验应试者对算法的基础知识和编程能力。当然，笔者在编写这些程序也花了多个小时，在有限的面试时间中不太可能写这么多。但也可以用简单文字描述，或在交流中讲解一些思考方向。个人认为，理想的工程人员不但能解决问题，还会知道有其他解的存在，并去实验、分析、选择最适合某场合的解。

如果读者也想到其他的解，或对上述解的改善，希望不吝告之，本人也会尽量整理于此。

[下载源文件](/downloads/idalloc20100903.zip)
