# 关于 Red/System 中 `struct!` 的注意点

## 缘由

[上一篇](https://github.com/haolloyin/blog/blob/master/Compare-Red-System-with-C.md)提到我在用 Red/System 抄写这个[纯 C 的 JSON 教程](https://github.com/miloyip/json-tutorial)，在解析对象时遇到函数内的 `/local` 局部结构体不管怎么递归调用，其内存地址总是一样。

由于会递归调用，导致上一次调用的内容被覆盖掉。相关代码在[这里](https://github.com/haolloyin/reds-json/blob/ch06/json.reds#L413-L417)。

下面是 C 代码。

```c
#include <stdio.h>

typedef struct st {
	int i;
} st;

void f(int i)
{
    if(i > 3) return;
    
    st s;		// 局部变量
    printf("loop i: %d, &s: %p\n", i, &s);
    f(i+1);     // 递归调用，保证调用栈不一样，因此局部变量 s 结构体的地址会不同
}

int main()
{
    f(0);
    return 0;
}
```

输出 `s` 结构体每次调用的地址确实不同。 

```shell
loop i: 0, &s: 0x7fff531db9e8
loop i: 1, &s: 0x7fff531db9c8
loop i: 2, &s: 0x7fff531db9a8
loop i: 3, &s: 0x7fff531db988
```

相应的 Red/System 代码如下：

```red
Red/System []

st!: alias struct! [
    i [integer!]
]

f: func [
    i       [integer!]
    return: [integer!]
    /local
        s   [st!]	;- 局部变量结构体
][
    if i > 3 [return 0]
    
    s: declare st! 
    printf ["loop i: %d, s: %d^/" i s]
    f (i + 1)   	;- 递归调用，保证调用栈不一样，按理本地变量 s 的地址应该不同
    return 0
]

f 0
```

输出如下，可以同样是递归调用，但 `s` 这个在函数内部声明的结构体却一直是相同的地址。

```shell
loop i: 0, s: 14020
loop i: 1, s: 14020
loop i: 2, s: 14020
loop i: 3, s: 14020
```



## 如何解决？

经 bitbegin 大侠提醒，上面 Red/System 函数内用 `/local` 修饰的局部变量（仅限 `struct!` ），表现很类似 C 里用 `static` 修饰的静态变量。

且不说 Red/System 为何这么设计，我能想到的办法就是赶紧用动态内存分配来初始化这个局部 `struct!`，问题解决了。

```red
s: as st! system/stack/allocate (size? st!) ;- 栈分配

s: as st! allocate (size? st!)				;- 堆分配，即 malloc 函数
```

但是这样有点别扭，性能不是最优。

再请教晴天大侠，确认了这一点：**函数内用 /local 修饰的 struct! 结构体默认就是类似 C 的 static 变量，除非结构体内加上 value 来修饰，才会变成 C 默认 auto 自由变量。**

即上面的 `f` 函数的 `/local` 局部参数声明应该写成：

```red
f: func [
    i       [integer!]
    return: [integer!]
    /local
        s   [st! value]		;- 局部变量结构体，加了 value 修饰
][
	...
]
```

这样就完美解决问题了，不用在运行时动态分配内存了，直接在编译期决定了内存分配。



## 为何这么诡异？

在[上一篇](https://github.com/haolloyin/blog/blob/master/Compare-Red-System-with-C.md)我说过 [Red/System 的语言规范](https://static.red-lang.org/red-system-specs-light.html#section-4.7) 本身就是结构体内存的起始地址，也支持嵌套 `struct!`。

但是嵌套 `struct!` 有两种情况：

1. 如果内层的 `struct!` 成员没用 `value` 修饰，那么是保存这个 `struct!` 的地址（类似引用）。
2. 只有用 `value` 修饰时，**内层**的 `struct!` 才会把__自身整块内存__包含在**外层的 `struct!` 内。



我当初在看[语言规范的形式化定义](https://static.red-lang.org/red-system-specs-light.html#section-4.7.1)时没看得太细，它也没有覆盖各种写法：

```red
declare struct! [
   <member> [<datatype>]
   ...
]
<member>   : a valid identifier
<datatype> : integer! | byte! | pointer! [integer! | byte!] | logic! |
             float! | float32! | c-string! |
             struct! [<members>] |		;-- 重新声明一个临时的 struct!，而不是用别名
             struct! [<members>] value | function! [<spec>]
```

它举的例子是：

```red
s2: declare struct! [		;- 临时声明一个结构体变量
   a   [integer!]
   b   [c-string!]
   c   [struct! [d [integer!] e [float!]] value]	;- c 是临时声明的 struct!，末尾用 value 修饰
]
```

但其实如果再写一个例子把 c 这种临时声明的结构体简化一下就很清晰了：

```red
st!: alias [i [integer!]]	;- 声明一个 struct! 别名，可以复用这个声明

s2: declare struct! [		;- 临时声明一个结构体变量
   a   [integer!]
   b   [c-string!]
   c   [st! value]			;- c 是 st! 结构体，且用 value 修饰后变成占用空间，而不是结构体的引用
]
```



上面第二个例子，跟晴天提示的函数内的 `/local` 结构体加上 `value` 修饰才会变成类似 C 的 `auto` 变量，写法是一样的，即末尾加 `value` 来修饰结构体。

看 Red/System 关于[函数参数列表的形式化表示](https://static.red-lang.org/red-system-specs-light.html#section-6.1)：

```
<name>: func | function [
   [<attributes>]                      ;-- optional part
   "<function purpose>"                ;-- optional doc-string
   <argument> [<datatype>]
   "<argument description>"            ;-- optional doc-string
   ...
   return: [<datatype>]                ;-- returned value type (optional part)
   "<returned value description>"      ;-- optional doc-string
   /local                              ;-- local variables (optional part)
   <local> [<datatype>]
   ...
][
   <body>
]

<name>       : function's name
<attributes> : special attributes
<argument>   : function's argument indentifier
<datatype>   : integer! | byte! | logic! | pointer! [integer! | byte!] |
               float! | float32! | c-string! | struct! [<members>] |
               struct! [<members>] value	;------------ 果然如此 ------------
<local>      : local variable
<body>       : function's body code
```

果然如此，可以推测函数参数列表中如果有 `struct!` 类型的参数（不管是形参还是本地变量），它的声明跟上面的嵌套结构体声明一样，没有用 `value` 修饰就是结构体的地址（相当于引用？），否则就是结构体本身的整个内存块。

虽然上面这个结论不能解释没加 `value` 修饰局部的 `struct!` 会默认是静态局部变量，也就是不知道 Red/System 是怎么实现的，但至少能够统一起来理解了。



## 后续？

我在想既然 Red/System 的语言规范没有面面俱到，但上面我遇到的嵌套结构体、函数参数或局部变量包含结构体而显得比较反常规的情况，单元测试里应该会覆盖到的。于是用 `grep -nr '! value]'` 来搜索代码，果然有一堆用 `value` 修饰结构体的用法，只不过以前看代码没有遇到而已。

更重要的是单元测试里就有明确的[例子](https://github.com/red/red/blob/master/system/tests/source/units/struct-test.reds)，还有注释，摘抄几个如下：

1. 嵌套结构体，https://github.com/red/red/blob/master/system/tests/source/units/struct-test.reds#L66-L80

```
===start-group=== "Nested structs read/write tests"

	--test-- "s-nested-1"
	struct3: declare struct! [
		d [byte!]
		b [integer!]
		c [c-string!]
		sub [					;-- this is a reference to a struct! not a struct value
			struct! [
				e [integer!]
				f [c-string!]
			]
		]
		g [integer!]
	]
```

2. 函数参数包含 `struct!` 类型的变量，https://github.com/red/red/blob/master/system/tests/source/units/struct-test.reds#L586-L642

```red
===start-group=== "Struct passed/returned by value"	;-- 传值，但这里的值是整个结构体

	tiny!:  alias struct! [b1 [byte!]]
	small!: alias struct! [one [integer!] two [integer!]]
	big!:   alias struct! [one [integer!] two [integer!] three [float!]]

	nested1!: alias struct! [f1	[integer!] sub [tiny! value] f2	[integer!]]
	nested2!: alias struct! [f1	[integer!] sub [small! value] f2 [integer!]]
	nested3!: alias struct! [f1	[integer!] sub [big! value] f2 [integer!]]
	
	#switch OS [
		Windows  [#define STRUCTLIB-file "structlib.dll"]
		macOS	 [#define STRUCTLIB-file "libstructlib.dylib"]
		FreeBSD  [#define STRUCTLIB-file "libstruct.so"]
		#default [#define STRUCTLIB-file "libstructlib.so"]
	]

	#import [
		STRUCTLIB-file cdecl [
			returnTiny:  "returnTiny"  [return: [tiny! value]]		;-- 用 value 修饰
			returnSmall: "returnSmall" [return: [small! value]]
			returnBig:	 "returnBig"   [return: [big! value]]
			returnHuge:  "returnHuge"  [a [integer!] b [integer!] return: [huge! value]]
			returnHuge2: "returnHuge2" [h [huge! value] a [integer!] b [integer!] return: [huge! value]]
		]
	]
	
	sbvf1: func [s [tiny! value] v [integer!]][		;-- 用 value 修饰
		s/b1: #"x"
		--assert s/b1 = #"x"
		--assert v = 741
		s
	]
```



只要试过并理解单元测试里的例子 assert 的目的，就可以得出结论：

1. 没有 `value` 修饰的结构体，不管是传入参数，还是作为返回，它**都是传地址（即结构体的起始地址，相当于引用）**
2. 被 `value` 修饰的结构体，不管是传入参数，还是作为返回，它**都是传值（即复制整个结构体的内存）**

