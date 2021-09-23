> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/225190602) ![](https://pic1.zhimg.com/v2-c54c38ef454db9df906eb590a96643a8_r.jpg)

现代高级编程语言管理内存的方式分为两种：自动和手动，像 C、C++ 等编程语言使用手动管理内存的方式，工程师编写代码过程中需要主动申请或者释放内存；而 PHP、Java 和 Go 等语言使用自动的内存管理系统，有内存分配器和垃圾收集器来代为分配和回收内存。

这篇文章会结合图示用每个人都能听懂的大白话解释清楚 Go 的内存分配原理。

关于 Go 的内存分配
-----------

在`Go`语言里，从内存的分配到不再使用后内存的回收等等这些内存管理工作都是由`Go`在底层完成的。虽然开发者在写代码时不必过度关心内存从分配到回收这个过程，但是`Go`的内存分配策略里有不少有意思的设计，通过了解他们有助于我们自身的提高，也让我们能写出更高效的`Go`程序。

Go 内存管理的设计旨在在并发环境中快速运行，并与垃圾回收器集成在一起。让我们看一个简单的示例：

```
package main

type smallStruct struct {
   a, b int64
   c, d float64
}

func main() {
   smallAllocation()
}

//go:noinline
func smallAllocation() *smallStruct {
   return &smallStruct{}
}
```

函数上面的注释`//go:noinline`将禁止`Go`对该函数进行内联，这样`main`函数就会使用`smallAllocation`函数返回的指针变量，因为被多个函数使用，返回的这个变量将被分配到堆上。

关于内联的概念之前的文章有说过：

> **内联是一种手动或编译器优化，用于将简短函数的调用替换为函数体本身。这么做的原因是它可以消除函数调用本身的开销，也使得编译器能更高效地执行其他的优化策略。**  

所以如果上面的例子不干预编译器的话，编译器通过内联将`smallAllocation`函数体里的内容直接放到`main`函数里，这样就不会产生`smallAllocation`这个函数的调用了，所有的变量都是`main`函数内这个范围使用的，也就不在需要将变量往堆上分配了。

继续说上面那个例子，通过逃逸分析命令 **go tool compile -m main.go** 可以确认我们上面的分析，`&smallStruct{}`会被分配到堆上去。

```
➜ go tool compile -m main.go
main.go:12:6: can inline main
main.go:10:9: &smallStruct literal escapes to heap
```

借助命令 **go tool compile -S main.go**，可以显示该程序的汇编代码，也可以明确地向我们展示内存的分配：

```
0x001d 00029 (main.go:10)       LEAQ    type."".smallStruct(SB), AX
0x0024 00036 (main.go:10)       PCDATA  $2, $0
0x0024 00036 (main.go:10)       MOVQ    AX, (SP)
0x0028 00040 (main.go:10)       CALL    runtime.newobject(SB)
```

内置函数`newobject`会通过调用另外一个内置函数`mallocgc`在堆上分配新内存。在 Go 里面有两种内存分配策略，一种适用于程序里小内存块的申请，另一种适用于大内存块的申请，大内存块指的是大于 32KB。

下面我们来细聊一下这两种策略。

小于 32KB 内存块的分配策略
----------------

当程序里发生了`32kb`以下的小块内存申请时，Go 会从一个叫做的`mcache`的本地缓存给程序分配内存。这个本地缓存`mcache`持有一系列的大小为`32kb`的内存块，这样的一个内存块里叫做`mspan`，它是要给程序分配内存时的分配单元。

![](https://pic1.zhimg.com/v2-37d0de4191cfaa410d8a84f1b625e30c_r.jpg)

在 Go 的调度器模型里，每个线程`M`会绑定给一个处理器`P`，在单一粒度的时间里只能做多处理运行一个`goroutine`，每个`P`都会绑定一个上面说的本地缓存`mcache`。当需要进行内存分配时，当前运行的`goroutine`会从`mcache`中查找可用的`mspan`。从本地`mcache`里分配内存时不需要加锁，这种分配策略效率更高。

那么有人就会问了，有的变量很小就是数字，有的却是一个复杂的结构体，申请内存时都分给他们一个`mspan`这样的单元会不会产生浪费。其实`mcache`持有的这一系列的`mspan`并不都是统一大小的，而是按照大小，从 8 字节到 32KB 分了大概 70 类的`msapn`。

![](https://pic2.zhimg.com/v2-b9a81ceb0ce847704860ad5e64add7c5_b.jpg)

就文章开始的那个例子来说，那个结构体的大小是 32 字节，正好 32 字节的这种`mspan`能满足需求，那么分配内存的时候就会给它分配一个 32 字节大小的`mspan`。

![](https://pic2.zhimg.com/v2-2936fba72393198ee4057cb378c29b91_r.jpg)

现在，我们可能会好奇，如果分配内存时`mcachce`里没有空闲的 32 字节的`mspan`了该怎么办？`Go`里还为每种类别的`mspan`维护着一个`mcentral`。

`mcentral`的作用是为所有`mcache`提供切分好的`mspan`资源。每个`central`会持有一种特定大小的全局`mspan`列表，包括已分配出去的和未分配出去的。 每个`mcentral`对应一种`mspan`，当工作线程的`mcache`中没有合适（也就是特定大小的）的`mspan`时就会从`mcentral` 去获取。`mcentral`被所有的工作线程共同享有，存在多个`goroutine`竞争的情况，因此从`mcentral`获取资源时需要加锁。

`mcentral`的定义如下：

```
//runtime/mcentral.go

type mcentral struct {
    // 互斥锁
    lock mutex 

    // 规格
    sizeclass int32 

    // 尚有空闲object的mspan链表
    nonempty mSpanList 

    // 没有空闲object的mspan链表，或者是已被mcache取走的msapn链表
    empty mSpanList 

    // 已累计分配的对象个数
    nmalloc uint64 
}
```

`mcentral`里维护着两个双向链表，**nonempty** 表示链表里还有空闲的`mspan`待分配。**empty** 表示这条链表里的`mspan`都被分配了`object`。

![](https://pic4.zhimg.com/v2-cf90e6add06c4d2c1b318503d1330a83_r.jpg)

如果上面我们那个程序申请内存的时候，`mcache`里已经没有合适的空闲`mspan`了，那么工作线程就会像下图这样去`mcentral`里去申请。

简单说下`mcache`从`mcentral`获取和归还`mspan`的流程：

*   获取 加锁；从`nonempty`链表找到一个可用的`mspan`；并将其从`nonempty`链表删除；将取出的`mspan`加入到`empty`链表；将`mspan`返回给工作线程；解锁。
*   归还 加锁；将`mspan`从`empty`链表删除；将`mspan`加入到`nonempty`链表；解锁。

![](https://pic4.zhimg.com/v2-13c262f71e7b3b28c35212001138b7fb_r.jpg)

当`mcentral`没有空闲的`mspan`时，会向`mheap`申请。而`mheap`没有资源时，会向操作系统申请新内存。`mheap`主要用于大对象的内存分配，以及管理未切割的`mspan`，用于给`mcentral`切割成小对象。

![](https://pic2.zhimg.com/v2-5e475c825106b890397e6a1bd09d6999_r.jpg)

同时我们也看到，`mheap`中含有所有规格的`mcentral`，所以，当一个`mcache`从`mcentral`申请`mspan`时，只需要在独立的`mcentral`中使用锁，并不会影响申请其他规格的`mspan`。

上面说了每种尺寸的`mspan`都有一个全局的列表存放在`mcentral`里供所有线程使用，所有`mcentral`的集合则是存放于`mheap`中的。 `mheap`里的`arena` 区域是真正的堆区，运行时会将 `8KB` 看做一页，这些内存页中存储了所有在堆上初始化的对象。运行时使用二维的 `[runtime.heapArena](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go%23L217)` 数组管理所有的内存，每个 `[runtime.heapArena](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go%23L217)` 都会管理 64MB 的内存。

![](https://pic4.zhimg.com/v2-97daf59507fd1d0aae21c0365c7d7f5b_b.jpg)

如果 `arena` 区域没有足够的空间，会调用 `[runtime.mheap.sysAlloc](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/6dd11bcb35cba37f5994c1b9aaaf7d2dc13fd7cf/src/runtime/malloc.go%23L617)` 从操作系统中申请更多的内存。

大于 32KB 内存块的分配策略
----------------

Go 没法使用工作线程的本地缓存`mcache`和全局中心缓存`mcentral`上管理超过 32KB 的内存分配，所以对于那些超过 32KB 的内存申请，会直接从堆上 (`mheap`) 上分配对应的数量的内存页（每页大小是 8KB）给程序。

![](https://pic1.zhimg.com/v2-7414f28dd531b5cb69a033e44080a664_r.jpg)

总结
--

我们把内存分配管理涉及的所有概念串起来，可以勾画出 Go 内存管理的一个全局视图：

![](https://pic3.zhimg.com/v2-bfb8df4d82cc8b654a2e12d0ff8cd19a_r.jpg)

Go 语言的内存分配非常复杂，这个文章从一个比较粗的角度来看 Go 的内存分配，并没有深入细节。一般而言，了解它的原理，到这个程度也就可以了（应付面试）。

总结起来关于 Go 内存分配管理的策略有如下几点：

*   Go 在程序启动时，会向操作系统申请一大块内存，由`mheap`结构全局管理。
*   Go 内存管理的基本单元是`mspan`，每种`mspan`可以分配特定大小的`object`。
*   `mcache`, `mcentral`, `mheap`是`Go`内存管理的三大组件，`mcache`管理线程在本地缓存的`mspan`；`mcentral`管理全局的`mspan`供所有线程使用；`mheap`管理`Go`的所有动态分配内存。
*   一般小对象通过`mspan`分配内存；大对象则直接由`mheap`分配内存。

### 相关阅读

[白话 Go 语言内存管理三部曲（二）解密栈内存管理](https://zhuanlan.zhihu.com/p/237870981)

[白话 Go 语言内存管理三部曲（三）垃圾回收原理](https://zhuanlan.zhihu.com/p/264789260)

### 参考链接

> 看到这里了，如果喜欢我的文章可以帮我点个赞，我会每周通过技术文章分享我的所学所见，感谢你的支持。微信搜索公众号「网管叨 bi 叨」第一时间获取我的文章推送。