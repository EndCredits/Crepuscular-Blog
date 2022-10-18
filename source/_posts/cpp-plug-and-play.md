---
title: C++ 光速入手
date: 2022-10-12 22:56:55
tags: CSP/ACM
---

光速入门 C++ ，方便算法竞赛使用... 好吧... 其实是老师让的，给学校后来有志于参加算法竞赛的同志们留下前人的经验...

该帖中所有的内容均于 ```clang 15.0.0``` ( llvm ```7c7702b318503e27ebc3c795a827aa85135cc130``` ) 下编译通过

( 虽然人很菜就是了 )

( 又菜又爱玩.JPG )

---
## Hello World!

> "欢迎来到世界"

```C++
#include <iostream>

using namespace std;

int main()
{
    cout<<"Hello World!"<<endl;
    return 0;
}
```
听说这是所有程序员的第一个程序？

## C++ 语法基础

### 预处理

预处理是编译的一个过程，通常由编译器的预处理器完成，C++ 中，传递给预处理器的指令通常用 ```#``` 标记出来

最常见的预处理指令应该就是 ```#include``` 了，它的作用是告诉预处理器去包含一个文件的内容，这样你就可以使用里面预先定义好的内容，比如

```C++
#include <iostream>
```
这样便是包含了 ```iostream``` 这个库文件，它会在预处理时被包含入你的程序文件，对于竞赛中常用的头文件会有

```C++
#include <iostream>
#include <cstdio>
#include <cstdlib>
#include <algorithm>
#include <stack>
#include <queue>
#include <vector>
#include <set>
#include <map>
```

以及一些竞赛中会允许你使用所谓的万能头文件，它包含了整个 C++ 标准库

```C++
#include <bits/stdc++.h>
```
另一个比较常用的预处理指令是 ```#define```，它的作用是定义一个宏

```C++
#define MATH_PI 3.14
```

这样我们就定义了一个叫做 ```MATH_PI``` 的宏，它指代的意思就是 ```3.14```，在预处理阶段，代码里所有的 ```MATH_PI``` 都会被替换成 ```3.14```

:::warning 不要用 ```#define``` 指令来替代常量的定义
有的同学学会 ```#define``` 之后就总想用它来替代常量的定义，这实际上是非常不好的，因为 ```#define``` 它是一个预处理指令，也就是说你定义的这个宏最终不会被导出到符号表之中，对调试过程会造成很大的麻烦，调试工具或者编译器只会告诉你是什么值不正确，而不知道是哪个符号出了问题，特别是在实际的工程项目之中，常量的定义跟具体的代码实现通常放在不同的文件中，调试就更加困难了，所以请尽量避免使用 ```#define``` 定义常量，而应该使用 ```const``` 关键字来修饰一个常量
:::

它也可以定义一个符号，这个通常与条件编译一起使用

```C++
#define LOCAL_BUILD_TYPE_DEBUG

#ifdef LOCAL_BUILD_TYPE_DEBUG
// do someting to help debug
#endif
// other function code block
```

也就是说，当你定义 ```LOCAL_BUILD_TYPE_DEBUG``` 这个符号时，```#ifdef``` 和 ```endif``` 之间的语句就会被编译，如果没有这个符号，那其中的内容就会被预处理器去掉，对于调试用途来说，这样很方便

### 基本结构

每一个 C++ 程序都以 ```main()``` 函数作为该程序的入口点，每个文件有且只能有一个 ```main()``` 函数

### 标准 IO 操作

C++ 中，我们使用 ```cout``` 向标准输出 (stdout) 上输出信息，使用 ```cin``` 从标准输入 (stdin) 上输入信息，它们都被定义于 ```iostream``` 库中，下面的小例子实现了整数 echo 功能

```C++
#include <iostream>

using namespace std;

int main()
{
    int a, b;
    cin>>a>>b;
    cout<<a<<b<<endl;
    return 0;
}
```
使用非常方便

当然我们也可以使用 C 标准库中的输入输出函数，也就是 ```printf``` 和 ```scanf``` ，定义于 ```cstdio``` 头文件中，下面的小程序实现了与上面的程序相同的功能

```C++
#include <cstdio>

int main()
{
    int a, b;
    scanf("%d%d", &a, &b);
    printf("%d%d", a, b);
    return 0;
}
```

上面的代码块中，```printf``` 与 ```scanf``` 接收几个参数，双引号内的内容是输出的内容以及格式化输出的内容，后面是要输入/输出的操作对象

你也许会注意到 ```scanf``` 中接受的变量参数带 ```&``` 而 ```printf``` 中不带，这一点会在以后讲解，现在就先记住这个特性，它与函数参数的传递方式和指针有关

:::warning 关于 ```cin/cout``` 与 ```scanf/printf``` 的效率问题
相比于  ```scanf/printf``` ， ```cin/cout``` 的执行速度要快，如果某个题目有巨量的数据需要输入，直接使用 ```cin/cout``` 可能会 TLE，这时候在确保算法及优化正确的情况下，可以考虑换用 ```scanf/printf``` 或者对 ```cin/cout``` 进行优化
:::

:::tip 关于 ```iostream``` 的优化方法
这部分暂时不看也没关系，第一次读的读者可以跳过阅读下一小节
C++ 为保证对 C 的兼容性做了许多努力，其中就包括 ```iostream``` 与 ```cstdio``` 的兼容

```C++
std::ios::sync_with_stdio(false)
```

这一项默认是打开的，目的是绑定 ```iostream``` 与 ```cstdio``` 使用的 ```stdin/stdout``` 流，以此保证 ```printf/scanf``` 与 ```cout/cin``` 同时使用时不会导致混乱，这也是 ```iostream``` 慢的主要原因
需要注意的是，关闭同步之后 ```scanf``` 就不能与 ```cout``` 一同使用，```printf``` 也不能与 ```cin``` 一起使用，但是 ```scanf``` 可以与 ```cin``` 一同使用，```printf``` 可以与 ```cout``` 一同使用
:::

至于其中的 ```%d``` 和 ```\n``` ，分别是格式化输出字符和转义字符，下面列出常用的格式化输出字符

| 格式化字符     | 作用                             |
|-------------:|:--------------------------------|
|%c            | 输入/输出单个字符                  |
|%s            | 输入/输出一个字符串                |
|%d            | 转换有符号整数为十进制表示          |
|%nd           | 转换 n 位有符号整数为十进制表示，输入/输出位数少于 n 则补前导零，超出则丢弃 |
|%lf           | 转换
|%
