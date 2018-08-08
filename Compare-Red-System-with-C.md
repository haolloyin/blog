# 对比 Red/System 与 C


## 资料

- [red，不红不专，但性感](https://zhuanlan.zhihu.com/p/27998473)，从各方面介绍 Red 和 Red/System
- [Red/System 语言规范](https://static.red-lang.org/red-system-specs-light.html)，前半部分的[中文翻译](https://github.com/red/red/wiki/%5Bzh-hans%5D-Red-System-Language-Specification-Chinese-Traslation)
- [Notes about Porting C code to Red System](https://github.com/red/red/wiki/Notes-about-Porting-C-code-to-Red-System)，晴天刚写的关于 Red/System 与 C/C++ 的一些兼容事项，必看
- [用 C 从零开始实现 JSON 库的教程](https://github.com/miloyip/json-tutorial)，本篇记录源于我用 `Red/System` [抄了一遍](https://github.com/haolloyin/reds-json)之后的体会
- [Red 官网](https://red-lang.org)，[Red 代码风格](https://doc.red-lang.org/zh-hans/style-guide.html)（Red/System 与此类似）

## 为什么用 Red/System？

Red/System 是与 C 相同层次的编程语言，所以我才能用 Red/System 来练习上述的 JSON 教程。
虽然我的 Red/System 是现学现卖，也没有 C 的经验，但两者对比一下还是能够衬托出 Red/System 的一些优点（或者说是我个人偏好的特点）。

- **语法简洁、明确、易懂**
- **几乎没有语法糖** -- 这也导致 Red/System 写法单一，不用纠结指针的指针、函数指针（虽然短期内不会遇到）。
- **工具链小巧，支持跨平台交叉编译** -- 1.2 MB 的二进制程序足矣，不用折腾其他东西。
- **与 C/C++ 无缝兼容** -- 只要 import 对应的 dll 文件，写上对应的桥接代码，就能在 Red/System 中使用。
- **有命名空间**

## 使用 Red/System 时遇到的问题

- **嵌套结构体不支持[前向声明](https://zh.wikipedia.org/wiki/%E5%89%8D%E5%90%91%E5%A3%B0%E6%98%8E)**，只能用通用指针 `byte-ptr!` 来代替，使用时再转型成对应的结构体。
```rebol
A!: alias struct! [s [B!]]  ; Compilation Error: invalid struct syntax: [s [B!]]
B!: alias struct! [s [A!]]

A!: alias struct! [p [byte-ptr!]] ; p 是通用指针，用 as B! p 来转型成对应的 B!
B!: alias struct! [s [A!]]
```

- **无法导入 C 标准库里的常量** -- 例如 C 标准库函数 [strtod](https://zh.cppreference.com/w/c/string/byte/strtod) 会设置 `errno` 宏为 `HUGE_VAL / HUGE_VALF / HUGE_VALL`，但这 4 个常量似乎都无法获取到。

- **不支持[左值](https://zh.wikipedia.org/wiki/%E5%80%BC_(%E9%9B%BB%E8%85%A6%E7%A7%91%E5%AD%B8))** -- 这会导致有些赋值语句必须通用一个临时变量来中转。例如：
```rebol
a: [1 2 3]
i: 1
a/i: 11     ; ok
a/(i+1): 22 ; Compilation Error: attempt to use pointer indexing with a non-integer! value
```

- **没有 const、static、extern 等修饰符** -- 不过我认为 `const` 还是可以考虑支持的，否则写代码时真得注意。

- **部分转义字符不知道在哪里可以找到** -- 在 [Rebol 的文档](http://www.rebol.com/docs/core23/rebolcore-16.html#section-2.11.2) 和 [Red/System 的源码](https://github.com/red/red/blob/master/system/runtime/common.reds#L68-L75) 可以找到一部分。

- **不能生成汇编，没有符号表，不能 gdb，没有 __FILE__、__LINE__ 等等** -- 这些跟调试相关的几乎为零，毕竟是 Red/System 不是 Red 生态的最重点，对 Red 开发团队来说够用就行。


## Red/System 的特点

1. 一切都是传值
2. 基础数据类型只有 `integer! | byte! | float! | float32! | pointer!`，指针只能用于前面 4 个
3. 没有指针的指针
4. 结构体 `struct!` 本身就是指针，指向结构体所在内存的起始地址
5. 综上，如果要实现指针的指针，或函数返回多个值，可以考虑用 `struct!` 来包一层，例如 `Red/System` 里的 [str-array](https://github.com/red/red/blob/master/system/runtime/common.reds#L79-L81)，[源码实例](https://static.red-lang.org/red-system-specs-light.html#section-13.2)
6. 函数内的 `/local` 局部变量支持类型推断，可以稍微偷懒一下
7. [c-string!](https://static.red-lang.org/red-system-specs-light.html#section-4.6) 是 `[pointer! [byte!]]` 末尾加 `null` 字节（即 [#"^(00)"](https://github.com/red/red/blob/master/system/runtime/common.reds#L34)）
8. `Red/System` 的源码中已经定义了一些[常用的宏](https://github.com/red/red/blob/master/system/runtime/common.reds#L13-L81)，在语言规范文档里不一定能看到
9. `Red/System` 的嵌套 `struct!` 支持另一个结构体的指针，或包含整个结构体，这个在[文档](https://static.red-lang.org/red-system-specs-light.html#section-4.7)里也不容易看出来，如下：
```rebol

A!: alias struct! [
    i [integer!]
    i [integer!]
    k [int-ptr!]
]

B!: alias struct! [
    i [integer!]
    a [A!]          ;- 保存 A! 结构体的地址
]

C!: alias struct! [
    i [integer!]
    a [A! value]    ;- 加 value 修饰
]

print-line ["size? integer! is " size? integer!]    ;- 4
print-line ["size? A! is " size? A!]                ;- 12
print-line ["size? B! is " size? B!]                ;- 8
print-line ["size? C! is " size? C!]                ;- 16
```

