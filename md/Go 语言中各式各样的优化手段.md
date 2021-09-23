> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/403417640)

作者：korzhao，腾讯 QQ 音乐后台开发工程师

总结了一些在维护 go 基础库过程中, 用到或者见到的性能优化技巧。时间仓促，可能有一些错误，欢迎一起讨论

### **常规手段**

### **1.sync.Pool**

临时对象池应该是对可读性影响最小且优化效果显著的手段。基本上，业内以高性能著称的开源库，都会使用到。

最典型的就是 **[fasthttp](https://link.zhihu.com/?target=https%3A//github.com/valyala/fasthttp/)** 了，它几乎把所有的对象都用`sync.Pool`维护。但这样的复用不一定全是合理的。比如在`fasthttp`中，传递上下文相关信息的`RequestCtx`就是用`sync.Pool`维护的，这就导致了你不能把它传递给其他的`goroutine`。如果要在`fasthttp`中实现类似接受请求 -> 异步处理的逻辑, 必须得拷贝一份`RequestCtx`再传递。这对不熟悉`fasthttp`原理的使用者来讲，很容易就踩坑了。

还有一种利用`sync.Pool`特性，来减少锁竞争的优化手段，也非常巧妙。另外，在优化前要善用`go逃逸检查`分析对象是否逃逸到堆上，防止负优化。

### **2.string2bytes & bytes2string**

这也是两个比较常规的优化手段，核心还是复用对象，减少内存分配。在 go 标准库中也有类似的用法 **[gostringnocopy](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/runtime/string.go%23L461)**，要注意`string2bytes`后，不能对其修改。

`unsafe.Pointer`经常出现在各种优化方案中，使用时要非常小心。这类操作引发的异常，通常是不能`recover`的。

### **3. 协程池**

绝大部分应用场景，go 是不需要协程池的。当然，协程池还是有一些自己的优势：

1.  可以限制`goroutine`数量，避免无限制的增长。
2.  减少栈扩容的次数。
3.  频繁创建`goroutine`的场景下，资源复用，节省内存。（需要一定规模。一般场景下，效果不太明显）

go 对`goroutine`有一定的复用能力。所以要根据场景选择是否使用协程池，不恰当的场景不仅得不到收益，反而增加系统复杂性。

### **4. 反射**

go 里面的反射代码可读性本来就差，常见的优化手段进一步牺牲可读性。而且后续马上就有泛型的支持，所以若非必要，建议不要优化反射部分的代码

比较常见的优化手段有：

1.  缓存反射结果，减少不必要的反射次数。例如 **[json-iterator](https://link.zhihu.com/?target=https%3A//github.com/json-iterator/go)**
2.  直接使用`unsafe.Pointer`根据各个字段偏移赋值
3.  消除一般的`struct`反射内存消耗 **[go-reflect](https://link.zhihu.com/?target=https%3A//github.com/goccy/go-reflect)**
4.  避免一些类型转换，如`interface->[]byte`。可以参考 **[zerolog](https://link.zhihu.com/?target=https%3A//github.com/rs/zerolog/blob/master/array.go%23L39)**

### **5. 减小锁消耗**

并发场景下，对临界区加锁比较常见。带来的性能隐患也必须重视。常见的优化手段有：

1.  减小锁粒度: **[go 标准库](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/math/rand/rand.go%23L387)**当中，`math.rand`就有这么一处隐患。当我们直接使用`rand`库生成随机数时，实际上由全局的`globalRand`对象负责生成。`globalRand`加锁后生成随机数，会导致我们在高频使用随机数的场景下效率低下。  
    
2.  atomic: 适当场景下，用原子操作代替互斥锁也是一种经典的`lock-free`技巧。  
    标准库中`sync.map`针对`读操作`的优化消除了`rwlock`，是一个标准的案例。对它的介绍文章也比较多，不在赘述。  
    `prometheus`里的组件`histograms直方图`也是一个非常巧妙的设计。  
    一般的开源库，比如 **[go-metrics](https://link.zhihu.com/?target=https%3A//github.com/rcrowley/go-metrics)**, trpc-go-metrics 都是直接在这里使用了`互斥锁`。指标上报作为一个高频操作，在这里加锁，对系统性能影响可想而知。  
    参考`sync.map`里冗余 map 的做法，`prometheus`把原来`histograms`的计数器也分为两个：`cold`和`hot`，还有一个`hotIdx`用来表示哪个计数器是`hot`。业务代码上报指标时，用`atomic`原子操作对`hot`计数器累加向`prometheus`服务上报数据时，更改`hotIdx`，把原来的热数据变为冷数据，作为上报的数据。然后把现在冷数据里的值，累加到热数据里，完成一次冷热数据的更新替换。还有一些状态等待，结构体内存布局的介绍，不再赘述。具体可以参考 **[Lock-free Observations for Prometheus Histograms](https://link.zhihu.com/?target=https%3A//grafana.com/blog/2020/01/08/lock-free-observations-for-prometheus-histograms/)**  
    

### **另类手段**

### **1. golink**

**[golink](https://link.zhihu.com/?target=https%3A//golang.org/cmd/compile/)** 在官方的文档里有介绍，使用格式：

```
//go:linkname FastRand runtime.fastrand
func FastRand() uint32
```

主要功能就是让编译器编译的时候，把当前符号指向到目标符号。上面的函数`FastRand`被指向到`runtime.fastrand`

`runtime`包生成的也是伪随机数，和`math`包不同的是，它的随机数生成使用的上下文是来自当前`goroutine`的，所以它不用加锁。正因如此，一些开源库选择直接使用`runtime`的随机数生成函数。性能对比如下：

```
Benchmark_MathRand-12       84419976            13.98 ns/op
Benchmark_Runtime-12        505765551           2.158 ns/op
```

还有很多这样的例子，比如我们要拿时间戳的话，可以标准库中的`time.Now()`，这个库在会有两次系统调用`runtime.walltime1`和`runtime.nanotime`，分别获取时间戳和程序运行时间。大部分场景下，我们只需要时间戳，这时候就可以直接使用`runtime.walltime1`。性能对比如下：

```
Benchmark_Time-12       16323418            73.30 ns/op
Benchmark_Runtime-12    29912856            38.10 ns/op
```

同理，如果我们需要统计某个函数的耗时，也可以直接调用两次`runtime.nanotime`然后相减，不用再调用两次`time.Now`

```
//go:linkname nanotime1 runtime.nanotime1
func nanotime1() int64
func main() {
    defer func( begin int64) {
        cost := (nanotime1() - begin)/1000/1000
        fmt.Printf("cost = %dms \n" ,cost)
    }(nanotime1())

    time.Sleep(time.Second)
}

运行结果：cost = 1000ms
```

系统调用在 go 里面相对来讲是比较重的。`runtime`会切换到`g0`栈中去执行这部分代码，`time.Now`方法在`go<=1.16`中有两次连续的系统调用。

不过，go 官方团队的 lan 大佬已经发现并提交优化 **[pr](https://link.zhihu.com/?target=https%3A//go-review.googlesource.com/c/go/%2B/314277)**。优化后，这两次系统调将会合并在一起，减少一次`g0`栈的切换。

> `g0`栈切换背景可以参考`GMP`调度相关知识，不再赘述  

linkname 为我们提供了一种方法，可以直接调用 go 标准库里的`未导出方法`，可以读取`未导出变量`。使用时要注意 go 版本更新后，是否有兼容问题，毕竟 go 团队并没有保证这些未导出的方法变量后续不会变更。

还有一些其他奇奇怪怪的用法：

1.  **[reflect2](https://link.zhihu.com/?target=https%3A//github.com/modern-go/reflect2/blob/master/type_map.go%23L12)** 包，创建`reflect.typelinks`的引用，用来读取所有包中`struct`的定义
2.  创建`panic`的引用后，用一些`hook`函数重定向`panic`，这样你的程序`panic` 后会走到你的自定义逻辑里
3.  `runtime.main_inittask`保存了程序初始化时，`init`函数的执行顺序，之前版本没有`init`过程 debug 功能时，可以用它来打印程序`init`调用链。最新版本已经有官方的调试方案：`GODEBUG=inittracing=1`开启`init`
4.  `runtime.asmcgocall`是`cgo`代码的实际调用入口。有时候我们可以直接用它来调用`cgo`代码，避免`goroutine`切换, 具体会在`cgo`优化部分展开

### **2. log - 函数名称行号的获取**

虽然很多高性能的日志库，默认都不开启记录行号。但实际业务场景中，我们还是觉得能打印最好。

在 **[runtime](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/runtime/extern.go%23L220)** 中，函数行号和函数名称的获取分为两步：

1.  `runtime`回溯`goroutine`栈，获取上层调用方函数的的程序计数器（pc）。
2.  根据 pc，找到对应的`funcInfo`, 然后返回行号名称

经过 pprof 分析。第二步性能占比最大，约 60%。针对第一步，我们经过多次尝试，并没有找到有效的办法。但是第二步很明显，我们不需要每次都调用`runtime`函数去查找`pc`和函数信息的，我们可以把第一次的结果缓存起来，后面直接使用。这样。第二步约 60% 的消耗就可以去掉。

```
var(
    m sync.Map
)
func Caller(skip int)(pc uintptr, file string, line int, ok bool){
    rpc := [1]uintptr{}
    n := runtime.Callers(skip+1, rpc[:])
    if n < 1 {
        return
    }
    var (
        frame  runtime.Frame
        )
    pc  = rpc[0]
    if item,ok:=m.Load(pc);ok{
        frame = item.(runtime.Frame)
    }else{
        tmprpc := []uintptr{
            pc,
        }
        frame, _ = runtime.CallersFrames(tmprpc).Next()
        m.Store(pc,frame)
    }
    return frame.PC,frame.File,frame.Line,frame.PC!=0
}
```

压测数据如下，优化后稍微减轻这部分的负担，同时消除掉不必要的内存分配。

```
BenchmarkCaller-8       2765967        431.7 ns/op         0 B/op          0 allocs/op
BenchmarkRuntime-8      1000000       1085 ns/op         216 B/op          2 allocs/op
```

### **3.cgo**

`cgo`的支持让我们可以在 go 中调用`c++`和`c`的代码，但`cgo`的代码在运行期间不受 go 调度器的管理，为了防止`cgo`调用引起调度阻塞，`cgo`调用会切换到`g0`栈执行，并独占`m`。由于`runtime`设计时没有考虑`m`的回收，所以运行时间久了之后，会发现有`cgo`代码的程序，线程数都比较多。

用 go 的编译器转换包含`cgo`的代码：

```
go tool cgo main.go
```

转换后看代码，`cgo`调用实际上是由`runtime.cgocall`发起，而`runtime.cgocall`调用过程主要分为以下几步：

1.  entersyscall(): 保存上下文，标记当前 m`incgo`独占`m`，跳过垃圾回收，
2.  osPreemptExtEnter：标记异步抢占，使异步抢占逻辑失效
3.  asmcgocall：真正的 cgo call 入口，切换到`g0`执行`c`代码
4.  恢复之前的上下文，清理标记

对于一些简单的`c`函数，我们可以直接用`asmcgocall`调用，避免来回切换

```
package main

/*
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
struct args{
    int p1,p2;
    int r;
};
int add(struct args* arg) {
    arg->r= arg->p1 + arg->p2;
    return 100;
}
*/
import "C"
import (
    "fmt"
    "unsafe"
)
//go:linkname asmcgocall runtime.asmcgocall
func asmcgocall(unsafe.Pointer, uintptr) int32

func main() {
    arg := C.struct_args{}
    arg.p1 = 100
    arg.p2 = 200
    //C.add(&arg)
    asmcgocall(C.add,uintptr(unsafe.Pointer(&arg)))
    fmt.Println(arg.r)
}
```

压测数据如下：

```
BenchmarkCgo-12             16143393    73.01 ns/op     16 B/op        1 allocs/op

BenchmarkAsmCgoCall-12      119081407   9.505 ns/op     0 B/op         0 allocs/op
```

### **4.epoll**

`runtime`对网络 io，以及定时器的管理，会放到自己维护的一个 epoll 里，具体可以参考`runtime/netpool`。在一些高并发的网络 io 中，有以下几个问题：

1.  需要维护大量的协程去处理读写事件
2.  对连接的状态无感知，必须要等待`read`或者`write`返回错误才能知道对端状态，其余时间只能等待
3.  原生的`netpool`只维护一个`epoll`，没有充分发挥多核优势

基于此，有很多项目用`x/unix`扩展包实现了自己的基于 epoll 的网络库，比如潘神的 **[gnet](https://link.zhihu.com/?target=https%3A//github.com/panjf2000/gnet)**，还有字节跳动的 **[netpoll](https://link.zhihu.com/?target=https%3A//github.com/cloudwego/netpoll)**。

在我们的项目中，也有尝试过使用。最终我们还是觉得基于标准库的实现已经足够。理由如下：

1.  用户态的`goroutine`优先级没有 go`netpool`的调度优先级高。带来的问题就是毛刺多了。近期字节跳动也开源了自己的`netpool`，并且通过优化扩展包内`epoll`的使用方式来优化这个问题，具体效果未知
2.  效果不明显，我们绝大部分业务的 QPS 主要受限于其他的 RPC 调用，或者 CPU 计算。收发包的优化效果很难体现。
3.  增加了系统复杂性，虽然标准库慢一点点，但是足够稳定和简单。

### **5. 包大小优化**

我们 CI 是用蓝盾流水线实现的，有一次业务反馈说蓝盾编译的二进制会比自己开发机编译的体积大 50% 左右。对比了操作系统和 go 版本都是一样的,`tlinux2.2 golang1.15`。我们在用 linux 命令`size —A`对两个文件各个`section`做对比时，发现了`debug`相关的`section size`明显不一致，而且`section`的名称也不一样:

```
size -A test-30MB
section                  size       addr
.interp                    28    4194928
.note.ABI-tag              32    4194956
... ... ... ...
.zdebug_aranges          1565          0
.zdebug_pubnames        56185          0
.zdebug_info          2506085          0
.zdebug_abbrev          13448          0
.zdebug_line          1250753          0
.zdebug_frame          298110          0
.zdebug_str             40806          0
.zdebug_loc           1199790          0
.zdebug_pubtypes       151567          0
.zdebug_ranges         371590          0
.debug_gdb_scripts         42          0
Total                93653020

size -A test-50MB
section                   size       addr
.interp                     28    4194928
.note.ABI-tag               32    4194956
.note.go.buildid           100    4194988
... ... ...
.debug_aranges            6272          0
.debug_pubnames         289151          0
.debug_info            8527395          0
.debug_abbrev            73457          0
.debug_line            4329334          0
.debug_frame           1235304          0
.debug_str              336499          0
.debug_loc             8018952          0
.debug_pubtypes        1072157          0
.debug_ranges          2256576          0
.debug_gdb_scripts          62          0
Total                113920274
```

通过查找`debug`和`zdebug`的区别了解到，`zdebug`是对`debug`段做了`zip`压缩，所以压缩后包体积会更小。查看 **[go 的源码](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/cmd/link/internal/ld/dwarf.go%23L2210)**，发现链接器默认已经对`debug`段做了`zip`压缩。

看来，未压缩的`debug`段不是 go 自己干的。我们很容易就猜到，由于代码中引入了`cgo`，可能是`c++`的链接器没有压缩导致的。

> 代码引入`cgo`后，go 代码由 go 编译器编译，c 代码由`g++`编译，后续由`ld`链接成可执行文件。所以包含`cgo`的代码在跨平台编译时，需要更改对应平台的 c 代码编译器, 链接器。具体过程可以翻阅 go 编译过程相关资料，不再赘述。  

但是我们再次寻找相关**[源码](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/cmd/link/internal/ld/lib.go%23L1471)**发现，go 在使用`ld`链接时，已经指定了参数`--compress-debug-sections=zlib-gnu`用来压缩`debug`相关信息。

再次寻找原因，我们猜测可能跟`tlinux2.2`支持`go 1.16`有关，之前我们发现升级 go 版本之后，在开发机上无法编译。最后发现是因为`go1.16`优化了一部分编译指令，导致我们的`ld`版本太低不支持。所以我们用`yum install -y binutils`升级了`ld`的版本。果然，在翻阅了`ld`的文档之后，我们确认了`tlinux2.2`自带的`ld`不支持`--compress-debug-sections=zlib-gnu`这个指令，升级后`ld`才支持。

总结：在包含`cgo`的代码编译时，将`ld`升级到`2.27`版本，编译后的体积可以减少约 50%。

### **6.simd**

首先，go 链接器支持 simd 指令，但 go 编译器不支持`simd`指令的生成。所以在 go 中使用`simd`一般来说有三种方式：

1.  手写汇编
2.  `llvm`
3.  `cgo`（如果用`cgo`的方式来调用，会受限于`cgo`的性能，达不到加速的目的）

目前比较流行的做法是`llvm`：

1.  用`c`来写`simd`相关的函数，然后用`llvm`编译成 c 汇编
2.  用工具把 c 汇编转换成 go 的汇编格式，保存为`.s`文件
3.  在 go 中调用`.s`里的方法，最后用 go 编译器编译

以下开源库用到了 simd，可以参考：

1.  **[simdjson-go](https://link.zhihu.com/?target=https%3A//github.com/minio/simdjson-go)**
2.  **[sonic](https://link.zhihu.com/?target=https%3A//github.com/bytedance/sonic)**
3.  **[sha256-simd](https://link.zhihu.com/?target=https%3A//github.com/minio/sha256-simd)**

合理的使用`simd`可以充分发挥 cpu 特性，但是存在以下弊端：

1.  难以维护，要么需要懂汇编的大神，要么需要引入第三方语言
2.  跨平台支持不够，需要对不同平台汇编指令做适配
3.  汇编代码很难调试，作为使用方来讲，完全黑盒

### **7.jit**

go 中使用 jit 的方式可以参考 **[Writing a JIT compiler in Golang](https://link.zhihu.com/?target=https%3A//medium.com/kokster/writing-a-jit-compiler-in-golang-964b61295f)**

目前只有在字节跳动刚开源的`json`解析库中发现了使用场景 **[sonic](https://link.zhihu.com/?target=https%3A//github.com/bytedance/sonic)**

这种使用方式个人感觉在 go 中意义不大，仅供参考

### **总结**

过早的优化是万恶之源，千万不要为了优化而优化

1.  pprof 分析，竞态分析，逃逸分析，这些基础的手段是必须要学会的
2.  常规的优化技巧是比较实用的，他们往往能解决大部分的性能问题并且足够安全。
3.  在一些着重性能的基础库中，使用一些非常规的优化手段也是可以的，但必须要权衡利弊，不要过早放弃可读性，兼容性和稳定性。

"QQ 音乐作为国内最大的音乐流媒体平台，QQ 音乐平台产品工程技术团队，打造了稳定高效的音乐流媒体平台，服务每天数十亿的用户听歌需求。并且在音频媒体能力、智能推荐算法、信息检索等领域深耕十几年，为用户提供更极致的音乐享受和发掘更多的音乐兴趣。欢迎对技术和音乐感兴趣，渴望通过技术创造音乐更大可能的技术同学加入。"

腾讯技术交流群已建立，交流讨论可加 **QQ 群：**160315980**（备注腾讯技术） ，微信交流群加：**teg_helper**。**