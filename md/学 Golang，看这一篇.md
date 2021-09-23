> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/276052046)

作者：xixie，腾讯 IEG 后台开发工程师

> 去年学了一遍 Golang，发现都给整忘了， 好饭不怕晚，再次二刷。 其实学好 Golang 并不难，关键是要找到它和其它语言不同和众里寻他千百度相通的微妙之处，就能很优雅地使用 Golang，以下会涉及较多知识点。  

### **特殊类型**

1： 空结构体 **类型** `struct{}`, 空结构体的实例 `struct{}{}`

2： 空接口 **类型** `interface{}`

### **会自动执行的函数**

```
fun init(){}  // 会自动执行
```

*   init 函数先于 main 函数自动执行，不能被其他函数调用；
*   init 函数没有输入参数、返回值；
*   每个包可以有多个 init 函数；
*   **包的每个源文件也可以有多个 init 函数**，这点比较特殊；
*   同一个包的 init 执行顺序，golang 没有明确定义，编程时要注意程序不要依赖这个执行顺序。
*   不同包的 init 函数按照包导入的依赖关系决定执行顺序。

### **匿名函数**

1: 需要传递匿名函数的地方：

```
once.Do(func() { // 该函数只会执行一次
  fmt.Println("Create Obj")
  singleInstance = new(Singleton)
 })
```

2: 如果一个函数需要返回一个匿名函数:

```
type retFunc func()  // 通过 type 创建一个  别名

func xxx() retFunc{
 return func(){
  return 1;
 }
}
```

更凶残的匿名函数:

```
writeDataFn := func() {
  var data []int // 往 slice 里 写100 个数；
  for i := 0; i < 100; i++ {
   data = append(data, i)
  }
  atomic.StorePointer(&shareBufPtr, unsafe.Pointer(&data)) // 写完后，将共享缓存的指针指向它；
 }

 readDataFn := func() {
  data := atomic.LoadPointer(&shareBufPtr)
  fmt.Println(data, *(*[]int)(data)) // 打印共享缓存里的值
 }
```

### **接口**

1： 注意接口是 `type xxx interface{}` 2： 传入接口的地方不能是指针 `p *Programmer`:

```
func writeFirstProgram2(p *Programmer) {
 fmt.Printf("%T %v\n", p, p.WriteHelloWorld())
}
```

如果接口要传入这个，则必须定义返回值为 `interface{}` 的方法：

```
type Programmer interface {
    WriteHelloWorld() interface{}
}
```

> 注意： 1：父接口，只能是实例，不能是指针，可以看成父接口本身代表的就是指针实例 2：子类以 指针 / 实例 实现的接口，传入时，就得以对应的指针实例传入接口 3: 一个接口，通常 只定义 一个 方法，尽量保证 **接口精简**：  

### **行为**

1：行为的定义时 `type xxx struct{}`

2: 行为的方法实现，决定了最终传入的实例是什么

```
type Programmer interface {
    WriteHelloWorld() string
}
```

第一种： 子类实现 `func (p *NoTypeProgrammer) WriteHelloWorld()`， 则 只能被 指针调用

```
// 将 子行为 传入接口
type NoTypeProgrammer struct {
}

// 标识，要看最终这里的实现 ！！！
func (p *NoTypeProgrammer) WriteHelloWorld() string {
    return "System.out.Println(\"Hello World!\")"
}

// 传入接口的  方法， 这里传 指针or实例都可以的
func writeFirstProgram(p Programmer) {
    fmt.Printf("%T %v\n", p, p.WriteHelloWorld())
}
```

> 方法前面传入对象，是为了通过对象. s 调用内部的成员变量 or 成员函数  

调用：

```
noTypeProg := new(NoTypeProgrammer)
    writeFirstProgram(noTypeProg)
    //writeFirstProgram(NoTypeProgrammer{}) // error
```

第二种：子类实现 `func (p oTypeProgrammer) WriteHelloWorld()` ， 则指针和实例都可以：

```
// 标识，要看最终这里的实现 ！！！
func (p NoTypeProgrammer) WriteHelloWorld() string {
    return "System.out.Println(\"Hello World!\")"
}
```

调用：

```
noTypeProg := new(NoTypeProgrammer)
    writeFirstProgram(noTypeProg)
    writeFirstProgram(NoTypeProgrammer{}) // right
```

### **空接口**

1： 空接口，代表 object 类型。需要通过**断言**判断类型

```
func DoSomething(p interface{}) {
 switch v := p.(type) {
 case int:
  fmt.Println("Integer", v)
 case string:
  fmt.Println("String", v)
 default:
  fmt.Println("Unknow Type")
 }
}

// 使用
DoSomething("10")
```

2：空接口可以接受任何类型的值:

> var yzj interface{} // 空接口的使用，空接口类型的变量可以保存任何类型的值, 空格口类型的变量非常类似于弱类型语言中的变量，未被初始化的 interface 默认初始值为 nil。  

### **错误处理**

1：使用多返回值，错误码来判断程序的处理；类似于 C ，通过引用或者指针传出来错误码；【最佳的是返回码 + 引用】；

### **模块**

*   同一个目录下只能有一个包名；
*   包名推荐与目录名保持一致；
*   每一个包内，只能有一个同名函数；
*   同一个 package 内的 函数， 变量可以随意调用，不用导入

### **CSP**

1：Communication sequential process: 进程间通过管道进行通信。

2：channel 的两种通信方式：

*   buffer- channel: 发送者和接受者有更松的耦合关系；
*   容量没满， 放消息的人可以一直往里面放； 若容量已满， 直到接受者收走一个消息，能继续放入消息了，才能继续往下执行；
*   接收者也类似： 只要是非空，能一直拿消息，向下执行；如果没有消息，需要一直等待，然后再向下继续执行；

单通道： `chan string`， `chan int`, 一次只能放入一个值， 在 值 被取走前， 通道是阻塞的。

3: 创建一个协程，除了 `go func(){}` 还有更简洁的方式：

```
go agt.EventProcessGroutine()   // 直接go 后面接一个 实名函数 也可以
```

协程是异步的， 主线程只会因为 **通道阻塞**。

![](https://pic3.zhimg.com/v2-9d1aed5e38fd0d4261bf8b6edb39023a_r.jpg)![](https://pic1.zhimg.com/v2-b018e710840886274b20022106bfaea0_r.jpg)

```
sequenceDiagram
    main ->> main:

    main ->> + AsyncService : 异步执行
    main ->> +otherTask: 和 AsyncService 同步执行
    otherTask-->>- main: return
    main ->> main: 等待 <-channel
    AsyncService ->> AsyncService: sleep
    AsyncService ->> AsyncService: sleep
    AsyncService ->>main: retCh <- ret， 通道在左边，是给通道赋值
    main ->> main: 唤起执行
    AsyncService ->>AsyncService: 继续执行
	main ->> main: End
	AsyncService ->>AsyncService: 继续执行
	AsyncService -->>- main: return
```

3：主线程可通过 `var wg sync.WaitGroup()` 管理多个协程的并发问题；

4： 生产者和消费者之间通过生产者的 `close(chan)` 来广播结束通道，

![](https://pic4.zhimg.com/v2-494318ff3a2e765e332e4beacd816db7_r.jpg)

5： // 通道没关闭，其它协程会陷入死循环；

`close(chan)` 给其它协程广播的是 `struct{}{}` 空消息；

6： 可用于**任意任务完成**的场景。

一个 buffer 会阻塞其它协程的 写， N 个协程创建 N 个 Buffer， 就是非阻塞的了。

```
ch := make(chan string)
```

7： `sync.WaitGroup()` 等待所有协程返回：

![](https://pic2.zhimg.com/v2-9cad9cf7f9690ee5a5d8ae988436ed55_r.jpg)

8： 利用 channel 自身取的阻塞，有多少个协程，就循环多少次：

![](https://pic3.zhimg.com/v2-4d9f6abd8556a994698204b934de84a6_r.jpg)

### **Select**

![](https://pic3.zhimg.com/v2-a6e7f4a96cebd1e5a9272032f2ebd206_r.jpg)

### **Context**

1： 可通过父协程取消所有子协程（**非 Channel 实现**）。

### **Sync.Once.Do()**

1： `sync.Once` 的 `Do（）` 方法，可保证匿名函数只被执行一次

### **对象池**

1： 对象池中如果需要放入任意的数据类型，就放入 `interface{}`

![](https://pic1.zhimg.com/v2-7fe27554a5fa692bd7cbdb875f69e48c_r.jpg)

### **sync.Pool**

1: 生命周期不可控，随时会被 GC

![](https://pic2.zhimg.com/v2-7a4ab8a707d70dde6df4852a0f5dad7d_r.jpg)

### **bench**

1:

![](https://pic2.zhimg.com/v2-60a4cea1a3ef0e2b13e0056942ebc645_r.jpg)

2： bench 只能做参考，**在不同的方式下，运行的次数不同。**

3： bench 的使用

```
func BenchSyncmap(b *testing.B) {
 // bench 需要传入一个函数
 b.Run("map with RWLock", func(b *testing.B) {
  hm := CreateRWLockMap()
  benchMap(b, hm)
 })
}
```

### **反射**

1： 可反射成员方法，编写更灵活的代码;

2: 特别是在解析 json 的时候；

3： 名字可以不一样，但是类型一样即可。 【万能程序】

### **不安全的编程**

1： 类型转换

![](https://pic4.zhimg.com/v2-9efb04a872e8c28ab35a27e111e87027_r.jpg)

2: 可通过 `atomic.StorePointer()` 和 `actomic.LoadPointer()` 对多个协程并发读写：

实际上读和写是分开的两个 Buffer：

![](https://pic4.zhimg.com/v2-9a9563f10b67140e2cb6da6288bf1f67_r.jpg)

### **pipe 和 Filter**

1： 定义接口

![](https://pic4.zhimg.com/v2-4422ef7093a8a2489a92caaf6d7f4127_r.jpg)

2： 定义每个接口`Filter`的 **工厂方法** + 数据处理方法 【工厂方法 在很多 库里都用到了，这种**设计方式可成为定式** 】

![](https://pic4.zhimg.com/v2-3cfeed503137c2d390419af71317999b_r.jpg)

3： 串起来调用：

![](https://pic2.zhimg.com/v2-8240b28b0c6aeb16e392f8a87b031ad5_r.jpg)![](https://pic4.zhimg.com/v2-9dfe1cd830ba1762cf5c12fab60e3783_r.jpg)

一个更常用的测试用例：

需要测试的接口，可以将各种实例放进去：

![](https://pic4.zhimg.com/v2-ba0236230151108489fcb92609586c03_r.jpg)

实现一个 接口的经典例子【golang 的类组织方式，采用的是**更扁平化的方式**】：

![](https://pic2.zhimg.com/v2-4533452e1002f0fd93ecd9c97e7f59c5_r.jpg)

创建对应的接口，并最终调用它：

![](https://pic3.zhimg.com/v2-08ae1c376be9e38ae6251dfdc0a3a6ce_r.jpg)

### **microKernel**

1： 微服务模式， 总类管理子类的过程；

![](https://pic2.zhimg.com/v2-a30c7e154c86538ceae2f0e89e31baf5_r.jpg)

### **JSON**

1：通过 struct tag 来解析，相对比较简单，有点类似那个万能程序；

2：easyjson 需要手动生成 marsh 和 unmarsh 文件；

### **HTTP 服务**

1： ROA：面向资源的架构

![](https://pic3.zhimg.com/v2-a2934c4b69e30491e6f6f56184129572_r.jpg)

### **性能分析**

1： 常见性能调优过程：

![](https://pic1.zhimg.com/v2-970605e0c68fd460bfbe0113cbf87d60_r.jpg)

2： 性能分析的常见指标：

*   1：Wall Time;
*   2: CPU Time;
*   3: Block Time;
*   4: Memory Allocation;
*   5: GC Times/Time Spent

3： prof 常用指令：

*   1： 查看 CPU：

```
go test -bench=.
 go test -bench=. -cpuprofile=cpu.prof
 go tool pprof cpu.prof top -cum list function
```

*   2: 查看 Mem:

```
go test -bench=.
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof
top
list function
```

也可调用 对应的 API，对某个函数 精准的输出 prof 文件

4: 网络查看： `http://localhost:8081/debug/pprof/`

5： 影响性能的几个要点：

文本处理和字符串连接

*   json 解析；
*   字符串连接 用 bytes.Buffer 或 strings.Builder

协程锁：

*   少用读锁；
*   多用 concurrent-map, 实现协程安全的 数据共享；

GC 友好代码【尽量复用内存，减少内存分配】：

*   大数组和结构体，在函数中传递时，使用指针；
*   切片 最好初始化到合适的大小； 或者 用数组；

### **面向错误的设计**

1： 隔离错误：

*   从设计上隔离： Micro-kernal 设计；
*   物理隔离，部署；

2： 重用和冗余的适当取舍；

3： 限流；

4： 快速拒绝比慢响应更优；（从 web 的角度）

5： 不要无休止的等待；

6： 断路器： 利用好缓存，适当容错；

### **面向恢复的设计**

1： 注意僵尸进程： 池化资源耗尽 和 死锁；

2： Let it crash 大多数 比 recover 程序更优（尽早发现错误尽早解决）；

3： 构建可恢复的系统：

*   拒绝单实例；
*   减少服务之间依赖；
*   快速启动；
*   尽量 无状态；

4：与客户端协商访问频次， 类似限流；

**更多干货尽在[腾讯技术](https://www.zhihu.com/org/teng-xun-ji-zhu-gong-cheng)，官方微信交流群已建立，交流讨论可加：Journeylife1900（备注腾讯技术） 。**