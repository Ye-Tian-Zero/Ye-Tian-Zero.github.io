---
title: 编译期二进制转换
date: 2019-10-08 10:53:58
tags: 
- c++
- meta-programming
- demo
categories:
- programming
- c++
top_img: /img/c++_template_binary_banner.png
cover: /img/c++_template_binary_index.png
description: 
- C++ 实现编译期的二进制转换。
---
<!-- more -->
# 编译期二进制转换

第一篇博客，仅用于测试和学习 hexo 系统，代码与内容都非常简单。
实现如下

```c++
template<size_t N, size_t base>
struct BinaryImpl {
  BinaryImpl() = delete;
  ~BinaryImpl() = delete;
  //data check
  static_assert(N % 10 == 0 || N % 10 == 1, "Value must be 0 or 1.");
  using type = BinaryImpl<N, base>;
  static const size_t value = (N % 10) * base + BinaryImpl<N / 10, base << 1>::value;
};

template<size_t base>
struct BinaryImpl<0, base> {
  BinaryImpl() = delete;
  ~BinaryImpl() = delete;
  using type = BinaryImpl<0, base>;
  static const size_t value = 0;
};

template<size_t N>
struct Binary{
  Binary() = delete;
  ~Binary() = delete;
  using type = Binary<N>;
  static const size_t value = BinaryImpl<N, 1>::value;
};
```

很低级，不描述了，就这么用：

```c++
std::cout << binary<11110011>::value << std::endl; //输出243
```

二进制长度不能超过 size_t 的长度。

vim 的 markdown 语法默认设置真反人类。
