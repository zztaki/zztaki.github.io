---
title: C++ primer笔记
date: 2023-3-5 21:26:46
description: C++ primer笔记
tags:
- C/C++
categories:
- C/C++
---

# C++ primer笔记

## 1 开始

### 1.2 初识输入输出

1. cout 输出语句时，尽量要以 endl 结束，从而刷新缓存区，否则如果出现程序崩溃，输出可能还留在缓冲区中。

## 2 变量和基本类型

### 2.2 变量

1. extern：
    - 只能作用在函数体外。
    - extern 声明一个变量，如果赋予变量初值，则表示定义。

### 2.5 处理类型

1. 类型别名：

    - typedef double wages;
    - using my_double = double;
    - 在使用包含指针的类型别名的时候，不能简单地将类型别名替换成原类型，见下图。

    {% asset_img typedef指针类型别名与const同时使用.jpg typedef指针类型别名与const同时使用 %}

2. auto

3. decltype：`decltype(expression)` 返回表达式结果类型（可能是左值）。

## 3字符串、向量和数组

## 4 表达式

### 4.11 类型转换

1. 隐式转换
2. 显示转换
    - 强制类型转换：(double)
    - 命名的强制类型转换：cast-name\<type\>(expression)
        - static_cast：常用于 void* 的具体化，较大的算术类型降级。
        
        - dynamic_cast：支持运行时类型识别。详细见 19.2 节。
        
        - const_cast：将 const 对象转换为非 const 对象，常常用于有函数重载的上下文。
        
            {% asset_img const_cast在函数重载中的应用.jpg %}
        
        - reinterpret_cast：**最好别用**。

## 5 语句

## 6 函数

## 7 类

一个好的习惯是：将类中的成员函数声明和定义分离，并尽量设置类的成员变量为私有。如此一来，用户代码只能通过声明的接口对类进行操作，类的实现者对类接口的修改不会影响到用户代码。

{% asset_img 将类的声明和定义分离.jpg %}

此时，将类以及与该类相关的友元接口声明在 `*.h` 文件中，并暴露给用户代码，即用户代码只需 `include "*.h"` 即可；类及相关友元接口的实现则定义在 `*.cpp` 中。

### 7.3 类的其它特性

1. mutable 数据成员可以被 const 成员函数修改。

    {% asset_img mutable.jpg mutable %}

### 7.5 再探构造函数

#### 7.5.3 使用默认构造函数

`Sales_data obj;` √

`Sales_data obj();` ×，这是在声明函数。

#### 7.5.4 隐式的类类型转换

1. 只接受一个实参的构造函数，也叫做**转换构造函数**。

    `Sales_data obj = string("123");`

2. 但是，只允许**一步类类型转化**，比如 `Sales_data obj = "123"；` 就不被允许。

3. 抑制构造函数定义的隐式转换：使用 `explicit` 关键字

    {% asset_img explicit.jpg explicit %}

## 8 IO 库

## 9 顺序容器

## 10 泛型

